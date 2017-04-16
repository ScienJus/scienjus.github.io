---
title: '[Functional Programming Principles in Scala] 学习笔记（一） 函数调用'
date: 2017-04-04 23:03:21
permalink: profun1-week1
---

# 开始之前

1.  这篇笔记是 Coursera 上的 [Functional Programming Principles in Scala](https://www.coursera.org/learn/progfun1) 课程，主讲者是 Scala 的作者 Martin Odersky，所以还是相当推荐的。
2.  英文能力渣 + 中文表达能力渣，如果错误信息欢迎指正

* * *

# 编程范式

在目前的编程中主要有三种范式：

1.  函数式编程（Functional Programming）
2.  命令式编程（Imperative Programming）
3.  逻辑式编程（Logic Programming）

除此之外，还有一些人认为面向对象编程（Object Oriented Programming）也是一种范式。

## 命令式编程

命令式编程主要由可变变量、赋值语句和程序控制语句（例如 if..then..else 和 loops）组成。最普遍的理解方式是冯诺依曼计算机的计算序列，它们有这样的对应关系：

1.  可变变量 ≈ 内存空间
2.  变量引用 ≈ 加载指令
3.  变量赋值 ≈ 存储指令
4.  控制结构 ≈ 指令跳转

命令式编程的问题是：如何能够避免逐字逐句的进行编程？

所以我们需要更高级别的抽象，例如多项式、集合、字符串、文档等，最好能够开发其相关的法则。该法则包含一个或多个数据类型、如何操作这些数据类型以及描述值和操作之间的关系。

通常情况下，法则并不会改变数据本身。例如以下是一个多项式的法则：

```
(a * x + b) + (c * x + d) = (a + c) * x + (b + d)
```

它并不会通过某个操作符改变系数本身使这个法则成立。

另一个例子是字符串通过 `++` 串联的法则：

```
(a ++ b) ++ c = a ++ (b ++ c)
```

它也不会通过操作符修改字符串中的字符序列而保证法则成立（例如在 Java 中，字符串是不可变的）。

## 函数式编程

1.  狭义而言，函数式编程是没有赋值、可变变量、循环等控制结构的编程手段。
2.  广义而言，函数式编程以函数为重点。函数可以作为一个值被创建、使用和组合。
3.  在函数式编程语言中，这一切都会变得很容易。

## 函数式编程语言

1.  狭义而言，函数式编程语言是没有赋值、可变变量、循环等控制结构的编程语言。
2.  广义而言，函数式编程语言允许构建以函数为重点的程序。
3.  函数在函数式编程语言中是第一等公民（First-Class Citizens），这意味着：
    1.  函数可以定义在任何地方，无论是在其他函数内部或外部
    2.  像其他值一样，函数可以作为另一个函数的参数或是返回值
    3.  像其他值一样，存在一些操作用于组合函数

## 现有的函数式编程语言

狭义而言：

1.  Pure Lisp, XSLT, XPath, XQuery, FP
2.  Haskell（不包含 I/O 单元或 UnsafePerformIO）

广义而言：

1.  Lisp, Scheme, Racket, Clojure
2.  SML, Ocaml, F#
3.  Haskell (整个语言)
4.  Scala
5.  Smalltalk, Ruby

在通常情况下 Smalltalk 和 Ruby 被视为面向对象语言，但是这两种语言都拥有构建语句块（本质上是一等公民函数）的能力。

他们的历史是：

*   1959 Lisp
*   1975-77 ML, FP, Scheme
*   1978 Smalltalk
*   1986 Standard ML
*   1990 Haskell, Erlang
*   1999 XSLT
*   2000 OCaml
*   2003 Scala, XQuery
*   2005 F#
*   2007 Clojure

# 求值策略

## 传值调用（Call by value）

假设有一个函数 `sumOfSquares` 用于求两个数字平方之和，在传值调用时，它的计算步骤为：

```
sumOfSquares(3, 2 + 2)
sumOfSquares(3, 4)
square(3) + square(4) 
3 * 3 + square(4) 
9 + square(4) 
9 + 4 * 4 
9 + 16 
25
```

整个执行流程为从左至右的计算函数的所有的表达式参数，然后将参数替换为计算后的值。

这种表达式求值方案称为替代模型（Substitution Model），该模型的核心思想是将所有的表达式计算为值，只要这些表达式没有副作用（Side Effects）。

## 传名称调用（Call by name)

传名称调用在实际使用参数前并不会计算表达式的值，同样是上面的函数，计算步骤为：

```
sumOfSquares(3, 2 + 2) 
square(3) + square(2 + 2)
3 * 3 + square(2 + 2) 
9 + square(2 + 2)
9 + (2 + 2) * (2 + 2)
9 + 4 * (2 + 2) 
9 + 4 * 4
25
```

相比之下，传值调用的优点是每个参数的值只会计算一次，而传名称调用的优点是参数如果没有在函数体中被用到，则根本不需要去计算。

## 终止条件

一个疑问：所有的表达式都可以在有限的步骤内计算为一个值么？

答案：否，例如 `def loop: Int = loop`。

所以在某些情况下，使用传名称调用函数可以顺利结束并得到一个返回值，使用传值调用则不能，例如：

```
def first(x: Int, y: Int) = x

first(1, loop)
```

Scala 在通常情况下使用传值调用，但是通过 `=>` 定义参数可以改为传名称调用：

```
def constOne(x: Int, y: => Int) = 1
```

# 条件和值定义

## 条件表达式

Scala 拥有条件表达式 if-else，其用起来很像 Java，但是在 Scala 中它是表达式而不是语句，这意味着可以这样使用：

```
def abs(x: Int) = if (x >= 0) x else -x
```

## 值定义

就像函数的参数可以通过传值或是传名称调用一样，值的定义也可以通过 `def` 或是 `val` 进行按名称或是按值的定义。

例如当使用 `def x = loop` 时，这是一个有效的赋值。而当时用 `val y = loop` 时，则会导致一个死循环。

# 嵌套函数

好的函数式编程风格是将一个任务分解为多个小函数。但是其中很多函数我们并不希望用户去直接使用它，基于此可以将这些小函数定义在真正使用的函数内部。

就像是：

```
def sqrt(x: Double) = {
  def sqrtIter(guess: Double, x: Double): Double =
    if (isGoodEnough(guess, x)) guess
    else sqrtIter(improve(guess, x), x)

  def improve(guess: Double, x: Double) =
    (guess + x / guess) / 2

  def isGoodEnough(guess: Double, x: Double) =
    abs(square(guess) - x) < 0.001

  sqrtIter(1.0, x)
}
```

其中 `sqrtIter`、`improve`、`isGoodEnough` 都只是 `sqrt` 中逻辑的一部分，所以对外只需要暴露这一个函数。

# 语句块

在 Scala 中通过 `{ ... }` 定义一个语句块。例如：

```
{
  val x = f(3)
  x * x
}
```

语句块有以下特性：

1.  语句块中可以包含一系列的定义和表达式
2.  语句块中最后的表达式代表这个块的值
3.  或者也可以在这之前用 return 表达式返回值
4.  语句块可以出现在任何一般表达式可以出现的地方

语句块中的变量有以下特性：

1.  语句块中定义的变量只有在语句块中才可以访问
2.  语句块中可以定义和语句块外同名的变量
3.  语句块内可以访问语句块外的变量

例如：

```
val x = 0 
def f(y: Int) = y + 1 
val result = { 
  val x = f(3) 
  x * x 
} + x
```

`result` 的值为 `16`，因为语句块内的 `x` 为 4，而语句块外的 `x` 始终为 `0`。

# 尾递归

有两个通过递归进行计算的函数，一个是 gcd：

```
def gcd(a: Int, b: Int): Int =
    if (b == 0) a else gcd(b, a % b)
```

当调用时它的执行步骤为：

```
gcd(14, 21)
if (21 == 0) 14 else gcd(21, 14 % 21)
if (false) 14 else gcd(21, 14 % 21)
gcd(21, 14 % 21)
gcd(21, 14)
if (14 == 0) 21 else gcd(14, 21 % 14)
gcd(14, 7)
gcd(7, 0)
if (0 == 0) 7 else gcd(0, 7 % 0)
7
```

另一个函数是 factorial：

```
def factorial(n: Int): Int =
    if (n == 0) 1 else n * factorial(n - 1)
```

它的执行步骤为：

```
factorial(4)
if (4 == 0) 1 else 4 * factorial(4 - 1)
4 * factorial(3)
4 * (3 * factorial(2))
4 * (3 * (2 * factorial(1)))
4 * (3 * (2 * (1 * factorial(0)))
4 * (3 * (2 * (1 * 1)))
120
```

它们的区别是：如果一个函数最后的动作只是调用它自己，那么这个函数的栈帧就可以被复用，这被称为尾递归（Tail Recursion）。

上面两个函数中，gcd 就是一个尾递归函数，而 factorial 并不是，因为它需要递归函数的返回值额外进行计算。

另外，如果一个函数最后的动作只是调用另一个函数（可能为它自己），那么一个栈帧就足够被这两个函数使用，这被称为尾调用（Tail Calls）。

factorial 也可以被改为使用尾递归的形式，只需要将函数中暂存的乘积作为参数传到下一次调用即可：

```
def factorial(n : Int): Int = {
  def loop(acc: Int, n: Int): Int =
    if (n == 0) acc
    else loop(acc * n, n - 1)
  loop(1, n)
}
```

在 Scala 中，只有在当前函数中直接进行递归调用的函数才会做尾递归优化。

也可以通过 `@tailrec` 指定该函数为尾递归函数，但如果函数本身就不是尾递归函数，则会抛出一个编译错误。

