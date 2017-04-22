---
title: 'Spring Boot使用心得'
date: 2015-11-03 07:19:16
permalink: spring-boot-summary
---

Spring Boot 并不是一个全新的框架，而是将已有的 Spring 组件整合起来。特点是去掉了繁琐的 XML 配置，改使用约定或注解。所以熟悉了 Spring Boot 之后，开发效率将会提升一个档次。

 约定优于配置的这种做法在如今越来越流行了，它的特点是简单、快速、便捷。但是这是建立在程序员熟悉这些约定的前提上。而 Spring 拥有一个庞大的生态体系，刚开始转到 Spring Boot 完全舍弃 XML 时肯定是不习惯的，所以也会造成一些困扰。这里介绍一下一些常用的心得。

### 运行方式

`spring-boot-starter-web`包含了 Spring MVC 的相关依赖（包括 Json 支持的 Jackson 和数据校验的 Hibernate Validator）和一个内置的 Tomcat 容器，这使得在开发阶段可以直接通过`main`方法或是 JAR 包独立运行一个 WEB 项目。而在部署阶段也可以打成 WAR 包放到生产环境运行。

```
@SpringBootApplication
public class Application extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure (SpringApplicationBuilder application) {
        return application.sources (Application.class);
    }

    public static void main (String [] args) throws Exception {
        SpringApplication.run (Application.class, args);
    }

}
```

 在拥有`@SpringBootApplication`注解的类中，使用`SpringApplication`的`run`方法可以通过 JAR 启动项目。

 继承`SpringBootServletInitializer`类并实现`configure`方法，使用`application`的`sources`方法可以通过 WAR 启动项目。

### 配置文件

Spring boot 的默认配置文件是`resources`下的`application.properties`和`application.yml`。

 我曾在项目中遇到过`application.properties`出现中文乱码问题，当时尝试了很多办法都没有解决。Spring Boot 总是会以`iso-8859`的编码方式读取该文件，后来改用 YAML 了就再也没有出现过乱码了。并且它也拥有更简洁的语法，所以在此也更推荐使用`application.yml`作为默认的配置文件。

 配置文件中可以定义一个叫做`spring.profiles.active`的属性，该属性可以根据运行环境自动读取不同的配置文件。例如将该属性定义为`dev`的话，Spring Boot 会额外从`application-dev.yml`文件中读取该环境的配置。

Spring Boot 注入配置文件属性的方法有两种，一种是通过`@Value`注解接受配置文件中的属性，另外一种是通过`@ConfigurationProperties`注解通过`set`方法自动为 Bean 注入对应的属性。

 通过`@Value`注入属性，接收者既可以是方法参数，也可以是成员变量。例如配置文件为：

```
dataSource:
    url: jdbc:mysql://127.0.0.1:3306/test
    username: test
    password: test
    filters: stat,slf4j

redis:
    host: 192.168.1.222
    port: 6379
```

 通过`@Value`接受方法参数初始化 Bean：

```
@Bean
public JedisPool jedisPool (@Value ("${redis.host}") String host,
                            @Value ("${redis.port}") int port) {
    return new JedisPool (host, port);
}
```

 通过`@ConfigurationProperties`读取配置初始化 Bean，会直接调用对应的`set`方法注入：

```
@Bean (initMethod="init",destroyMethod="close")
@ConfigurationProperties (prefix="dataSource")
public DataSource dataSource () {
    return new DruidDataSource ();
}
```

Spring Boot 目前还无法直接注入的静态变量。我目前使用的方法是专门建立一个读取配置文件的 Bean，然后使用`@PostConstruct`注解修饰的方法对这些静态属性进行初始化，例如：

```
@Configuration
public class ConstantsInitializer {

    @Value ("${paging_size}")
    private String pagingSize;

    @PostConstruct
    public void initConstants () {
        Constants.PAGING_SIZE = this.pagingSize;
    }
}
```

### Servlet

