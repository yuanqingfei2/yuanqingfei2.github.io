---
title: JS同步与异步
date: "2019-05-17T15:17:00.000Z"
description: "JS同步与异步的实现与交换"
---

JS本身是单线程的，默认当然就是同步的。具体实现原理，请阅这篇[文章](https://medium.com/@siddharthac6/javascript-execution-of-synchronous-and-asynchronous-codes-40f3a199e687)。但是可以很方便的实现异步函数。同步函数异步化以及异步函数同步化是我们经常遇到的问题，下面逐一解释。

Update on 20190620: 增加async/await

## 同步函数异步化　

* setTimeout

```javascript
 setTimeout(function(){ alert("Hello"); }, 3000);
```

* [promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)

创造promise。 

```javascript
var promise = new Promise(function(resolve, reject) {
    // some heavy opertion
    console.log('Heavy operation');
    // resolve or reject
    resolve(results)
    if(err) reject(err, null);
});
console.log(promise);
```

* [Q](https://github.com/kriskowal/q)

同样是创造promise，方式稍微不同。

```javascript
var promise = function(){
    var deferred = Q.defer();

    // resolve or reject
    if(err) {
        deferred.reject(err, null);
    } else {
        deferred.resolve(result);
    }

    return deferred.promise;
}
```

* Async/Await

这个新特性是[ECMAScript 2017 (ECMA-262)](https://www.ecma-international.org/ecma-262/8.0/#sec-async-function-definitions)引入的。它几乎是一个革命性的改进，使得JS对异步的支持更加优雅。[这篇文章](https://hackernoon.com/6-reasons-why-javascripts-async-await-blows-promises-away-tutorial-c7ec10518dd9) 对此作了很好的总结。

Promise Version:

```javascript
const makeRequest = () => {
  try {
    getJSON()
      .then(result => {
        // this parse may fail
        const data = JSON.parse(result)
        console.log(data)
      })
      // uncomment this block to handle asynchronous errors
      // .catch((err) => {
      //   console.log(err)
      // })
  } catch (err) {
    console.log(err)
  }
}
```

Async/Awiat Version:

```javascript
const makeRequest = async () => {
  try {
    // this parse may fail
    const data = JSON.parse(await getJSON())
    console.log(data)
  } catch (err) {
    console.log(err)
  }
}
```

## 异步函数同步化

callback是最常用的办法。

* [Callback](https://codeburst.io/javascript-what-the-heck-is-a-callback-aba4da2deced)

```javascript
some_3secs_function(some_value, function() {
  some_5secs_function(other_value, function() {
    some_8secs_function(third_value, function() {
      //All three functions have completed, in order.
    });
  });
});
```

ES6支持promise了，在promise之后利用then来连接确保顺序执行。如果是多个promise，需要在完成前一个promise的同时返回下一个promise，这样可以顺序执行并处理其结果。

* [promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)

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

* [Q](https://github.com/kriskowal/q)

Q同样是利用then，逻辑上和上文的Promise一样。

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

* 另外一个[Promises](https://www.promisejs.org/)库

没有本质区别，语法稍有不同。

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

## 感谢

* https://medium.com/@siddharthac6/javascript-execution-of-synchronous-and-asynchronous-codes-40f3a199e687

* https://stackoverflow.com/questions/5187968/how-should-i-call-3-functions-in-order-to-execute-them-one-after-the-other

* https://coderwall.com/p/ijy61g/promise-chains-with-node-js

* https://hackernoon.com/6-reasons-why-javascripts-async-await-blows-promises-away-tutorial-c7ec10518dd9
