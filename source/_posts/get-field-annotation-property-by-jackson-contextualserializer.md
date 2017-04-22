---
title: 'Jackson使用ContextualSerializer在序列化时获取字段注解的属性'
date: 2016-02-23 02:59:26
permalink: get-field-annotation-property-by-jackson-contextualserializer
---

这个资料挺少了，所以记录一下。

 需求举例：

 在某一个项目中，数据库存储图片文件的路径，服务端将域名和路径拼接之后返回给客户端。例如数据库存储的路径为`/1.jpg`，服务端配置文件中配置的域名为`http://img.xxx.com`，最终返回给客户端`http://img.xxx.com/1.jpg`

 最简单的方法是在建立 Model（返回给客户端的 JSON 对象）时，直接拼接并赋值，例如：

```
class UserModel {

    private String avatar

    public UserModel (User user) {
        this.avatar = Config.HOST_NAME + user.getAvatar;
    }
}
```

 但是这样做过于硬编码，不易修改和维护，更好的办法是使用注解显式的声明序列化方式，在 Jackson 中就是：

```
class UserModel {

    @JsonSerialize (using = ImageURLSerialize.class)
    private String avatar

    public UserModel (User user) {
        this.avatar = user.getAvatar;
    }
}

class ImageURLSerialize extends JsonSerializer<String> {

    @Override
    public void serialize (String value, JsonGenerator jgen, SerializerProvider provider) throws IOException {
        if (!StringUtils.isEmpty (value)) {
            jgen.writeString (Config.HOST_NAME + value);
        }
        jgen.writeString (value);
    }
}
```

 这样 Jackson 会使用`ImageURLSerialize`序列化`avatar`字段 ，该类实现的`serialize`方法会自动在路径前添加域名。

 可以进一步封装为一个独立的注解使其更容易使用：

```
@Retention (RetentionPolicy.RUNTIME)
@JacksonAnnotationsInside
@JsonSerialize (using = ImageURLSerialize.class)
public @interface ImageURL {
}
```

 这样在 Model 类中只需要使用`@ImageURL`注解字段就可以了。

```
class UserModel {

    @ImageURL
    private String avatar

    public UserModel (User user) {
        this.avatar = user.getAvatar;
    }
}
```

 到此为止处理的都还算不错，直到有一天出现了新需求：图片服务器从本地搬到了阿里云 OSS 上。由于开通了 CDN 服务，图片可以在获取时生成缩略图了，只需要在图片链接后添加类似于`@100w_100h`的参数。

 在之前的基础上，最好的方法肯定是给`@ImageURL`增加一个字段标识需要附加的参数，例如：

```
@Retention (RetentionPolicy.RUNTIME)
@JacksonAnnotationsInside
@JsonSerialize (using = ImageURLSerialize.class)
public @interface ImageURL {

    // 请求图片附加的参数，会追加在图片地址后面
    String param () default "";
}
```

 这时候问题出现了，在`ImageURLSerialize`的`serialize`方法中根本无法获得`@ImageURL`注解，更别说获得注解中的参数了。

 接下来就要介绍本文的重点了，`ContextualSerializer`是 Jackson 提供的另一个序列化相关的接口，它的作用是通过字段已知的上下文信息定制`JsonSerializer`，只需要实现`createContextual`方法即可：

```
class ImageURLSerialize extends JsonSerializer<String> implements ContextualSerializer {

    private String param;

    // 必须要保留无参构造方法
    public ImageURLSerialize () {
        this ("");
    }

    public ImageURLSerialize (String param) {
        this.param = param;
    }

    @Override
    public void serialize (String value, JsonGenerator jgen, SerializerProvider provider) throws IOException {
        String url = "";
        if (!StringUtils.isEmpty (value)) {
            url = Config.HOST_NAME + value;
            if (!StringUtils.isEmpty (param)) {
                url = url + param;
            }
        }
        jgen.writeString (url);
    }

    @Override
    public JsonSerializer<?> createContextual (SerializerProvider serializerProvider, BeanProperty beanProperty) throws JsonMappingException {
        if (beanProperty != null) { // 为空直接跳过
            if (Objects.equals (beanProperty.getType ().getRawClass (), String.class)) { // 非 String 类直接跳过
                ImageURL imageURL = beanProperty.getAnnotation (ImageURL.class);
                if (imageURL == null) {
                    imageURL = beanProperty.getContextAnnotation (ImageURL.class);
                }
                if (imageURL != null) { // 如果能得到注解，就将注解的 value 传入 ImageURLSerialize
                    return new ImageURLSerialize (imageURL.value ());
                }
            }
            return serializerProvider.findValueSerializer (beanProperty.getType (), beanProperty);
        }
        return serializerProvider.findNullValueSerializer (beanProperty);
    }
}
```

`createContextual`可以获得字段的类型以及注解。当字段为`String`类型并且拥有`@ImageURL`注解时，取出注解中的值创建定制的`ImageURLSerialize`，这样在`serialize`方法中便可以得到这个值了。`createContextual`方法只会在第一次序列化字段时调用（因为字段的上下文信息在运行期不会改变），所以不用担心影响性能。

 更多的使用方式可以看 [Jackson 官方的测试用例](https://github.com/FasterXML/jackson-databind/blob/master/src/test/java/com/fasterxml/jackson/databind/contextual/TestContextualSerialization.java)。


