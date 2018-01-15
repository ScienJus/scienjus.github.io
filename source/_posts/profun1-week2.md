---
title: '[Functional Programming Principles in Scala] 学习笔记（二） 高阶函数和类'
date: 2017-05-01 21:03:21
permalink: profun1-week2
---

本周主要介绍了 Scala 中的高阶函数和类的相关定义，包含高阶函数和柯里化、类的构造与抽象等内容。

<!--more-->

## 高阶函数

函数式语言将函数作为一等公民，这意味着函数可以像其他值一样作为参数或是返回值，这种做法提高了程序的灵活性。

将其他函数作为参数或者返回值的函数被称为高阶函数（Higher Order Functions）。

一个例子，下面这个方法用于计算两个整数之间的所有整数之和：

```scala
def sumInts(a: Int, b: Int): Int =
  if (a > b) 0 else a + sumInts(a + 1, b)
```

另一个方法用于计算两个整数之间的所有整数立方的和：

```scala
def cube(x: Int): Int = x * x * x

def sumCubes(a: Int, b: Int): Int =
  if (a > b) 0 else cube(a) + sumCubes(a + 1, b)
```

还有一个方法用于计算两个整数之间所有整数阶乘的和：

```scala
def fact(x: Int): Int = if (x == 0) 1 else x * fact(x - 1)

def sumFactorials(a: Int, b: Int): Int =
  if (a > b) 0 else fact(a) + sumFactorials(a + 1, b)
```

可以看出，这三个方法的大部分模式都是相同的，它们都是通过递归获得 a 到 b 的所有整数，通过某个方法进行转换，最后将转换得到的值进行累加。

那么就可以将这个转换的函数提取成为方法参数：

```scala
def sum(f: Int => Int, a: Int, b: Int): Int =
  if (a > b) 0
  else f(a) + sum(f, a + 1, b)
```

使用时在不同的实现中传入不同的函数即可：

```scala
def id(x: Int): Int = x
def sumInts(a: Int, b: Int) = sum(id, a, b)

def cube(x: Int): Int = x * x * x
def sumCubes(a: Int, b: Int) = sum(cube, a, b)

def fact(x: Int): Int = if (x == 0) 1 else x * fact(x - 1)
def sumFactorials(a: Int, b: Int) = sum(fact, a, b)
```

在 Scala 中， `A => B` 代表一个接受一个 `A` 类型参数，并返回一个 `B` 类型参数的方法。例如上文中的 `Int => Int` 代表将一个整数转换为另一个整数的方法。

## 匿名函数

在使用高阶函数时，不可避免的需要定义很多小函数，但是其实很多时候不需要通过 `def` 定义函数并为其起一个名字。

以字符串举例，当需要打印一个常量字符串时，以下的代码是多余的：

```scala
def str = "ABC"
println(str)
```

它可以直接写为：

```scala
println("ABC")
```

就像字符串一样，函数也可以作为一个常量存在，它们被称为匿名函数（Anonymous Functions）。

上文中 `cube` 的匿名函数形式为：

```scala
(x: Int) => x * x * x
```

其中 `(x: Int)` 是该函数的参数，`x * x * x` 是该函数的函数体。

如果函数有多个参数，那么彼此之间需要用 `,` 分隔，例如：


```scala
(x: Int, y: Int) => x + y
```

如果函数的类型可以通过上下文推断得出，那么是可以省略的。

一个匿名函数 

```
(x1 : T1, ..., xn : Tn) => E
``` 

可以被定义为一个形如 

```
{ def f(x1 : T1, ..., xn : Tn) = E; f }
``` 

的表达式，其中 `f` 是一个任意的未被占用的名称。所以可以把匿名函数当做一个语法糖（Syntactic Sugar）。

通过匿名函数，上文中的方法又可以进一步简化：

```scala
def sumInts(a: Int, b: Int) = sum(x => x, a, b)

def sumCubes(a: Int, b: Int) = sum(x => x * x * x, a, b)
```

## 柯里化

再次观察上面的函数，它们是否还有进一步优化的空间？

在上面的函数实现中，参数 `a` 和 `b` 都没有经过任何处理，而是直接传到了 `sum` 函数中，是否有更好的写法隐藏这些参数呢？

