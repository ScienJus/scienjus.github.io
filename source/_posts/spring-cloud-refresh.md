---
title: 'Spring Cloud 是如何实现热更新的'
date: 2017-07-25 20:12:31
permalink: spring-cloud-refresh
---

作为一篇源码分析的文章，本文虽然介绍 Spring Cloud 的热更新机制，但是实际全文内容都不会与 Spring Cloud Config 以及 Spring Cloud Bus 有关，因为前者只是提供了一个远端的配置源，而后者也只是提供了集群环境下的事件触发机制，与核心流程均无太大关系。

<!--more-->

## ContextRefresher

顾名思义，`ContextRefresher` 用于刷新 Spring 上下文，在以下场景会调用其 `refresh` 方法。

1. 请求 `/refresh` Endpoint。
2. 集成 Spring Cloud Bus 后，收到 `RefreshRemoteApplicationEvent` 事件（任意集成 Bus 的应用，请求 `/bus/refresh` Endpoint 后都会将事件推送到整个集群）。

这个方法包含了整个刷新逻辑，也是本文分析的重点。

首先看一下这个方法的实现：

```
public synchronized Set<String> refresh() {
  Map<String, Object> before = extract(
      this.context.getEnvironment().getPropertySources());
  addConfigFilesToEnvironment();
  Set<String> keys = changes(before,
      extract(this.context.getEnvironment().getPropertySources())).keySet();
  this.context.publishEvent(new EnvironmentChangeEvent(keys));
  this.scope.refreshAll();
  return keys;
}
```

首先是第一步 `extract`，这个方法接收了当前环境中的所有属性源（PropertySource），并将其中的非标准属性源的所有属性汇总到一个 Map 中返回。

这里的标准属性源指的是 `StandardEnvironment` 和 `StandardServletEnvironment`，前者会注册系统变量（System Properties）和环境变量（System Environment），后者会注册 Servlet 环境下的 Servlet Context 和 Servlet Config 的初始参数（Init Params）和 JNDI 的属性。个人理解是因为这些属性无法改变，所以不进行刷新。

第二步 `addConfigFilesToEnvironment` 是核心逻辑，它创建了一个新的 Spring Boot 应用并初始化：

```
SpringApplicationBuilder builder = new SpringApplicationBuilder(Empty.class)
    .bannerMode(Banner.Mode.OFF).web(false).environment(environment);
// Just the listeners that affect the environment (e.g. excluding logging
// listener because it has side effects)
builder.application()
    .setListeners(
        Arrays.asList(new BootstrapApplicationListener(),
            new ConfigFileApplicationListener()));
capture = builder.run();
```

这个应用只是为了重新加载一遍属性源，所以只配置了 `BootstrapApplicationListener` 和 `ConfigFileApplicationListener`，最后将新加载的属性源替换掉原属性源，至此属性源本身已经完成更新了。

此时属性源虽然已经更新了，但是配置项都已经注入到了对应的 Spring Bean 中，需要重新进行绑定，所以又触发了两个操作：

1. 将刷新后发生更改的 Key 收集起来，发送一个 `EnvironmentChangeEvent` 事件。

2. 调用 `RefreshScope.refreshAll` 方法。

## EnvironmentChangeEvent

在上文中，`ContextRefresher` 发布了一个 `EnvironmentChangeEvent` 事件，接下来看看这个事件产生了哪些影响。

> The application will listen for an EnvironmentChangeEvent and react to the change in a couple of standard ways (additional ApplicationListeners can be added as @Beans by the user in the normal way). When an EnvironmentChangeEvent is observed it will have a list of key values that have changed, and the application will use those to:

> 1. Re-bind any @ConfigurationProperties beans in the context

> 2. Set the logger levels for any properties in logging.level.*

官方文档的介绍中提到，这个事件主要会触发两个行为：

1. 重新绑定上下文中所有使用了 `@ConfigurationProperties` 注解的 Spring Bean。
2. 如果 `logging.level.*` 配置发生了改变，重新设置日志级别。

这两段逻辑分别可以在 `ConfigurationPropertiesRebinder` 和 `LoggingRebinder` 中看到。

### ConfigurationPropertiesRebinder

这个类乍一看代码量特别少，只需要一个 `ConfigurationPropertiesBeans` 和一个 `ConfigurationPropertiesBindingPostProcessor`，然后调用 `rebind` 每个 Bean 即可。但是这两个对象是从哪里来的呢？

