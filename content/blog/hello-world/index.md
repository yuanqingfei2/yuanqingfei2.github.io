---
title: Hello World
date: "2019-05-04T22:12:03.284Z"
description: "解释了下之前blog的构建以及现在的构建。说明为什么又另起炉灶。"
---
## 前情提要

两年前(2017)搞了一个， 也就是[为艺术而技术](http://yuanqingfei.me),采用的当时流行的[Jekyll](https://jekyllrb.com/)，它是基于[Ruby](https://www.ruby-lang.org/en/)语言的。但是它有个很诱人的特点就是可以很方便的**免费**使用[Github Pages](https://pages.github.com/)来做静态网站。步骤如下：

* 选择一个theme，[Jekyllthemes](http://jekyllthemes.org/)里有很多，我采用的是[Minimal Mistakes](https://mademistakes.com/work/minimal-mistakes-jekyll-theme/)。

* 创建一个域名，没啥说的。我使用的GoDaddy，但并不推荐。

* 配置DNS，一个是A，指向github pages的。 一个是CNAME，指向你github的网站。

*  ~~当时的github pages还没有支持SSL，所以中间又转了一层，加了[Cloudflare](https://www.cloudflare.com/)，这里有[教程](https://www.codementor.io/landonpatmore/how-to-setup-a-static-website-using-github-pages-and-cloudflare-with-your-own-domain-name-jb99nbuoe)~~

简单来说： Github Pages + ~~Cloudflare/~~Godaddy + Jkeyll

## 目前动机

上面网站我写了一些文章后发现，主要都是生活方面的。 还是想打算找个地方专门写技术文章。所以转到这里[青梅嗅](htts://yuanqingfei.com)

这次采用的平台是[Gatsby](https://www.gatsbyjs.org)。本来还是享用Github Pages来做，结果不支持，也就是说你必须每次手动把生成的东西放上去。还好[Netlify](https://www.netlify.com/)支持，也就是自动发现Github Repo的更新并发布。另外还有个bonus：SSL，不用再麻烦Cloudflare，其实现在的Github Pages也已经支持SSL了，只是Cloudflare还可以看访客数量信息，所以之前的网站就不变了。

简单来说: Netlify + Godaddy + Gatsby

目的就是写 **有趣** 并 **有用** 的文章

## 语法高亮显示

```scala
def foldRight[A, B](as: List[A], z: B)(f: (A, B) => B): B = 
  as match {
    case Nil => z
    case Cons(x, xs) => f(x, foldRight(xs, z)(f))
```

```javascript
var s = "JavaScript syntax highlighting";
alert(s);
```

```python
s = "Python syntax highlighting"
print s
```

```java
class Test{
  public static void main(String args[]){
    System.out.println("Java syntax highlighting");
  }
}
```

默认[starter blog](https://github.com/gatsbyjs/gatsby-starter-blog) 不支持上面的语言高亮显示，根据这篇[文章](https://reactgo.com/gatsbyblog/syntaxhighlighting/)需要在 gatsby-browser.js 里面加上下面这行：

```javascript
require("prismjs/themes/prism-solarizedlight.css")
```

## 加上google Analysitics

使用[gatsby-plugin-google-analytics](https://www.gatsbyjs.org/packages/gatsby-plugin-google-analytics/)。

在gatsby-config.js 加上下面：

```javascript
{
      resolve: `gatsby-plugin-google-analytics`,
      options: {
        trackingId: "UA-XXXXXXXXX-X",
      },
    },
```

## 加上Disqus评论

* 在[Disqus](https://disqus.com/)注册而获得一个shortname

* 增加包
```bash
yarn add disqus-react
```
或者
```bash
npm install disqus-react --save
```

* 修改src/templates/blog-posts.js
```javascript
import { DiscussionEmbed } from "disqus-react";
```
在render()方法里添加
```javascript
    const disqusShortname = "yourdisqusshortname";
    const disqusConfig = {
      identifier: post.id,
      title: post.frontmatter.title,
    };
```
在UI部分添加
```html
    <DiscussionEmbed shortname={disqusShortname} config={disqusConfig} />
```

## 增加adsense

采用[gatsby-plugin-google-adsense](https://www.npmjs.com/package/gatsby-plugin-google-adsense)

```javascript
// In your gatsby-config.js file
plugins: [
    {
      resolve: `gatsby-plugin-google-adsense`,
      options: {
        publisherId: `ca-pub-xxxxxxxxxx`
      },
    },
]
```

## 致谢

https://daveceddia.com/start-blog-gatsby-netlify/

https://daveceddia.com/how-to-blog/

https://mk.gg/add-disqus-comments-to-gatsby-blog/
