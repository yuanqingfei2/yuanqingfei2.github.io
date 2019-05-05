---
title: Hello World
date: "2015-05-01T22:12:03.284Z"
description: "try to explain why new blog"
---
两年前搞了一个， 也就是[为艺术而技术]（http://yuaniqngfei.me）,采用的当时流行的[Jkeyll](https://jekyllrb.com/)，它是基于[Ruby](https://www.ruby-lang.org/en/)语言的。但是它有个很诱人的特点就是可以很方便的**免费**使用[Github Pages](https://pages.github.com/)来做静态网站。步骤如下：

* 选择一个theme，[Jekyllthemes](http://jekyllthemes.org/)里有很多，我采用的是[Minimal Mistakes](https://mademistakes.com/work/minimal-mistakes-jekyll-theme/)。

* 创建一个域名，没啥说的。我使用的GoDaddy，但并不推荐。

* 配置DNS，一个是A，指向github pages的。 一个是CNAME，指向你github的网站。

* 当时的github pages还没有支持SSL，所以中间又转了一层，加了[Cloudflare](https://www.cloudflare.com/)，这里有[教程](https://www.codementor.io/landonpatmore/how-to-setup-a-static-website-using-github-pages-and-cloudflare-with-your-own-domain-name-jb99nbuoe)

简单来说： Github Pages + Cloudflare/Godaddy + Jkeyll
-----------

上面网站我写了一些文章后发现，质量偏低，主要都是生活方面的。 还是想打算找个地方专门写技术文章。所以转到这里[青梅嗅](htts://yuanqingfei.com)

这次采用的平台是[Gatsby]。

简单来说: Netlify + Godaddy + Gatsby

目的就是写 **有趣** 并 **有用** 的文章

```scala
def foldRight[A, B](as: List[A], z: B)(f: (A, B) => B): B = 
  as match {
    case Nil => z
    case Cons(x, xs) => f(x, foldRight(xs, z)(f))
```

![Chinese Salty Egg](./salty_egg.jpg)

