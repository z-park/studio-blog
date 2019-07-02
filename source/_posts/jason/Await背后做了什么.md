---
title: Await的背后做了什么？
date: 2019-07-03 11:30
tags:
    - Jason
---

## 前言

首先先来看下代码: 
```javascript
const p = Promise.resolve()

;(async () => {
  await p
  console.log('after:await')
})()

p.then(() => console.log('tick:a'))
 .then(() => console.log('tick:b'))
```

V8团队的博客中说到这段代码有两种运行结果: 

![await-bug-node-8](http://qiniu.enboest.com/await-bug-node-8.png)

> 执行结果以node v10为准，其中node v8的结果是不符合规范的


为什么结果正确的结果是这样？，所以我们要理解就要深入了解await背后干了什么，首先得了解下前置知识


## Promise状态和术语

我们先简单了解下Promise的几种状态和术语: 
### 状态: 
- pending (等待态)
处于等待态时，promise 需满足以下条件：
	- 可以迁移至执行态或拒绝态
- fulfilled  (执行态)
处于执行态时，promise 需满足以下条件：
	- 不能迁移至其他任何状态
	- 必须拥有一个不可变的终值
- reject      (拒绝态)
处于拒绝态时，promise 需满足以下条件：
	- 不能迁移至其他任何状态
	- 必须拥有一个不可变的据因


### 术语 
- thenable
是一个定义了 `then` 方法的对象或函数
- settled
一个promise被settled，指的是它脱离了`pending`状态，或者被`fulfilled`或者被`rejected`
- resolved
resolve/reject一个promise
指的是调用`new Promise((resolve,reject)=>{...}`中的`resolve/reject`函数
调用了`resolve`函数之后的promise，称为`resolved`




## 状态继承和"locked in"
比如有2个Promise(代指promiseA, promiseB), 状态继承就是promiseB继承promiseA的状态，也就是将promiseA传入到promiseB的resolve函数中调用。如下代码所示:

```javascript
const promiseA = new Promise((resolve) => {
  setTimeout(() => {
    resolve('a')
  }, 3000)
})

const promiseB = new Promise((resolve) => resolve(promiseA))

promiseB.then((val) => {
  console.log(val)  // 三秒后打印'a'
})
```

在ES文档中有两句话可以帮助我们理解状态继承:
> A promise is `resolved` if it is settled or if it has been`“locked in” `to match the state of another promise. 
>  A `resolved` promise may be pending, fulfilled or rejected.

第一句话提到了`"locked in"`，要理解它，首先我们要知道，resolve函数接受三种类型的值作为参数即(`promise, thenable`, 其他值)。
第二句话说的是一个已经被处理的promise，他的状态可能是pending，fulfilled，rejected。

这里的`"locked in"`指的是resolve函数接受`promise`或者`thenable`参数的情况，以这种方式进行resolve，会使得当前的promise仍然处于pending状态。其settled结果，和参数对象的最终状态有关。也就是说`"locked in"`是指状态继承的过程中被锁定在pending状态。

下面用伪代码来表示状态继承: 
```javascript
const promiseB = new Promise((resolve) => resolve(promiseA)) //状态继承

// 伪代码
// 向mircotask queue中添加PromiseResolveThenableJob
addToMicroTaskQueue(() => PromiseResolveThenableJob())

const PromiseResolveThenableJob = () => {
  promiseA.then(  // 向mircotask queue中添加PromiseReactionJob
	resolvePromisB,
    rejectPromiseB
  )
}

```

为了更好的理解上面的代码，需要继续了解下一些概念:

## Tasks和microtasks
>JavaScript 中有 task 和 microtask。task 用于处理 I/O 和计时器等事件，每次执行一个。microtask 为 async/await 和 Promise 实现延迟执行，并在每个 task 结束时执行。在每一个事件循环之前，microtask 队列总是被清空（执行）。

![task和microtasks](http://qiniu.enboest.com/20190703002312.png)

> 有些地方将Task称为MacroTask,将MicroTask称为Jobs。

### Promise Jobs
promise有两种job，都属于microtask
- [PromiseReactionJob](https://tc39.es/ecma262/#sec-promisereactionjob)
- [PromiseResolveThenableJob](https://tc39.es/ecma262/#sec-promiseresolvethenablejob)


**PromiseReactionJob**
如果promise还没被settled,调用`promiseA.then(res, rej)`，那么promise会将`res/rej`回调放入promise的`[[PromiseFulfillReactions]]/[[PromiseRejectReactions]]`列表尾部


如果promiseA已经被settled，会向一个名为`PromiseJobs`的`mircotask queue`中[添加](https://tc39.es/ecma262/#sec-performpromisethen)一个用来处理fulled/rejected的`PromiseReactionJob`

>` [[PromiseFulfillReactions]]`和`[[PromiseRejectReactions]]`的作用是，当promise处于pending状态时，保存`promiseA.then(res, rej)`所添加的处理器函数`res`和`rej`。

因此，一个已经settled的promise，每次调用then会向`PromiseJobs`的`mircotask queue`中添加一个`PromiseReactionJob`任务，PromiseReactionJob可以理解为promise调用then后传入的回调函数

**PromiseResolveThenableJob**
在promise对象中用promise为参数调用resolve函数，会向名为`PromiseJobs`的`mircotask queue`中[添加](https://tc39.es/ecma262/#sec-promise-resolve-functions)一个`PromiseResolveThenableJob`

现在我们回头看看之前的代码


## 状态继承内部实现
```javascript
const promiseA = new Promise((resolve) => {
  setTimeout(() => {
    resolve('a')
  }, 3000)
})

const promiseB = new Promise((resolve) => resolve(promiseA))  // <----注意看这行

promiseB.then((val) => {
  console.log(val)  // 三秒后打印'a'
})
```

当promiseB用promiseA为参数调用resolve函数时向名为`PromiseJobs`的`mircotask queue`中[添加](https://tc39.es/ecma262/#sec-promise-resolve-functions)一个`PromiseResolveThenableJob`，这就是状态继承


PromiseResolveThenableJob所做的事情也就是上面提到的代码: 

```javascript
const promiseB = new Promise((resolve) => resolve(promiseA)) //状态继承

// 伪代码
// 向mircotask queue中添加PromiseResolveThenableJob
addToMicroTaskQueue(() => PromiseResolveThenableJob())

// PromiseResolveThenableJob大概做了如下的事情
const PromiseResolveThenableJob = () => {
  promiseA.then(  // 向mircotask queue中添加PromiseReactionJob
    resolvePromisB,
    rejectPromiseB
  )
}
```

当resolvePromiseB执行后， promiseB的状态才变成resolve，也就是B追随A的状态

现在我们回到正题

## Await内部实现
V8团队的博客中是以伪代码的方式解释await的执行逻辑:
![await-under-the-hood](http://qiniu.enboest.com/20190702221749.png)

await的执行步骤是: 
1. 将v包裹promise中，v代表传递给await的值
2. 调用then给promise附加处理程序
3. 暂停执行foo，并返回implicit_promise
4. 一旦promise变为fulfilled，恢复异步函数执行，并将promise的值赋给w，而且这个 w 也是 implicit_promise 被 resolved 后的值


我们可以用promise语法写成：
```javascript
function foo(v) {
  const implicit_promise = new Promise(resolve => {
    const promise = new Promise(res => res(v));
    promise.then(w => resolve(w));
  });

  return implicit_promise;
}
```

同样我们把文章开头的代码转换成: 
```javascript
const p = Promise.resolve();

(() => {
  const implicit_promise = new Promise(resolve => {
    const promise = new Promise(res => res(p))  // 这里是状态继承
    promise.then(() => {
      console.log("after:await")
      resolve()
    })
  })

  return implicit_promise
})()

p.then(() => console.log("tick:a"))
 .then(() => console.log("tick:b"))
```

通过前面我们已经了解状态继承会有两个microtask queue ticks，而await至少需要三个microtask queue ticks，并且必须创建两个额外的Promise

![await-overhead](http://qiniu.enboest.com/20190702230637.png)

这时候，我们终于可以理清开头代码的顺序了


**mircotask[]**

p处于fulfilled状态，到状态继承调用resolve(p)向microtask queue插入任务`PromiseResolveThenableJob`

**mircotask[PromiseResolveThenableJob]**

promise处于pending状态，promise.then(...)调用，将回调放入promise的`[[PromiseFulfillReactions]]/[[PromiseRejectReactions]]`列表尾部

**mircotask[PromiseResolveThenableJob]**

p处于fulfilled状态，p.then(...)调用，向microtask queue插入任务`PromiseReactionJob(tick:a)`

**mircotask[PromiseResolveThenableJob, PromiseReactionJob(tick:a)]**

执行微任务PromiseResolveThenableJob向microtask queue插入任务`PromiseReactionJob(resolvePromise)`
执行微任务PromiseReactionJob(tick:a) ---> `输出tick:a` 返回一个promise向microtask queue插入任务`PromiseReactionJob(tick:b)`

**mircotask[PromiseReactionJob(resolvePromise),PromiseReactionJob(tick:b)]**

执行微任务PromiseReactionJob(resolvePromise)即状态继承结束，此时promise处于fulfilled状态，从`[PromiseFulfillReactions]`去取出回调向microtask queue插入任务`PromiseReactionJob('after await')`

**mircotask[PromiseReactionJob(tick:b), PromiseReactionJob('after await')]**

执行PromiseReactionJob(tick:b) ---> `输出tick:a`
执行PromiseReactionJob('after await') --->` 输出after await`


## 更快的await
之前提到的Node.js 8 中有一个 bug 导致 await 在某些情况下跳过 microticks，虽然更快但是不符合规范

V8团队的博客中提到通过[promiseResolve](https://tc39.es/ecma262/#sec-promise-resolve)来替换resolvePrmise操作
![await-new-step-2](http://qiniu.enboest.com/20190702233929.png)

也就是将const promise = new Promise(res => res(p)) 替换成Promise.resolve(p)
根据[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/resolve)文档， 当 p 是一个 promise 时，Promise.resolve(p)直接返回 p，而这是大概率事件。

![/await-overhead-removed](http://qiniu.enboest.com/20190702234435.png)
通过这种方式，减少了两次tick的开销

并且v8团队继续对额外创建throwaway promise进行了优化
![node-10-vs-node-12](http://qiniu.enboest.com/20190702234736.png)

> throwaway Promise 只是为了满足 performPromiseThen 规范中内部操作的 API 约束。

因此又减少了一个promise的开销，虽然行为类似Node.js8 的bug，但背后的逻辑不一样。它现在是一个正在标准化的优化！

## 参考
[更快的异步函数和 Promise](https://v8.js.cn/blog/fast-async/##)
[v8是怎么实现更快的 await ？深入理解 await 的运行机制](https://zhuanlan.zhihu.com/p/53944576#)
[剖析Promise内部结构，一步一步实现一个完整的、能通过所有Test case的Promise类](https://github.com/xieranmaya/blog/issues/3)
[ECMAScript 标准 - Promise Objects](https://tc39.es/ecma262/#sec-promise-objects)
[Promises/A+](https://promisesaplus.com/)