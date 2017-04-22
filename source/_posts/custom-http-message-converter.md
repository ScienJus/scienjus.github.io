---
title: '使用自定义HttpMessageConverter对返回内容进行加密'
date: 2015-08-21 08:57:09
permalink: custom-http-message-converter
---

今天上午技术群里的一个人问” 如何在 Spring MVC 中统一对返回的 Json 进行加密？”。

 大部分人的第一反应是通过 Spring 拦截器（Interceptor）中的`postHandler`方法处理。实际这是行不通的，因为当程序运行到该方法，是在返回数据之后，渲染页面之前，所以这时候 Response 中的输出流已经关闭了，自然无法在对返回数据进行处理。

 其实这个问题用几行代码就可以搞定，因为 Spring 提供了非常丰富的扩展支持，无论是之前提到的`Interceptor`和`MethodArgumentResolver`，还是接下来要提到的`HttpMessageConverter`。

 在 Spring MVC 的 Controller 层经常会用到`@RequestBody`和`@ResponseBody`，通过这两个注解，可以在 Controller 中直接使用 Java 对象作为请求参数和返回内容，而完成这之间转换作用的便是`HttpMessageConverter`。

`HttpMessageConverter`接口提供了 5 个方法：

- `canRead`：判断该转换器是否能将请求内容转换成 Java 对象
- `canWrite`：判断该转换器是否可以将 Java 对象转换成返回内容
- `getSupportedMediaTypes`：获得该转换器支持的 MediaType 类型
- `read`：读取请求内容并转换成 Java 对象
- `write`：将 Java 对象转换后写入返回内容

 其中`read`和`write`方法的参数分别有有`HttpInputMessage`和`HttpOutputMessage`对象，这两个对象分别代表着一次 Http 通讯中的请求和响应部分，可以通过`getBody`方法获得对应的输入流和输出流。

 这里通过实现该接口自定义一个 Json 转换器作为示例：

```
class CustomJsonHttpMessageConverter implements HttpMessageConverter {

    //Jackson 的 Json 映射类
    private ObjectMapper mapper = new ObjectMapper ();

    // 该转换器的支持类型：application/json
    private List supportedMediaTypes = Arrays.asList (MediaType.APPLICATION_JSON);

    /**
     * 判断转换器是否可以将输入内容转换成 Java 类型
     * @param clazz 需要转换的 Java 类型
     * @param mediaType 该请求的 MediaType
     * @return
     */
    @Override
    public boolean canRead (Class clazz, MediaType mediaType) {
        if (mediaType == null) {
            return true;
        }
        for (MediaType supportedMediaType : getSupportedMediaTypes ()) {
            if (supportedMediaType.includes (mediaType)) {
                return true;
            }
        }
        return false;
    }

    /**
     * 判断转换器是否可以将 Java 类型转换成指定输出内容
     * @param clazz 需要转换的 Java 类型
     * @param mediaType 该请求的 MediaType
     * @return
     */
    @Override
    public boolean canWrite (Class clazz, MediaType mediaType) {
        if (mediaType == null || MediaType.ALL.equals (mediaType)) {
            return true;
        }
        for (MediaType supportedMediaType : getSupportedMediaTypes ()) {
            if (supportedMediaType.includes (mediaType)) {
                return true;
            }
        }
        return false;
    }

    /**
     * 获得该转换器支持的 MediaType
     * @return
     */
    @Override
    public List getSupportedMediaTypes () {
        return supportedMediaTypes;
    }

    /**
     * 读取请求内容，将其中的 Json 转换成 Java 对象
     * @param clazz 需要转换的 Java 类型
     * @param inputMessage 请求对象
     * @return
     * @throws IOException
     * @throws HttpMessageNotReadableException
     */
    @Override
    public Object read (Class clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        return mapper.readValue (inputMessage.getBody (), clazz);
    }

    /**
     * 将 Java 对象转换成 Json 返回内容
     * @param o 需要转换的对象
     * @param contentType 返回类型
     * @param outputMessage 回执对象
     * @throws IOException
     * @throws HttpMessageNotWritableException
     */
    @Override
    public void write (Object o, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        mapper.writeValue (outputMessage.getBody (), o);
    }
}
```

 当前 Spring 中已经默认提供了相当多的转换器，分别有：

| 名称 | 作用 | 读支持 MediaType | 写支持 MediaType |
| --- | --- | --- | --- |
| ByteArrayHttpMessageConverter | 数据与字节数组的相互转换 | \\*/\\* | application/octet-stream |
| StringHttpMessageConverter | 数据与 String 类型的相互转换 | text/\\* | text/plain |
| FormHttpMessageConverter | 表单与 MultiValueMap<string string>的相互转换</string> | application/x-www-form-urlencoded | application/x-www-form-urlencoded |
| SourceHttpMessageConverter | 数据与 javax.xml.transform.Source 的相互转换 | text/xml 和 application/xml | text/xml 和 application/xml |
| MarshallingHttpMessageConverter | 使用 Spring 的 Marshaller/Unmarshaller 转换 XML 数据 | text/xml 和 application/xml | text/xml 和 application/xml |
| MappingJackson2HttpMessageConverter | 使用 Jackson 的 ObjectMapper 转换 Json 数据 | application/json | application/json |
| MappingJackson2XmlHttpMessageConverter | 使用 Jackson 的 XmlMapper 转换 XML 数据 | application/xml | application/xml |
| BufferedImageHttpMessageConverter | 数据与 java.awt.image.BufferedImage 的相互转换 | Java I/O API 支持的所有类型 | Java I/O API 支持的所有类型 |

 回到最开始所提的需求，既然要对返回的 Json 内容进行加密，肯定是对`MappingJackson2HttpMessageConverter`进行改造，并且只需要重写`write`方法。

 从`MappingJackson2HttpMessageConverter`的父类`AbstractHttpMessageConverter`中的`write`方法可以看出，该方法通过`writeInternal`方法向返回结果的输出流中写入数据，所以只需要重写该方法即可:

```
@Bean
public MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter () {
    return new MappingJackson2HttpMessageConverter () {
        // 重写 writeInternal 方法，在返回内容前首先进行加密
        @Override
        protected void writeInternal (Object object,
                                     HttpOutputMessage outputMessage) throws IOException,
                HttpMessageNotWritableException {
            // 使用 Jackson 的 ObjectMapper 将 Java 对象转换成 Json String
            ObjectMapper mapper = new ObjectMapper ();
            String json = mapper.writeValueAsString (object);
            LOGGER.error (json);
            // 加密
            String result = json + "加密了！";
            LOGGER.error (result);
            // 输出
            outputMessage.getBody ().write (result.getBytes ());
        }
    };
}
```

 在这之后还需要将这个自定义的转换器配置到 Spring 中，这里通过重写`WebMvcConfigurer`中的`configureMessageConverters`方法添加自定义转换器：

```
// 添加自定义转换器
@Override
public void configureMessageConverters (List> converters) {
    converters.add (mappingJackson2HttpMessageConverter ());
    super.configureMessageConverters (converters);
}
```

 测试一下：

![测试结果](/uploads/2015/08/result.png)

 如此便简单的完成了对返回内容进行加密的功能。


