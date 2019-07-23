---
layout: post
title: 关于Vue中nextTick的理解
subtitle: 关于Vue中nextTick的理解
date: 2019-7-23
author: OYH
header-img: img/js/necttick.jpg
catalog: true
tags:
  - js
---

### [官网说明](https://cn.vuejs.org/v2/guide/reactivity.html#%E5%BC%82%E6%AD%A5%E6%9B%B4%E6%96%B0%E9%98%9F%E5%88%97)

> 当你设置 vm.someData = 'new value' ，该组件不会立即重新渲染。当刷新队列时，组件会在事件循环队列清空时的下一个“tick”更新。多数情况我们不需要关心这个过程，但是如果你想在 DOM 状态更新后做点什么，这就可能会有些棘手。虽然 Vue.js 通常鼓励开发人员沿着“数据驱动”的方式思考，避免直接接触 DOM，但是有时我们确实要这么做。为了在数据变化之后等待 Vue 完成更新 DOM ，可以在数据变化之后立即使用 Vue.nextTick(callback) 。这样回调函数在 DOM 更新完成后就会调用。

### 任务队列

js 是单线程语言，所有任务执行都要排队，这样就会有一个先后顺序。如果前一个任务需要花费大量时间来计算，那么后一个任务必须等待它执行完后才会执行。而 js 的任务分为两种，同步任务和异步任务

- 同步任务就是按照顺序一个一个执行任务，后一个任务执行必须等待它前一个任务执行完成
- 异步任务（回调）不会占用主线程，会被赛道任务队列，等主线程的任务执行完毕，就会把这个异步任务队列里的任务放到主线程一次执行

### 事件队列

除了主线程外，任务队列分为 `microtask（微任务）和` `macrotask（宏任务），`

- 执行优先级为：主线程 > microtask > macrotask
- 典型的`macrotask`有`setTimeout`和`setInterval`,以及只有 IE 支持的 setImmediate，还有 MessageChannel 等，ES6 的 Promise 则是属于 microtask

### ES6 中的 Promise

遇到 Promise 中的 then 之前部分立即执行，而 then 则属于 microtask，进入任务队列

### nodejs 中的 process.nextTick

nodejs 中内置的 nextTick 属于 microtask

### nextTick

从 [源码](https://github.com/vuejs/vue/blob/dev/src/core/util/next-tick.js) 不难发现，Vue 在内部尝试对异步队列使用原生的 setImmediate Promise.then 和 MessageChannel，如果当前执行环境不支持，就采用 setTimeout(fn, 0)代替。

```js
if (typeof Promise !== "undefined" && isNative(Promise)) {
  const p = Promise.resolve();
  timerFunc = () => {
    p.then(flushCallbacks);
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop);
  };
  isUsingMicroTask = true;
} else if (
  !isIE &&
  typeof MutationObserver !== "undefined" &&
  (isNative(MutationObserver) ||
    // PhantomJS and iOS 7.x
    MutationObserver.toString() === "[object MutationObserverConstructor]")
) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1;
  const observer = new MutationObserver(flushCallbacks);
  const textNode = document.createTextNode(String(counter));
  observer.observe(textNode, {
    characterData: true
  });
  timerFunc = () => {
    counter = (counter + 1) % 2;
    textNode.data = String(counter);
  };
  isUsingMicroTask = true;
} else if (typeof setImmediate !== "undefined" && isNative(setImmediate)) {
  // Fallback to setImmediate.
  // Techinically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = () => {
    setImmediate(flushCallbacks);
  };
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0);
  };
}
```

### 终极问题

```js
console.log(1);
setTimeout(function() {
  console.log("2");
  process.nextTick(function() {
    console.log("3");
  });
  new Promise(function(resolve) {
    console.log("4");
    resolve();
  }).then(function() {
    console.log("5");
  });
});
process.nextTick(function() {
  console.log("6");
});
new Promise(function(resolve) {
  console.log("7");
  resolve();
}).then(function() {
  console.log("8");
});

setTimeout(function() {
  console.log("9");
  process.nextTick(function() {
    console.log("10");
  });
  new Promise(function(resolve) {
    console.log("11");
    resolve();
  }).then(function() {
    console.log("12");
  });
});
console.log(13);
```

### 答案

> 1 7 13 6 8 2 4 3 5 9 11 10 12

这里的答案在 nodejs 中测试和浏览器中测试会有区别
