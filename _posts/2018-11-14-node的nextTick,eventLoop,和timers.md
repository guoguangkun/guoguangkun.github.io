---
title : nodejs的eventloop，timers和process.nextTick()【译】
layout: post
date:   2018-11-14 
categories: 技术文章
---


### eventloop详解

当启动nodejs时，它会初始化eventloop，处理提供的输入脚本（或者是丢入REPL，本文档中没有涉及到REPL相关知识，REPL详情请查看https://nodejs.org/api/repl.html#repl_repl）这会使用 async API  calls（异步api 调用），安排定时器，或者调用 process.nextTick(),然后开始处理eventloop（事件循环）

下面的表格展示了一个event loop执行的操作顺序的简化概述
```
   ┌───────────────────────────┐
┌─>│           timers 计时器         │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks  等待中的回调   │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare  空转，准备     │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll    轮询        │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check  检查         │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks   关闭回调   │
   └───────────────────────────┘
```
 


> 注意：每个盒子都被比作是eventloop的一个阶段

每个阶段都会执行一个FIFO（first in first out）的回调队列。当每个阶段有自己特殊的方法，一般来说，当事件循环进入一个阶段的时候，会执行这个阶段所有的特殊操作，然后执行当前阶段的队列中的回调，直到队列执行完，或者是达到回调的最大数量。当一个队列被执行完毕之后或者达到回调的限制，事件循环会移动到下一个阶段然后继续。


因为任何的这些操作可能会安排更多的操作，并且新的事件在轮询阶段被内核插入队列，轮询事件可以被插入队列

