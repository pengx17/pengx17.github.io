---
layout: post
keywords: javascript, babel, async, await, promise
description: blog
title: "ES7 async/await 使用剖析"
categories: [frontend]
tags: [javascript, es6]
icon: file-o
---

## 简介
`async/await` 特性目前做为TC39的一个提议，由微软公司提出，预计加入下一个版本的 ECMAScript 中。
在浏览器大规模支持新的ES版本前 （[也许永远不会跟上脚步](http://kangax.github.io/compat-table/es7/)），
我们可以使用流行的 `babeljs` 转译工具提前享用最新的语言特性。
> 如果你想试一下 `babeljs` 简单的测试环境并尝试一下这个语法，不妨 clone 一下我的这个项目:
> [babel-playground](https://github.com/pengx17/babel_playground)。

`async/await` 的灵感来自于 C# ([wiki](https://en.wikipedia.org/wiki/Await))，
在某些其他语言 (比如 Scala, Python) 也可以发现他的身影。
此特性依赖于新加入ES6 的 generator 和 Promise 特性。
开发者可以组合这两个特性，以类似于同步的方式写出优美的非同步的代码。

这篇文章将以 `async/await` 的转译过程开始介绍这个特性的实现原理，并在之后列举几个在开发中经常碰到的问题。

### TL;DR
> async 函数是一个返回值为 Promise 的函数。借助生成器我们可以不用回调函数嵌套的方式编写非同步代码。

## async/await 转译过程
`async/await` 这个便利的特性背后有着一个精巧的实现原理。
我的建议是，为了更有信心的使用这个特性，我们有必要简单了解一下 `async/await` 是如何转译、工作的。

`async/await` 本身可以看作 Promise + generator 的语法糖。
它的关键在于 `spawn` 函数。
Reference: [Informative Desugaring](http://tc39.github.io/ecmascript-asyncawait/#desugaring):

```javascript
function spawn(genF, self) {
    return new Promise(function(resolve, reject) {
        var gen = genF.call(self);
        function step(next) {
            try {
                next = nextF();
            } catch(e) {
                // finished with failure, reject the promise
                reject(e);
                return;
            }
            if (next.done) {
                // finished with success, resolve the promise
                resolve(next.value);
                return;
            } else {
              // not finished, chain off the yielded promise and `step` again
              Promise.resolve(next.value).then(function(v) {
                // resolve
                step(gen.next(v));
              }, function(e) {
                // reject
                step(gen.throw(e));
              });
            }
        }
        step(gen.next());
    })
});
```

这个函数代码可能初看不好理解。总的来说，函数分为几个部分:

1. 调用 ES6 生成器函数 `genF`，获得一个生成器对象 `gen`。
2. 调用 `gen` 的 `.next()` 方法。
3. 当生成器未执行完毕时，这次的运行结果为一个 `Promise` 对象。
将其结果返回给生成器，使得生成器能够拿到 `Promise` 的最终结果，并把新的生成器放入下一次递归中。
4. 当生成器执行完毕时，把最后的结果放入 `resovle` 函数的调用中。

值得注意的是，为了防止生成器 `yield` 的值不是一个 `Promise`，上面的代码中我们把这个值放入
`Promise.resolve(next.value)` 中，以保持代码的连贯性。这一点在文章稍后也有提到。

### 实际调用例子：

```javascript
spawn(function* () {
  const res0 = yield Promise.resolve(123);
  const res1 = yield Promise.resolve(res0 + 456);
  return res1; // should be 123 + 456
});
```

### 上面的代码使用 `async/await` 语法糖的写法：

```javascript
async function myAsyncFunction() {
  const res0 = await Promise.resolve(123);
  const res1 = await Promise.resolve(res0 + 456);
  return res1; // should be 123 + 456
}

// 而不用async/await的写法是...
function myAsyncFunction2() {
  function p1 = function(res) {
    return Promise.resolve(res + 456);
  }

  Promise.resolve(123).then(value => {
    return p1(value);
  });

  // 注意，此函数的返回类型为 Promise(Promise)，但 Promise 对象在调用 Promise.resolve 方法时
  // 会追随给定的 Promise 执行到 resolve/reject 为止。因此在使用上和 myAsyncFunction 是等效的。
}
```
上面的例子比较简单，不过还是能大概的看出使用 `async/await` 后代码显得更清晰易懂。

总的来讲，`async/await` 语法就是把 `async` 函数替换成一个生成器函数，并把函数内的 `await`
替换为 `yield`，再将函数作为参数传入 `spawn` 函数中。也就是说，隐含的转移过程为：

```javascript
async function <name>?<argumentlist><body>
=>
function <name>?<argumentlist>{ return spawn(function*() <body>, this); }
```

`async` 函数的大体转移原理如上，不过实际的转译工具对于生成器也有额外的转译过程（比如用循环迭代代替递归）和运行时需要添加的依赖。
比如 `babel` 使用 facebook 的 [`regenerater runtime`](https://babeljs.io/docs/usage/polyfill/)，这里不多做赘述。

-----
## PITFALLS!
实际项目使用中，本人由于刚上手时不熟悉 `async/await`，踩到很多坑。
这里列一下我总结的使用这个特性时需要注意的点:

### 警惕 "Race condition"
使用 `async/await` 的一个常见错误是把 `async` 函数内的调用看作是同步的。
实际情况是，在每个 `await` 开始到下一个 `await` 之前，程序的执行才是同步的。
这一点在 `async` 函数内部修改外部的状态时需要格外小心。比如下面的例子：

```javascript

async function foo(uri) {
  // 请求某一资源
  const res = await $.ajax(uri);
  // 用res做一些其他事情
  generateView(res);
}

// 由用户操作触发的两次连续foo调用
foo(slowURI); // 返回较慢
foo(fastURI); // 返回较快

// 视图最终被渲染为 slowURI 的内容
```

这是我开发过程中经常碰到的一个问题:
如果第一次的调用返回比较慢，就会将视图渲染为第一次的结果，然而这显然不是我们想要的。

#### 比较简单的解决方案是：

```javascript
let lastRequestedURI = null;

async function foo(uri) {
  lastRequestedURI = uri;
  // 请求某一资源
  const res = await $.ajax(uri);
  // 当全局的请求URI跟这次的请求一致时才重新渲染视图
  if (lastRequestedURI === uri) {
    // 用res做一些其他事情
    generateView(res);
  }
}

// 由用户操作触发的两次连续foo调用
foo(slowURI); // 返回较慢
foo(fastURI); // 返回较快

// 视图最终被渲染为 fastURI 的内容
```

### 正确处理 async 函数中的异常
根据转译过程的代码看出，想要让 `async` 函数中的 `Promise` 抛出的异常正常的被捕捉，一定要在调用
Promise 的时候与 `await` 组合。

如下面的代码所示：

```javascript
function testExceptions() {
  // 在 Node.js 中处理未捕捉的 Promise 异常
  process.on('unhandledRejection', (reason, p) => {
    console.log("Unhandled Rejection - " + reason + " reason");
  });

  async function foo() {
    throw 'some error';
  }

  // 异常未被捕获
  try {
    foo();
  } catch(e) {
    console.log("normal try/catch - " + e);
  }

  // 异常未被捕获
  (async () => {
    try {
      foo();
    } catch(e) {
      console.log("async try/catch without await - " + e);
    }
  })();

  // 异常被捕获
  (async () => {
    try {
      await foo();
    } catch(e) {
      console.log("async try/catch with await - " + e);
    }
  })();
}
// 执行结果：
// async try/catch with await - some error
// Unhandled Rejection - some error reason
// Unhandled Rejection - some error reason
```

谨记一点：`async` 函数中的 `Promise` 如果不是跟 `await` 组合，那么他的返回值还是一个 `Promise`。
开发过程中请谨慎对待 **"游离状态"** - 也就是没有外部引用的 - `Promise`。
虽然对 NodeJs 来说，可以通过对于 `process` 的 `unhandledRejection` 事件进行监听来处理没有被处理的 rejected Promise。
但这种用法会失去对于 `Promise` 执行状态的追踪，使得代码的容错水平降低。

### await 一个非 Promise 值

例子：

```javascript
function bar() {
  return Math.random() * 2 > 1 ? 123; somePromise;
}

async function foo() {
  await bar();
}
```

在之前的转译过程解释中有提到，`await` 的值实际会被包裹在一个 `Promise.resolve()` 中。
因此上面的代码可以正常工作。

#### 注意
Promise.resolve(somePromise) 等效于 somePromise。

> [*Promise.resolve*](Promise.resolve(value)):
Returns a Promise object that is resolved with the given value.
If the value is a thenable (i.e. has a then method),
the returned promise will "follow" that thenable,
adopting its eventual state; otherwise the returned promise will be fulfilled with the value.
**Generally, if you want to know if a value is a promise or not - Promise.resolve(value)
it instead and work with the return value as a promise.**

类似的，下面的代码也可正常工作：

```javascript
async function foo() {
  return somePromise; // 没有调用 await
}
```

### 并行（parallel）执行多个 async 函数
利用 `async` 函数我们可以方便的顺序执行 Promise。比如下面的例子：

```javascript
// 返回一个在 milliseconds 毫秒后完成的一个 Promise
function delay(milliseconds, index) {
  return new Promise(res => {
    setTimeout(() => {
      res(`[${index}]: Res after ${milliseconds} milliseconds`);
    }, milliseconds);
  });
}

// 顺序执行
async function sequence() {
  const start = new Date();
  for (let i = 0; i < 10; i ++) {
    console.log(`${await delay(Math.random() * 1000, i)}`);
  }
  console.log(`sequence done in ${new Date() - start} ms`);
}

/* 输出
[0]: Res after 986.4941246341914 milliseconds
[1]: Res after 333.2838623318821 milliseconds
[2]: Res after 354.3416520114988 milliseconds
[3]: Res after 834.6803605090827 milliseconds
[4]: Res after 215.9734272863716 milliseconds
[5]: Res after 221.03742230683565 milliseconds
[6]: Res after 114.20689150691032 milliseconds
[7]: Res after 28.70347397401929 milliseconds
[8]: Res after 524.5535324793309 milliseconds
[9]: Res after 669.4943546317518 milliseconds
sequence done in 4318 ms
*/
```

然而开发中，我们经常需要并行执行多个 Promise。
比如典型的网络请求情形，一般来说我们需要同时发出多个网络请求，并在所有请求返回时进行下一步操作。
如果用上面的顺序执行方案的话，JavaScript 的非阻塞特性没有被充分利用。

如果不同的 Promise 之间并没有依赖，就可以用并行的方式执行他们。

```javascript
async function parallel() {
  const delays = [];
  const start = new Date();
  for (let i = 0; i < 10; i ++) {
    delays.push(delay(Math.random() * 1000));
  }
  console.log(await Promise.all(delays));
  console.log(`parallel done in ${new Date() - start} ms`);
}

/*
[ '[0]: Res after 193.9734136685729 milliseconds',
  '[1]: Res after 323.2925720512867 milliseconds',
  '[2]: Res after 935.3614274878055 milliseconds',
  '[3]: Res after 422.59012744762003 milliseconds',
  '[4]: Res after 318.91681230627 milliseconds',
  '[5]: Res after 39.97510578483343 milliseconds',
  '[6]: Res after 616.469515254721 milliseconds',
  '[7]: Res after 696.0562060121447 milliseconds',
  '[8]: Res after 859.5389637630433 milliseconds',
  '[9]: Res after 473.90571935102344 milliseconds' ]
parallel done in 938 ms
*/
```

并行执行的诀窍在于 [`Promise.all`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) 方法。
我们需要事先把需要并行的 Promise 放入一个数组内，然后传入这个方法。
当所有的 Promise 执行完毕后，`all` 会把所有 Promise 的结果按顺序放入最终结果的数组内。

-----
## 参考资料
- [TC39 asyn/await proposal](http://tc39.github.io/ecmascript-asyncawait)
- [HTML5 rocks Promise tutorial](http://www.html5rocks.com/en/tutorials/es6/promises)
- [MDN Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
- [Babel Online REPL](http://babeljs.io/repl/)
