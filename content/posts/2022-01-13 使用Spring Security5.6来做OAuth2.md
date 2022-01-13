---
title: 使用Spring Security5.6来做OAuth2
date: "2022-01-13T11:41:00.000Z"
---

上一篇完成了Gateway相关部件的更新，本以为没事了，可是Security又有问题：

```
2022-01-12 14:50:41.716 ERROR [auth-service,354d1a5f8ec29f97,18024b06d903b198] 20172 --- [nio-5000-exec-2] o.a.c.c.C.[.[.[.[dispatcherServlet]      : Servlet.service() for servlet [dispatcherServlet] in context with path [/uaa] threw exception [Request processing failed; nested exception is java.lang.IllegalArgumentException: Failed to evaluate expression '#oauth2.hasScope('server')'] with root cause

org.springframework.expression.spel.SpelEvaluationException: EL1011E: Method call: Attempted to call method hasScope(java.lang.String) on null context object
	at org.springframework.expression.spel.ast.MethodReference.throwIfNotNullSafe(MethodReference.java:154) ~[spring-expression-5.3.14.jar:5.3.14]
	at org.springframework.expression.spel.ast.MethodReference.getValueRef(MethodReference.java:83) ~[spring-expression-5.3.14.jar:5.3.14]
```

原因[在这](https://github.com/spring-projects/spring-security/wiki/OAuth-2.0-Migration-Guide)(Spring Security OAuth 2.x to Spring Security 5.x):

>Spring Security converts scopes that follow the granted authority naming convention. To authorize requests or methods based on scope, you write an expression like hasAuthority("SCOPE_scope").

所以要把所有的`#oauth2.hasScope('server')` 更改为`hasAuthority('SCOPE_server')`

完成之后，确实工作了，可是被拒绝(403 Error)。如下：

```
2022-01-12 15:32:09.896 ERROR [account-service,95c58f5219e6dde1,66f7867f03f9fe8e] 28016 --- [nio-6000-exec-1] o.a.c.c.C.[.[.[.[dispatcherServlet]      : Servlet.service() for servlet [dispatcherServlet] in context with path [/accounts] threw exception [Request processing failed; nested exception is org.springframework.cloud.client.circuitbreaker.NoFallbackAvailableException: No fallback available.] with root cause

feign.FeignException$Forbidden: [403] during [POST] to [http://auth-service/uaa/users] [AuthServiceClient#createUser(User)]: [{"error":"access_denied","error_description":"Access is denied"}]
	at feign.FeignException.clientErrorStatus(FeignException.java:217) ~[feign-core-11.7.jar:na]
	at feign.FeignException.errorStatus(FeignException.java:194) ~[feign-core-11.7.jar:na]
	at feign.FeignException.errorStatus(FeignException.java:185) ~[feign-core-11.7.jar:na]
```

弄了半天，发现这个问题归根到底还是由于混用最新的Spring Cloud和要退休的Spring Security Oauth2混用造成的，因为在最新的Spring中，已经把Spring Security OAuth2都迁移到Spring Security中去了，所以应该抛弃所有Spring Security OAuth2相关依赖。当然，这也是个大动作，所有的Resource Server 和 Authorization Server 都应该重写。如下：

移除：

```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-oauth2</artifactId>
        <version>2.2.5.RELEASE</version>
    </dependency>
```

增加

```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>	
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>
```

接下来就要更改代码了，因为新版Spring Security对OAuth2的支持是重新写的，所以之前那些`@EnableResourceServer`、`OAuth2RestTemplate`、`@EnableOAuth2Client`都不再被支持，因为需要相应地大幅修改代码。写到这里不由地感叹下，之前Hoxton版本的piggymetrics真的已经比较完善了，除非特别需要紧跟Spring Cloud的脚步，否则真没有必要像我一样来踩这么多的坑。

