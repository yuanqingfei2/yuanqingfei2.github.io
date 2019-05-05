---
title: Hello World
date: "2015-05-01T22:12:03.284Z"
description: "try to explain why new blog"
---

为什么又要搞一个blog呢，主要是想把一些经验分享出来。遵循以下几个原则：

* 有趣
* 有用

```scala
def foldRight[A, B](as: List[A], z: B)(f: (A, B) => B): B = 
  as match {
    case Nil => z
    case Cons(x, xs) => f(x, foldRight(xs, z)(f))
```

![Chinese Salty Egg](./salty_egg.jpg)

