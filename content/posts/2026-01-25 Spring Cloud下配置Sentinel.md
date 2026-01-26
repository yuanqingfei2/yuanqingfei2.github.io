---
title: Spring Cloud 下配置Sentinel
date: "2026-01-25T21:56:00.000Z"
---

4年前偶尔[提了一嘴](https://www.yuanqingfei.com/2022-01-16%20Spring%20Cloud%20Alibaba/)，没想到今天又忙活了一回，虽然有了AI助力，还是踩了不少坑。

步骤比较简单，关键是其中几个环节调通花了不少时间。先说步骤

SENTINEL-DASHBOARD <----- 微服务 <----- NACOS

# 微服务需要加依赖
```gradle
    implementation("com.alibaba.cloud:spring-cloud-starter-alibaba-sentinel")
    implementation("com.alibaba.csp:sentinel-datasource-nacos")
```

如果是gateway，要特殊一点，还要额外加一个依赖
```gradle
implementation("com.alibaba.cloud:spring-cloud-alibaba-sentinel-gateway")
```

# Dashboard要跑起来

```yml
  monitor-sentinel:
    image: bladex/sentinel-dashboard:1.8.6
    <<: *default-log
    container_name: monitor-sentinel
    ports:
      - "8858:8858"
    restart: always
```

# 微服务的配置 --- 这个是关键

## 微服务的Docker配置
```yml
    hostname: ${SVC_NAME_GATEWAY}
    environment:
      CSP_SENTINEL_APP_NAME: ${SVC_NAME_GATEWAY}
```
hostname是对应dashboard里面的机器名，而APP_NAME就是对应dashboard左侧的应用名字。如果你有几十个微服务，区分一下是必要的。

## 微服务的启动配置

这个分成两块，一个放在common-config.yml里面所有微服务共享,就是告诉微服务dashboard在那里，启动之后去找它，把从NACOS拿到的规则都在那里显示出来。当然也可以在Dashboard里面创建新规则，但是这个规则只能实时生效，无法直接保存到NACOS那里去，也就是说配置是单向流动的。当然另外一块就是通用的fallback配置，一旦任何规则被违反，流量就先指向这个页面。

```yml
sprng:  
  cloud:
    sentinel:
      transport:
        dashboard: monitor-sentinel:8858 # 指向控制台容器地址
        port: 8719 # 客户端监控端口，保持默认即可
      scg:
        fallback:
          mode: response
          response-status: 429
          response-body: '{"code":-1,"msg":"当前访问人数过多，请稍后再试"}'
          content-type: application/json
```

另一个就是微服务自己的bootstrap.yml

```yml
spring:
  cloud:
    sentinel:
      # 1. 强制在容器启动后立刻发送心跳，不等待流量
      eager: true
      filter:
        # 3. 强制网关适配器初始化
        enabled: true
      datasource:
        # 这个是自定义名称，可以随便起
        gw-flow-rules:
          nacos:
            server-addr: ${NACOS_SERVER}
            dataId: ${spring.application.name}-gw-flow-rules.json
            groupId: SENTINEL_GROUP
            rule-type: gw-flow  
```

犯的第一个错误就是把datasource没有直接放在sentinel下面，而是放在了sentinel/transport下面了。这个配置和dashboard不同，dashboard是放在transport下面，而datasource和transport是同一个级别的。

第二个就是要保证你在NACOS中创建的规则要和你这里定义的一模一样，包括groupId(特别注意是SENTINEL_GROUP，不是DEFAULT_GROUP)，dataId(要包含最后的.json)，如果是使用python脚本publish规则还要特别注意类型。最后应该如下：

```python
data_id = f"{app_profile}.{file_type}"
publish_url = f"{NACOS_HOST}/nacos/v1/cs/configs"

group = "DEFAULT_GROUP"
my_type = "yaml" if file_type in ["yml", "yaml"] else file_type
if file_type == 'json':
    group = "SENTINEL_GROUP"

params = {
    "dataId": data_id,
    "group": group,
    "content": content,
    "type": my_type
}

# 添加命名空间参数（非public时需要）
if NAMESPACE != "public":
    params["namespaceId"] = NAMESPACE

response = session.post(publish_url, params=params)

```

总体思路是Gateway控流，而微服务之间如果有必要就加上降级。就目前而言，控流更重要，大概500到1000QPS就足够了。