```scala
def sum(f: Int => Int): (Int, Int) => Int = {
  def sumF(a: Int, b: Int): Int =
    if (a > b) 0
    else f(a) + sumF(a + 1, b)
  sumF
}
```

这个重写的函数不再接受两个 Int 类型的参数，而是直接将另一个函数作为了返回值，这个返回的函数才接受两个 Int 类型参数，并返回最终的结果。

上文中的函数定义将会变得更加简单：

```scala
def sumInts = sum(x => x)
def sumCubes = sum(x => x * x * x)
def sumFactorials = sum(fact)
```

甚至可以避免定义这些中间变量，直接通过原始方法调用：

```scala
sum(cube)(1, 10)
```

在这当中 `sum(cube)` 返回了一个计算阶乘之和的方法，它和 `sumCubes` 是完全一样的，并且可以直接通过紧接着的 `(1, 10)` 参数调用这个方法。

在函数中返回另一个函数是非常有用的，为此 Scala 有一种特殊的语法：

```scala
def sum(f: Int => Int)(a: Int, b: Int): Int =
  if (a > b) 0 
  else f(a) + sum(f)(a + 1, b)
```

这段方法和上面返回 `sumF` 的实现几乎是一样的，但是写起来更简洁。

如果定义了一个含有多个参数列表的方法：

```scala
def f(args1)...(argsn) = E
```

它实际等同于：

```scala
def f(args1)...(argsn−1) = { def g(argsn) = E; g }
```

或是像匿名函数一样：

```scala
def f(args1)...(argsn−1) =  (argsn ⇒ E)
```

往复替换 n 次之后，就会变为：

```scala
def f = (args1 ⇒ (args2 ⇒ ...(argsn ⇒ E)...)
```

这种风格被称为柯里化（Currying）。

最终定义的 `sum` 方法的类型为：

```scala
(Int => Int) => (Int, Int) => Int
``` 

因为函数类型的关联是从右向左的，所以实际等同于：

```scala
(Int => Int) => ((Int, Int) => Int)
```

## Scala 语法汇总

以下定义中所用到的符号的含义为：

- `|` ：替代关系
- `[...]` 0 或 1 个
- `{...}` 0 或 多个

### 类型（Types）

![Types](/uploads/types.png)

类型可以是：

- 数字：Int、Double（Byte、Short、Char、Long、Float）
- 布尔：true 或 false
- 字符串
- 函数：像是 `Int => Int` 或者 `(Int, Int) => Int`

### 表达式（Expressions）

![Expressions](/uploads/expressions.png)

表达式可以是：

- 标识符：例如 `x` 或是 `isGoodEnough`
- 常量：例如 `0`、`1.0` 或是 `"abc"`
- 执行函数：例如 `sqrt(x)`
- 执行运算符：例如 `-x` 、`x + y`
- 选择表达式：例如 `math.abs` （这里不太懂 `selection`是指的什么，该方法的内部实现是用的选择表达式？）
- 条件表达式：例如 `if (x < 0) -x else x`
- 代码块：例如 `{ val x = math.abs(y) ; x * 2 }`
- 匿名函数：例如 `x => x + 1`

### 定义（Definitions）

![Definitions](/uploads/definitions.png)

定义可以是：

- 方法定义：例如 `def square(x: Int) = x`
- 值定义：例如 `val y = square(2)`

其中参数可以是：

- 值调用：例如 `(x: Int)`
- 名称调用：例如 `(y: => Double)`

## 函数和数据

本节通过一个例子介绍如何在 Scala 中使用函数创建和封装结构体。

一个分数由一个整数分子和另一个整数分母组成。如果需要计算两个分数的和，就需要定义两个如下的方法：

```scala
def addRationalNumerator(n1: Int, d1: Int, n2: Int, d2: Int): Int
def addRationalDenominator(n1: Int, d1: Int, n2: Int, d2: Int): Int
```

但是这样做明显增加了代码的维护成本，一种更好的方式是将分子和分母共同维护在一个结构体中。

### 类

在 Scala 中，可以用下面这种方式定义一个类（Classes）：

```scala
class Rational(x: Int, y: Int) {
  def numer = x
  def denom = y
}
```

