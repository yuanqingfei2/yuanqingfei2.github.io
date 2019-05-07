---
title: 协变与逆协变
date: "2019-05-06T13:22:00.000Z"
description: "范型中常见的两个概念解析以及在Java和Scala中的应用，最后介绍了PECS法则"
---

假定A <: B (A是B的子类)，如果T[A] <: T[B]，那么我们就说T是协变的(**covariant**)。如果T[B] <: T[A]，那么我们说T是逆协变的(**contravariant**)。如果T[B]和T[A]没有从属关系，我们就说T是非协变的(**invariant**)。

## Java

Java中数组是协变的，意味着下面的代码没有问题，但是你需要自己清楚实际数组放的是什么类型。否则运行时可能要出问题。
```java
Object testObj = null;
String[] arrayB = { "a", "b", "c" };
Object[] arrayA = arrayB;
testObj = arrayA[0];
```

Java中的Generic是Java5引入的，默认是invariant的。 `MyClass<String>`不是`MyClass<Object>`的子类或者父类。同样原因，下面的代码会有编译错误。 
```java
List<Car> cars = new ArrayList<>();
List<Vehicle> garage = cars;        // compilation error
```

Java使用use-site方式来实现协变或者逆协变。也就是在使用的时候才知道。这依赖于Java5中引入的[wildcard](https://docs.oracle.com/javase/tutorial/extra/generics/wildcards.html)。
```java
public void process(List<? extends Car> list) { ... }
```
也就是说这个List参数变成协变的了，因为它接受所有Car以及Car子类的List。 

与之相对，
```java
public void process(List<? super Car> list) { ... }
```
这个参数就是逆协变了。

## Scala

Scala可以使用和Java同样的use-site方式来实现协变或逆协变。
```scala
A :< B
L[A] :< L[_ <: B]   // covariant
L[A] :> L[_ >: B]   // contravariant
```

Scala还可以使用delcare-site方式来实现。
```scala
A :< B
L[A] :< L[+B]      // covariant
L[A] >: L[-B]      // contravariant
```

## Producer Extends, Consumer Super （PECS）

先说结论:  
producer === extends === read only === covariant  
consumer ==== super ==== write only === contravariant

从Java集合的角度, 从集合中取数据就是producer，你就要用extends；向里面存就是consumer，需要使用super。
```java
List<? extends Vehicle> garage = new ArrayList<>();
garage.add(new Vehicle());  // compilation error
garage.add(new Car());      // compilation error
garage.add(new Bus());      // compilation error
// reading behavior
Vehicle vehicle = garageB.get(1);
```
上面这个例子用的是extends，所以只读，不可写。为什么呢，很简单，因为编译器不知道你到底是哪个子类型，所以不让写。但是你读出来的肯定是Vehicle的子类，所以完全没问题。

```java
List<? super Car> garage = new ArrayList<>();
garage.add(new BMW());
garage.add(new Alto());
garage.add(new Vehicle());    // compilation error
// reading behavior
Object object = garage.get(0);    // I don't get a Car, why?
```
同样，上面的例子可以写入，但是由于不知道具体类型，所以只能给出最高级类型也就是Object。

从Scala的角度，参数就是Consumer，所以应该是逆协变，而返回值就是Producer，所以就是协变。这样理解下面的[Signature](https://www.scala-lang.org/api/2.9.2/scala/Function1.html)就容易了。
```scala
trait Function1[-T, +R] extends AnyRef
trait Function2[-T1, -T2, +R] extends AnyRef
```

Scala支持高阶函数，因此函数本身也就有了范型。
```scala
abstract class Animal {
  def name: String
}
case class Cat(name: String) extends Animal
case class Dog(name: String) extends Animal
abstract class SmallAnimal extends Animal
case class Mouse(name: String) extends SmallAnimal
```
基于上面的解释，Animal是Cat的父类，而Mouse是SmallAnimal的子类，因此我们可以说 **Animal => Mouse** 是 **Cat => SmallAnimal** 的子类型。

## 致谢

https://medium.com/@sinisalouc/variance-in-java-and-scala-63af925d21dc 

https://docs.scala-lang.org/tour/variances.html 

https://medium.com/@isuru89/java-producer-extends-consumer-super-9fbb0e7dd268


