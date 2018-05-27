---
title: '在 Java 单元测试中 mock gRPC'
date: 2018-05-27 14:00:53
permalink: mock-grpc-in-java-unit-test
---

最近团队内在推广单元测试，我主要做一些 Java 框架和 CI 环境的支持。我们内部的 RPC 框架主要有 HTTP（Spring Cloud Feign）和 gRPC 两种，而在单元测试中一般需要 mock 跨服务之间的请求，相比之下 gRPC 的 mock 较为复杂，在此详细介绍一下。

## Mock Client?

对于大多数 RPC 框架来说，都会有一个封装抽象的比较上层的接口，即不需要考虑序列化以及通信相关的实现。所以只需要直接 mock 这类的接口，作为本地方法调用并返回对应的结果即可，不必进行真实的 RPC 请求。

以 Spring Cloud Feign 为例，Feign 的定义本身就是完全抽象的 Java 接口，同时每一个 Feign Client 又会注册成一个 Spring Bean，所以就可以通过 Spring 原生提供的 `@MockBean` 进行 mock，例如：

```
@SpringBootTest
@RunWith(SpringRunner.class)
public class UserServiceTest {

    @MockBean
    private UserInfoFeignClient client;

    @Autowired
    private UserInfoService service;

    @Test
    public void updateUserBasicInfos() {
        doReturn(null).when(client).getByMobile(any());
        UserInfoDetail user = service.register("18812345678");
        assertTrue("user should register success when mobile not exists", user != null);
    }
}
```

在最开始，我以为 gRPC 也会有类似的支持，可以通过现有的框架 mock 一个 Stub，使其返回指定的 protobuf 对象。

不幸的是，由于 gRPC 的所有源码都是由 protobuf 文件生成而来，而最重要的是：其生成的 Java Class 都是 final 的，这导致我们没有办法使用基于动态代理实现的 mock 框架去直接代理一个 Stub。

