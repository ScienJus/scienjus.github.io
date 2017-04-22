---
title: '使用FastRoute重写Spring的路由'
date: 2016-04-05 09:41:14
permalink: fastroute-spring
---

### Spring 的路由实现

 在 Spring 中路由被抽象为`MappingHandler`接口，而最常用的实现类是处理`@RequestMapping`的`RequestMappingHandlerMapping`。

`MappingHandler`接口只有一个方法`getHandler`，该方法接受标准的`HttpServletRequest`并返回`HandlerExecutionChain`，这个对象包含一组拦截器和具体需要执行的处理器。

```
HandlerExecutionChain getHandler (HttpServletRequest request) throws Exception;
```

`AbstractHandlerMapping`实现了这个方法，并将其定义为`final`方法，但是事实上只是添加了一些处理逻辑，并提供了一个抽象方法`getHandlerInternal`由子类实现。

 接下来，在`AbstractHandlerMethodMapping`中，这个方法被完整的实现了，最核心的业务在`lookupHandlerMethod`方法中，让我们来看一下这个方法具体是如何实现路由的：

```
protected HandlerMethod lookupHandlerMethod (String lookupPath, HttpServletRequest request) throws Exception {
    List matches = new ArrayList ();
    List directPathMatches = this.mappingRegistry.getMappingsByUrl (lookupPath);
    if (directPathMatches != null) {
        addMatchingMappings (directPathMatches, matches, request);
    }
    if (matches.isEmpty ()) {
        // No choice but to go through all mappings...
        addMatchingMappings (this.mappingRegistry.getMappings ().keySet (), matches, request);
    }

    if (!matches.isEmpty ()) {
        Comparator comparator = new MatchComparator (getMappingComparator (request));
        Collections.sort (matches, comparator);
        if (logger.isTraceEnabled ()) {
            logger.trace ("Found " + matches.size () + " matching mapping (s) for [" +
                    lookupPath + "] : " + matches);
        }
        Match bestMatch = matches.get (0);
        if (matches.size () > 1) {
            if (CorsUtils.isPreFlightRequest (request)) {
                return PREFLIGHT_AMBIGUOUS_MATCH;
            }
            Match secondBestMatch = matches.get (1);
            if (comparator.compare (bestMatch, secondBestMatch) == 0) {
                Method m1 = bestMatch.handlerMethod.getMethod ();
                Method m2 = secondBestMatch.handlerMethod.getMethod ();
                throw new IllegalStateException ("Ambiguous handler methods mapped for HTTP path '" +
                        request.getRequestURL () + "': {" + m1 + ", " + m2 + "}");
            }
        }
        handleMatch (bestMatch.mapping, lookupPath, request);
        return bestMatch.handlerMethod;
    }
    else {
        return handleNoMatch (this.mappingRegistry.getMappings ().keySet (), lookupPath, request);
    }
}
```

 首先它会调用`getMappingsByUrl`去寻找静态路由，这个方法实际就是一个 Map，以路由地址和路由对象组成的键值对。如果`@RequestMapping`中注册的路径没有包含参数或正则（例如`{id}`或`/**`），这个方法会很快的查询到匹配的路由。

 但是如果注册的路径包含了参数或正则，这个方法就会返回一个空列表，之后 Spring 会对所有的路由进行遍历，寻找匹配的路由。

 匹配大致分两步：首先将路由中除了路径以外的信息进行对比，包括请求方式（HTTP Method）、参数（Param）、请求头（Header）、内容类型（Content-Type）。如果所有信息都符合才会进行比较耗时的路径匹配，Spring 默认使用的是`AntPathMatcher`而不是正则表达式匹配路径，性能上应该会比单纯使用正则表达式要更高。

 最后，Spring 会将所有匹配的路由进行排序，得到匹配度最高的路由（如果有多个匹配度相同且最高的路由，会抛出异常）。接着对该路由进行解析，获得路径中的动态参数。

 从这个流程中可以看出，遍历整个列表绝对是一个低效的实现方式，即使`AntPathMatcher`的性能要高于正则表达式，每次都要从头扫描 URL 却是没有必要的。

 接下来介绍一种更加快速的路由匹配方式：FastRoute。

