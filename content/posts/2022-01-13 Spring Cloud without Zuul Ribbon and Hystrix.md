---
title: Spring Cloud without Zuul Ribbon and Hystrix
date: "2022-01-13T00:01:00.000Z"
---

上次说到，2021版本的Spring Cloud出于netflix不再开源新版本组件而被迫放弃对Zuul，Ribbon以及部分Hystrix，那么我们尝试下在没有他们的情况下升级piggymetrics。也就是基于Spring所推荐的替代品列表：

当前组件 |替代品 
--- | --- 
Hystrix|Resilience4j
Hystrix Dashboard / Turbine|Micrometer + Monitoring System
Ribbon|Spring Cloud Loadbalancer
Zuul 1|Spring Cloud Gateway
Archaius 1|Spring Boot external config + Spring Cloud Config

第一步是抛弃所有应该抛弃Hystrix的库，也就是如下几个：

```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        <version>${spring-cloud-netflix-hystrix.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-netflix-hystrix-stream</artifactId>
        <version>${spring-cloud-netflix-hystrix.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        <version>${spring-cloud-netflix-hystrix.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
        <version>${spring-cloud-netflix-hystrix.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-turbine-stream</artifactId>
        <version>${spring-cloud-netflix-hystrix.version}</version>
    </dependency>
```

事情并没有太糟糕，只有Gateway和TubineStream两个子项目编译不能通过，显然是由于`@EnableZuulProxy`和`@EnableTurbineStream`两个注解失效。而且由于后台都去除了`spring-cloud-netflix-hystrix-stream`，自然也失去了向RabitMQ发消息的功能。当然也没有了Ribbon所带来的好处。最重要的是要先解决Zuul的问题，这样系统才能跑起来，也就是要动用Spring Gateway和Spring LB了。在之前，启动的时候还遇到下面的错误：

```
java.lang.IllegalStateException: Annotation @EnableCircuitBreaker found, but there are no implementations. Did you forget to include a starter?
	at org.springframework.cloud.commons.util.SpringFactoryImportSelector.selectImports(SpringFactoryImportSelector.java:77) ~[spring-cloud-commons-3.1.0.jar:3.1.0]
```

很显然，`@EnableCircuitBreaker`也是因为只支持hystrix而变成了行尸走肉，需要去除。 

## Spring Circuit Breaker / Feign

加上如下依赖：

```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>
```

删去部分：

```yml
feign:
  hystrix:
    enabled: true

hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 10000    
```

增加部分:

```yml
spring:
  cloud:
    circuitbreaker:
      resilience4j:
        enabled: true

feign:
  circuitbreaker:
    enabled: true
```

再增加一个默认Circuit Breaker的默认配置（不确定系统自带默认配置）

```java
	@Bean
	public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
	    return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
	            .timeLimiterConfig(TimeLimiterConfig.custom().timeoutDuration(Duration.ofSeconds(4)).build())
	            .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
	            .build());
	}
```

其他代码不用动，依然是Fallback模式。

另外，在notification的项目中，由于用到了`@RefreshScope`，所以要显式地增加如下配置（默认是disabled），

```yml
feign:
  client:
    refresh-enabled: true
```

## Spring Load Balancer

加上如下依赖

```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
```

## Spring Gateway

加上如下依赖

```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
```

配置调整比较大

删除部分

```yml
ribbon:
  ReadTimeout: 60000
  ConnectTimeout: 60000
  eureka:
    enable: true  

zuul:
  ignoredServices: '*'
  host:
    connect-timeout-millis: 60000
    socket-timeout-millis: 60000

  routes:
    account-service: /accounts/**

    auth-service:
        path: /uaa/**
        url: http://localhost:5000

    statistics-service: /statistics/**

    notification-service:
        path: /notifications/**
        serviceId: notification-service
        stripPrefix: false
        sensitiveHeaders:
```

增加部分

```yml
spring:
  cloud:
    loadbalancer:
      ribbon: 
        enabled: false   
    gateway:
      routes:
      - id: account-service
        uri: lb://account-service
        predicates:
        - Path=/accounts/**
      - id: auth-service
        uri: http://localhost:5000
        predicates:
        - Path=/uaa/**
      - id: statistics-service
        uri: lb://statistics-service
        predicates:
        - Path=/statistics/**
      - id: notification-service
        uri: lb://notification-service
        predicates:
        - Path=/notifications/** 
```

基于上面的更改，调用前几步都成功了，至此，使用新版本大功告成。

