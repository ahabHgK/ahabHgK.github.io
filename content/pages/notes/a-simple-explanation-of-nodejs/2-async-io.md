---
title: 异步 I/O
slug: /notes/a-simple-explanation-of-rxjs/async-io
date: 2020-10-21
description: 《深入浅出 NodeJS》读书笔记
tags:
  - Note
  - A Simple Explanation of NodeJS
---

为什么要异步 I/O？

1. 用户体验
2. 资源分配
    - 单线程串行：阻塞，CPU 利用效率低
    - 多线程并行：创建线程和执行期线程上下文切换开销较大；锁、状态同步等问题
    - 异步 I/O：利用单线程远离锁、状态同步等问题；同时让单线程远离阻塞

阻塞/非阻塞与同步/异步（[怎样理解阻塞非阻塞与同步异步的区别？- 知乎](https://www.zhihu.com/question/19732473/answer/20851256)）

- 操作系统对于 I/O 只有两种方式：阻塞和非阻塞
  - 阻塞 I/O 调用之后要等系统内核完成所有操作后，调用才结束
  - 非阻塞 I/O 调用之后会立即返回
    - OS 将计算机所有输入输出设备抽象为文件，内核在 I/O 操作时通过文件描述符进行管理，文件描述符类似于应用程序与系统内核之间的凭证
    - 非阻塞 I/O 返回之后，CPU 可以用来处理其他事务
    - 完整 I/O 没有完成，业务层为了得到完整的数据，需要进行轮询（poll、epoll）
- 理想的非阻塞异步 I/O：在 I/O 完成后通过信号或回调将数据传递给应用程序
- 现实的异步 I/O：让部分线程进行阻塞 I/O 或非阻塞 I/O + 轮询来完成数据获取，让主线程进行数据处理，通过线程之间的通信传递 I/O 得到的数据，模拟异步 I/O

    ![现实中的异步 I/O](./images/async-io.png)

## Node 的异步 I/O

![基于 libuv 的架构图](./images/node-libuv.png)

libuv 在 *uix 和 Windows 上都是通过线程池实现的异步 I/O

```text
lib/fs.js          ( fs.open()    )
src/node_file.cc   ( Open         )
deps/uv/unix/fs.c  ( uv_fs_open() )
```

1. uv_fs_open 调用过程中会创建一个请求对象，把传入的 callback 放到这个请求对象上
2. 组装好请求对象后会放入 I/O 线程池中
3. 线程池中的 I/O 完毕后，把结果放到请求对象上，并通过 I/O 观察者（每次 Tick 会检查线程池中是否有执行完的请求）把请求对象放到任务队列中，将其当作事件处理

```text
   ┌───────────────────────────┐
┌─>│           timers          │  (setTimeout, setInterval)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  (I/O callbacks deferred to the next loop)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  (only use internally, process.nextTick)
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │  (retrieve new I/O events; execute I/O related callbacks (almost all with the exception of close callbacks, the ones scheduled by timers, and setImmediate()); node will **block** here when appropriate)
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │  (setImmediate)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  (some close callbacks)
   └───────────────────────────┘
```

**每个阶段都有一个 FIFO 队列**来执行回调

```cpp
// [uv_run 事件循环源码](https://github.com/nodejs/node/blob/master/deps/uv/src/unix/core.c#L365)
while (r != 0 && loop->stop_flag == 0) {
  uv__update_time(loop);
  uv__run_timers(loop);
  ran_pending = uv__run_pending(loop);
  uv__run_idle(loop);
  uv__run_prepare(loop);

  timeout = 0;
  if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
    timeout = uv_backend_timeout(loop);

  uv__io_poll(loop, timeout);
  
  // ...

  uv__run_check(loop);
  uv__run_closing_handles(loop);
  
  // ...
}
```

uv__io_poll 之前会调用 uv_backend_timeout 回去 timeout，uv_backend_timeout 中检测 uv__has_active_handles、uv__has_active_reqs、idle_handles、pending_queue、closing_handles 都没有就返回 0，否则 uv__next_timeout 获取 timeout

uv__next_timeout 中获取**距离此时此刻最先到期的一个 timer 的时间**，如果大于根据系统设置的 INT_MAX 就返回 INT_MAX，否则返回这个相差的时间

之后 uv__io_poll 调用时获得这个 timeout（有被注册的任务没有完成的时存在），uv__io_poll 执行时会阻塞 timeout 时间等待这段时间内会不会有任务完成，当 timeout 是最先到期的一个 timer 的时间时，就会在这段时间完成这个 timer，直接执行 timer 的回调函数（实现一旦轮询队列为空，事件循环将查看**已达到时间阈值的计时器**。如果一个或多个计时器已准备就绪，则事件循环将绕回计时器阶段以执行这些计时器的回调）

## process.nextTick 和 setImmediate 区别

nextTick 的回调函数保存在数组上，setImmediate 的回调函数保存在链表上，nextTick 会在每轮循环将数组中所有的回调全部执行完，setImmediate 每轮循环只执行链表中的一个回调函数

nextTick 在 idle 阶段，setImmediate 在 check 阶段，优先级 idle > poll > check

实际上这两个名字换一下更合适，但这是过去遗留问题不能随意改变

## 浏览器中的事件循环



## ref

[The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/zh-cn/docs/guides/event-loop-timers-and-nexttick/)

[从 libuv 看 nodejs 事件循环](https://set.sh/post/200317-how-nodejs-event-loop-works)