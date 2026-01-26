---
title: Spring Cloud Balancer找问题记录
date: "2025-12-20T09:34:00.000Z"
---

项目中，使用的是比较老的东西，主要是spring cloud 2021.0.8，以及对应的Ali Cloud 2021.0.5.0。最近在setup Cloud Gateway的时候，loadbalancer死活不工作，我一直使用DeekSeek来解决问题，这次它却无法给我正确提示了，我试了试google，它两次就告诉了我正确的原有，让我刮目相看。这个简单的问题花了我整整5个小时，所以记下来给后人参考。

问题很简单，如下

```yml
        - id: abc_share
          uri: lb://abc_share
          predicates:
            - Path=/share/**
          filters:
            - StripPrefix=1
```

一开始就是这样的，死活走不通，说无效uri

```
java.lang.IllegalStateException: Invalid host: lb://abc_share
        at org.springframework.cloud.gateway.filter.RouteToRequestUrlFilter.filter(RouteToRequestUrlFilter.java:85) ~[spring-cloud-gateway-server-3.1.8.jar:3.1.8]
        Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException:

```

根据Deepseek的提示，加了很多调试代码，全部无功而返。最后google的提示一阵见血：

```
在 Spring Cloud 2021.0.8 (对应 Spring Cloud Gateway 3.1.x) 中，出现 java.lang.IllegalStateException: Invalid host: lb://abc_share 的报错，结合你提供的服务名，最直接的原因是：服务名中包含下划线 _。
在 Spring Cloud Gateway 的源码逻辑中（如 RouteToRequestUrlFilter.java:85），由于 lb:// 协议需要被解析为标准 URI 格式，而标准 RFC 规范中 URI 的 Host 部分是不允许出现下划线的。当 Host 包含下划线时，Java 的 java.net.URI 类会解析失败，导致 Host 为空，从而触发“Invalid host”异常。 
```

以后打算主要依赖google，这个AI确实更厉害！