---
title: 'Armeria 学习笔记之断路器'
date: 2016-12-24 09:28:50
permalink: armeria-circuit-breaker
---

## 为什么需要断路器

 在分布式系统中，应用之间通过 RPC 调用，这意味着一个服务的健康状态不再只与该服务所在的应用有关，还与其所依赖的所有服务有关。这将会出现两个问题：

1. 一个服务被多个服务所依赖，如果该服务出现了故障，依赖其的服务都会受到影响
2. 一个服务依赖多个服务，如果其中一个依赖服务出现了故障，该服务就会受到影响

 也就是意味着在一整条服务的调用链路中，任何一个单元出现了故障（程序问题或是网络问题），都会造成连锁失败，这实际是非常不可靠的。

![](/uploads/2016/12/cascading_failure.jpg)

 断路器是一个基于客户端的自我保护行为，它会统计依赖服务的健康状态，在依赖服务不可靠时快速失败，而避免等待超时等行为对自身造成太大影响。

![](/uploads/2016/12/fail_fast.jpg)

## 如何实现一个断路器

 一个简单的断路器实现其实就是状态机，通过服务调用的成功、失败次数在开启和关闭之间切换。

 断路器的状态有：

- 关闭：所有请求都正常进行
- 开启：阻止所有请求并快速失败
- 半开启：在开启和关闭状态的中间切换状态

 断路器的生命周期为：

1. 初始为关闭状态，所有请求正常执行，并统计成功、失败数
2. 当失败率高于容忍的阈值后，切换为开启状态，阻止请求执行
3. 开启状态持续一定时间后，切换为半开启状态，并通过一个请求
4. 等待该请求的结果，如果成功切换为关闭状态，失败则重新恢复为开启状态

![](/uploads/2016/12/fsm.jpg)

## Armeria 源码分析

### 断路器的设计

 断路器的接口结构如下：

```
public interface CircuitBreaker {

    /**
     * 成功时执行该方法
     */
    void onSuccess ();

    /**
     * 失败时执行该方法
     * @param cause 捕获的异常
     */

    void onFailure (Throwable cause);

    /**
     * 失败时执行该方法
     */
    void onFailure ();

    /**
     * 通过当前状态判断是否可以执行该次请求
     */

    boolean canRequest ();

}
```

 对请求的封装：

```
// 通过断路器判断是否可以执行该次请求
if (circuitBreaker.canRequest ()) {
    // 发起请求
    final O response;
    try {
        response = delegate ().execute (ctx, req);
    } catch (Throwable cause) {
        // 发生异常时，记录失败并抛出异常
        circuitBreaker.onFailure (cause);
        throw cause;
    }

    response.closeFuture ().handle (voidFunction ((res, cause) -> {
        // 通过请求结果记录成功、失败
        if (cause == null) {
            circuitBreaker.onSuccess ();
        } else {
            circuitBreaker.onFailure (cause);
        }
    })).exceptionally (CompletionActions::log);

    return response;
} else {
    // 如果断路器不允许进行该次请求，直接快速失败
    throw new FailFastException (circuitBreaker);
}
```

 调用时：

```
try {
    productClient.getProduct (productId);
} catch (TException e) {
    // 错误处理
} catch (FailFastException e) {
    // 对于快速失败的错误，可以返回本地缓存中的值，或是准备一个默认值
}
```

### 断路器的实现

Armeria 的默认断路器实现是 `NonBlockingCircuitBreaker`，通过内部维护 `State` 用于状态切换。

`canRequest` 实现：

```
@Override
public boolean canRequest () {
    final State currentState = state.get ();
    if (currentState.isClosed ()) {
        // all requests are allowed during CLOSED
        return true;
    } else if (currentState.isHalfOpen () || currentState.isOpen ()) {
        if (currentState.checkTimeout () && state.compareAndSet (currentState, newHalfOpenState ())) {
            // changes to HALF_OPEN if OPEN state has timed out
            logStateTransition (CircuitState.HALF_OPEN, null);
            notifyStateChanged (CircuitState.HALF_OPEN);
            return true;
        }
        // all other requests are refused
        notifyRequestRejected ();
        return false;
    }
    return true;
}
```

