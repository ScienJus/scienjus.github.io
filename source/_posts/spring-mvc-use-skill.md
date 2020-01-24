---
title: '一些Spring MVC的使用技巧'
date: 2015-10-17 07:12:54
permalink: spring-mvc-use-skill
---

### APP 服务端的 Token 验证

 通过拦截器对使用了`@Authorization`注解的方法进行请求拦截，从 http header 中取出 token 信息，验证其是否合法。非法直接返回 401 错误，合法将 token 对应的 user key 存入 request 中后继续执行。具体实现代码：

```
public boolean preHandle (HttpServletRequest request,
                         HttpServletResponse response, Object handler) throws Exception {
    // 如果不是映射到方法直接通过
    if (!(handler instanceof HandlerMethod)) {
        return true;
    }
    HandlerMethod handlerMethod = (HandlerMethod) handler;
    Method method = handlerMethod.getMethod ();
    // 从 header 中得到 token
    String token = request.getHeader (httpHeaderName);
    if (token != null && token.startsWith (httpHeaderPrefix) && token.length () > 0) {
        token = token.substring (httpHeaderPrefix.length ());
        // 验证 token
        String key = manager.getKey (token);
        if (key != null) {
            // 如果 token 验证成功，将 token 对应的用户 id 存在 request 中，便于之后注入
            request.setAttribute (REQUEST_CURRENT_KEY, key);
            return true;
        }
    }
    // 如果验证 token 失败，并且方法注明了 Authorization，返回 401 错误
    if (method.getAnnotation (Authorization.class) != null) {
        response.setStatus (HttpServletResponse.SC_UNAUTHORIZED);
        response.setCharacterEncoding ("gbk");
        response.getWriter ().write (unauthorizedErrorMessage);
        response.getWriter ().close ();
        return false;
    }
    // 为了防止以某种直接在 REQUEST_CURRENT_KEY 写入 key，将其设为 null
    request.setAttribute (REQUEST_CURRENT_KEY, null);
    return true;
}
```

 通过拦截器后，使用解析器对修饰了`@CurrentUser`的参数进行注入。从 request 中取出之前存入的 user key，得到对应的 user 对象并注入到参数中。具体实现代码：