这段定义包含了两部分：

1. 一个新的类型（Type）：Rational
2. 一个可以用于创建 Rational 实例的构造方法（Constructor）

Scala 会保证定义的名称和值在不同的命名空间（Namespace）中，所以多个 Rational 定义彼此之间不会冲突（？）

### 对象

每个类型的元素被称为对象（Objects），通过 `new` 加上构造方法可以创建一个新的对象：

```scala
new Rational(1, 2)
```

每个 Rational 对象都有两个成员变量（Members）：`numer` 和 `denom`。通过 `.` 操作符可以获取对象的成员变量：

```scala
val x = new Rational(1, 2) > x: Rational = Rational@2abe0e27
x.numer                    > 1
x.denom                    > 2
```

### 方法

在拥有 Rational 对象之后，就可以对其定义一些计算函数了：

```scala
def addRational(r: Rational, s: Rational): Rational =
  new Rational(r.numer * s.denom + s.numer * r.denom, r.denom * s.denom)

def makeString(r: Rational) =
  r.numer + ”/” + r.denom
  
makeString(addRational(new Rational(1, 2), new Rational(2, 3))) > 7/6
```

在此之上，还可以直接将函数抽象为结构体本身，这样的函数被称为方法（Methods）。

Rational 类本身就可以有 `add` 和 `toString` 方法：

```scala
class Rational(x: Int, y: Int) {
  def numer = x
  def denom = y

  def add(r: Rational) =
    new Rational(numer * r.denom + r.numer * denom, denom * r.denom)
  
  override def toString = numer + ”/” + denom
}
```

注意：`toString` 是由 `java.lang.Object` 继承而来的方法，所以需要加上 `override` 关键词。

这样调用时就可以变为：

```scala
val x = new Rational(1, 3)
val y = new Rational(5, 7)
val z = new Rational(3, 2)
x.add(y).add(z)
```

### 抽象

在上面的例子中，可以发现通过计算而得出的分数有可能不是最简形态（例如 `3/6` 可以被约为 `1/2`）。

为此我们可以在每一个计算分数的方法中都加入化简的逻辑，但是这会使代码难以维护，很有可能在某个计算中忘记加入这部分逻辑。

一个更好的办法是直接在构造分数时就进行化简：

```scala
class Rational(x: Int, y: Int) {
  private def gcd(a: Int, b: Int): Int = if (b == 0) a else gcd(b, a % b)
  private val g = gcd(x, y)

  def numer = x / g
  def denom = y / g

}
```

上面方法中的 `gcd` 和 `g` 都是私有成员，它们只能在该对象内部被访问到。

另一种方式是将 `numer` 和 `denom` 都声明为 `val`，然后直接用 `gcd` 方法去计算，这样可以保证 `numer` 和 `denom` 只会初始化一次：

```scala
class Rational(x: Int, y: Int) {
  private def gcd(a: Int, b: Int): Int = if (b == 0) a else gcd(b, a % b)

  val numer = x / gcd(x, y)
  val denom = y / gcd(x, y)

}
```

上面两种方式对调用方都是无感知的，但是可以通过具体的情况选择不同的实现方案，这种方式称之为抽象（Abstraction）。

抽象是软件工程中的基石。

### 自引用

在类的内部可以使用 `this` 关键词指代当前执行方法的对象，也就是自引用（Self Reference）。

例如为 Rational 添加 `less` 和 `max` 方法：

```scala
class Rational(x: Int, y: Int) {  ...  def less(that: Rational) =    this.numer * that.denom < that.numer * this.denom

  def max(that: Rational) =    if (this.less(that)) that else this}
```

### 前提检验

假设 Rational 类要求分母必须是一个正整数，就可以通过 `require` 方法进行校验：

```scala
class Rational(x: Int, y: Int) {  require(y > 0, ”denominator must be positive”)  ...}
```

`require` 是一个预定义方法，它需要一个条件以及可选的提示信息。当条件为假时，将会抛出一个携带提示信息的 `IllegalArgumentException` 异常。

### 断言

另一种校验的方式是使用断言（Assert），它同样接受一个条件和可选的提示信息，而当条件不满足时，它会抛出 `AssertionError` 异常。

