---
title: 追逐圣杯
date: "2019-09-15T17:06:00.000Z"
description: "记录下软件跨平台之路"
---

在软件行业，有句名言：如果有什么复杂性不能解决的话，就加一层，如果还不能解决的话，就再加一层。没错，分层这个万金油在我们这个靠抽象架构吃饭的行当里无往不胜。

某年某月某日，Java诞生了，戴着光环来的，因为它自诞生起20多年里就一直雄霸编程语言排行榜[第一名直到现在](https://www.tiobe.com/tiobe-index/)，不能不说它有些神奇，其中最大神奇之处可能就是它带来了JVM。有了它，也就是在机器语言和普通的高级编程语言中间加了一层所谓bytecode层，不要小看这一层，有了这一层，Java语言就不仅仅是Java语言了，它还可以被成为JVM平台语言**之一**，高大上吧。当年只有Java语言的年代，感觉就是一句废话，谁能想到，历史的车轮滚滚向前，Java背后的Sun公司倒了？没有关系，新老大Oracle貌似更重视律师团队而非Java技术团队？没有关系。只要JVM在，所有的历史库和生态圈就都可以为我所用，社区纷纷努力，诞生了一大批JVM语言来与Java争高下，Groovy, Scala, Clojure, Kotlin... 这种情况下生意只能越做越大嘛。这就是所谓的三流的公司做产品，二流的公司做平台，一流的公司做标准。JVM标准一出，谁与争锋?!

某年某月某日，浏览器诞生了，也是戴着光环来了，它成了互联网时代的操作系统，事实上，google就推出过基于[Chromium](https://www.chromium.org/)项目的笔记本。插一句,江河轮流转阿，想当年大型机小型机时代，标准的瘦客户端，标准的分时操作系统。随着Gates的一句“我有一个梦想，梦想世界上每个家庭都有一台Windows的电脑”，他成了世界首富，客户端也是越来越胖了，电脑城里一个个超频装机，不亦乐乎。随着互联网的兴起以及网络宽带的极大提升，以及对未来5G，6G 以及NG的期待，我们可以想象Sun公司的那句“网络就是计算机”的口号是多么的有先见之明阿，可惜它死的太早。总之，瘦客户端又再次回归，当然这次根据历史螺旋式上升的万能定律，这次必定是重装上线。这不，重中之重的浏览器开始革命了，IE，Chorme, Firefox, Edge,一波推一波，性能不断提升，而浏览器唯一支持的语言就是Javascript可跟着开始[飞升](https://en.wikipedia.org/wiki/ECMAScript)，1999年就出版本3了，知道10年之后才出5（中间的4直接没出来），又过了6年也就是2015才出了6，然后就一年一个标准了。里面的Javascript引擎和语言标准一样也飞速升级,从大名鼎鼎的V8,到SpiderMonkey,到Chakra。各大厂商极尽各种优化之能事。但是.........这还不够，因为某人说了他有另外一个梦想.....

我找不到第一个这么说的人是谁，反正我几年前在第一次看到大神[Lihaoyi](http://www.lihaoyi.com/)在[文章](http://www.lihaoyi.com/hands-on-scala-js/#SharingCode)里这么写到:

>**Shared code is one of the holy-grails of web development**. Traditionally the client-side code and server-side code has been written in separate languages: PHP or Perl or Python or Ruby or Java on the server, with only Javascript on the client. **This means that algorithms were often implemented twice, constants copied-&-pasted, or awkward Ajax calls are made in an attempt to centralize the logic in one place (the server)**. With the advent of Node.js in the last few years, you can finally re-use the same code on the server as you can on the client, but with the cost of having all the previously client-only problems with Javascript now inflicted upon your server code base. Node.js expanded your range-of-options for writing shared client/server logic from "Write everything twice" to "Write everything twice, or write everything in Javascript". More options is always good, but it's not clear which of the two choices is more painful!

>Scala.js provides an alternative to this dilemma. **With Scala.js, you can utilize the same libraries you use writing your Scala servers when writing your Scala web clients**! 

我不知道微软是不是也看到类似的话而去发明了Typescript从而使得前后端都可以使用同样的语言，反正大量的语言开始都抽风似玩命编译成javascript，[这里](https://github.com/jashkenas/coffeescript/wiki/List-of-languages-that-compile-to-JS)有个不完整长的吓人的列表。话说回来，这真的好吗？ 我是说把一个原本只是写小程序的浏览器小语言事实上变成了互联网操作系统的汇编语言了！不，果然，又有一群好事者（我最喜欢这样的好事者了），他们打出了“王侯将相，宁有种乎”的旗帜，搞出了[WebAssembly](https://webassembly.org/)，当然我夸张了点，人家还是很peace的，人家网站上说只是想和JS共存，其实司马昭之心，谁不知到呢。你看人家名字起的，就叫网络汇编语言，大气，敞亮，就喜欢这样的。一句话说就是，有了我在浏览器里呢，就相当与在浏览器里有了个执行bytecode的虚拟机，各种语言只要编译成bytecode就行了（是不是有种似曾相似的感觉，没错，就是和JVM一样的加一层的套路!）。当然现在这个伟大的宏图还只是处于早期阶段，我们先静观其变就好，这里有篇不错的[介绍](http://kripken.github.io/mloc_emscripten_talk/#/)。我有空开始搞搞Kotlin的WebAssembly，也算是挖个坑。顺便说下，今天这个文也算填了[这篇文](https://www.yuanqingfei.com/Java%20Web%20Framework%E7%9A%84%E5%86%8D%E6%AC%A1%E5%85%B4%E7%9B%9B/)的坑。打完收工。

