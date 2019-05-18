---
title: JS同步与异步
date: "2019-05-17T15:17:00.000Z"
description: "JS同步与异步的实现与交换"
---

JS本身是单线程的，默认当然就是同步的。具体实现原理，请阅这篇[文章](https://medium.com/@siddharthac6/javascript-execution-of-synchronous-and-asynchronous-codes-40f3a199e687)。但是可以很方便的实现异步函数。同步函数异步化以及异步函数同步化是我们经常遇到的问题，下面逐一解释。

## 异步函数同步化

* 通常，我们就用Callback来实现。

```javascript
some_3secs_function(some_value, function() {
  some_5secs_function(other_value, function() {
    some_8secs_function(third_value, function() {
      //All three functions have completed, in order.
    });
  });
});
```

* 使用[promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)

这篇[文章](https://javascript.info/promise-basics)很好。

```javascript
new Promise(function(fulfill, reject){
    fulfill(result);
}).then(function(result){
    // deal with the result of the first promise
}).then(function(){
   // start the second promise
    return new Promise(function(fulfill, reject){
        fulfill(result);
    });
}).then(function(result){
    //do something with the result of the second promise
}).catch(function(error){
   // deal with error
};
```

* 使用[Q](https://github.com/kriskowal/q)

```javascript
Q.fcall(function(){
    // create promise with fcall in step 1 
}).then(function(){
    // create promise with deferred in step 2
    var deferred = Q.defer();
    FS.readFile("foo.txt", "utf-8", function (error, text) {
        if (error) {
            deferred.reject(new Error(error));
        } else {
            deferred.resolve(text);
        }
    });
    return deferred.promise;
}).then(function (text) {
    // Do something with text
}).catch(function (error) {
    // Handle any error from all above steps
}).done();
```

* promises 与 Q 在chain上的区别

我的观察是promises在处理完之前的promise后需要return一个新的promise然后继续。而Q则是把事先定义好的defered的东西直接连接起来，不需要再return了。

他们的区别这篇[文章](https://lucybain.com/blog/2016/js-promises-vs-deferred/)写的比较好。

* 使用[Promises](https://www.promisejs.org/)

```javascript
function readFile(filename, enc){
  return new Promise(function (fulfill, reject){
    fs.readFile(filename, enc, function (err, res){
      if (err) reject(err);
      else fulfill(res);
    });
  });
}

function readJSON(filename){
  return readFile(filename, 'utf8').then(JSON.parse);
}
```

## 同步函数异步化　

* setTimeout

```javascript
setTimeout(doSomething, 10);
setTimeout(doSomethingElse, 10);
setTimeout(doSomethingUsefulThisTime, 10);
```

## 感谢

* https://medium.com/@siddharthac6/javascript-execution-of-synchronous-and-asynchronous-codes-40f3a199e687

* https://stackoverflow.com/questions/5187968/how-should-i-call-3-functions-in-order-to-execute-them-one-after-the-other

* https://coderwall.com/p/ijy61g/promise-chains-with-node-js