两个异常的不同代表着这两种方式分别适合用于不同的场景：

- `require` 适合在方法执行前校验外部传入的参数
- `assert` 用于校验方法执行过程中的逻辑

### 构造函数

在 Scala 中，类定义就会隐式的引入一个构造函数，它被称为主构造函数（Primary Constructor）。

构造函数的主要用途是：

- 接收传入的参数
- 执行类体中的所有语句

除了主构造函数以外，还可以定义辅助构造函数，例如：

```scala
class Rational(x: Int, y: Int) {  def this(x: Int) = this(x, 1)  ...}new Rational(2) > 2/1
```

## 类中的代换模型

在之前的笔记中有提到 Scala 的函数执行是通过一种称为代换模型的计算模型，在类和对象中也是如此。

当构建一个类实例 `new C(e1, ..., em)` 时，它的表达式参数依旧会像普通函数一样会被返回值所替代，成为 `new C(v1, ..., vm)`。

假设有一个包含方法的类定义：

```scala
class C(x1, ..., xm){ ... def f(y1, ..., yn) = b ... }
```

它拥有类的形参 `x1, ..., xn` 和类实例方法的形参 `y1, ..., yn`，那么当执行 `new C(v1, ..., vm).f(w1, ..., wn)` 时，这整个表达式会被重写为：

```scala
[w1/y1, ..., wn/yn][v1/x1, ..., vm/xm][new C(v1, ..., vm)/this] b
```

这里有三处地方被代换了：

- `w1, ..., wn` 被代换为了方法 `f` 的形参 `y1, ..., yn`
- `v1, ..., vn` 被代换为了类 `C` 的形参 `x1, ..., xm`
- 表达式 `new C(v1, ..., vn)` 被代换为了自引用 `this`

以 Rational 作为一个具体的例子，当调用以下方法时：

```scala
new Rational(1, 2).less(new Rational(2, 3))
```

首先会进行代换：

```scala
[1/x, 2/y] [newRational(2, 3)/that] [new Rational(1, 2)/this]
```

于是该方法的实现：

```scala
this.numer * that.denom < that.numer * this.denom
```

就会被替换为：

```scala
new Rational(1, 2).numer * new Rational(2, 3).denom < new Rational(2, 3).numer * new Rational(1, 2).denom
```

最后可以轻松的计算出结果：

```scala
1 * 3 < 2 * 2
true
```

## 运算符

原则上来说，通过 Rational 定义的分数和整数没有什么区别，但是在使用时却有一些差异。

当我们想要计算两个整数的和时，只需要调用 `x + y`，而当需要计算两个 Rational 的和时，却需要调用 `r.add(s)`。

在 Scala 中，可以通过两步消除这种差异。

### 中缀运算

任何只有一个参数的方法都可以使用中缀运算符（Infix Operator）的方式进行调用：

```scala
r add s  = r.add(s)
r less s = r.less(s)
r max s  = r.max(s)
```

### 标识符

在 Scala 中标识符可以有两种形态：

- 字母数字（Alphanumeric）：以字母为起始字符，字母和数字组成的序列
- 符号（Symbolic）：以一个运算符为起始字符，后面可以跟着其他的运算符
- 下划线（`_`）也算是字母的一种
- 字母数字的标识符可以以下划线结尾，之后跟着一些运算符，例如 `vector_++`

所以 Rational 类中的部分方法可以通过运算符进行替换：

```scala
class Rational(x: Int, y: Int) {
  private def gcd(a: Int, b: Int): Int = if (b == 0) a else gcd(b, a % b)
  private val g = gcd(x, y)
  
  def numer = x / g
  def denom = y / g
  
  def + (r: Rational) =
    new Rational(numer * r.denom + r.numer * denom, denom * r.denom)
  
  def - (r: Rational) = ...
  
  def * (r: Rational) = ...
  ...
}
```

这样使用时就可以像 Int 或是 Double 一样了：

```
val x = new Rational(1, 2)
val y = new Rational(1, 3)

(x * x) + (y * y)
```

运算符的优先级由其第一个字符决定，下图为优先级有低至高的运算符。

![precedence](/uploads/precedence.png)