Servlet 中最重要的配置文件就是`web.xml`，它的主要用途是配置 Servlet 映射和过滤器。而在 Spring Boot 中这将简单很多，只需要将对应的`Servlet`和`Filter`定义为 Bean 即可。

 声明一个映射根路径的 Servlet ，例如 Spring MVC 的`DispatcherServlet`：

```
@Bean
public DispatcherServlet dispatcherServlet () {
    return new DispatcherServlet ();
}
```

 需要注意的是，Spring Boot 默认会自动创建`DispatcherServlet`的映射。但这是在项目中没有手动声明其他 Servlet Bean 的情况下，否则就需要也将这个 Bean 一起声明。

 声明一个映射特定路径的 Servlet ，或是需要配置初始化参数的话，则需要使用`ServletRegistrationBean`。例如 Druid 的`StatViewServlet`：

```
@Bean
public ServletRegistrationBean statViewServlet () {
    ServletRegistrationBean reg = new ServletRegistrationBean ();
    reg.setServlet (new StatViewServlet ());
    reg.addUrlMappings ("/druid/*");
    return reg;
}
```

 声明过滤器也是如此，例如 Spring MVC 的`CharacterEncodingFilter`：

```
@Bean
public CharacterEncodingFilter characterEncodingFilter () {
    CharacterEncodingFilter filter = new CharacterEncodingFilter ();
    filter.setEncoding ("UTF-8");
    filter.setForceEncoding (true);
    return filter;
}
```

 复杂一点的同样是通过类似的`FilterRegistrationBean`，例如：

```
@Bean
public FilterRegistrationBean appFilter () {
    FilterRegistrationBean reg = new FilterRegistrationBean ();
    reg.setFilter (new LoggingFilter ());
    reg.addUrlPatterns ("/api/*");
    return reg;
}
```

### Spring MVC

Spring MVC 主要的配置都可以通过继承`WebMvcConfigurerAdapter`（或者`WebMvcConfigurationSupport`）类进行修改，这两个类的主要方法有：

- `addFormatters`：增加格式化工具（用于接收参数）
- `configureMessageConverters`：配置消息转换器（用于`@RequestBody`和`@ResponseBody`）
- `configurePathMatch`：配置路径映射
- `addArgumentResolvers`：配置参数解析器（用于接收参数）
- `addInterceptors`：添加拦截器

 总之几乎所有关于 Spring MVC 都可以在这个类中配置。之后只需要将其设为`@Configuration`，Spring Boot 就会在运行时加载这些配置。

 还有一些常用的 Bean 默认会自动创建，但是可以通过自定义进行覆盖，例如负责 `@RequestBody` 和 `@RequestBody` 进行转换的 `MappingJackson2HttpMessageConverter` 和 `ObjectMapper`，可以直接这样覆盖掉：

```
@Bean
public MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter () {
    return new CustomMappingJackson2HttpMessageConverter ();
}

@Bean
public ObjectMapper jsonMapper (){
    ObjectMapper objectMapper = new ObjectMapper ();
    //null 输出空字符串
    objectMapper.getSerializerProvider ().setNullValueSerializer (new JsonSerializer<Object>() {
        @Override
        public void serialize (Object value, JsonGenerator jgen, SerializerProvider provider) throws IOException {
            jgen.writeString ("");
        }
    });
    return objectMapper;
}
```

### DataSource

 如果使用了`spring-boot-starter-data-jpa`，Spring Boot 将会自动创建一个 DataSource Bean。可以直接在配置文件中定义它的属性，前缀是`spring.datasource`。并且无需指定数据库的方言，这个 Bean 会自动根据项目中依赖的数据库驱动判断使用的哪种数据库。

 同样的，如果使用了`spring-boot-starter-data-redis`，也会自动创建`RedisTemplate`、`ConnectionFactory`等 Bean。也同样可以在配置文件中定义属性，前缀是`spring.redis`。


