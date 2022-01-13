---
title: Spring Cloud的那些Annotation
date: "2022-01-10T02:10:00.000Z"
---

学习Spring Cloud这段时间，发现Spring把大量的抽象都放在Annotation里面，这样做的好处是对于普通用户只需要对普通Java class进行标注就可以变成具备某一特性的服务型class，列举一下。

## @EnableDiscoveryClient / @EnableEurekaClient

是最常用的，也是最基础的，加上这个标注就可以被Cloud管理机制所识别，即便将来更换管理机制，也无需更改代码，这是Spring的惯用手法。

## @EnableEurekaServer 

Eureka的容器管理标注，用来注册用。

## @EnableConfigServer

集中配置进行管理的标注，用来把所有云模块相关的configuration集中进行配置管理。这个没有啥好说的，配置集中管理可以避免大量重复甚至危险的配置，是云服务平台的核心基础设施之一。

## @EnableFeignClients / @FeignClient(name = "xxx-service", fallback = XXXClientFallback.class)

启用Feign调用机制。Feign在我看来相当神奇，思路是老思路，如同过去的CORBA，EJB，COM一样都想实现分布式调用，可是又要摆脱之前那种强耦合，不能跨防火墙等缺点，于是引入了Feign，它基础的功能是已经集成了Ribbon的负载均衡，因为云调用和过去的调用最大的不同就是要做到容易扩展，大量的微服务的Service端口进行负载均衡就变成了一个基础性的需求。然后在这基础上通过声明一个接口，从而实现方法的声明式跨服务调用，大大方便了云服务的整合，这其中的关键点有两个：

  1. 跨服务的模型肯定会有一定程度的重合，但是这是微服务所相应做出来的牺牲，而且由于没有编译期间的检查，接口正确与否完全看人工检查，因为跨服务的调用时基于json这种所有平台都接受的格式，所以带来松耦合的好处。当然服务的名字也必须人工保证是匹配的。

  2. 当和安全机制如OAUTH2等整合时，需要保证Feign调用也要保持SecurityContext。目前时通过`OAuth2FeignRequestInterceptor`来达到这一目的，期望将来可以把这个也隐含在Spring里面以避免手工配置。

## @EnableCircuitBreaker或@EnableHystrix 以及 @HystrixCommand(fallbackMethod = "fallback")

和负载均衡一样，在云平台下，由于对于网络环境的天然不信任，服务容错的能力需要提升到和其他基础设施一样重要的地位。目前是通过引入Hystrix库来实现，通过加上上面的标注来实现服务降级，依赖隔离以及断路器的功能，这样可以保护整个云平台的调用网络在某一点卡壳时候不会崩溃。这个容错机制是如此重要，以至于已经成为默认Spring Cloud的配置。如下所示

@SpringCloudApplication = @SpringBootApplication + @EnableDiscoveryClient + @EnableCircuitBreaker

## @EnableZuulProxy

同样，当大量微服务需要暴露出来时，这里面需要统一处理其中的mapping以及安全机制等，这样前端代理也成了一级基础设施。目前是通过Zuul库来实现的。

## @EnableZipkinServer

大量微服务还带来另外一个问题，如何Debug，如何跟踪这其中的调用链条，并在需要的时候很快发现新能瓶颈呢，就是利用Sleuth库以及Zipkin的信息收集功能来实现的

## @EnableTurbine

大量微服务还需要对所有服务的状况信息进行集群监控，这里就要用到Turbine服务，通过上面这个标记就可以启动了。

## @EnableResourceServer / @EnableAuthorizationServer / @EnableOAuth2Client / @EnableGlobalMethodSecurity(prePostEnabled = true) / @PreAuthorize("hasRole('ROLE_XYZ')

当然还要有基于Spring Security和OAUTH2的安全机制，具体请参考上一篇。

--------

以上就是最基本的服务设施了，但是如果我们想实现消息驱动的架构，Spring对此也做了类似的抽象，通过`@EnableBinding`来使得业务逻辑脱离具体的消息容器。如下图所示。

![Binder](https://blog.didispace.com/assets/SCSt-with-binder.png)

顺便说下，消息驱动之所以能应用在需要大量节点的云平台上，是借助了消息分组和消息分区实现的期望多重消费和期望指定节点消费。在这程中， 同时也达到了负载均衡的效果。

上面的有些服务设施也有相应的消息驱动版。比如集群监控就是`@EnableTurbineStream`, 




