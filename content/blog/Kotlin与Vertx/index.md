---
title: Kotlin与Vertx
date: "2019-09-13T17:39:00.000Z"
description: "记录下一点学习感想"
---

做一个东西久了，就很容易形成思维定势，好在我所处的行业是所有行业中最不推崇墨守成规的。在Java Web开发行业中，Servlet可谓源源流长，系出名门，最新的版本已经到了[4.0版本了](https://en.wikipedia.org/wiki/Java_servlet)。一个web容器如果说自己不支持Servlet都不好意思出门。可是谁能想到，当初Servlet也是从CGI手里夺得江山的，一旦封王，横行至今。天下大道，后浪推前浪，自从一帮不安份的人把JS的性能通过[JIT](https://hacks.mozilla.org/2017/02/a-crash-course-in-just-in-time-jit-compilers/)提高了10倍之后，就有大神把JS直接推到了服务端，也就是大名鼎鼎的NodeJS，生生从Servlet的手中夺得一片天下。还好JVM这边也不是吃素的，赶紧的学习借鉴，你不是有Functional Programming吗，我也搞出来了Scala，Kotlin，你不是Reactive Programming吗，嘿嘿，我也跟着[Reactive Streams](http://www.reactive-streams.org/)有[RxJava](https://github.com/ReactiveX/RxJava)，就连JVM业界翘楚Spring也都赶紧表示自己也没有落下，推出了基于[Reactor](https://github.com/reactor/reactor)的[webflux](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux)，一时间大家都取长补短，一派欣欣向荣之状。可天下总有那种好事之徒，动不动就掀桌子重干，业内叫“重新发明轮子”。这不，有人一不做二不休，干脆从头到位来个React，你不是要观察者模式吗？干脆，咱重启炉灶，直接搞了一个JVM版本（也有其他语言版本）的NodeJS，就这份豪气和干劲，佩服佩服。这就是今天这位大神[Vert.x](https://vertx.io/)，从1.0干到3.0，业务越做越大，直接形成了一个巨大的生态圈(GraphQL, Web Client, Web API/Swagger, Data Access...)，不客气的说，有点想当年Spring出来的气势，事实上，已经有人做了[对比](http://www.tellmehow.co/comparison-between-spring-boot-and-vertx/)，Vert.x优势不小阿。特别是性能方面，[这里](https://dreamix.eu/blog/java/reactive-java-vert-x-vs-spring-framework-5)和[这里](https://vironit.com/comparison-of-web-frameworks-spring-boot-vs-vert-x/)都说Vertx的性能要好不少。究其根源，可能和不用那么多的Annotion以及Reflection有关。

稍微说下Reactive Programming这个东西，本来只不过是观察者模式的一个实现，为何能越做越大呢，说到底，还是思维方式的转变，工厂重要，还是产品重要？ 当然还是产品重要，所以最终是工厂围着产品转，在IT业内这叫以数据为导向，一切都是要为数据服务，而数据流最讨厌的就是死板。想想以下城市的自来水管道和多米诺骨牌，本质上都是构建反应模式，然后数据驱动。而在今天这个Functional Programming加持下，这种模式如鱼得水，当然这里必须要表扬以下首开风气之先的NodeJS.

废话不多说，今天直接用Vert.x搞了个简配版的CRUD练练手，感受一下。

## Gradle 配置

关键是如下几个：

```groovy
plugins {
  id 'org.jetbrains.kotlin.jvm' version '1.3.20'
  id 'application'
  id 'com.github.johnrengelman.shadow' version '5.0.0'
}

dependencies {
  implementation "io.vertx:vertx-web:$vertxVersion"
  implementation "io.vertx:vertx-lang-kotlin:$vertxVersion"

  testImplementation "io.vertx:vertx-junit5:$vertxVersion"
  testRuntimeOnly "org.junit.jupiter:junit-jupiter-engine:$junitJupiterEngineVersion"
  testImplementation "org.junit.jupiter:junit-jupiter-api:$junitJupiterEngineVersion"
}

shadowJar {
  classifier = 'fat'
  manifest {
    attributes 'Main-Verticle': mainVerticleName
  }
  mergeServiceFiles {
    include 'META-INF/services/io.vertx.core.spi.VerticleFactory'
  }
}

run {
  args = ['run', mainVerticleName, "--redeploy=$watchForChange", "--launcher-class=$mainClassName", "--on-redeploy=$doOnChange"]
}
```

可以看出，里面根本就没有Jetty容器的事，后台看了下加载包，才知道，默认用了[Netty](https://netty.io/)。也不错，好马，新鞍。

## Http Rest

```kotlin
    var products = mutableMapOf<String, Any?>()
    fun addProduct(product: JsonObject) {
        products[product.getString("id")] = product
    }
    fun setUpInitialData() {
        addProduct(json {
            obj(
                "id" to "prod3568",
                "name" to "Egg Whisk",
                "price" to 3.99,
                "weight" to 150
            )
        })
    }
    fun handleGetProduct(routingContext: RoutingContext) {
        var productID = routingContext.request().getParam("productID")
        var response = routingContext.response()
        if (productID == null) {
            sendError(400, response)
        } else {
            var product = products[productID]
            if (product == null) {
                sendError(404, response)
            } else {
                response.putHeader("content-type", "application/json").end(product.toString())
            }
        }
    }
    fun sendError(statusCode: Int, response: HttpServerResponse) {
        response.setStatusCode(statusCode).end()
    }
    fun handleListProducts(routingContext: RoutingContext) {
        var arr = json {
            array()
        }
        for ((_, v) in products) {
            arr.add(v)
        }

        routingContext.response().putHeader("content-type", "application/json").end(arr.toString())
    }
    fun handleAddProduct(routingContext: RoutingContext) {
        var productID = routingContext.request().getParam("productID")
        var response = routingContext.response()
        if (productID == null) {
            sendError(400, response)
        } else {
            var product = routingContext.getBodyAsJson()
            if (product == null) {
                sendError(400, response)
            } else {
                products[productID] = product
                response.end()
            }
        }
    }
    override fun start() {

        setUpInitialData()

        var router = Router.router(vertx)

        router.route().handler(BodyHandler.create())
        router.get("/products/:productID").handler({ handleGetProduct(it) })
        router.put("/products/:productID").handler({ handleAddProduct(it) })
        router.get("/products").handler({ handleListProducts(it) })

        vertx.createHttpServer().requestHandler(router).listen(8080)
    }
```

看看代码就知道，相当简单粗暴，直接就是router.get().handler(...)就是这样，CRUD直接搞定，你要怎样action，自己随便定义就是。虽然现在代码还有点多，估计随着[vert.x-web](https://github.com/vert-x3/vertx-web)的升级，还可以表现更好。