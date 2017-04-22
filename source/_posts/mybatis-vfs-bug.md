---
title: 'MyBatis无法扫描Spring Boot别名的Bug'
date: 2016-03-25 10:25:41
permalink: mybatis-vfs-bug
---

这个问题发生的原因比较复杂，主要条件有 4 个：

1. 使用 Spring Boot，并使用 Spring Boot 的 Maven 插件打包
2. 使用 MyBatis（目前最新的`3.3.1`版本仍有这个问题）
3. 将 Domain 配置在单独的 Jar 包中（例如 Maven 多模块）
4. 使用`SqlSessionFactoryBean.setTypeAliasesPackage`指定包扫描 Domain

 然后你会发现：在开发时直接使用 IDEA 执行`main`方法运行时一切正常，但是打成 Jar 包后使用`java -jar`启动时配置的 Domain 别名均会失效。

 例如我有一个 Spring Boot 项目，其中分为三个 Maven 模块：

```
scienjus
----scienjus-domain
--------com.scienjus.domain.User
----scienjus-mapper
--------com.scienjus.mapper.UserMapper
--------UserMapper.xml
----scienjus-web
--------SqlSessionFactoryConfig
```

 在 SqlSessionFactoryConfig 中配置 SqlSesstionFactory：

```
@Bean
public SqlSessionFactory sqlSessionFactory () throws Exception {
    final SqlSessionFactoryBean sqlSessionFactory = new SqlSessionFactoryBean ();
    sqlSessionFactory.setDataSource (dataSource ());
    // 配置别名
    sqlSessionFactory.setTypeAliasesPackage ("com.scienjus.domain");
    return sqlSessionFactory.getObject ();
}
```

 在 UserMapper.xml 使用别名：

```
<select id="get" resultType="User">
    select * from user u where id = \#{id}
</select>
```

 开发时使用 IDEA 启动一切都会正常运行，但是如果等到运行时通过命令行启动，将会出现以下错误信息：

```
org.apache.ibatis.builder.BuilderException: Error resolving class. Cause:
org.apache.ibatis.type.TypeException: Could not resolve type alias 'User'. Cause:
java.lang.ClassNotFoundException: Cannot find class: User
```

 这个错误的大概意思是生成 Mapper 时出错了，原因是无法识别`User`这个别名，也找不到`User`这个 class。可以看出之前配的包扫描根本没有扫描到`com.scienjus.domain.User`这个类。

 为了证明这点，我翻了一下 MyBatis 的源码，然后在`org.apache.ibatis.type.TypeAliasRegistry`的`registerAliases (String packageName, Class> superType)`方法中发现了 MyBatis 是如何通过包名扫描别名类的。直接将这部分逻辑搬到`main`方法中执行试试：

```
public static void main (String [] args) {
    ResolverUtil resolverUtil = new ResolverUtil ();
    resolverUtil.find (new IsA (Object.class), "com.scienjus.domain");
    Set typeSet = resolverUtil.getClasses ();
    Iterator i$ = typeSet.iterator ();

    while (i$.hasNext ()) {
        Class type = (Class) i$.next ();
        if (!type.isAnonymousClass () && !type.isInterface () && !type.isMemberClass ()) {
            System.out.println (type.getName ());
        }
    }
}
```

 分别在 IDEA 和 Jar 中执行，会发现前者将会打印出`com.scienjus.domain.User`，后者却无任何输出结果，说明问题出在这里。

 既然锁定了问题出现的地方，就可以仔细看看这是如何发生的了。查看`ResolverUtil.find`方法，其通过`VFS.getInstance ().list (path)`方法获得 Class 文件，而`VFS.getInstance ()`默认情况下返回的是`DefaultVFS`，也就是说原因是这个类的`list`方法无法扫描到 Spring Boot 依赖 Jar 包中的类。

 再细化一下调用逻辑，就可以准备断点调试了：