```
@Override
public boolean supportsParameter (MethodParameter parameter) {
    Class clazz;
    try {
        clazz = Class.forName (userModelClass);
    } catch (ClassNotFoundException e) {
        return false;
    }
    // 如果参数类型是 User 并且有 CurrentUser 注解则支持
    if (parameter.getParameterType ().isAssignableFrom (clazz) &&
            parameter.hasParameterAnnotation (CurrentUser.class)) {
        return true;
    }
    return false;
}

@Override
public Object resolveArgument (MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
    // 取出鉴权时存入的登录用户 Id
    Object object = webRequest.getAttribute (AuthorizationInterceptor.REQUEST_CURRENT_KEY, RequestAttributes.SCOPE_REQUEST);
    if (object != null) {
        String key = String.valueOf (object);
        // 从数据库中查询并返回
        Object userModel = userModelRepository.getCurrentUser (key);
        if (userModel != null) {
            return userModel;
        }
        // 有 key 但是得不到用户，抛出异常
        throw new MissingServletRequestPartException (AuthorizationInterceptor.REQUEST_CURRENT_KEY);
    }
    // 没有 key 就直接返回 null
    return null;
}
```

 详细分析：[RESTful 登录设计（基于 Spring 及 Redis 的 Token 鉴权）](http://www.scienjus.com/restful-token-authorization/)

 源码见：[ScienJus/spring-restful-authorization](https://github.com/ScienJus/spring-restful-authorization)

 封装好的工具类：[ScienJus/spring-authorization-manager](https://github.com/ScienJus/spring-authorization-manager)

### 使用别名接受对象的参数

 请求中的参数名和代码中定义的参数名不同是很常见的情况，对于这种情况 Spring 提供了几种原生的方法：

 对于`@RequestParam`可以直接指定 value 值为别名（`@RequestHeader`也是一样），例如：

```
public String home (@RequestParam ("user_id") long userId) {
    return "hello " + userId;
}
```

 对于`@RequestBody`，由于其使使用 Jackson 将 Json 转换为对象，所以可以使用`@JsonProperty`的 value 指定别名，例如：

```
public String home (@RequestBody User user) {
    return "hello " + user.getUserId ();
}

class User {
    @JsonProperty ("user_id")
    private long userId;
}
```

 但是使用对象的属性接受参数时，就无法直接通过上面的办法指定别名了，例如：

```
public String home (User user) {
    return "hello " + user.getUserId ();
}
```

 这时候需要使用 DataBinder 手动绑定属性和别名，我在 Stack Overflow 上找到的 [这篇文章](http://stackoverflow.com/questions/8986593/how-to-customize-parameter-names-when-binding-spring-mvc-command-objects) 是个不错的办法，这里就不重复造轮子了。

### 关闭默认通过请求的后缀名判断 Content-Type

 之前接手的项目的开发习惯是使用.html 作为请求的后缀名，这在 Struts2 上是没有问题的（因为本身 Struts2 处理 Json 的几种方法就都很烂）。但是我接手换成 Spring MVC 后，使用`@ResponseBody`返回对象时就会报找不到转换器错误。

 这是因为 Spring MVC 默认会将后缀名为.html 的请求的 Content-Type 认为是`text/html`，而`@ResponseBody`返回的 Content-Type 是`application/json`，没有任何一种转换器支持这样的转换。所以需要手动将通过后缀名判断 Content-Type 的设置关掉，并将默认的 Content-Type 设置为`application/json`：

```
@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {

    @Override
    public void configureContentNegotiation (ContentNegotiationConfigurer configurer) {
        configurer.favorPathExtension (false).
                defaultContentType (MediaType.APPLICATION_JSON);
    }
}
```

### 更改默认的 Json 序列化方案

 项目中有时候会有自己独特的 Json 序列化方案，例如比较常用的使用`0`/`1`替代`false`/`true`，或是通过`""`代替`null`，由于`@ResponseBody`默认使用的是`MappingJackson2HttpMessageConverter`，只需要将自己实现的`ObjectMapper`传入这个转换器：

```
public class CustomObjectMapper extends ObjectMapper {

    public CustomObjectMapper () {
        super ();
        this.getSerializerProvider ().setNullValueSerializer (new JsonSerializer<Object>() {
            @Override
            public void serialize (Object value, JsonGenerator jgen, SerializerProvider provider) throws IOException {
                jgen.writeString ("");
            }
        });
        SimpleModule module = new SimpleModule ();
        module.addSerializer (boolean.class, new JsonSerializer<Boolean>() {
            @Override
            public void serialize (Boolean value, JsonGenerator jgen, SerializerProvider provider) throws IOException {
                jgen.writeNumber (value ? 1 : 0);
            }
        });
        this.registerModule (module);
    }
}
```

### 自动加密 / 解密请求中的 Json

 涉及到`@RequestBody`和`@ResponseBody`的类型转换问题一般都在`MappingJackson2HttpMessageConverter`中解决，想要自动加密 / 解密只需要继承这个类并重写`readInternal`/`writeInternal`方法即可：

```
@Override
protected Object readInternal (Class<?> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
    // 解密
    String json = AESUtil.decrypt (inputMessage.getBody ());
    JavaType javaType = getJavaType (clazz, null);
    // 转换
    return this.objectMapper.readValue (json, javaType);
}

@Override
protected void writeInternal (Object object, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
    // 使用 Jackson 的 ObjectMapper 将 Java 对象转换成 Json String
    ObjectMapper mapper = new ObjectMapper ();
    String json = mapper.writeValueAsString (object);
    // 加密
    String result = AESUtil.encrypt (json);
    // 输出
    outputMessage.getBody ().write (result.getBytes ());
}
```

### 基于注解的敏感词过滤功能

 项目需要对用户发布的内容进行过滤，将其中的敏感词替换为`*`等特殊字符。大部分 Web 项目在处理这方面需求时都会选择过滤器（`Filter`），在过滤器中将`Request`包上一层`Wrapper`，并重写其`getParameter`等方法，例如：

```
public class SafeTextRequestWrapper extends HttpServletRequestWrapper {
    public SafeTextRequestWrapper (HttpServletRequest req) {
        super (req);
    }

    @Override
    public Map<String, String []> getParameterMap () {
        Map<String, String []> paramMap = super.getParameterMap ();
        for (String [] values : paramMap.values ()) {
            for (int i = 0; i < values.length; i++) {
                values [i] = SensitiveUtil.filter (values [i]);
            }
        }
        return paramMap ;
    }

    @Override
    public String getParameter (String name) {
        return SensitiveUtil.filter (super.getParameter (name));
    }
}

public class SafeTextFilter implements Filter {
    @Override
    public void init (FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter (ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        SafeTextRequestWrapper safeTextRequestWrapper = new SafeTextRequestWrapper ((HttpServletRequest) request);
        chain.doFilter (safeTextRequestWrapper, response);
    }

    @Override
    public void destroy () {

    }
}
```

 但是这样做会有一些明显的问题，比如无法控制具体对哪些信息进行过滤。如果用户注册的邮箱或是密码中也带有`fuck`之类的敏感词，那就属于误伤了。

 所以改用 Spring MVC 的 Formatter 进行拓展，只需要在`@RequestParam`的参数上使用`@SensitiveFormat`注解，Spring MVC 就会在注入该属性时自动进行敏感词过滤。既方便又不会误伤，实现方法如下：

 声明`@SensitiveFormat`注解：

```
@Target ({ElementType.FIELD, ElementType.PARAMETER})
@Retention (RetentionPolicy.RUNTIME)
@Documented
public @interface SensitiveFormat {
}
```

 创建`SensitiveFormatter`类。实现`Formatter`接口，重写`parse`方法（将接收到的内容转换成对象的方法），在该方法中对接收内容进行过滤：

```
public class SensitiveFormatter implements Formatter<String> {
    @Override
    public String parse (String text, Locale locale) throws ParseException {
        return SensitiveUtil.filter (text);
    }

    @Override
    public String print (String object, Locale locale) {
        return object;
    }
}
```

 创建`SensitiveFormatAnnotationFormatterFactory`类，实现`AnnotationFormatterFactory`接口，将`@SensitiveFormat`与`SensitiveFormatter`绑定：

```
public class SensitiveFormatAnnotationFormatterFactory implements AnnotationFormatterFactory<SensitiveFormat> {

    @Override
    public Set<Class<?>> getFieldTypes () {
        Set<Class<?>> fieldTypes = new HashSet<>();
        fieldTypes.add (String.class);
        return fieldTypes;
    }

    @Override
    public Printer<?> getPrinter (SensitiveFormat annotation, Class<?> fieldType) {
        return new SensitiveFormatter ();
    }

    @Override
    public Parser<?> getParser (SensitiveFormat annotation, Class<?> fieldType) {
        return new SensitiveFormatter ();
    }
}
```

 最后将`SensitiveFormatAnnotationFormatterFactory`注册到 Spring MVC 中：

```
@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {

    @Override
    public void addFormatters (FormatterRegistry registry) {
        registry.addFormatterForFieldAnnotation (new SensitiveFormatAnnotationFormatterFactory ());
        super.addFormatters (registry);
    }
}
```

### 记录请求的返回内容

 这里提供一种比较通用的方法，基于过滤器实现，所以在非 Spring MVC 的项目也可以使用。

 首先导入`commons-io`：

```
<dependency>
  <groupId>commons-io</groupId>
  <artifactId>commons-io</artifactId>
  <version>2.4</version>
</dependency>
```

 需要用到这个库中的`TeeOutputStream`，这个类可以将一个将内容同时输出到两个分支的输出流，将其封装为`ServletOutputStream`：

```
public class TeeServletOutputStream extends ServletOutputStream {

    private final TeeOutputStream teeOutputStream;

    public TeeServletOutputStream (OutputStream one, OutputStream two) {
        this.teeOutputStream = new TeeOutputStream (one, two);
    }

    @Override
    public boolean isReady () {
        return false;
    }

    @Override
    public void setWriteListener (WriteListener listener) {

    }

    @Override
    public void write (int b) throws IOException {
        this.teeOutputStream.write (b);
    }

    @Override
    public void flush () throws IOException {
        super.flush ();
        this.teeOutputStream.flush ();
    }

    @Override
    public void close () throws IOException {
        super.close ();
        this.teeOutputStream.close ();
    }
}
```

 然后创建一个过滤器，将原有的`response`的`getOutputStream`方法重写：

```
public class LoggingFilter implements Filter {

    private static final Logger LOGGER = LoggerFactory.getLogger (LoggingFilter.class);

    @Override
    public void init (FilterConfig filterConfig) throws ServletException {

    }

    public void doFilter (ServletRequest request, final ServletResponse response, FilterChain chain) throws IOException, ServletException {
        final ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream ();

        HttpServletResponseWrapper responseWrapper = new HttpServletResponseWrapper ((HttpServletResponse) response) {

            private TeeServletOutputStream teeServletOutputStream;

            @Override
            public ServletOutputStream getOutputStream () throws IOException {
                return new TeeServletOutputStream (super.getOutputStream (), byteArrayOutputStream);
            }
        };
        chain.doFilter (request, responseWrapper);
        String responseLog = byteArrayOutputStream.toString ();
        if (LOGGER.isInfoEnabled () && !StringUtil.isEmpty (responseLog)) {
            LOGGER.info (responseLog);
        }
    }

    @Override
    public void destroy () {

    }
}
```

 将`super.getOutputStream ()`和`ByteArrayOutputStream`分别作为两个分支流，前者会将内容返回给客户端，后者使用`toString`方法即可获得输出内容。

 其他的等想到再补充…