```
public void rebind() {
  for (String name : this.beans.getBeanNames()) {
    rebind(name);
  }
}
```

`ConfigurationPropertiesBeans` 需要一个 `ConfigurationBeanFactoryMetaData`， 这个类逻辑很简单，它是一个 `BeanFactoryPostProcessor` 的实现，将所有的 Bean 都存在了内部的一个 Map 中。

而 ConfigurationPropertiesBeans 获得这个 Map 后，会查找每一个 Bean 是否有 `@ConfigurationProperties` 注解，如果有的话就放到自己的 Map 中。

绕了一圈好不容易拿到所有需要重新绑定的 Bean 后，绑定的逻辑就要简单许多了：

```
public boolean rebind(String name) {
  if (!this.beans.getBeanNames().contains(name)) {
    return false;
  }
  if (this.applicationContext != null) {
    try {
      Object bean = this.applicationContext.getBean(name);
      if (AopUtils.isCglibProxy(bean)) {
        bean = getTargetObject(bean);
      }
      this.binder.postProcessBeforeInitialization(bean, name);
      this.applicationContext.getAutowireCapableBeanFactory()
          .initializeBean(bean, name);
      return true;
    }
    catch (RuntimeException e) {
      this.errors.put(name, e);
      throw e;
    }
  }
  return false;
}
```

其中 `postProcessBeforeInitialization` 方法将 Bean 重新绑定了所有属性，并做了校验等操作。

而 `initializeBean` 的实现如下：

```
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
  Object wrappedBean = bean;
  if(mbd == null || !mbd.isSynthetic()) {
    wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
  }

  try {
    this.invokeInitMethods(beanName, wrappedBean, mbd);
  } catch (Throwable var6) {
    throw new BeanCreationException(mbd != null?mbd.getResourceDescription():null, beanName, "Invocation of init method failed", var6);
  }

  if(mbd == null || !mbd.isSynthetic()) {
    wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
  }

  return wrappedBean;
}
```

其中主要做了三件事：

1. `applyBeanPostProcessorsBeforeInitialization`：调用所有 `BeanPostProcessor` 的 `postProcessBeforeInitialization` 方法。
2. `invokeInitMethods`：如果 Bean 继承了 `InitializingBean`，执行 `afterPropertiesSet` 方法，或是如果 Bean 指定了 `init-method` 属性，如果有则调用对应方法
3. `applyBeanPostProcessorsAfterInitialization`：调用所有 `BeanPostProcessor` 的 `postProcessAfterInitialization` 方法。

之后 `ConfigurationPropertiesRebinder` 就完成整个重新绑定流程了。

### LoggingRebinder

相比之下 `LoggingRebinder` 的逻辑要简单许多，它只是调用了 `LoggingSystem` 的方法重新设置了日志级别，具体逻辑就不在本文详述了。

## RefreshScope

首先看看这个类的注释：

> Note that all beans in this scope are <em>only</em> initialized when first accessed, so the scope forces lazy initialization semantics. The implementation involves creating a proxy for every bean in the scope, so there is a flag

> If a bean is refreshed then the next time the bean is accessed (i.e. a method is executed) a new instance is created. All lifecycle methods are applied to the bean instances, so any destruction callbacks that were registered in the bean factory are called when it is refreshed, and then the initialization callbacks are invoked as normal when the new instance is created. A new bean instance is created from the original bean definition, so any externalized content (property placeholders or expressions in string literals) is re-evaluated when it is created.

这里提到了两个重点：

1. 所有 `@RefreshScope` 的 Bean 都是延迟加载的，只有在第一次访问时才会初始化
2. 刷新 Bean 也是同理，下次访问时会创建一个新的对象

再看一下方法实现：

```
public void refreshAll() {
  super.destroy();
  this.context.publishEvent(new RefreshScopeRefreshedEvent());
}
```

这个类中有一个成员变量 `cache`，用于缓存所有已经生成的 Bean，在调用 `get` 方法时尝试从缓存加载，如果没有的话就生成一个新对象放入缓存，并通过 `getBean` 初始化其对应的 Bean：

```
public Object get(String name, ObjectFactory<?> objectFactory) {
  if (this.lifecycle == null) {
    this.lifecycle = new StandardBeanLifecycleDecorator(this.proxyTargetClass);
  }
  BeanLifecycleWrapper value = this.cache.put(name,
      new BeanLifecycleWrapper(name, objectFactory, this.lifecycle));
  try {
    return value.getBean();
  }
  catch (RuntimeException e) {
    this.errors.put(name, e);
    throw e;
  }
}
```

