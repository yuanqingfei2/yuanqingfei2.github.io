---
title: NodeJs连接Sybase
date: "2019-07-31T15:26:00.000Z"
description: "连接时遇到的问题以及解决"
---

项目遇到一个新需求，把一个传统的Java API用NodeJS来实现，由于要连接远程的Sybase数据库。这个过程中就出现了好几个问题。记录一下。

## 数据库的问题

一开始尝试的是生产数据库，用的是[DBVisulizer](https://www.dbvis.com/). 没有想到的是无论怎样都不成功，后来才知道，服务端做了某种配置，导致如下这种"Connection is closed"的错误。 

![连接错误](./DBVisulizer4_error.PNG)

连接的时候需要加密密码以及设置加密类，一定也要把那个加密类的jar放进去。 如下图所示

![连接属性](./DBVisulizer3_properties.PNG)

![加密类的jar](./DBVisulizer2_driver.PNG)

## NodeJs库的问题

连接Sybase好像也没有多少选择，用的就是[node-sybase](https://github.com/rodhoward/node-sybase)，这个库本质上就是把一个Java库包装了一下。一开始老是出现一个问题: `error : Unable to access the jarfile. ./javasybaseLink/dist/JavasybaseLink.jar.`, 搜了一圈，发现[stackoverflow](https://stackoverflow.com/questions/40459800/connecting-to-sybase-using-node) 也有人问。仔细看了下才知道是由于默认npm按照的不是最新版本(1.2.1)，而是老的版本(1.0.8)，它最新的版本不知道什么原因没有发布出来，现在解决的办法是用原来的依赖来下载库。 把最新的代码引入进来调用. 但是需要做如下调整:

```javascript
if (this.pathToJavaBridge === undefined)
{
    this.pathToJavaBridge = path.resolve(__dirname, "..", "..", "node_modules", "sybase", "JavaSybaseLink", "dist", "JavaSybaseLink.jar");
}
```

另外，这个Java库需要一个sybaseConfig.properties，它需要放在NodeJS项目的主目录上才能被找到。

```
ENCRYPT_PASSWORD=true
CHARSET=utf8
JCE_PROVIDER_CLASS=org.bouncycastle.jce.provider.BouncyCastleProvider
```

## 调用代码问题

调用的时候，就需要使用你自己引入的最新代码。 如下所示。

```javascript
var Sybase = require('../util/SybaseDB');
function querySybase(query, next){
    var db = new Sybase(nconf.get('sybase:host'), nconf.get('sybase:port'), 'master', nconf.get('sybase:user'), nconf.get('sybase:password')); 
    db.connect(function (err) {
        if (err) next(err);   
        db.query(query, function (err, data) {
            if (err) next(err);
            next(null, data);        
            db.disconnect();
            winston.info('query finished');
        });
    });
}
```

另外代码中由于每次调用后都把连接关掉了，连接需要重新生成，可是测试发现db也不能重复使用，每次调用都需要声称一个新的。这也算一个坑。

## 感谢

* https://github.com/rodhoward/node-sybase
