---
title: Intelij Idea Proxy 问题
date: "2019-12-20T09:41:00.000Z"
description: "记录一下自己如何解决这个问题"
---

由于是在公司，无论是Gradle还是Maven之前使用时都需要先设好Proxy，一直没有问题，最近发现突然不行了。

## 症状

首先是build.gradle.kts，如果使用最新的语法

```kotlin
plugins {
    kotlin("jvm") version "1.3.50"
}
```

或者

```kotlin
plugins {
    id("org.jetbrains.kotlin.jvm") version "1.3.50"
}
```

无论是设不设Proxy，都只给你下面这样模糊的出错提示：

```
* Where:
Build file 'C:\Users\qy24155\Documents\workspace\adr2\build.gradle.kts' line: 14

* What went wrong:
Plugin [id: 'org.jetbrains.kotlin.jvm', version: '1.3.50'] was not found in any of the following sources:

- Gradle Core Plugins (plugin is not in 'org.gradle' namespace)
- Plugin Repositories (could not resolve plugin artifact 'org.jetbrains.kotlin.jvm:org.jetbrains.kotlin.jvm.gradle.plugin:1.3.50')
  Searched in the following repositories:
    Gradle Central Plugin Repository
```

这个时候，切换回老的语法格式反而能得到更多有效出错提示：

```kotlin
buildscript {
    repositories {
        maven {
            url = uri("https://plugins.gradle.org/m2/")
        }
    }
    dependencies {
        classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:1.3.50")
    }
}

apply(plugin = "org.jetbrains.kotlin.jvm")
```

### 无Proxy

也就是 ~/.gradle.properties里面为空，出错提示为

```
* What went wrong:
A problem occurred configuring root project 'adr'.
> Could not resolve all artifacts for configuration ':classpath'.
   > Could not resolve org.jetbrains.kotlin:kotlin-gradle-plugin:1.3.50.
     Required by:
         project :
      > Could not resolve org.jetbrains.kotlin:kotlin-gradle-plugin:1.3.50.
         > Could not get resource 'https://plugins.gradle.org/m2/org/jetbrains/kotlin/kotlin-gradle-plugin/1.3.50/kotlin-gradle-plugin-1.3.50.pom'.
            > Could not GET 'https://plugins.gradle.org/m2/org/jetbrains/kotlin/kotlin-gradle-plugin/1.3.50/kotlin-gradle-plugin-1.3.50.pom'.
               > plugins.gradle.org

```

### 需要用户名密码的Proxy

```
systemProp.http.proxyHost=webproxy2.xxxx
systemProp.http.proxyPort=8080
systemProp.https.proxyHost=webproxy2.xxx
systemProp.https.proxyPort=8080
```

```
* What went wrong:
A problem occurred configuring root project 'adr'.
> Could not resolve all artifacts for configuration ':classpath'.
   > Could not resolve org.jetbrains.kotlin:kotlin-gradle-plugin:1.3.50.
     Required by:
         project :
      > Could not resolve org.jetbrains.kotlin:kotlin-gradle-plugin:1.3.50.
         > Could not get resource 'https://plugins.gradle.org/m2/org/jetbrains/kotlin/kotlin-gradle-plugin/1.3.50/kotlin-gradle-plugin-1.3.50.pom'.
            > Could not GET 'https://plugins.gradle.org/m2/org/jetbrains/kotlin/kotlin-gradle-plugin/1.3.50/kotlin-gradle-plugin-1.3.50.pom'. Received status code 407 from server: Proxy Authentication Required
```

### 有不需要用户名密码的Proxy

```
systemProp.http.proxyHost=webproxy.xxxx
systemProp.http.proxyPort=8080
systemProp.https.proxyHost=webproxy.xxx
systemProp.https.proxyPort=8080
```

```
* What went wrong:
A problem occurred configuring root project 'adr'.
> Could not resolve all artifacts for configuration ':classpath'.
   > Could not resolve org.jetbrains.kotlin:kotlin-gradle-plugin:1.3.50.
     Required by:
         project :
      > Could not resolve org.jetbrains.kotlin:kotlin-gradle-plugin:1.3.50.
         > Could not get resource 'https://plugins.gradle.org/m2/org/jetbrains/kotlin/kotlin-gradle-plugin/1.3.50/kotlin-gradle-plugin-1.3.50.pom'.
            > Could not GET 'https://plugins.gradle.org/m2/org/jetbrains/kotlin/kotlin-gradle-plugin/1.3.50/kotlin-gradle-plugin-1.3.50.pom'.
               > sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
```

## 解决办法

到了上面这一步，就说明我们的JVM需要一个安全证书了，所以主要工作就是按照这个安全证书。

### 导入公司证书

在你使用的JDK（比如：/c/Program Files/Java/jdk1.8.0_144/bin）下面执行如下

```bash
keytool -import -file C:/Users/xxx/Downloads/XXX_CAs_Base64/XXXDeviceCA1G2_b64.cer -alias XXXDeviceCA1G2_b64 -keystore C:/temp/certificates/mavenKeystore
```

如果还有其他证书，同样导入，中间会问你证书密码。

### 修改gradle.properties

加入如下几行

```
systemProp.javax.net.ssl.trustStore=C:\\temp\\certificates\\mavenKeystore
systemProp.javax.net.ssl.trustStorePassword=XXX
```

## 尾声

这个问题在上面的操作后就解决了，然后你最好还是要改回最新的脚步模式，也即

```kotlin
plugins {
    id("org.jetbrains.kotlin.jvm") version "1.3.50"
}
```