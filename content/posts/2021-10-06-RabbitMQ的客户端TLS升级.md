---
title: RabbitMQ的客户端TLS升级
date: "2021-10-06T15:58:00.000Z"
description: "记录一下自己完成的经历"
---

最近项目中连接的RabbitMQ升级到TLS登录了，这样就不用设置用户名和密码了，更安全了，下面把升级中的过程记录一下。

## 背景

当涉及安全性的时候，一般都要聊两个概念： Authentication和Authorization。前者指的是系统需要确认你是你，后者是指系统应该知道的你的权限如何。这里还有另外一环，就是中间的传输加密，一般就是SSL。而这次RabbitMQ的升级就是这两个方面。首先是系统如何知道你是你。 公司内部一般都有一个公告的自签名证书，如果有钱，也可以去买一个证书，基于这个母证书，你可以生成自己的专用证书(密钥)以及相应的密码。应为服务端也是拥有这个专有证书的另一半（公钥）。所以当你带着这个证书登录时，系统就知道你是谁了。SSL也是类似，但是SSL是用客户端公钥加密，服务端用私钥解密，所以TLS一般都是对称加密。具体请看到[wiki](https://en.wikipedia.org/wiki/Public-key_cryptography)

## 步骤

在公司这个系统中，使用的是NodeJs，所以库使用的是[amqplib](http://www.squaremobius.net/amqp.node)

### 连接

注意这个连接中的协议已经从amqp变成了**amqps**了

```javascript
var open = require('amqplib').connect('amqps://localhost', opts);
open.then(function(conn){
    // ...
}).then(null, console.warn)
```

### 公钥私钥

这个是定义在options这个连接参数对象里。

```javascript
options = {
    pfx: fs.readFileSync(pfxFile),
    passphrase: passphrase,
    ca: [fs.readFileSync(uatPemFile), fs.readFileSync(prodPemFile)],
}
```

### SSL

这个也是通过options来实现的，官方例子也有[具体解释](https://github.com/squaremo/amqp.node/blob/main/examples/ssl.js) Line 54

```javascript
options = {
    credentials: amqp.credentials.external(),
}
```

顺便列下C#和Java版本的对应配置

```C#
public override ConnectionFactory GetConnectionFactory(){
    return new ConnectionFactory{
        Uri = new Uri(this.GetConnectionString("secureNoCredentials")),
        Ssl = {
            Enabled = true,
            Version = SslProtocols.Tls12,
            ServerName = GetServerName(),
            CertPath = GetCertPath(),
            CertPassphrase = GetCertPassphrase(),
        },
        AuthMechanisms = new AuthMechanismFactory[] {new ExternalMechanismFacotry()}
    };
}
```

```java
private void SetSaslConfigToExternal(ConnectionFactory connectionFactory){
    CachingConnectionFactory cachingConnectionFactory = (CachingConnectionFactory) connectionFactory;
    if(cachingConnectionFactory != null){
        if(cachingConnectionFactory.getRabbitConnectionFactory().isSSL()){
            cachingConnectionFactory.getRabbitConnectionFactory().setSaslConfig(DefaultSaslConfig.EXTERNAL);
        }
    }
}
```

如果没有这个设定，就会出现下面的错误：

```
Error: Handshake terminated by server: 403(ACCESS-REFUSED) with message \"ACCESS_REFUSED - Login was refused using authentication mechanism PLAIN. For details see the broker logfile.\""}
```

## 其他

关于在服务端激活TLS，可以参考[官方文档](https://wwww.rabbitmq.com/ssl.html#enabling-tls)。本质上和客户端是对应的，也是三个key

```properties
listeners.ssl.default = 5671

ssl_options.cacertfile = /path/to/ca_certificate.pem
ssl_options.certfile   = /path/to/server_certificate.pem
ssl_options.keyfile    = /path/to/server_key.pem
ssl_options.verify     = verify_peer
ssl_options.fail_if_no_peer_cert = true
```



