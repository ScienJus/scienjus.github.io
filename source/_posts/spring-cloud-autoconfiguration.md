---
title: 'Spring Cloud AutoConfiguration 简介'
date: 2017-08-21 23:31:29
permalink: spring-cloud-autoconfiguration
---

将公司内部分享的一个 Slide 拆解为两部分。本文是第一部分，主要介绍一下 Spring Boot AutoConfiguration 的组成和原理。

<!--more-->

# 什么是 AutoConfiguration

Spring Boot 作为加快 Spring 项目开发的扩展框架，其中一个很重要的特性就是引入了 Starter 套件。Starter 可以在程序启动时自动初始化程序所需要的 Bean，开发者只需要关注如何使用组件本身。

Spring Boot 通过 AutoConfiguration 机制使得应用可以在启动时根据引入的类和配置，自动加载配置类（Configuration），从而在这些类中初始化所需的 Spring Bean。

一个完善的 AutoConfiguration 组件应该由四部分组成：

1. 配置类：就像开发者自己在项目中编写的一样，定义了初始化 Spring Bean 的方法。
2. 自动装载：开发者只需要引入配置类，不用像使用组件扫描（Component Scan）的方式显式的初始化配置类，避免开发者关注组件配置具体的实现。
3. 条件化加载：通过判断当前应用中引入的类库或是配置项，动态的判断项目是否需要加载某一个配置类，或是初始化某个 Bean。
4. 配置项映射：定义每个组件可以通过哪些配置项进行配置，遵从约定大于配置的原则，开发者只需要按照定义对应的值。

# 配置类

从 Spring 3 开始，开发者就可以通过 Java Config 的方式配置 Bean 了。

一个最简单的配置类由 `@Configuration` 和 `@Bean` 组成，其中前者将该类声明为一个配置类，同时也作为一个 Spring 的组件以便被扫描、注入。后者作为方法上的注解可以将方法的返回值注册为 Spring Bean。

```
@Configuration
public class GsonConfiguration {

	@Bean
	public Gson gson() {
		return new Gson();
	}

}
```

Spring Boot 中的所有配置类都在 `spring-boot-autoconfigure` 项目中，在这个项目中可以看到 Spring 为每个组件定义了哪些类、初始化了哪些 Bean，以及是如何进行配置的。

# 自动装载

对于传统的配置类，在 Spring 项目中一般都是由组件扫描（`@ComponentScan`）或是主动引入（`@Import`）的方式去进行装载。这种方式使得使用者被迫的去了解组件的配制方法和源码，极易出现配置错误等问题。

而在 Spring Boot 中，则提供了具有自动装载功能的 `@EnableAutoConfiguration` 注解，只要在项目中使用了该注解（或是其作为元注解的 `@SpringBootApplication`），就会在应用启动时自动装载所有 Spring Boot 提供的配置类。

其实现原理也是基于 Spring 现有的组件。

## 原理

![](/uploads/15032215137843.jpg)

逻辑的入口是 `@EnableAutoConfiguration`，它唯一的功能就是使用了 `@Import` 作为元注解，并引入了 `EnableAutoConfigurationImportSelector` 类。

`@Import` 是在 Spring 3 中作为 Java Config 功能所引入的，其最初作用是为了实现 XML 配置中 `<import>` 标签的功能：将多个配置引入到主配置中。所以它最开始的用途也是为了引入一些 `@Configuration` 类。

其实 Java Config 的配置方式具有更强大的表现力，例如使用者可以在注解中设置不同的值，或是通过运行一段 Java 代码动态的加载不同的配置，于是在 Spring 3.1 中就引入了两个新的功能：

- 配合 `ImportSelector`：通过读取注解属性，动态引入一些配置。
- 配合 `ImportBeanDefinitionRegistrar`：通过读取注解属性，动态注册 Bean。

