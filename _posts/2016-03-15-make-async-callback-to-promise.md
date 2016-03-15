---
layout: post
keywords: blog
description: blog
title: "将 Node 的异步函数调用转化为 Promise"
categories: [frontend]
tags: [javascript, es6]
icon: code
---
借助于 ES7 的 `async` 语法，我们可以方便的使用 Promise 来书写优美的异步调用代码。
不过现有的 Node 版本，大部分的异步函数都还是回调函数方式接口的，比如 `fs.read(fd, buffer, offset, length, position, callback)`。

笔者认为，使用 `Promise` 而不是回调函数的一大优势是: **将函数调用的责任转移到了使用异步函数者身上，而不是转嫁到异步函数去负责调用函数**。
对于异步函数，我们其实最关心的是它的返回结果，然后根据结果做一些有用的事情。这样的调用才是更自然，符合直觉的编程方式。

好消息是，通过 Node 新版本原生支持的 `Promise` 类，我们可以很方便的将一个回调函数接口的函数转化成 `Promise` 版本的接口。
以最简单的 `setTimeout` 为例，我们使用它的一大作用是在当前的异步调用环境中加一个等待时间，然后接下来做一些别的事情。

### 改写 setTimeout

```js

function foo() {
  setTimeout(() => {
    doSomething1();
    setTimeout(() => {
      doSomething2();
    }, 100);
  }, 100);
}

```

上面的形式就是回调函数经常会出现的问题。借助 `Promise`， 我们可以用下面的方式来改写：

```js
// Returns a promise which will resolve in $milliseconds
function delay(milliseconds) {
  return new Promise(res => {
    setTimeout(res, milliseconds);
  });
}

async function awesomeFoo() {
  await delay(100);
  doSomething1();
  await delay(100);
  doSomething2();
}
```

是不是看起来清爽很多了呢？

### 改写 fs.readFile
对于更复杂一点的 `fs.readFile(file, callback)` 函数，我们可以将其以类似的方式改写成 `Promise` 版本。

传统方式调用如下

```js
function foo() {
  fs.readFile('/etc/passwd', (err, data) => {
    if (err) throw err;
    console.log(data);
  });
}
```

改写后：

```js

// Promise based fs.readFile
function awesomeReadFile(path) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, (err, data) => {
      if (err) reject(err);
      resolve(data);
    });
  });
}

async function awesomeFoo() {
  try {
    const data = await awesomeReadFile();
    console.log(data);
  } catch (err) {
    throw err;
  }
}

```

最后，希望新版本的 `Node` 可以提供原生的 `Promise` API。<del>别做梦了</del>