```
public static void main (String [] args) throws IOException {
    DefaultVFS defaultVFS = new DefaultVFS ();
    List children = defaultVFS.list ("com/scienjus/domain");
    for (String child : children) {
        System.out.println (child);
    }
}
```

 断点调试使用`java -jar`启动的程序并没有想象中困难，IDEA 和 Eclipse 都内置了非常优秀的调试工具，略微介绍一下 IDEA 中的使用方法：

 开启 Debug 模式运行 Jar 包，并且监听一个特定的端口：

```
java -Xdebug -Xrunjdwp:transport=dt_socket,address=5005,server=y,suspend=y -jar scienjus-web.jar
```

IDEA 端在 Run -\\> Edit Configurations 中创建一个 Remote 应用，填写 IP 和监听的端口号，然后启动就可以了。

 通过断点调试我在`findJarForResource`发现了一块比较有意思的代码：

```
// If the file part of the URL is itself a URL, then that URL probably points to the JAR
try {
  for (;;) {
    url = new URL (url.getFile ());
    if (log.isDebugEnabled ()) {
      log.debug ("Inner URL: " + url);
    }
  }
} catch (MalformedURLException e) {
  // This will happen at some point and serves as a break in the loop
}
```

 这是一个死循环，唯有抛出`MalformedURLException`异常时才会跳出循环，根据上面的注释我们可以得知，这件事是必然发生的，且会将`url`指向一个想要的结果。对比一下两种方式运行时`url`最后的结果：

IDEA 中直接运行：

```
scienjus-domain/target/classes/com/scienjus/domain
```

 命令行运行：

```
scienjus-web/target/scienjus-web.jar!/lib/scienjus-domain.jar!/com/scienjus/domain
```

 之后将变量`jarUrl`的值赋为`scienjus-web/target/scienjus-web.jar!/lib/scienjus-domain.jar`，但是最后`listResources`方法会返回`null`。

 而调用这个方法时的注释则是这样说的：

```
// First, try to find the URL of a JAR file containing the requested resource. If a JAR
// file is found, then we'll list child resources by reading the JAR.
```

 也就是说，如果扫描的文件确实在一个 Jar 包中，这个方法应该返回这个 Jar 包的 URL，于是尝试一个比较粗暴的改进：

```
public static void main (String [] args) throws IOException {
    DefaultVFS defaultVFS = new DefaultVFS () {
        @Override
        protected URL findJarForResource (URL url) throws MalformedURLException {
            String urlStr = url.toString ();
            if (urlStr.contains ("jar!")) {
                return new URL (urlStr.substring (0, urlStr.lastIndexOf ("jar") + "jar".length ()));
            }
            return super.findJarForResource (url);
        }
    };
    List children = defaultVFS.list ("com/scienjus/domain");
    for (String child : children) {
        System.out.println (child);
    }
}
```

 如果这个 URL 中含有`jar!`的标识，就直接返回这个 Jar 包的地址。我不太确定这样做是否有隐患，不过我只有在扫描 Domain 别名时会用到这个类，并且这时候是正常工作的。

 既然这个类可以正常工作了，只需要将它设为默认的 VFS。在 MyBatis 的 [文档](http://www.mybatis.org/mybatis-3/zh/configuration.html#properties) 中写着可以通过配置文件更改`vfsImpl`属性更换 VFS 实现类，我这里用这个配置没有效果，原因是 Spring 的配置会在 MyBatis 配置文件之前执行，所以在读取这个配置之前`VFS.getInstall ()`已经实例化了。然后我给 MyBatis 提了个 Issue，顺道还发现这个扫描不到类的 Bug 早在去年10月 就有人提出了，也早就有解决办法了，只是需要到`3.4.1`版本才会发布。

MyBatis 官方的解决办法首先是推荐使用 [mybatis-spring-boot](https://github.com/mybatis/mybatis-spring-boot) 的`1.0.1`版本，默认已经配置了一个兼容 Spring Boot 的 VFS 实现类。或是将 [这个实现类](https://github.com/mybatis/mybatis-spring-boot/commit/35be747a4c4e53121e4c4c32d08418864095adf7) 添加到你的项目中，并手动配置。

 为了不让这些瞎折腾白费，我决定将这整个过程发布出来，教各位在使用开源项目遇到 bug 时如何定（zuo）位（si），这可能也是本文的仅剩的一点价值了。


