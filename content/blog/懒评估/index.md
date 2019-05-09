---
title: 懒评估
date: "2019-05-09T13:45:00.000Z"
description: "是什么，为什么以及在Java、Scala中的应用"
---

懒评估(Lazy Evaluation，下面简称LE)顾名思义就是和紧评估(Strict Evaluation, SE)对应，是指延迟计算直到需要的时候，如果只是这样，我们称它为“按名”(by name)，但是还可以更深一层，如果我们能够缓存第一次评估的结果，这样就可以节约大量的计算。到了这一层，我们就叫“按需”(by need)。

LE在Functional Programming中很有用，因为它很理想的实现了递归模型。比如
```scala
def addOne(n) = [n] + addOne(n + 1)
// [1, 2, 3, 4, 5, 6, ...]
list = addOne(1)

oneToThree = list.takeFirst(3)
print(oneToThree) // [1, 2, 3]
```

## Java
在Java 8之前，只支持很有限的一个LE实现，就是[短路评估(Short-circuit_evaluation)](https://en.wikipedia.org/wiki/Short-circuit_evaluation)
```java
public boolean isTrue() {
    return isBTrue() || isATrue();
}
```
如果isBTrue()为true，那么isATrue()就不会被执行。

在Java 8之后，由于引入了[lambda表达式](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)，也可以实现“按需”LE，[这里](https://dzone.com/articles/leveraging-lambda-expressions-for-lazy-evaluation)提供了一个很好的例子。

## Scala

Scala默认和Java一样都是SE的，但是可以显示声明LE。

* 变量
Scala可以使用关键词lazy来实现“按需”LE。
```scala
val lazy v = 12
```

* 参数
```scala
def foo(x: Int): Int={}  //SE 
def foo(x: => Int): Int={}   //LE

// The arrow `=>` in front of the argument type `B` means that the function `f` takes its second argument by name and may choose not to evaluate it.
def foldRight[B](z: => B)(f: (A, => B) => B): B = 
this match {
    // If `f` doesn't evaluate its second argument, the recursion never occurs.
    case Cons(h,t) => f(h(), t().foldRight(z)(f)) 
    case _ => z
}
```

* 例子

Stream



##　致谢
http://matt.might.net/articles/implementing-laziness/

https://dzone.com/articles/leveraging-lambda-expressions-for-lazy-evaluation

https://medium.com/background-thread/what-is-lazy-evaluation-programming-word-of-the-day-8a6f4410053f
