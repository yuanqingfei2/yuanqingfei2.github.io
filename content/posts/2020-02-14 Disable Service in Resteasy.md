---
title: Disable Service in Resteasy
date: "2020-02-14T15:35:00.000Z"
description: "Disable Service in Resteasy"
---

最近遇到一个需求，老项目要退休，告知了客户，可是客户并不放在心上，对于迁移到新服务并不上心。为了给用户一个提前警告，并且不中断其他Rest服务，需要找个办法来把某些RestService暂时下线。怎么做呢？经过一番摸索，如下。

## 环境

```xml
<!-- spring version -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-beans</artifactId>
    <version>4.3.22.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>4.3.22.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>4.3.22.RELEASE</version>
</dependency>

<!-- resteasy version mainly resteasy-jaxrs version matters -->
<dependency>
    <groupId>org.jboss.resteasy</groupId>
    <artifactId>resteasy-jaxrs</artifactId>
    <version>3.0.9.Final</version>
</dependency>    
<dependency>
    <groupId>org.jboss.resteasy</groupId>
    <artifactId>resteasy-spring</artifactId>
    <version>3.0.11.Final</version>
</dependency>
<dependency>
    <groupId>org.jboss.resteasy</groupId>
    <artifactId>resteasy-jaxb-provider</artifactId>
    <version>2.2.1.GA</version>
</dependency>
```

## Java 代码

```java
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

import javax.ws.rs.container.ContainerRequestContext;
import javax.ws.rs.container.ContainerRequestFilter;
import javax.ws.rs.container.PreMatching;
import javax.ws.rs.core.Response;
import javax.ws.rs.ext.Provider;

@Provider
@PreMatching
public class DisabledEndpointsFilter implements ContainerRequestFilter {
    @Override
    public void filter(ContainerRequestContext request) throws IOException {
    	final List<String> disabledEndpoints = new ArrayList<String>();
    	disabledEndpoints.add("agreement/ns");
        final String path = stripLeadingSlash(request.getUriInfo().getPath());
        
        System.out.println("PATH:   " + path);

        for (String endpoint: disabledEndpoints) {
            endpoint = stripLeadingSlash(endpoint);
            if (path.startsWith(endpoint)) {
                request.abortWith(Response.status(404).build());
                return;
            }
        }
    }

    private String stripLeadingSlash(String path) {
        return path.charAt(0) == '/'
                ? path.substring(1)
                : path;
    }
}
```

基本上就是告诉服务器，如果遇到配置的URL模式，就直接中断访问，返回404。　


## 配置

如果你只是写了上面的这个类，尽管有RS的Annotation，并不会自动加载，根据[文档](https://docs.jboss.org/resteasy/docs/3.0.9.Final/userguide/html_single/#Installation_Configuration)，需要在web.xml里面加上如下配置。　

```xml
<context-param>
    <param-name>resteasy.providers</param-name>
    <param-value>com.xxx.provider.DisabledEndpointsFilter</param-value>
</context-param>
```
如果有多个Filter, 用逗号分割即可。另外需要注意，由于这里已经使用了Spring来管理所有Bean，所以不能使用`resteasy.scan.providers`。　否则就会报如下错误：

```
java.lang.RuntimeException: You cannot use resteasy.scan, resteasy.scan.resources, or resteasy.scan.providers with the SpringContextLoaderLister as this may cause serious deployment errors in your application
	at org.jboss.resteasy.plugins.spring.SpringContextLoaderListener.contextInitialized(SpringContextLoaderListener.java:44)
```

## 背后故事

![workflow](https://www.baeldung.com/wp-content/uploads/2018/03/Jersey2.png)

[这里](https://www.baeldung.com/jersey-filters-interceptors)有篇很好的文章，尽管讲的Jersey，但是都差不多。上图就来自于此。

## 感谢

* https://stackoverflow.com/questions/43614223/need-jersey2-technique-to-enable-or-disable-services-during-runtime