在 gRPC Java 的 [Github Issues](https://github.com/grpc/grpc-java/issues/2160) 中也有着一些类似的讨论，一部分开发者认为 Stub 不应该被定义为 final 类型，这样就可以进行 mock 了。而核心开发者认为 mock Stub 的做法本身就是错误的，真正的作法应该是 mock 一个 Server 实现，并通过 in-process 的传输方式和 Client 进行通信。

## Mock Server!

在明确了 gRPC 的 mock 只能在 Server 端进行之后，官方为此也提供了一些对应的支持，其中最核心的实现是一个 Junit4 的 Rule `GrpcServerRule`。

在这个 Rule 中，每次进行测试之前都会启动一个 in-process Server 以及一个 `MutableHandlerRegistry` 作为注册中心。之后使用者可以 mock 对应的 Server 实现并将其添加到其中，之后再使用 in-process Server 返回的 Channel 构造 Stub，最终调用该 Stub 的对应方法就可以进入到对应的 Server 逻辑中了。

下面是一个最简单的代码实现：

```
@RunWith(MockitoJUnitRunner.class)
public class SimpleGrpcClientTests {

    @Rule
    public final GrpcServerRule grpcServerRule = new GrpcServerRule().directExecutor();

    @Spy
    private SimpleGrpc.SimpleImplBase server;

    @Captor
    private ArgumentCaptor<HelloRequest> requestArgumentCaptor;
    
    private SimpleGrpc.SimpleBlockingStub stub;
    
    private GrpcClientService grpcClientService;

    @Before
    public void setup() {
        // 1. 在 before 阶段将 server 添加到 grpcServerRule 中
        grpcServerRule.getServiceRegistry().addService(server);
        // 2. 使用 grpcServerRule 的 channel 生成 stub
        stub = SimpleGrpc.newBlockingStub(grpcServerRule.getChannel());
        // 3. 通过 stub 生成一个 service
        grpcClientService = new GrpcClientService(stub);
    }

    @Test
    public void testSendMessage() {
        // 4. 为 server 添加 mock 的返回值，第一个参数是 request，第二个参数是 observer
        doAnswer((invocationOnMock) -> {
            StreamObserver<HelloReply> argument = invocationOnMock.getArgumentAt(1, StreamObserver.class);
            HelloReply reply = HelloReply.newBuilder().setMessage("World").build();
            argument.onNext(reply);
            argument.onCompleted();
            return null;
        }).when(server).sayHello(requestArgumentCaptor.capture(), any());

        String message = "Hello my world";

        // 5. 调用方法判断返回值是否是 mock 的值
        assertThat(grpcClientService.sendMessage(message)).isEqualTo("World");

        // 6. 判断 mock 方法被调用
        verify(server).sayHello(requestArgumentCaptor.capture(), any());
        
        // 7. 判断 mock 方法被调用的参数为 service 传入的参数
        assertThat(requestArgumentCaptor.getValue().getName()).isEqualTo(message);
    }
}
```

使用这种方式需要注意的是，Server 的实现必须严格的使用 `StreamObserver.class` 进行结果返回，否则会一直卡在请求中，无法正确的得到结果。

## Spring Style

当了解了最核心的 mock 实现后，让我们回到真实世界。

在大多数情况下的实际场景并没有这么简单，例如我们使用了 [yidongnan/grpc-spring-boot-starter](https://github.com/yidongnan/grpc-spring-boot-starter) 将 gRPC 和 Spring 所结合，其实现了一个 PostBeanProcessor 用于将 Channel 或是 Stub 注入到 Bean 的字段中，例如：

```
@Service
public class GrpcClientService {

    @GrpcClient("local-grpc-server")
    private Channel serverChannel;

    @GrpcStub("local-grpc-server")
    private SimpleBlockingStub stub;
}
```

在这种场景下 Mockito 的 `@InjectMocks` 和 Spring Boot 的 `@MockBean` 都是非常优秀的实现，但是由于篇幅有限，这里只展示一个参考 `MockitoAnnotations#initMocks` 的类似实现。

这个方法只需要做三件事：

1. 找到测试类中所有包含 `@Mock` 或是 `@Spy` 的字段，如果其是一个 gRPC Server 实现（继承了 `BindableService`），则将其添加到 `grpcServiceRule` 中。
2. 找到测试类中所有包含 `@Autowired` 的字段，递归遍历所有包含 `@GrpcClient` 和 `@GrpcStub` 的字段，将 `grpcServiceRule` 中的 Channel 注入到其中。
3. 在实际情况中可能会出现只有部分 Stub、Channel 需要注入的情况，所以在第一步的时候需要收集所有 mock 对象所对应的名称，而在第二步时只注入含有对应名称的字段

下面是代码示例：

```
@Slf4j
public class GrpcAnnotations {

    public static void initMocks(Object testClass, GrpcServerRule grpcServerRule) {
        if (testClass == null) {
            throw new IllegalArgumentException("testClass cannot be null");
        }
        // 扫描所有的字段，如果含有 @MockGrpc 则注册到 grpcServerRule 中
        Set<String> bindingGrpcServiceName = new HashSet<>();
        for (Field field : testClass.getClass().getDeclaredFields()) {
            if (BindableService.class.isAssignableFrom(field.getType()) && field.isAnnotationPresent(MockGrpc.class)) {
                MockGrpc mockGrpc = field.getAnnotation(MockGrpc.class);
                if (!bindingGrpcServiceName.add(mockGrpc.value())) {
                    throw new IllegalStateException("Multiple gRPC services have the same name.");
                }
                try {
                    Object instance = Mockito.spy(field.getType());
                    ReflectionUtils.makeAccessible(field);
                    field.set(testClass, instance);
                    grpcServerRule.getServiceRegistry().addService((BindableService) instance);
                } catch (Exception e) {
                    throw new IllegalStateException("Unable to inject because of reflection failure.", e);
                }
            }
        }

        // 扫描所有的字段，如果含有 @InjectGrpc 则尝试注入对应的 Channel/Stub
        for (Field field : testClass.getClass().getDeclaredFields()) {
            if (field.isAnnotationPresent(InjectGrpc.class)) {
                ReflectionUtils.makeAccessible(field);
                try {
                    injectGrpcFields(field.get(testClass), grpcServerRule, bindingGrpcServiceName);
                } catch (Exception e) {
                    throw new IllegalStateException("Unable to inject because of reflection failure.", e);
                }
            }

        }
    }

    private static void injectGrpcFields(Object instance,
                                         GrpcServerRule grpcServerRule,
                                         Set<String> bindingGrpcServiceName) {
        if (instance == null) {
            return;
        }
        for (Field field: instance.getClass().getDeclaredFields()) {
            if (Channel.class.isAssignableFrom(field.getType()) &&
                    field.isAnnotationPresent(GrpcClient.class)) {
                GrpcClient grpcClient = field.getAnnotation(GrpcClient.class);
                if (bindingGrpcServiceName.contains(grpcClient.value())) {
                    ReflectionUtils.makeAccessible(field);
                    ReflectionUtils.setField(field, instance, grpcServerRule.getChannel());
                }
            } else if (AbstractStub.class.isAssignableFrom(field.getType()) && field.isAnnotationPresent(
                    GrpcStub.class)) {
                GrpcStub grpcClient = field.getAnnotation(GrpcStub.class);
                if (bindingGrpcServiceName.contains(grpcClient.value())) {
                    ReflectionUtils.makeAccessible(field);
                    ReflectionUtils.setField(field, instance, GrpcClientUtils.createGrpcStub(field, grpcServerRule.getChannel()));
                }
            } else {
                ReflectionUtils.makeAccessible(field);
                try {
                    injectGrpcFields(field.get(instance), grpcServerRule, bindingGrpcServiceName);
                } catch (IllegalAccessException e) {
                    log.warn("Unable to inject because of reflection failure.");
                }
            }
        }
    }
}
```

如此一来，使用者只需要在每个测试运行前调用下 `GrpcAnnotations#initMocks` 即可完成所有 Server 的 mock 声明和对应 Client 的注入了。

```
@SpringBootTest
@RunWith(SpringRunner.class)
public class SpringGrpcClientIntegrationTests {

    @Rule
    public final GrpcServerRule grpcServerRule = new GrpcServerRule().directExecutor();

    @MockGrpc("local-grpc-server")
    private SimpleGrpc.SimpleImplBase server;

    @Autowired
    @InjectGrpc
    private GrpcClientService grpcClientService;

    @Before
    public void setUp() {
        GrpcAnnotations.initMocks(this, grpcServerRule);
    }

    @Test
    public void testSendMessage() {
        // ...
    }
}
```

