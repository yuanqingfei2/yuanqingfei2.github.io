---
title: Kotlin Vertx React 1
date: "2019-09-22T21:17:00.000Z"
description: "探索一"
---

如同[上文](https://www.yuanqingfei.com/%E8%BF%BD%E9%80%90%E5%9C%A3%E6%9D%AF/)所说，我也一直想尝试基于共享代码的web编程，之前尝试过[Scala平台的](https://github.com/yuanqingfei/gdbscan-akka-d3js)。刚刚发现Kotlin自从1.2开始也[支持了](https://kotlinlang.org/docs/reference/multiplatform.html)，于是也想尝试以下。

另外如同[上文](https://www.yuanqingfei.com/Java%20Web%20Framework%E7%9A%84%E5%86%8D%E6%AC%A1%E5%85%B4%E7%9B%9B/)所述，Vertx发展不错，基于尝鲜的念头，想把Vertx引入到Kotlin的MPP中，不过到现在还没有搞成功，但是记录以下自己踩过的坑如下：

## JVM Target的更改

默认情况虾，Kotlin的JVM plugin是编译到Java 1.6的，但是很多情况下，当我们用到Kotlin自己的一些扩展时，需要编译到1.8才可以。 基于目前的搜索，IDE（Idea） 和 命令行(grdle.kts)的情况分述如下：

### IDE

* Settings -> Build -> Compiler -> Kotlin 点选 1.8

* Project Struure -> Facets 都使用project configuration

来源： [stackoverflow](https://stackoverflow.com/questions/48601549/why-kotlin-gradle-plugin-cannot-build-with-1-8-target)

### Console

```kotlin
tasks.withType<KotlinCompile> {
    kotlinOptions {
        jvmTarget = "1.8"
    }
}
```

Or

```kotlin
tasks.withType(KotlinCompile::class).all {
    kotlinOptions {
        jvmTarget = "1.8"
    }
}
```

来源：[github](https://github.com/gradle/kotlin-dsl-samples/issues/1368) 

## 多平台支持

In Intellij in Settings -> Build, Execution, Deployment-> Compiler -> Kotlin Compiler append to Additional command line parameters: field command `-Xmulti-platform`.

来源：[stackoverflow](https://stackoverflow.com/questions/48852066/kotlin-multi-platform-feature )

## Vertx Gradle Plugin

本来打算使用 Vertx 的[plugin](https://github.com/jponge/vertx-gradle-plugin)来启动后台的，结果发现即使配置如下没问题的情况下，执行 `gradle build vertxRun`还是说找不到类，后来在[这里](https://github.com/jponge/vertx-gradle-plugin/issues/30)才发现目前这个plugin根本就还不支持变化的文件结构。 

`gist:yuanqingfei/76afa0c6b179b32565b435a875548b8e`

## Run 的方式来启动后端

费了半天劲学习Gradle DSL，掌握一点皮毛。最麻烦转的就是下面这点

```kotlin
jvmJar {
    dependsOn(jsBrowserWebpack)
    from(new File(jsBrowserWebpack.entry.name, jsBrowserWebpack.outputPath))
}

task run(type: JavaExec, dependsOn: [jvmJar]) {
    group = "application"
    main = "sample.MainKt"
    classpath(configurations.jvmRuntimeClasspath, jvmJar)
    args = []
}
```

转好之后

`gist:yuanqingfei/e89a90ee163b7d2468cd04eec90b44fb`

转的过程关键是你要给出类型，而类型的获取可以使用如下命令(以jsBrowserWebpack为例))：

```bash
aaron@aaron-w530:/var/docker/project/kotlin-vertx-react$ gradle help --task jsBrowserWebpack

Path
     :jsBrowserWebpack

Type
     KotlinWebpack (org.jetbrains.kotlin.gradle.targets.js.webpack.KotlinWebpack)

```

来源1： [Kotlin Official](https://guides.gradle.org/migrating-build-logic-from-groovy-to-kotlin/)

来源2： [jnizet](https://github.com/jnizet/gradle-kotlin-dsl-migration-guide)

## 后端的Netty目前版本需要JAVA 8, JAVA 9不行

不行不是说啥都不行，而是你把log的DEBUG模式打开后，会有错误信息出现

## 使用H2内存数据库需要配置

一开始就用默认`jdbc:h2:mem:regular` 死活不行，需要更改为 `jdbc:h2:mem:regular;DB_CLOSE_DELAY=-1;`

## Kotlin data class在json互相转的时候需要特殊依赖

正常情况下会报错误说是没有默认构造体，需要增加依赖`implementation("com.fasterxml.jackson.module:jackson-module-kotlin:2.9.7")`， 然后代码里添加`Json.mapper.registerModule(KotlinModule())` 即可

## Exposed 库的一个问题

当数据库是H2时（其他数据库不知道），创建自增主键表时，只能`val id = integer("id").primaryKey().autoIncrement()` , 顺序颠倒一下 `val id = integer("id").autoIncrement().primaryKey()`，就会报如下错误

```
Caused by: org.h2.jdbc.JdbcSQLException: Syntax error in SQL statement "ALTER TABLE USERS MODIFY COLUMN ID INT AUTO_INCREMENT PRIMARY[*] KEY"; SQL statement:
ALTER TABLE USERS MODIFY COLUMN ID INT AUTO_INCREMENT PRIMARY KEY [42000-197]
```

如果使用IntIdTable也是如此。