其中 `EnableAutoConfigurationImportSelector` 就是一个 `ImportSelector` 的实现类。该接口只有一个方法 `selectImports`，这个方法会传入注解的元信息，最后返回需要装载的配置类的类名。

在这个实现中，最终会调用 `SpringFactoriesLoader.loadFactoryNames` 加载配置类的类名。

![](/uploads/15032270247665.jpg)

而 `SpringFactoriesLoader.loadFactoryNames` 则会去寻找项目中所有 `META-INF/spring.factories` 文件，并将其转化为 Properties，读取注解类名对应的值作为配置类的列表返回。

![](/uploads/15032279356954.jpg)

查看 spring-boot-autoconfigure 项目，确实能发现其中包含着该文件，以及对应的配置项。

![](/uploads/15032279639040.jpg)
 
至此可以得出一个结论，如果想要将一个配置类变为自动装载，只需要在项目中增加 `META-INF/spring.factories` 文件，并该类的类名作为 `ENableAutoConfiguration` 的值即可。

> 这个文件不光可以定义配置类，还可以定义 `ApplicationListener` 或是 `ApplicationContextInitializer`，一样用来实现组件自动装载的功能。

## 阻止 AutoConfiguration

阻止 AutoConfiguration 加载的方式有两种，一种是在 `@EnableAutoConfiguration` 的 `exclude` 属性中定义这个配置类的类名，另一种方式是在 `spring.autoconfigure.exclude` 配置中定义配置类的类名。一般更推荐前者，因为这种场景一般是在开发期都可以确定的。

# 条件化加载

条件化加载是在 Spring 4 中引入的新特性，而到了 Spring Boot 配合自动装载才真正发挥出其强大的功能。

这个特性就是在配置类上或是某个配置 Bean 的方法上定义一系列条件。而只有这些条件满足时，这个配置类才会进行装载，或是这个方法才会被执行。

在传统的 Spring 项目中，一般配置类都是由自己定义，所以基本上定义的配置类都是实际需要使用的，也就自然不需要添加额外的加载条件。

而 Spring Boot 遵从约定大于配置，每一个 AutoConfiguration 都会根据项目中是否引入了必要的依赖，以及是否配置了必须的配置项决定是否加载，完美的契合了条件化加载的使用场景。

## 举例

![](/uploads/15032299342952.jpg)

例如上图中初始化 Freemarker 的配置类，一共使用了 4 个条件化注解：

- `@ConditionalOnClass`：指定的 Class 存在于当前 Classpath 
- `@ConditionalOnWebApplication`：是一个 Web 应用
- `@ConditionalOnMissingBean`：指定的 Bean 不存在
- `@ConditionalOnProperty`：指定的配置项满足条件（存在、等于某个值、不等于某个值等）

## @ConditionalOnClass

由于用户定义的配置类永远会在 AutoConfiguration 之前进行装载，所以 `@ConditionalOnMissingBean` 可以很轻易的实现使用者自定义的 Bean 替代掉自动生成的 Bean 的功能。

## @Profile

另一个比较特殊的注解是 `@Profile`，这个注解允许使用者通过 `spring.profiles.active` 来控制一些 Bean 是否初始化，或是初始化不同的实例。

`@Profile` 实际是在 Spring 3 就有的功能，但是在 Spring 4 条件化注解出现后，其实现也通过相关 API 进行重写了。

## 自定义条件

虽然 Spring Boot 已经提供了很多常用的条件实现，但是在某些特殊场景依旧需要自定义加载条件。

所有的条件注解实现都是由 `@Conditional` 这个注解作为元注解，并指向一个 `Condition` 接口的实现类。

以一个具体场景举例，在 Java 应用中开发者一般使用 Slf4j 作为通用的日志桥接，从而隐藏具体的 Log4j 或是 Logback 实现。而当需要开发与日志相关的配置类时，就需要根据不同日志实现加载相关的配置类，就需要自定义一个条件注解。

