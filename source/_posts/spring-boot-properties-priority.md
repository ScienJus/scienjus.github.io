---
title: Spring Boot 的配置文件优先级
date: 2018-02-14 20:55:46
permalink: spring-boot-properties-priority
---

公司 Config Server 的逻辑越来越复杂了，新同事很难确定多个配置文件的关系和优先级。由于 Spring Cloud Config 是通过创建一个临时的 Spring Boot Application 加载配置文件，完全复用了 Spring Boot 本身的逻辑，于是写了这篇文章介绍一下 Spring Boot 中配置文件的优先级。

<!--more-->

## 一个真实的栗子

那么，Spring Cloud Config 究竟会配置多么复杂的规则呢？举一个真实的栗子：

```
spring:
  cloud:
    config:
      server:
        git:
          repos:
            uri: /opt/repos/common/config
            searchPaths: 'common,{application}'
            user:
              pattern: user-*
              uri: /opt/repos/user/config
              searchPaths: 'common,{application}'
```

上面是 Spring Cloud Config Server 的配置，它做了两个特殊的配置：

1. 通过设置 `pattern` 将不同的应用映射到不同的配置仓库，实现应用间的配置独立管理。
2. 通过 `searchPaths` 设置配置文件所在的文件夹，在默认情况 Config Server 只会搜索根目录下的配置文件，而上面的设置可以搜索 `/`、`common` 以及和应用同名的文件夹。

而请求配置时可以传入 `application`、`profile` 和 `label` 三个动态配置，例如：

```
# application/profile/label(optional)
http://configserver/user-api/production,beta/master
```

## 配置文件的匹配规则

这个请求会匹配到哪些配置文件呢？

在此首先忽略 `label`，因为在大部分情况下它只和 git 所使用的分支有关，所以它不涉及到具体配置文件的匹配。

在 Spring Boot 中，配置文件的搜索条件主要由三个参数组成：

1. `spring.config.name`：应用的名称，默认是 `application`（常量）加上 `spring.application.name` 的值，在 Spring Cloud Config 中对应请求的 `application`。
2. `spring.profiles.active`：当前生效的 profile，在 Spring Cloud Config 中对应请求的 `profile`
3. `spring.config.location`：搜索配置文件的路径，在 Spring Cloud Config 对应配置文件中的 `searchPaths`

Spring Cloud Config 本质是通过创建了一个临时的 Spring Boot Application，设置这些配置并复用 Spring Boot 的逻辑加载对应的配置文件。所以上面的请求最终的结果集为搜索路径 `['./', '/common', '/user-api']` 乘以应用名 `['application', 'user-api']` 乘以环境 `['', 'production', 'beta']`，共计 18 个配置文件，也就是：

```
./application.yml
./user-api.yml
./application-production.yml
./user-api-production.yml
./application-beta.yml
./user-api-beta.yml

./common/application.yml
./common/user-api.yml
./common/application-production.yml
./common/user-api-production.yml
./common/application-beta.yml
./common/user-api-beta.yml

./user-api/application.yml
./user-api/user-api.yml
./user-api/application-production.yml
./user-api/user-api-production.yml
./user-api/application-beta.yml
./user-api/user-api-beta.yml
```

## 配置文件的优先级

那么这些配置文件中的优先级是什么样呢？

Spring Boot 严格按照以下顺序进行排序：

1. profile
2. location
3. application

其中定义了多个 profile 和 location 时，越靠后的优先级越高，所以 `beta` 的优先级要大于 `production`，`./user-api` 的优先级要大于 `./common`。

所以最终的优先级为：

```
# profile/location/application 均为最高
./user-api/user-api-beta.yml

# profile/location 最高，application 次高
./user-api/application-beta.yml

# profile 最高，location 次高
./common/user-api-beta.yml
./common/application-beta.yml

# profile 最高，location 最低
./user-api-beta.yml
./application-beta.yml


# profile 次高
./user-api/user-api-production.yml
./user-api/application-production.yml
./common/user-api-production.yml
./common/application-production.yml
./user-api-production.yml
./application-production.yml

# 无 profile
./user-api/user-api.yml
./user-api/application.yml
./common/user-api.yml
./common/application.yml
./user-api.yml
./application.yml
```

## 源码解析

因为源码实现比较多而且绕，就不在这里大段的贴代码了，基本上只需要关注两处：

1. Spring Cloud Config 复用 Spring Boot 逻辑加载配置文件的实现可以看 `NativeEnvironmentRepository#findOne`。
2. Spring Boot 加载配置文件的实现可以看 `ConfigFileApplicationListener.Loader#load`。

## 更改优先级

默认情况 Spring Cloud Config 的配置会覆盖掉所有本地配置，包括命令行参数和环境变量，不过可以通过以下配置修改：

```
spring:
  cloud:
    config:
      allow-override: false # 远端配置是否允许覆盖，默认是 true
      override-none: true # 远端配置是否为最低优先级，不覆盖任何已存在配置，默认是 false
      override-system-properties: false # 远端配置是否可以覆盖系统配置，默认是 true
```

注意这些配置本身必须放在 Spring Cloud Config 中才会生效。

