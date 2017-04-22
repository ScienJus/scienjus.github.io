---
title: '尝试了一下Kotlin'
date: 2016-02-19 12:15:37
permalink: try-kotlin
---

前几天 Kotlin 正式发布了 1.0 版本，我初步了解并使用了一下，觉得确实很容易上手并且有很多非常好的特性，所以简单介绍一下。

Kotlin 由 Jetbrains 开发，在 GitHub 上开源，运行在 JVM 上，它的特点是可以与 Java 完全兼容且互相调用。所以它可以在 Java 庞大的第三方库支持下以更简洁、更安全的语法创建一个运行在 JVM 上的项目。并且 Kotlin 最低支持的 JDK 版本是 1.6。

 由于 JetBrains 同时也是 IDEA 的开发商，所以在 IDEA 上开发 Kotlin 项目是非常简单的，只需要安装插件（其中已经包含了对应的支持库），就可以创建项目开始编写程序了。

 下面介绍几个我认为还比较不错的特性（相对于 Java 而言，其实大部分特性都可以看到其他语言的影子）：

### 更安全的流程

Kotlin号称比 Java 拥有更高的安全性，这里主要举两个例子：

- 所有变量在声明时都需要指定是只读的，还是读写的
- 所有对象在声明时都需要指定该对象是否可以为`null`

 使用`var`和`val`可以声明一个变量，其中用`val`声明的变量一旦赋值后便不可以再改变了，而使用`var`声明的变量可以随时改变。例如：

```
val a = 1
a += 1 // 编译错误

var b = 1
b += 1 //2
```

 默认声明的对象是不允许为`null`的，如果你想声明一个可以为`null`的对象，需要在类型的后面添加一个`?`，例如：

```
var a: String = "abc"
a = null // 编译错误

var b: String? = "abc"
b = null //ok
```

 到这里还没有结束，当调用这两个对象的方法时，会发现`b`无法通过编译：

```
val length = a.length //ok

var length = b.length // 编译错误
```

 原因是`b`的类型是`String?`，有可能为`null`从而引发`NullPointException`，所以会编译错误。

 这时候如果是 Java，这时候就需要这样一段代码，先判断`b != null`再调用`b.length`，否则直接返回`0`：

```
int length = b != null ? b.length : 0;
```

 而在 Kotlin 中，只需要这样写就可以达到与上面 Java 代码一样的效果:

```
val length = b?.length : 0
```

`?.`的作用是一旦发现调用者为`null`时，直接停止并返回一个`null`，也许你会觉得这两句差别也不是很大，但是如果调用链很长呢？比如下面这行代码用 Java 写出来想必要费一番功夫了：

```
val name = post?.author?.friends?.get (0)?.name : ""
```

 还有一种情况是：你确信在正常逻辑中`b`不可能是`null`，如果发生了就干脆让它抛出来`NullPointException`，然后在对这个异常进行处理。那么就可以用另外一种写法：

```
val length = b!!.length
```

 归根结底，这两个特性的目的都是通过将一些运行时不可预知的错误转换成编译时可预知的错误，从而提高程序的安全性。

### 更简单的实体类

 在项目的开发中，实体类是非常常见的（例如许多人所说的 POJO、DTO、VO、Model、Form 等等），这种类的特征是只有一些私有属性和对应的`get`/`set`方法。而其中的`get`/`set`方法在大部分情况下又是毫无意义的（基本也都是通过 IDE 自动生成或是通过 lombok 在编译时生成的），然而在 Kotlin 中，实体类的定义将更加简单，只需要一行代码：

```
data class User (val name: String, var age: Int?, var country: String = 'Unknown')
```

 上面这一行简单的代码实际表达了很多的东西，首先可以看出，这个是一个`User`类，有三个字段`name`，`age`和`country`。由于它是一个`data class`，Kotlin 会自动为它生成与这三个属性相关的构造函数、`get`/`set`、`toString`、`equals`和`hashCode`等方法。

 并且由于每个字段都有一些不同的特征，所以他们之间还有一些不同：

- `name`被声明成`val`，因此它没有`set`方法，也不允许通过`name=`赋值
- `age`允许为空，所以在使用构造方法创建对象时，可以不设置这个属性的值
- `country`拥有一个默认值`Unknown`，所以它在构造方法中也是可选的

### 更强大的 switch

Kotlin 使用`when`代替 Java 中的`switch`，并且功能更加强大。举一个比较全面的例子：

```
when (obj) {
    1 -> print ("number 1")
    "Hello" -> print ("say hello")
    in 1..10 -> print ("between 1 and 10")
    is Int -> print ("number ${obj}")
    !is String -> print ("not string")
    obj.isEmpty () -> print ("empty string")
    else -> print ("Unknown")
}
```

`when`中的表达式既可以判断是否相等，也可以是条件表达式。所以不光可以代替`switch..case..`也可以代替多选项的`if..else..`，同时也没有惹人厌的`fallthrough`。

### 更方便的扩展

 在 Kotlin 中可以直接对已有的类添加新的方法，这一点比较像 Ruby。

 例如我们需要一个方法将日期类型转换成对应格式的字符串，一般在 Java 中我们会定义一个`DateUtils`类，并加入这样一个方法：

```
public static String format (Date date, String pattern) {
    if (pattern == null) {
        pattern = "yyyy-MM-dd"
    }
    return new SimpleDateFormat (pattern).format (date); // 这里只是为了写起来简单，实际不需要每次都创建 SimpleDateFormat
}
```

 使用时就是：

```
Date now = new Date ();
System.out.println (DateUtils.format (date));
```

 而在 Kotlin 中，只需要直接对原有的`Date`类进行扩展即可，写法是这样的：

```
fun Date.format (pattern: String = "yyyy-MM-dd"): String {
    return SimpleDateFormat (pattern).format (this)
}
```

 使用时直接在对象上调用即可：

```
var now = Date ();
println (now.format ())
```

 从语义上来说是不是这种方式更好呢？而且链式调用也要比传参的方式看起来更舒服些。

Kotlin 已经通过这种方式对 Java 中的许多类进行了拓展，例如深受诟病的集合类。Java 的集合类功能相当弱，虽然可以用但是用起来非常不爽。所以例如 Scala 就直接重写了一套自己的集合类给用户使用。

 与 Scala 不同的是，Kotlin 使用拓展方法来对 JDK 进行封装，这样能保证完全兼容 Java 代码。例如只需要下面这两行代码就可以实现`List.filter`方法：

```
public inline fun > Iterable.filterTo (destination: C, predicate: (T) -> Boolean): C {
    for (element in this) if (predicate (element)) destination.add (element)
    return destination
}
```

 这些内置的方法非常好用，但这并不是最重要的，最重要的是它提供了这种拓展的方式，这样即使没有这些方法，或是这些方法满足不了你的需求，你也可以自己在原有的类上定义新的方法。

### 更好的将来

Kotlin 还包含了很多特性，例如`lambda`表达式、字符串模板等，还有大量的语法糖。由于篇幅有限就不在这里一一介绍了。

 总而言之，目前 Java 语言已经有了相当长的历史了，在这点有好处有坏处。好处是它拥有庞大的生态圈，海量的资源，而坏处则是长时间的迭代和为了保持向后兼容导致语言层面留下了很多坑，而且也拖慢了语言本身的发展（Java8 已经推出很久了，但是大部分人都还在用着 Java6 和 Java7）。

 而 Kotlin 不一样，他站在 Java 这个巨人的肩膀上，可以随意的调用所有资源，但是在语言层面上它又是全新的，不用去兼容，不用去填坑，不用去向任何人妥协。我相信它的未来将是光明的。


