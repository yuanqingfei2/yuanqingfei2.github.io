---
title: JS同步与异步
date: "2019-05-１７T15:17:00.000Z"
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

```javascript
new Promise(function(fulfill, reject){
    //do something for 5 seconds
    fulfill(result);
}).then(function(result){
    return new Promise(function(fulfill, reject){
        //do something for 5 seconds
        fulfill(result);
    });
}).then(function(result){
    return new Promise(function(fulfill, reject){
        //do something for 8 seconds
        fulfill(result);
    });
}).then(function(result){
    //do something with the result
});
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

一般来说推荐使用第二种方式，因为它是官方标准。　如果不复杂，也可以采用第一种方式。　

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

* https://github.com/kriskowal/q
