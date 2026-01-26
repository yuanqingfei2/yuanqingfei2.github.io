---
title: WSL进行Docker开发一个注意事项
date: "2026-01-25T20:56:00.000Z"
---

Windows11自带的WSL2系统就是个Ubuntu,在上面跑Docker,在windows里面的IDEA进行开发，本来配合得挺好的，可是后来发现无论是数据库还是ActiveMQ都经常中断，而且不是因为忙中断(`HikariCP - Failed to validate connection (No operations allowed after connection closed.)`)，是系统一起来就中断，一开始疯狂调Hikari各种参数，结果都不管用。多种情况都表明这个事情是WSL引起来的，目前WSL的配置（.wslconfig）如下：
```
[wsl2]
memory=12GB  # Limits VM memory in WSL 2GB, also can be set to other values
swap=6GB
networkingMode=mirrored
[experimental]
autoMemoryReclaim=gradual  # 或者设置为 dropcache
hostAddressLoopback=true
```
这种配置的好处就是windows和Docker共享局域网地址，这样少了中间环节，而且从windows的浏览器或者数据库客户端就可以直接用局域网IP地址访问Docker的服务，非常好。但是就如一开头所说，连接不稳定，经常被中断连接，问了AI后知道，问题就出在loopback为true上面，因为进来的IP地址请求会在windows和Docker反复横跳。所以要更改为false，试验证实更改为false确实docker系统稳定了。但是带来的副作用是无法在windows系统中使用IP访问了，恰好我的Mysql和Nacos这两个系统都是暴露在局域网IP上的，直接就嘎了。我之所以把所有docker系统都暴露在局域网IP上而不是默认的`0.0.0.0`，原因是为了上了生产之后的安全性。但现在为了开发，权宜之计就是把这两个docker服务的端口映射临时更改为默认的，等到生产的时候再更改回来。

最后完整的配置如下
```
[wsl2]
memory=12GB  # Limits VM memory in WSL 2GB, also can be set to other values
swap=6GB
networkingMode=mirrored
[experimental]
autoMemoryReclaim=gradual  # 或者设置为 dropcache
hostAddressLoopback=false
ignoredPorts=53,67,68
dnsTunneling=true
firewall=false
```