首先编写一个自定义的条件注解 `@ConditionalOnSlf4jBinding`，这个注解用于指定期望 Slf4j 绑定的具体实现，在使用这个注解时可以将期望实现的类名放在 `value` 中。

```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
@Conditional(OnSlf4jBindingCondition.class)
public @interface ConditionalOnSlf4jBinding {

  String value();
}
```

同时按照规范，这个注解使用了 `@Conditional` 作为元注解，并指向了 `OnSlf4jBindingCondition`。这个类是一个 `SpringBootCondition` 的实现类，通过调用 Slf4j 的方法获取当前绑定的日志实现，并查看是否与期望值相同：

```
public class OnSlf4jBindingCondition extends SpringBootCondition {

  @Override
  public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
    String bindingClassName = attribute(metadata, "value");
    StaticLoggerBinder binder = StaticLoggerBinder.getSingleton();
    String loggerFactoryClassName = binder.getLoggerFactoryClassStr();

    return new ConditionOutcome(
        loggerFactoryClassName.equals(bindingClassName),
        String.format("Binding: %s, logger factory: %s", bindingClassName, loggerFactoryClassName));
  }

  private static String attribute(AnnotatedTypeMetadata metadata, String name) {
    return (String) metadata.getAnnotationAttributes(ConditionalOnSlf4jBinding.class.getName()).get(name);
  }

}
```

使用时就像这样：

```
@ConditionalOnSlf4jBinding("ch.qos.logback.classic.util.ContextSelectorStaticBinder")
public class LogbackSentryAppenderInitializer {
  // ...
}
```

或是更进一步的枚举出所有日志实现：

```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
@ConditionalOnSlf4jBinding("ch.qos.logback.classic.util.ContextSelectorStaticBinder")
public @interface ConditionalOnLogbackBinding {

}
```

使用时会更加简单：

```
@ConditionalOnLogbackBinding
public class LogbackSentryAppenderInitializer {
  // ...
}
```

## 加载状态

配置 `debug` 属性启动 Spring Boot 应用时，会打印一系列 AUTO-CONFIGURATION REPORT 的记录。

在这个记录中首先会把所有 AutoConfiguration 分为两组，一组为命中条件自动加载，另一组为未命中条件不进行自动加载。在这个报告中开发者可以很清晰的看到当前应用中每一个配置类因为满足/不满足某些条件最终会/不会被自动加载。

![](/uploads/15032348544518.jpg)

# 配置项映射

在配置文件中定义属性是 AutoConfiguration 唯一与使用者相关的功能。

一般在普通的应用开发中，开发者一般会使用 `@Value` 注入配置文件中的值。而在定义配置类时，更好的做法是定义一个与配置文件映射的 Model，并通过 `@ConfigurationProperties` 进行标识。

![](/uploads/15032355461124.jpg)

而当 Configuration 装载时，可以使用 `@EnableConfigurationProperties` 将对应的配置类引入，会自动注册为 Spring Bean。

![](/uploads/15032355276952.jpg)


使用这种做法的好处有很多：

- 结构化配置：支持 List、Map、URL 等复杂类型以及嵌套 Model 的映射。
- 校验：可以通过和 Hibernate Validator 等校验组件结合，通过注解进行自动校验。
- 热刷新支持：在 Spring Cloud 环境下可以在应用运行中刷新绑定的配置项。
- 配置文件提示：Spring Boot 提供了 `spring-configuration-metadata.json` 用来描述配置项，配合 IDE 可以在编写配置文件时有提示。

![](/uploads/15032352039035.jpg)

> `@Value` 实际也是支持热刷新的，但是必须定义为 `@RefreshScope`，而且实现也有不同，理论上 `@ConfigurationProperties` 的热刷新更加轻量级。

> 使用 `spring-boot-configuration-processor` 依赖可以在编译时扫描项目中的 `@ConfigurationProperties` 类，并自动生成 `spring-configuration-metadata.json` 文件。

