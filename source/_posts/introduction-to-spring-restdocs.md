---
title: 'Spring REST Docs 介绍'
date: 2018-02-25 18:14:55
permalink: introduction-to-spring-restdocs
---

Spring REST Docs 是一个为 Spring 项目生成 API 文档的框架，它通过在单元测试中额外添加 API 信息描述，从而自动生成对应的文档片段。

本文会以一个最简单的示例介绍如何在一个 Spring Boot 应用中使用 Spring REST Docs，并在最后与目前最常见的 SpringFox 进行一些对比，分别介绍其特点和优劣。

<!--more-->

## 使用示例

本文的示例源码托管在 Github 上，你可以通过[这个地址](https://github.com/ScienJus/learn-spring-restdocs)下载并在本地运行。

### 基础准备

首先需要一个 Spring Boot 项目，并通过 MockMvc 编写一些简单的测试。

```java
@RestController
public class HelloController {

    @GetMapping("hello")
    public Result hello(@RequestParam("name") String name) {
        return new Result(200, String.format("Hello %s!", name));
    }
}
```

在上面代码中提供了一个最简单的 Controller，其接收请求参数中的 `name` 属性，并返回一个包含 `code` 和 `msg` 的 Result 对象。

接下来需要为其编写一个测试：

```java
@WebMvcTest
@ExtendWith(SpringExtension.class)
public class HelloControllerTests {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void testHello() throws Exception {
        mockMvc.perform(get("/hello").param("name", "ScienJus"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("msg", "Hello ScienJus!").exists())
}
```

在这里使用了 JUnit5 和 Spring 的 MockMvc 编写 API 测试，只是简单的请求这个 API 并校验返回值。

完成以上工作，就可以开始通过修改测试代码，为这个 API 自动生成相关的描述文档了。

### 配置 Spring REST Docs

当使用 MockMvc 时，只需要添加 `spring-restdocs-mockmvc` 依赖：

```xml
<dependency>
    <groupId>org.springframework.restdocs</groupId>
    <artifactId>spring-restdocs-mockmvc</artifactId>
    <scope>test</scope>
</dependency>
```

之后，需要修改测试代码，添加对应的文档支持：

```java
@WebMvcTest
@ExtendWith({RestDocumentationExtension.class, SpringExtension.class}) <1>
public class HelloControllerTests {

    private MockMvc mockMvc;

    @BeforeEach <2>
    public void setUp(WebApplicationContext webApplicationContext,
                      RestDocumentationContextProvider restDocumentation) {
        this.mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext)
                .apply(documentationConfiguration(restDocumentation))
                .build();
    }

    @Test
    public void testHello() throws Exception {
        mockMvc.perform(get("/hello").param("name", "ScienJus"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("msg", "Hello ScienJus!").exists())
                .andDo(document("hello")); <3>
    }
}
```

1. 在 `@ExtendWith` 中增加 `RestDocumentationExtension`（JUnit5 的 Extension 相当于 JUnit4 中的 Rule）。
2. 将 `MockMvc` 由直接注入改为手动构建，增加 `documentationConfiguration(restDocumentation)` 配置。
3. 在执行测试的最后，调用 `andDo(document("hello"))` 给测试调用所生成的文档命名。

### 构建文档

完成配置后，运行 `mvn clean package` 进行构建，当测试运行成功后查看 `target/generated-snippets` 下出现的一系列 adoc 文档：

![](/uploads/15269065775245.jpg)

其中 `curl/httpie-request.adoc` 记录了测试请求通过 curl 和 httpie 的调用方式， `http-request/response.adoc` 记录了测试请求和返回的 raw 信息，`request/response-body.adoc` 记录了请求和返回的 Payload。

不过这些都只是一个个文档片段，还需要将其拼凑到一起才能成为一份完整的 API 文档，框架本身不提供直接生成完整文档的功能，所以需要编写一个文档主页并引入这些自动生成的文档片段。

默认的文档主页可以放在 `src/main/asciidoc/index.adoc` 中，例如：

```adoc
= Learn Spring REST Docs
:toc: left

Learn how to use Spring REST Docs based on Spring Boot2 and JUnit5.

== /hello: Say "Hello World!"

operation::hello[]
```

其中最重要的一行是 `operation::hello[]`，它表示将 hello 下的所有片段都引入进入，或者也可以指定 `operation::hello[snippets='curl-request,http-request,http-response']` 的方式只引入部分代码片段。

编写好文档主页后，需要使用 `asciidoctor-maven-plugin` 使其可以在打包时与片段整合起来，并生成最终的 HTML 文件：

```xml
<plugin>
    <groupId>org.asciidoctor</groupId>
    <artifactId>asciidoctor-maven-plugin</artifactId>
    <version>1.5.3</version>
    <executions>
        <execution>
            <id>generate-docs</id>
            <phase>prepare-package</phase>
            <goals>
                <goal>process-asciidoc</goal>
            </goals>
            <configuration>
                <backend>html</backend>
                <doctype>book</doctype>
            </configuration>
        </execution>
    </executions>
    <dependencies>
        <dependency>
            <groupId>org.springframework.restdocs</groupId>
            <artifactId>spring-restdocs-asciidoctor</artifactId>
            <version>2.0.1.RELEASE</version>
        </dependency>
    </dependencies>
</plugin>
```

此时再次运行 `mvn clean package` 之后，可以看到 `target/generated-docs` 下生成了最终的网页，其最终效果如下图所示。

![](/uploads/15269074752006.jpg)

至此，最简单的请求文档便构建完成了。

### 添加更多描述

对于目前这份文档来说，其仅仅记录了最原始的请求信息，却没有任何相关的文字描述，所以接下来需要给请求和返回增加额外的描述信息。

```java
mockMvc.perform(get("/hello").param("name", "ScienJus"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("msg", "Hello ScienJus!").exists())
        .andDo(document("hello",
                requestParameters(
                        parameterWithName("name").description("The name to retrieve")), <1>
                responseFields(
                        fieldWithPath("code").description("Code of the response"),
                        fieldWithPath("msg").description("Message of the response")))); <2>
```

在上面代码中增加了 `requestParameters` 定义请求参数的描述，以及通过 `responseFields` 定义返回值的描述，初次之外，还有 `pathParameters`、`requestHeaders`、`requestFields` 等分别用于描述路径变量、Header 信息、Payload 信息的方法。

需要注意的是，所有增加描述的字段都会在测试请求中进行校验，如果文档中定义的参数在实际的测试中并没有出现，测试会直接失败，这样可以保证文档描述和最终运行结果是一致的。

再次重新构建，可以看到生成的文档片段中多出了 `request-parameters.adoc` 和 `response-fields.adoc` 两个文件，就是在测试代码中定义的描述信息了。

![](/uploads/15269084559348.jpg)

介于篇幅有限，本文对 Spring REST Docs 的基本使用介绍就到此为止了，更多的配置和自定义项可以在[官方文档](https://docs.spring.io/spring-restdocs/docs/current/reference/html5/) 中查看。

## 和 SpringFox 的对比

相较于传统且更流行的 SpringFox（Swagger），Spring REST Docs 的实现方式相当新颖，而且有着鲜明的区别，那么不妨在此列举一下两者的区别以及优劣，以便更好的根据实际需求和使用场景选择最合适的工具。

首先，两者最大的区别就在于根本定位，SpringFox 的定位是和应用一起启动的在线文档，文档的浏览者可以很简单的填写表单并发起一个真实的请求，而 Spring REST Docs 更倾向于导出一份离线文档作为展示，并配合 curl、httpie 这种工具请求真实部署的服务。

其次，SpringFox 最大的特点是使用简单，只需要在源码中增加一些描述性的注解即可完成整份文档，而使用 Spring REST Docs 的前提条件是需要在项目中对 API 进行单元测试，并且要保证测试是可以稳定执行的，这对于很多团队来说无疑增加了很高的门槛。

但是对于已经有完整单元测试的团队来说，增加额外的文档描述几乎和 SpringFox 一样简单，并且还能完整的去除源码依赖。除此之外，依靠测试本身也正是 Spring REST Docs 的最大亮点：

首先，每一次测试都是一个真实的请求（不追究 MockMvc 具体实现细节），它所对应的请求和返回都是真实的，可以轻松将其记录下来作为 Demo 展示。而 SpringFox 只是对 Controller 层的方法进行了扫描，却无法感知 Interceptor、MethodArgumentResolver 这类中间件的存在，只能通过一些全局配置进行额外的描述。

其次，每一次测试也都是一个独立的请求，使得 Spring REST Docs 可以描述同一个 API 在不同请求参数中返回的不同结果的场景（例如成功或是各种失败情况），而 SpringFox 只能描述单一的方法签名和返回值 Model，却无法描述其具体可能出现的场景。

最后，错误的文档比没有文档还要糟糕，所以 Spring REST Docs 不仅仅是做 API 文档化，同时也是在做 API 契约化，如果 API 的实现修改破坏了已有的测试，哪怕仅仅是字段定义，都会导致测试的失败。这可以督促 API 的制定者保证对外提供的契约，也可以让 API 的使用者更加放心。

所以相比之下，如果一个技术氛围良好，对服务严格负责，且愿意尝试 API 单元测试和契约测试的团队来说，我更推荐使用 Spring REST Docs，而如果只是在已有的服务上增加描述性的文档，SpringFox 会是性价比更高的选择。

