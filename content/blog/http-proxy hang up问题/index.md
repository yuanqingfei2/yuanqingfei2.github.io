---
title: http-proxy hang up问题
date: "2019-09-06T09:00:00.000Z"
description: "http-proxy hang up问题 记录解决过程"
---

最近项目有个新的需求，需要程序能够屏蔽某个用户，由于前段有load-balacer的缘故，原本的客户ip地址已经拿不到。所以采用的办法就是根据用户登录用的token，解密后来确定身份，进而决定屏蔽与否，如果需要屏蔽，就不再重定向的代理服务器，直接返回403，并给出客户联系方式。

## 问题

原来的结构

```javascript
// http server
var server = https.createServer(credentials, requestListener).listen(port, listen);

// http proxy server
var proxyServer = httpProxy.createProxyServer(proxyHelper.options);

// requestListener function
function requestListener(req, res) {
    ...
     if (req.method === 'POST' || req.method === 'PUT') {
        var body = [];
        var msgBody = '';
        var msgFullBody = '';
        req.on('data', function (chunk) {
            body.push(chunk);
        })
            .on('end', function () {
                body = Buffer.concat(body).toString();

                if (JSON.stringify(body).toUpperCase().indexOf('PASSWORD') === -1) {
                    msgBody = 'BODY: ' + JSON.stringify(body).slice(0, 128);
                    msgFullBody = 'BODY: ' + JSON.stringify(body);
                } else {
                    msgBody = 'BODY: Request body redacted because it contains sensitive password information.';
                    msgFullBody = 'BODY: Request body redacted because it contains sensitive password information.';
                }

                logger.info(msgBody);
                requestLogger.info(msgFullBody);
            });
    } 
    ...
    proxyServer.web(req, res, { target: target }, function(err) {
    if (err) {
        logger.error('Error! ROUTER: proxy request failed', err.stack);
        jsonLogger.error('Proxy Request Failed', makeJsonMessage(jsonHeader, {stack: err.stack}));  
        res.writeHead(500);
        return res.end(err.message);
    }});
}
```

应该说结构比较清楚，就是在代理之前，如果是`POST`的请求，通过 `on('data')` 的callback来把内容打印出来。现在由于我们要先判别用户的身份再决定是否转向代理服务器，所以需要加一个判断:

```javascript
function requestListener(req, res) {
    ...
    var token = extractToken(req);
    // try to decode the user to decide if block it or not as we cannot get ip address due to load balancer.
    if (token) {
        xxx.decodeToken(token)
        .then(function (user) {  
            var id = user.user_name || user.client_id;
            logger.info('user ID: ' + id);
            if(blockedIdsCache.includes(id)){
                res.writeHead(403);
                res.end('You are not allowed to access. Please contact xxx');
                logger.info('block this user: ' + id);
            } else {
                makeProxyRequest(req, res, target, jsonHeader);
            }
        })
        .catch(function(err) {
            var msg = 'Check user failed: ' + JSON.stringify(err.message);
            logger.error(msg);
            res.writeHead(500);
            res.end(msg);
        });
    } else {
        makeProxyRequest(req, res, target, jsonHeader);
    }
}

function makeProxyRequest(req, res, target, jsonHeader){
    proxyServer.web(req, res, { target: target }, function(err) {
        if (err) {
            logger.error('Error! ROUTER: proxy request failed', err.stack);
            jsonLogger.error('Proxy Request Failed', makeJsonMessage(jsonHeader, {stack: err.stack}));  
            res.writeHead(500);
            return res.end(err.message);
        }});
}
```

问题出来了， GET完全没有问题，每次POST的时候，就迟迟不能转向代理服务器，直到超时，最后就打印出 hang up

```
error: [2019-09-05 15:24:59.051][][XXX][pid: 26956]: Error! ROUTER: proxy request failed Error: socket hang up
    at createHangUpError (_http_client.js:268:15)
    at TLSSocket.socketOnEnd (_http_client.js:360:23)
    at emitNone (events.js:91:20)
    at TLSSocket.emit (events.js:185:7)
    at endReadableNT (_stream_readable.js:978:12)
    at _combinedTickCallback (internal/process/next_tick.js:80:11)
    at process._tickCallback (internal/process/next_tick.js:104:9)
```

## 解决方案1

经过测试发现，问题就出在 `req.on('data', function (chunk) {...}` 上面，这个callback的主要目的就是打印log，可是原始的req挂了这么个钩子后,http-proxy就不答应代理了。最简单的办法就是直接删掉这个钩子，可是我们又真的想要打印出来这个信息，怎么办?

## 解决方案2

经过多方搜索，终于找到一个解决办法，就是把这个钩子挂到代理里边去。

```javascript
proxyServer.on('proxyReq', function(proxyReq, req, res, options){
    if (req.method === 'POST' || req.method === 'PUT') {
        var body = [];
        var msgBody = '';
        var msgFullBody = '';
        req.on('data', function (chunk) {
            body.push(chunk);
        })
            .on('end', function () {
                ...
            });
    } 
});
```

这样代理就没问题，可是为何之前没加if判断的时候，怎么就没有问题呢，我现在的观察和答案是时间延迟。没加if判断的时候，代理是一步到位，直接上去的，也就是说是在req结束之前就开始代理了，而加了判断之后，代理服务器是在一个同步动作之后(then)，中间有个时间延迟，这个延迟完全导致在代理发生之前req就结束了。那么代理也就失去了接受原始req的机会。

## 其他

在搜索解决方案的时候，发现了一个类似的场景，导致类似的问题，就是body-parser如果在http-proxy之前，也会导致这个问题。具体就请参考[这里](https://stackoverflow.com/questions/25207333/socket-hang-up-error-with-nodejs/25651651#25651651) 以及 [这里](https://github.com/chimurai/http-proxy-middleware/issues/40#issuecomment-249430255).

## 感谢

* https://github.com/chimurai/http-proxy-middleware/issues/40#issuecomment-249430255

* https://stackoverflow.com/questions/25207333/socket-hang-up-error-with-nodejs/25651651#25651651
