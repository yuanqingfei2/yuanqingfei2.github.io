---
title: 懒评估
date: "2019-05-09T13:45:00.000Z"
description: "是什么，为什么以及在Java、Scala中的应用"
---

懒评估(Lazy Evaluation，下面简称LE)顾名思义就是和紧评估(Strict Evaluation, SE)对应，是指延迟计算直到需要的时候，如果只是这样，我们称它为“按名”(by name)，但是还可以更深一层，如果我们能够缓存第一次评估的结果，这样就可以节约大量的计算。到了这一层，我们就叫“按需”(by need)。

LE在Functional Programming中很有用，因为它很理想地实现了递归模型而不会栈溢出(Stack Overflow)。比如
```scala
def addOne(n) = [n] + addOne(n + 1)
// [1, 2, 3, 4, 5, 6, ...]
list = addOne(1)

oneToThree = list.takeFirst(3)
print(oneToThree) // [1, 2, 3]
```

关于节约计算这块，在pipeline计算的时候会非常明显。

## Java
* Java 8之前
只支持很有限的一个LE实现，就是[短路评估(Short-circuit_evaluation)](https://en.wikipedia.org/wiki/Short-circuit_evaluation)
```java
public boolean isTrue() {
    return isBTrue() || isATrue();
}
```
如果isBTrue()为true，那么isATrue()就不会被执行。

* Java 8之后
由于引入了[lambda表达式](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)，也可以实现“按需”LE，[这里](https://dzone.com/articles/leveraging-lambda-expressions-for-lazy-evaluation)提供了一个很好的例子。

## Scala

Scala默认和Java一样都是SE的，但是可以显示声明LE。

* 变量
Scala可以使用关键词lazy来实现“按需”LE。
```bash
scala> val x = 15
x: Int = 15
scala> lazy val y = 13
y: Int = <lazy>
```

* 参数
```scala
def foo(x: Int): Int={}      //SE 
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

Stream : Lazy版的List。Java版的[Stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html#package.description); Scala版的[Stream](https://www.scala-lang.org/api/2.12.8/scala/collection/immutable/Stream.html)

以Scala版为例：

```bash
scala> val stream = (1 to 10000).toStream
stream: scala.collection.immutable.Stream[Int] = Stream(1, ?)

scala> stream.head
res13: Int = 1

scala> stream.tail
res14: scala.collection.immutable.Stream[Int] = Stream(2, ?)

scala> stream
res15: scala.collection.immutable.Stream[Int] = Stream(1, 2, ?)

scala> stream(9)
res16: Int = 10

scala> stream
res17: scala.collection.immutable.Stream[Int] = Stream(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, ?)

scala> List(1,2,3).toStream.take(3).foreach(x => print(x + " "))
1 2 3 
```

## Stream的Scala简化实现

```scala
sealed trait Stream[+A]
case object Empty extends Stream[Nothing]
case class Cons[+A](h: () => A, t: () => Stream[A]) extends Stream[A]

object Stream{
  def cons[A](hd: => A, tl: => Stream[A]): Stream[A] = {
    lazy val head = hd
    lazy val tail = tl
    Cons(() => head, () => tail)
  }
  def empty[A]: Stream[A] = Empty

  def apply[A](as: A*): Stream[A] = 
    if(as.isEmpty) empty else cons(as.head, apply(as.tail: _*))
}
```

## lazy的实现

```scala
class LazyTest {
  lazy val msg = "Lazy"
}
```

* Scala 2.7
```java
class LazyTest {
  public int bitmap$0;
  private String msg;

  public String msg() {
    if ((bitmap$0 & 1) == 0) {
        synchronized (this) {
            if ((bitmap$0 & 1) == 0) {
                synchronized (this) {
                    msg = "Lazy";
                }
            }
            bitmap$0 = bitmap$0 | 1;
        }
    }
    return msg;
  }
}
```

* Scala 2.10, 2.12
```java
public class LazyTest
{
  private String msg;
  private volatile boolean bitmap$0;
  
  public String msg()
  {
    return !this.bitmap$0 ? msg$lzycompute() : this.msg;
  }
  
  private String msg$lzycompute()
  {
    synchronized (this)
    {
      if (!this.bitmap$0)
      {
        this.msg = "Lazy";this.bitmap$0 = true;
      }
    }
    return this.msg;
  }
}
```

至于为什么抛弃了Double Checking而转用votatile，这就是另外一个[故事](https://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java)了。简短的说，就是如下的解释了。

> The real problem is that Thread A may assign a memory space for instance before it is finished constructing instance. Thread B will see that assignment and try to use it. This results in Thread B failing because it is using a partially constructed version of instance.


## 致谢

http://matt.might.net/articles/implementing-laziness/

https://dzone.com/articles/leveraging-lambda-expressions-for-lazy-evaluation

https://medium.com/background-thread/what-is-lazy-evaluation-programming-word-of-the-day-8a6f4410053f
