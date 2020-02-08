---
title: 几个小问题
date: "2020-02-07T15:41:00.000Z"
description: "最近几天解决的三个小问题"
---

## JAVA MAIL Send Failure

这个问题停奇怪的，原来没事的，最近突然出现了，有下面这个错误

```
javax.mail.SendFailedException: Send failure (javax.mail.MessagingException: Unable to obtain SASL authenticator)
        at javax.mail.Transport.send(Transport.java:163)
        at javax.mail.Transport.send(Transport.java:48)
        at com.xxx.xx.util.mail.SimpleMailSender.sendHtmlMail(SimpleMailSender.java:100)
        at com.xxx.xx.tasks.QuattroEmailSender.sendEmail(QuattroEmailSender.java:110)
        at com.xxx.xx.tasks.QuattroEmailSender.execute(QuattroEmailSender.java:56)

```

网上搜了一通，就看到这里：[https://stackoverflow.com/questions/25484961/send-failure-javax-mail-messagingexception-unable-to-obtain-sasl-authenticator](https://stackoverflow.com/questions/25484961/send-failure-javax-mail-messagingexception-unable-to-obtain-sasl-authenticator)， 虽然里面没有立即可取的答案，但是有人提醒要使用官方的JAVA包，抱着试试看的态度，把jar替换了一下，还真就没有问题了！

```
-        <dependency>
-            <groupId>org.apache.geronimo.specs</groupId>
-            <artifactId>geronimo-javamail_1.4_spec</artifactId>
-            <version>1.6</version>
-        </dependency>
-        <dependency>
-            <groupId>org.apache.geronimo.specs</groupId>
-            <artifactId>geronimo-activation_1.1_spec</artifactId>
-            <version>1.0.2</version>
-        </dependency>
-        <dependency>
-            <groupId>org.apache.geronimo.javamail</groupId>
-            <artifactId>geronimo-javamail_1.4_provider</artifactId>
-            <version>1.7</version>
-        </dependency>

+        <dependency>
+            <groupId>javax.mail</groupId>
+            <artifactId>mail</artifactId>
+            <version>1.4.7</version>
+        </dependency>        

```

## XJC 生成数组是字符串类型的问题

对于一个Double的list, XSD定义如下：

```xml
<simpleType name="Vector">
    <list itemType="double" />
</simpleType>
```

原来是使用的一个很低版本（1.2）来生成Java代码的，本来是没有问题的，最近升级，就用了一个较高版本的，其他都没有问题了，结果竟然这里出问题了。

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>jaxb2-maven-plugin</artifactId>
    <version>1.2</version>
    <executions>
        <execution>
            <goals>
                <goal>xjc</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <!--  do not change the version in source code because of too many will involved, instead, 
                as icva will migrate to quattra, we will do this in quattra project -->				
        <packageName>com.xxx.xxx.xml.v1_0</packageName>
        <schemaDirectory>${basedir}/src/main/resources</schemaDirectory>
        <bindingDirectory>${basedir}/src/main/resources</bindingDirectory>
        <clearOutputDir>true</clearOutputDir>
    </configuration>
</plugin>
```

升级后一开始使用的2.3发现不行，后来搜了下，找到了[https://github.com/mojohaus/jaxb2-maven-plugin/releases](https://github.com/mojohaus/jaxb2-maven-plugin/releases), 于是使用2.5.0， 问题解决。

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>jaxb2-maven-plugin</artifactId>
    <version>2.3</version>
    <executions>
        <execution>
            <id>xsd-v1</id>
            <goals>
                <goal>xjc</goal>
            </goals>
            <configuration>
                <sources>
                    <source>src/main/xsd/xxx/xxx-file-v4.2.xsd</source>
                </sources>
                <xjbSources>
                    <xjbSource>src/main/xsd/xx/numberEnum.xjb</xjbSource>
                </xjbSources>
                <packageName>com.xxx.xml.v1_0</packageName>
                <clearOutputDir>false</clearOutputDir>
            </configuration>
        </execution>
    <executions> 
</plugin>            
```                

## NOClassFound org/jboss/resteady/core/LoggerCategories

这个问题说来话长，老项目一来太复杂，为了在新项目里去掉老项目的依赖，把有些源代码直接搬过来了，另外为了满足变异，把相关jar的依赖也在maven上面加上了，可是毕竟不是依赖管理软件那么精确，结果就出现了运行时找不到Jar包的问题，后来仔细看了下，老项目中有两个版本的，我不幸用了低版本的，更改后问题解决. 

```xml
<dependency>
    <groupId>org.jboss.resteasy</groupId>
    <artifactId>resteasy-jaxb-provider</artifactId>
-   <version>1.2.1.GA</version>
+   <version>2.2.1.GA</version>
    <exclusions>
        <exclusion>
            <groupId>com.sun.xml.bind</groupId>

```

```xml
<dependency>
    <groupId>org.jboss.resteasy</groupId>
    <artifactId>resteasy-jettison-provider</artifactId>
-   <version>1.2.1.GA</version>
+   <version>2.2.1.GA</version>
</dependency>
```


自从公司网络管理更加严格之后，没有办法在公司内写文章了，只能勉强支撑吧。