当轮询事件被执行的时候。结果，长时间执行的回调会允许轮询阶段会被允许比定时器的时间更长。更多详情请查看[timers](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#timers)和[poll](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#poll)的相关章节。


注意：在实际调用中，windows 和Unix和Linux系统会有很小的差异，但是对此次示范操作无关紧要，最重要的地方在这儿。有实际上有七八个步骤但是我们实际关心的也就是nodejs实际上使用的是上面的那些。

### 阶段概览

* timer ：此阶段执行由setTimeout()和setInterval()规划的的回调
* pending callbacks：执行延迟到下次循环迭代的I/O回调D
* idle，prepare：仅限内部使用
* poll：检索新的I/O事件，执行I/O相关的回调（几乎全部和关闭回调概念相关，通过timers和setImmediate进行排期。）
* check：setImmediate（）回调将会在次执行
* close callbacks：一些执行关闭的函数，例如 socket.on('close', ...)


在每次事件循环执行的中间，nodejs会检查是否在等待一些定时器或者是异步I/O操纵，然后没有的话就会彻底关闭。


### 阶段详情


#### timers （定时器）


定时器具体说明了在一个被提供的回调被执行之后的时间而不是人像让它执行的确切时间。定时器回调回在声明的时候传入具体的时间之后就开始执行，然而，操作系统线程或者其他回调的执行会是它们延迟。

注意：技术上，轮询阶段控制定时器该何时执行

例如，你声明了一个在100ms之后执行的settimeout，然后你的脚本开始异步读取文件用掉了95ms：

```
const fs = require('fs');

function someAsyncOperation(callback) {
  // 假设这需要95ms来完成
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;

  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);


// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(() => {
  const startCallback = Date.now();

  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});
```



当事件循环进入轮询阶段，会有个空队列（fs.readFile()）未完成，所以会用剩余的时间等待，直到接下来的定时器的时间到了。当等待95ms之后，fs.readfile(）完成读取文件，并且他的回调回占用10ms来完成将它添加到队列并执行。当回调完成，就没有其他回调了，所以事件循环会看到接下来的定时器的极限时间已经到了，然后绕回定时器阶段来执行定时器的回调。在这个例子中，你会看到，在定时器被声明到他的回调被执行中的延迟时间是105ms。


注意：为了防止轮询阶段使事件循环等待，libuv（用来执行nodejs的事件循环和此平台其他所有的异步行为 的c 语言库）在它停止轮询更多事件之前，也会有一个严格的最大值（取决于系统）


#### 等待中的回调：

此阶段执行一些系统操作的回调，比如像TCP错误之类的。举例来说如果一个TCP socket当尝试连接时接收到 ECONNREFUSED（链接被拒绝），一些系统模块会等待着来报错，这就会在等待回调阶段执行。

#### poll 轮询

轮询阶段由两个主要的函数

1 计算需要阻塞多长时间，并且进行I/O轮询，然后

2 处理在轮询队列的事件

当事件循环进入轮询阶段并且没有定时器队列，两件事的其中之一会发生：


* 如果轮询队列不是空的，事件循环将会通过队列中的回调进行迭代同步的执行，直到要么队列满了，或者是达到系统最大限制
* 如果轮询队列是空的，那么就会发生更多的两件事：
  * * 1  如果脚本是用setImmediate（）来执行的，事件循环将会结束论序阶段然后继续检查阶段执行那些队
                   列脚本
  * * 2 如果脚本没有用setImmediate（），事件循环将会等待回调被加到队列中，然后立即执行

一旦轮询队列是空的，事件循环将会检查计时器，看谁的时间快到了，如果一个或者多个计时器准备执行了，事件循环将会绕回到计时器阶段，来执行那些定时器的回调。


#### Scheck 检查

这个阶段在轮询阶段结束之后允许人立即执行回调。如果轮询阶段变成空转并且脚本被setImmdediate（）插入到队列中了，事件循环将会继续检查阶段而不是空等。

setImmdediate（）实际上是一个特殊的定时器，运行在事件循环的一个分开的阶段。它使用了一个能安排回调在轮询阶段结束之后执行的libuvAPI.

总体来说，当代码被执行时，事件循环回最终会碰上等待联入链接，请求等等的轮询阶段。但是，如果一个回调被setImmediate（）安排执行，而且轮询阶段变成空转，就将会结束并且继续进入检查阶段而不是等待轮询事件


#### close callbacks 关闭回调函数


如果一个socket或者handle被突然关闭（e.g.socket.destory()）”close“事件会在这个阶段释放，否则将会在process.nextTick（）中释放


### setImmediate() vs setTimeout()


setImmediate和setTimeout很相似，但是当他们被调用的时候表现的确不一样

* setImmediate（）被设计来一旦当前的轮询阶段完成就执行一段脚本
* setTimeout（）安排一段脚本在一段很短的时间过后执行


这两个函数哪个先执行将会极大的取决当前被调用的上下文。如果两者都在主模块被调用，时间将会与进程的性能相关（这将会被其他在这台机器上跑的应用影响）


例如如果我运行下面这段没有I/O 循环的脚本（i.e. 主模块）这两个计时器的执行顺序是不确定的，因为受进程的性能影响。






```
// timer.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});

```
结果：
```
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```

但是入过你把这两个函数放在一个I/O 循环里去调用，那么immediate函数将会一直先执行

```
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```

结果：
```
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```


使用set immediate函数而不是settimeout（）的最主要原因是setImmediate如果在I/O循环中被调用，就会比任何计时器都要先执行，独立于不管有多少计时器。


### process.nextTick()
#### Understanding process.nextTick()理解 process.nextTick()

你可能会注意到process.nextTick()没有在表格中展示，即使它是异步API的一部分，这是因为技术上rocess.nextTick()不是事件循环的一部分。并且，nextTickQueue队列将会在当前操作完成后立即执行，不管当前是事件循环的哪个阶段。


回头看一下我们的表格，任何时候任何阶段当你调用process.nextTick()时，所有传给process.nextTick()的回调，在事件循环继续之前都将会被标记成已解决。这会造成一些很坏的情况，因为他允许你通过使用process.nextTick() 做递归调用来使I/O进程等待，这会阻止事件循环进入轮询阶段。


为什么会被允许呢？

为什么node中会有这种东西呢？其中的一部分是设计理念，即 API应该始终是异步操作，即使没必要是。下面是个例子：

```
function apiCall(arg, callback) {
  if (typeof arg !== 'string')
    return process.nextTick(callback,
                            new TypeError('argument should be string'));
}
```



这个小片段做了一个参数校验，如果不正确就会将错误对象传给回调，API最近才更新的允许传参数给process.nextTick()使他能够接收任何在回调函数扩散之后作为回调函数的参数传过来的参数，所以你就用不着进行函数嵌套了。


我们正在做的就是在允许用户的剩下的代码能继续执行的条件下，传递错误对象回去给用户。通过使用process.nextTick()我们保证那个APICall（）会一直在用户剩下的代码之后和事件循环被允许之前执行它的回调。为了实现这个目的，JS调用栈被允许来解开，然后立即执行被提供的回调，这允许用户做出递归调用来使用process.nextTick()，而不会报RangeError: Maximum call stack size exceeded from v8的错。


这个理念会导致可能的有问题的情况，就拿下面这段代码来说：

```
let bar;

// this has an asynchronous signature, but calls callback synchronously。 异步特征的却被同步调用
function someAsyncApiCall(callback) { callback(); }

// the callback is called before `someAsyncApiCall` completes.回调在someAsyncApiCall完成之前回调
someAsyncApiCall(() => {
  // since someAsyncApiCall has completed, bar hasn't been assigned any value 调用完成也还没赋上值
  console.log('bar', bar); // undefined
});

bar = 1;
```



用户定了someAsyncApiCall()来获取异步签名，但是实际上却是同步操作的，当被调用时，提供给函数的回调，在同一个事件循环的统一阶段被调用，因为函数没有做任何异步操作。结果，回调试图引用 bar这个变量，在作用域中甚至还没有那个变量，因为简本还没运行完。


通过把回调函数放在一个process.nextTick()中，代码还有能力继续运行完，允许所有变量，函数，等等在函数回调调用之前进行初始化。它还不让事件循环继续的优点。可以在让用户在事件循环被允许继续之前报出错误帮上忙。


这里先前的例子使用 process.nextTick：

```
let bar;

function someAsyncApiCall(callback) {
  process.nextTick(callback);
}

someAsyncApiCall(() => {
  console.log('bar', bar); // 1
});

bar = 1;

```



这里是现实中另一个例子：

```
const server = net.createServer(() => {}).listen(8080);

server.on('listening', () => {});
```


当仅传一个端口时，端口立即被绑定。所以，“listening”回调将会被立即调用。问题就是.on('listening')回调那会儿还没设置。


为了克服这个问题，listening事件被在nexttick（）中插入队列，这样就允许脚本能运行完。这允许用户设置任何他们想设置的方法。


### process.nextTick() vs setImmediate()


我们有两个回调，他们在用户考虑的范围内很相似，暗示他们的名字却很令人疑惑。

* process.nextTick()在当前阶段立即执行
* setImmediate()在事件循环的下一次迭代或者是tick的时候立即执行


本质上，名字应该互换一下，process.nextTick()比setImmediate()执行更加迅速，但是这是过去人为决定的不太可能更改。如果让它们交换将会破坏很大一部分的npm包。每天都有新的包添加到npm，意味着我们每等一天都会有更多可能会被破坏的包出现。虽然他们很疑惑，但是名字是不会变的。

我们建议开发者在任何情况下都使用setImmediate()因为更容易去解释为什么（并且会使代码在更多的环境中都能兼容，像浏览器中的js）


### 为什么使用process.nextTick()?


有两个主要原因：

1 允许用户解决报错，清理任何将来不需要的资源（垃圾清理）或者是在事件循环继续之前再次发送请求。

2 有时，允许回调函数在调用栈解开之后但是在事件循环继续之前运行时必要的。


一个满足用户期待的简单例子：

```
const server = net.createServer();
server.on('connection', (conn) => { });

server.listen(8080);
server.on('listening', () => { });

```

listen() 运行在生命周期的开始，但是监听回调被放在一个setImmediate()立即执行函数里，除非端口名被传进去，否则会立即绑定到端口。为了事件循环执行，他必须进入轮询阶段，也就是意味着有非零的几率，可能会接收到一个在监听事件开始之前就触发的连接事件


另一个例子就是运行一个函数的构造器，也就是说继承一个EventEmitter并且想在构造器里调用事件


```
const EventEmitter = require('events');
const util = require('util');

function MyEmitter() {
  EventEmitter.call(this);
  this.emit('event');
}
util.inherits(MyEmitter, EventEmitter);

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});


```

你不能在构造器里立即释放一个事件，以为你脚本还没运行到用户为事件定义回调函数的地方。所以在构造函数内部，你可以使用process.nextTick()来在构造器完成之后设置一个回调来释放这个事件，也就是下面这个例子所期望的结果：



```
const EventEmitter = require('events');
const util = require('util');

function MyEmitter() {
  EventEmitter.call(this);

  // use nextTick to emit the event once a handler is assigned
  process.nextTick(() => {
    this.emit('event');
  });
}
util.inherits(MyEmitter, EventEmitter);

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
```







原文链接：https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/


参考：http://www.ruanyifeng.com/blog/2013/10/event_loop.html