所以在销毁时只需要将整个缓存清空，下次获取对象时自然就可以重新生成新的对象，也就自然绑定了新的属性：

```
public void destroy() {
  List<Throwable> errors = new ArrayList<Throwable>();
  Collection<BeanLifecycleWrapper> wrappers = this.cache.clear();
  for (BeanLifecycleWrapper wrapper : wrappers) {
    try {
      wrapper.destroy();
    }
    catch (RuntimeException e) {
      errors.add(e);
    }
  }
  if (!errors.isEmpty()) {
    throw wrapIfNecessary(errors.get(0));
  }
  this.errors.clear();
}
```

清空缓存后，下次访问对象时就会重新创建新的对象并放入缓存了。

而在清空缓存后，它还会发出一个 `RefreshScopeRefreshedEvent` 事件，在某些 Spring Cloud 的组件中会监听这个事件并作出一些反馈。

### Zuul

Zuul 在收到这个事件后，会将自身的路由设置为 dirty 状态：

```
private static class ZuulRefreshListener implements ApplicationListener<ApplicationEvent> {

  @Autowired
  private ZuulHandlerMapping zuulHandlerMapping;
  
  @Override
  public void onApplicationEvent(ApplicationEvent event) {
    if (event instanceof ContextRefreshedEvent
        || event instanceof RefreshScopeRefreshedEvent
        || event instanceof RoutesRefreshedEvent) {
      this.zuulHandlerMapping.setDirty(true);
    }
  }
}
```

并且当路由实现为 `RefreshableRouteLocator`  时，会尝试刷新路由：

```
public void setDirty(boolean dirty) {
  this.dirty = dirty;
  if (this.routeLocator instanceof RefreshableRouteLocator) {
    ((RefreshableRouteLocator) this.routeLocator).refresh();
  }
}
```

当状态为 dirty 时，Zuul 会在下一次接受请求时重新注册路由，以更新配置：

```
if (this.dirty) {
  synchronized (this) {
    if (this.dirty) {
      registerHandlers();
      this.dirty = false;
    }
  }
}
```

### Eureka

在 Eureka 收到该事件时，对于客户端和服务端都有不同的处理方式：

```
protected static class EurekaClientConfigurationRefresher {

  @Autowired(required = false)
  private EurekaClient eurekaClient;

  @Autowired(required = false)
  private EurekaAutoServiceRegistration autoRegistration;

  @EventListener(RefreshScopeRefreshedEvent.class)
  public void onApplicationEvent(RefreshScopeRefreshedEvent event) {
    //This will force the creation of the EurkaClient bean if not already created
    //to make sure the client will be reregistered after a refresh event
    if(eurekaClient != null) {
      eurekaClient.getApplications();
    }
    if (autoRegistration != null) {
      // register in case meta data changed
      this.autoRegistration.stop();
      this.autoRegistration.start();
    }
  }
}
```

对于客户端来说，只是调用了下 `eurekaClient.getApplications`，理论上这个方法是没有任何效果的，但是查看上面的注释，以及联想到 `RefreshScope` 的延时初始化特性，这个方法调用应该只是为了强制初始化新的 `EurekaClient`。

事实上这里很有趣的是，在 `EurekaClientAutoConfiguration` 中，实际为了 `EurekaClient` 提供了两种初始化方案，分别对应是否有 `RefreshScope`，所以以上的猜测应该是正确的。

而对于服务端来说，`EurekaAutoServiceRegistration` 会将服务端先标记为下线，在进行重新上线。

## 总结

至此，Spring Cloud 的热更新流程就到此结束了，从这些源码中可以总结出以下结论：

1. 通过使用 `ContextRefresher` 可以进行手动的热更新，而不需要依靠 Bus 或是 Endpoint。
2. 热更新会对两类 Bean 进行配置刷新，一类是使用了 `@ConfigurationProperties` 的对象，另一类是使用了 `@RefreshScope` 的对象。
3. 这两种对象热更新的机制不同，前者在同一个对象中重新绑定了所有属性，后者则是利用了 `RefreshScope` 的缓存和延迟加载机制，生成了新的对象。
4. 通过自行监听 `EnvironmentChangeEvent` 事件，也可以获得更改的配置项，以便实现自己的热更新逻辑。
5. 在使用 Eureka 的项目中要谨慎的使用热更新，过于频繁的更新可能会使大量项目频繁的标记下线和上线，需要注意。





