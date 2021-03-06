## 单线程

首先，JS 的一大特点是单线程，也就是说同一时间只能处理一件事。单线程的好处是可以从根源规避掉一些多线程带来的同步问题，毕竟 DOM 是直接呈现给用户的，不同于 Java 的各种“锁”。

这里有个个人理解，单线程的意义是指，JS 引擎只会以一个线程来处理任务，但是监听请求返回、监听设备输入（暂且理解为接收器）应该是同时运行的，只是回调函数中的任务，还是要由引擎单线程执行。

## 任务队列

单线程的弊端就是会被阻塞，一个任务没有完成，那下一个任务自然无法开始。为了解决这个基本问题，js 将任务分成两种：一种是同步任务，一种是异步任务。同步任务就是正常在主线程上排队进行的任务。而异步任务不同，它没有直接进入主线程，而是进入了一个叫“任务队列”的地方。只有当任务队列通知主线程说，其中某一个任务可以执行了，这个异步任务才会被放到主线程中执行。

- （1）所有同步任务都在主线程上执行，形成一个执行栈（execution context stack）。
- （2）主线程之外，还存在一个"任务队列"（task queue）。只要异步任务有了运行结果，就在"任务队列"之中放置一个事件。
- （3）一旦"执行栈"中的所有同步任务执行完毕，系统就会读取"任务队列"，看看里面有哪些事件。那些对应的异步任务，于是结束等待状态，进入执行栈，开始执行。
- （4）主线程不断重复上面的第三步。

## 事件和回调函数

在我们写代码的时候，异步任务结束之后往往都会有一个回调函数。JS 在运行时，当执行一些异步的耗时操作（例如请求），或者响应一些监听时；当数据返回或者监听被响应，那么回调函数就会被放进任务队列，等待主线程在事件循环中执行。

与此同时，JS 中的定时器也会影响事件的执行。主线程在执行任务队列的时候，会检查一下执行事件，有些事件只有到了规定的时间，才能返回到执行栈被执行。

## 宏观任务和微观任务

通常，JS 的任务都是由我们发起的。由于 Promise 的引入，JS 引擎也可以自己发起任务了。因此，由宿主发起的任务就叫做宏观任务，由 JS 自己发起的任务就叫做微观任务。

联系上面的知识，我理解中，任务队列其实就是一个个的宏任务。也就是说，上面的任务队列的概念，可以对应到宏任务队列。

宏任务包括有：setTimeOut、setInterval、setImmediate、I/O、用户交互操作，UI 渲染

微任务包括有：Promise(重点)、process.nextTick(nodejs)、Object.observe(不推荐使用)

![任务环](https://zbd-image.oss-cn-hangzhou.aliyuncs.com/rumination/1802415781_26802454139_v8.gif)

## Promise

```js
var r = new Promise(function (resolve, reject) {
  console.log('a');
  resolve();
});
r.then(() => console.log('c'));
console.log('b');
// a b c
```

来分析一波这段代码，首先主线程会执行到 new Promise 里，然后打印出 a，然后到 resolve()；但是由于这个是 Promise 里的异步回调，所以打印 c 的代码会被放到任务队列中，主线程继续往下，执行打印 b。然后主线程的执行栈空掉了，就去任务队列找，发现有一个打印 c 的任务，也符合时间要求，就取出来到主线程执行了。

再来看这段代码，当 setTimeout 和 Promise 混合在一起，也就是宏任务和微任务混合到一起的时候。

```js
var r = new Promise(function (resolve, reject) {
  console.log('a');
  resolve();
});
setTimeout(() => console.log('d'), 0);
r.then(() => console.log('c'));
console.log('b');
// a b c d
```

和上面一样，首先主线程会运行到 new Promise 里，打印出 a；然后到 resolve()，打印 c 被放进任务队列，然后发现 setTimeout，会将打印 d 放进任务队列，之后执行打印 b。执行栈结束后调用任务队列，打印剩下的 c 和 d。
但是这个地方发现，不论怎么调整 c 和 d 的位置，都不会影响先打印 c 后打印 d；也就是说作为微任务的 c 会先于宏任务 d 执行

```js
setTimeout(() => console.log('d'), 0);
var r = new Promise(function (resolve, reject) {
  resolve();
});
r.then(() => {
  var begin = Date.now();
  while (Date.now() - begin < 1000);
  console.log('c1');
  new Promise(function (resolve, reject) {
    resolve();
  }).then(() => console.log('c2'));
});
// c1 c2 d
```

这段代码中，在打印 c1 之前，强制阻塞了 1s 的时间，但 d 仍然是在 c1 和 c2 之后被打印。这也再度证明了微任务的执行优先级是要高于宏任务。

最后简单得出一个结论：微任务队列优先于宏任务队列执行，微任务队列上创建的宏任务会被后添加到当前宏任务队列的尾端，微任务队列中创建的微任务会被添加到微任务队列的尾端。只要微任务队列中还有任务，宏任务队列就只会等待微任务队列执行完毕后再执行。

```js
new Promise((resolve, reject) => {
  console.log('promise1');
  resolve();
})
  .then(() => {
    console.log(1);
  })
  .then(() => {
    console.log(2);
  })
  .then(() => {
    console.log(3);
  });

new Promise((resolve, reject) => {
  console.log('promise2');
  resolve();
})
  .then(() => {
    console.log(4);
  })
  .then(() => {
    console.log(5);
  })
  .then(() => {
    console.log(6);
  });
// 1 4 2 5 3 6
```

需要注意的是，当在 promise 中，连续的几个 then()回调，并不是连续的创建了一系列的微任务并推入微任务队列，因为 then()的返回值必然是一个 Promise，而后续的 then()是上一步 then()返回的 Promise 的回调。也就是说，then 所创建的微任务不是一次性全部推进任务队列，而是陆续创建，事件循环也进行了很多次。

```js
function test1() {
  console.log(1);
}
function test2() {
  console.log(2);
}
async function test3() {
  console.log(3);
  await test1();
  await test2();
  console.log(4);
}
test3();
console.log(5);
// 3 1 5 2 4
```

async 修饰的函数会固定返回一个 promise，await 之前的代码和 await 的函数会被同步执行，await 之后的函数会被看做 promise.then()中执行，也就是添加到微任务栈中
