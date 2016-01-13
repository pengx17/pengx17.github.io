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

## async/await 转译原理
`async/await` 这个便利的特性背后有着并不简单的原理。
如果没有深入了解过它的实现原理，在开发过程中有可能会碰到奇奇怪怪的问题。

我的建议是，为了更有信心的使用这个特性，我们有必要简单了解一下 `async/await` 是怎么工作的。

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
`Promise.resolve(next.value)` 中，以保持代码的连贯性。

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
}
```
上面的例子比价简单，不过还是能大概的看出使用 `async/await` 后代码显得更清晰易懂。

总的来讲，`async/await` 语法就是把 `async` 函数替换成一个生成器函数，并把函数内的 `await`
替换为 `yield`，再将函数作为参数传入 `spawn` 函数中。也就是说，隐含的转移过程为：

```javascript
async function <name>?<argumentlist><body>
=>
function <name>?<argumentlist>{ return spawn(function*() <body>, this); }
```

`async` 函数的大体转移原理如上，不过实际的转译工具对于生成器也有额外的转译过程和运行时需要添加的额外依赖。
比如 `babel` 使用 facebook 的 [`regenerater runtime`](https://babeljs.io/docs/usage/polyfill/)，这里不多做赘述。

-----
## PITFALLS!
实际项目使用中，我由于开始不熟悉 `async/await` 的原理而踩到很多坑。
这里列一下我所总结的使用这个特性时需要注意的点:

### 警惕 "Race condition"
使用 `async/await` 的一个常见错误是把 `async` 函数内的调用看作是同步的。
实际情况是，在每个 `await` 开始到下一个 `await` 之前，程序才是同步的。
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

开发过程中这是我经常碰到的一个问题:
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

### 并行（parallel）执行多个 async 函数

## 图比肯忒牛德


-----
## 参考资料
- http://tc39.github.io/ecmascript-asyncawait
- http://www.html5rocks.com/en/tutorials/es6/promises
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/then
- http://www.2ality.com/2015/03/es6-generators.html
- http://babeljs.io/repl/
