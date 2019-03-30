---
 title: Node 异步 I/O
 tags: 
    - JavaScript
    - node
 author:
    nick: Lovae
    
 cover: /images/async/async_pic.png
 date: 2018-3-17
  
 subtitle:
    图解 Node 异步 I/O，事件循环机制
 categories: JavaScript

---
这篇文章主要讲 nodejs 中的异步 IO，关于同步、异步、阻塞、非阻塞 请移步[这里](https://nodejs.org/en/docs/guides/blocking-vs-non-blocking/)。

### 事件循环 和 消息队列

我们常说“JavaScript是单线程的”。

所谓单线程，是指在JS引擎中负责解释和执行JavaScript代码的线程只有一个。不妨叫它主线程。

但是实际上还存在其他的线程。例如：处理AJAX请求的线程、定时器线程、读写文件的线程等等。这些线程可能存在于JS引擎之内，也可能存在于JS引擎之外，在此我们不做区分。不妨叫它们工作线程。

![](/images/async/async_pic.png)

#### node 执行过程

![](/images/async/node_event.png)

处理并执行完 js 代码，main函数继续往下调用libuv的事件循环入口uv_run()，node进程进入事件循环。 `uv_run()` 的 while 循环做的就是一件事，判断 `default_loop_struct` 是否有存活的 io 观察者 或 定时器。

#### 事件循环

> 事件循环是指主线程重复从消息队列中取消息、执行的过程

事件循环对应上图 3 号标注的部分。用代码表示大概是这样的：

```Js
        while(true) {
            var message = queue.get();
            execute(message);
        }
```

![](/images/async/event_loop.png)

如上图，每一次执行一次循环体的过程称为 Tick。

**事件循环的阶段：**

```
   ┌───────────────────────┐
┌─>│        timers         │ 执行定时器(setTimeout/setInterval)注册的回调函数，也是进入事
│  └──────────┬────────────┘ 件循环第一个阶段。
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │ I/O 事件相关联的回调或者报错会在这里执行
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │ 内部使用，不讨论
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │ 最重要的一个阶段，I/O 观察者观察到线程池
│  │         poll          │<─────┤  connections, │	里有任务已经完成，就会在这里执行回调。
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │ 专门用来执行 setImmediate() 的回调
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │ 一个连接或 handle 突然被关闭，close 事件会被发送到这里执行回调
   └───────────────────────┘
```

如上图，共有六个阶段（官方称为 phase）。特别要说明的是 poll 阶段，在这个阶段，如果暂时没有事件到来，主线程便会阻塞在这里，等待事件发生。当然它不会一直等下去：

* 它首先会判断后面的 Check Phase 以及 Close Phase 是否还有等待处理的回调. 如果有, 则不等待, 直接进入下一个 Phase. 
* 如果没有其他回调等待执行, 它会给 epoll 这样的方法设置一个 timeout. 可以猜一下, 这个 timeout 设置为多少合适呢? 答案就是 Timer Phase 中最近要执行的回调启动时间到现在的差值, 假设这个差值是 delta. 因为 Poll Phase 后面没有等待执行的回调了. 所以这里最多等待 delta 时长, 如果期间有事件唤醒了消息循环, 那么就继续下一个 Phase 的工作; 如果期间什么都没发生, 那么到了 timeout 后, 消息循环依然要进入后面的Phase, 让下一个迭代的 Timer Phase 也能够得到执行.

来看一下流程：

![](/images/async/phases.png)

到这里你一定发现少了一些问题：process.nextTick() 和 Promise 都是异步的，它们对应以上哪个阶段呢？往下看 

#### 任务队列

```
1、运行主线程（函数调用栈）中的同步任务
2、主线程（函数调用栈）执行到任务源时，通知相应的webAPIs进行相应的执行异步任务，将任务源指定的异步任务放入任务队列中
3、主线程（函数调用栈）中的任务执行完毕后，然后执行所有的微任务，再执行宏任务，找到一个任务队列执行完毕，再执行所有的微任务
4、不断执行第三步
```

>  任务队列也叫消息队列。主要分两类任务：宏任务(macro-task)、微任务(micro-task)

宏任务：setTimeout	setInterval	setImmediate	I/O

微任务：process.nextTick	Promise 的回调

在上面的图中，各个 phase 完成了宏任务对应的事件。微任务的执行时机在**每一次进入下一个阶段之前**，process.nextTick	优先级大于 Promise 的回调。

#### FAQ

##### setTimeout 和 setImmediate 的比较

```js
setImmediate(() => console.log(2))
setTimeout(() => console.log(1))
```

这段代码的结果实际上是不确定的。可是，为什么？按照流程图，应该是 timer 先于 check 阶段，所以应该是 setTimeout 先执行，可是为什么结果不是这样呢？首先我们要知道：

```js
setTimeout(fn) ==> setTimeout(fn, 0) ==> setTimeout(fn, 1)
```

上面三个效果是一样的！前两个好理解，给定的默认值是0。其实在 node 源码中，最低为 1 ms，官方文档如下：

```
When delay is larger than 2147483647 or less than 1, the delay will be set to 1.
```

所以当进入 timer 阶段时，1ms 可能超时也可能没有，这个影响因素有很多。如果还没超时，则进入下一个 phase，依次往下，所以先输出 2 。如果已经超时，则先输出 1。

> 但是！如果它们在 I/O 事件回调中，那么输出顺序是固定了的，如下

```js
require('fs').readFile('path.txt', () => {
 setImmediate(() => console.log(2))
 setTimeout(() => console.log(1))
});
// 输出: 2 1
```

如果不知道为什么，答案就在循环图中。

(完)



### Reference

* [The Node.js Event Loop, Timers, and `process.nextTick()`](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