### FastRoute 的路由实现

 [FastRoute](https://github.com/nikic/FastRoute) 是 GitHub 上的一个 PHP 开源项目，并且应用于 [Lumen](https://github.com/laravel/lumen) 框架中。它提供了一种更加快速但是并不麻烦的路由匹配方式。

 之前我们提到，Spring 这种循环整个列表进行匹配的做法并不高效，而在 FastRoute 中，它将所有的路由拼成了一个完整的正则表达式，所以只要一次正则匹配就可以完成所有路由的查询。

 例如有以下三个路由：

```
/users/{id}

/users/{id}/{name}

/users/{uid}/posts/{pid}
```

 替换成正则表达式分别是：

```
^/users/([^/]+)$

^/users/([^/]+)/([^/]+)$

^/users/([^/]+)/posts/([^/]+)$
```

 按照以往的方法，当收到一个请求，就需要按顺序对每个正则表达式进行匹配，如果有三个路由就需要匹配三次，有一千个路由就需要匹配一千次。但是如果将所有的路由拼成一个完整的正则表达式，就会变成：

```
^(?:
 (/users/([^/]+))
|
 (/users/([^/]+)/([^/]+))
|
 (/users/([^/]+)/posts/([^/]+))
)$
```

 这样只需要一次就可以对所有的路由进行匹配了。例如：

```
Pattern pattern = Pattern.compile ("^(?:(/users/([^/]+))|(/users/([^/]+)/([^/]+))|(users/([^/]+)/posts/([^/]+)))$");

Matcher matcher = pattern.matcher ("/users/123/abc");

if (matcher.matches ()) {
  for (int i = 1; i
```

 将会输出结果：

```
null
null
/users/123/abc
123
abc
null
null
null
```

 而路由与匹配组的映射关系应该是这样一个数组：

```
 [
    Route ("/users/{id}"), Param ("id"),
    Route ("/users/{id}/{name}"), Param ("id"), Param ("name"),
    Route ("/users/{uid}/posts/{pid}"), Param ("uid"), Param ("pid")
]
```

 这样也就可以很轻易的通过组的序号得到对应的路由和参数信息。

 如果你想了解更多的信息，也可以看 [这篇作者写的分析文章](http://nikic.github.io/2014/02/18/Fast-request-routing-using-regular-expressions.html)，介绍的非常全面。

### 在 Spring 中实现 FastRoute

 我尝试通过继承`RequestMappingHandlerMapping`并重写`lookupHandlerMethod`方法添加自己的路由逻辑，但是这过程并不顺利，一方面是`AbstractHandlerMapping`定义了很多私有类和`final`方法，在实现类中都无法使用。另一方面是 FastRoute 需要在所有路由都注册后将它们编译成完整的正则表达式，而 Spring 中却没有这个事件，所以我只能在第一次请求的时候进行这个编译操作。

 [这是我目前完成的一个实现](https://github.com/ScienJus/fastroute-spring)，它的实现方式很糟糕，并且阉割了很多功能，不过已经可以进行最基本的性能测试了。我创建了大概 1000 个左右的路由（其中五分之二是静态路由，五分之三是正则路由）分别进行了两种测试：

 方式 1：使用 Jmeter 进行压力测试，线程数 200，业务逻辑仅仅是返回一个字符串。

`RequestMappingHandlerMapping`的结果：

![RequestMappingHandlerMapping](/uploads/2016/04/route2.jpg)

`FastRouteHandlerMapping`的结果

![RequestMappingHandlerMapping](/uploads/2016/04/route1.jpg)

 方式 2：直接在程序内部记录`lookupHandlerMethod`方法所消耗的时间，因为`FastRouteHandlerMapping`实际上只重写了这个方法。

`RequestMappingHandlerMapping`的结果：

```
50000 次调用时间：84370ms
```

`FastRouteHandlerMapping`的结果：

```
50000 次调用时间：2932ms
```

 可以看到性能提升还是非常明显的！