- 当前状态为关闭时，断路器对于所有请求都会返回 `true`
- 当状态为开启和半开启时，会先检查当前状态是否还在持续时间内（开启和半开启状态都有固定的持续时间），如果还在持续时间内就返回 `false`
- 如果当前状态为开启，且已经到期了，就尝试将状态改为半开启，同时返回 `true` 通过一个请求去校验服务器状态
- 如果当前状态为半开启，且已经到期了（这种情况出现的原因只可能是之前切换状态时，通过的那个请求没有得到反馈），就清空状态，再放出一个新的请求。

`onSuccess` 实现：

```
@Override
public void onSuccess () {
    final State currentState = state.get ();
    if (currentState.isClosed ()) {
        // fires success event
        final Optional<EventCount> updatedCount = currentState.counter ().onSuccess ();
        // notifies the count if it has been updated
        updatedCount.ifPresent (this::notifyCountUpdated);
    } else if (currentState.isHalfOpen ()) {
        // changes to CLOSED if at least one request succeeds during HALF_OPEN
        if (state.compareAndSet (currentState, newClosedState ())) {
            logStateTransition (CircuitState.CLOSED, null);
            notifyStateChanged (CircuitState.CLOSED);
        }
    }
}
```

- 当前状态为关闭时统计成功数，成功事件并不会造成关闭状态的状态变更
- 当前状态为半开启时，说明切换为半开启状态时通过的那个请求成功了，将状态切换为关闭

`onFailure` 实现：

```
@Override
public void onFailure () {
    final State currentState = state.get ();
    if (currentState.isClosed ()) {
        // fires failure event
        final Optional<EventCount> updatedCount = currentState.counter ().onFailure ();
        // checks the count if it has been updated
        updatedCount.ifPresent (count -> {
            // changes to OPEN if failure rate exceeds the threshold
            if (checkIfExceedingFailureThreshold (count) &&
                state.compareAndSet (currentState, newOpenState ())) {
                logStateTransition (CircuitState.OPEN, count);
                notifyStateChanged (CircuitState.OPEN);
            } else {
                notifyCountUpdated (count);
            }
        });
    } else if (currentState.isHalfOpen ()) {
        // returns to OPEN if a request fails during HALF_OPEN
        if (state.compareAndSet (currentState, newOpenState ())) {
            logStateTransition (CircuitState.OPEN, null);
            notifyStateChanged (CircuitState.OPEN);
        }
    }
}
```

- 当前状态为关闭时统计失败数，当失败率超过阈值时转换为半开启状态
- 当前状态为半开启时，说明切换为半开启状态时通过的那个请求失败了，重新将状态切换为关闭

`NonBlockingCircuitBreaker` 使用了 `AtomicReference` 的 `compareAndSet` 方法切换状态，这样可以保证在并发时多个线程中只会有一个线程真正的切换状态，实现了非阻塞且线程安全。

 在关闭状态时使用了 `SlidingWindowCounter` 的实现统计成功失败数，内部实现用 `LongAdder` 原子记录成功、失败数，并通过时间分段存储在一个 `ConcurrentLinkedQueue` 中。

* * *

 本文是通过阅读 Armeria 源码和 Line 技术博客整理出的笔记，其中还有一些内容并没有介绍，可以移步到原文阅读：

- [line / armeria](https://github.com/line/armeria)
- [Circuit breakers for distributed services](http://developers.linecorp.com/blog/?p=3882)
- [Applying CircuitBreaker to Channel Gateway](http://developers.linecorp.com/blog/?p=3918)

 一些其他资料：

- [从 LongAdder 看更高效的无锁实现](http://coolshell.cn/articles/11454.html)

