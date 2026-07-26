---
title: 【iOS】NSOperation：状态机、依赖与自定义并发 Operation
published: 2026-07-27
description: 一个自定义并发 Operation 少发一条 KVO 通知，整条队列停摆。实测告诉你队列真正在听的是哪一个 key path，以及 Apple 文档在 cancel 这件事上写错了一句话。
tags:
  - iOS
  - NSOperation
  - 并发
  - KVO
  - GCD
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 15
draft: true
---
# NSOperation：状态机、依赖与自定义并发 Operation

下面这个类把一次异步网络请求包成 Operation。四个 getter 一个不缺，`isAsynchronous` 也返回了 `YES`，看起来该做的都做了。

```objc
@implementation BrokenAsyncOp {
    BOOL _executing;
    BOOL _finished;
}
- (BOOL)isAsynchronous { return YES; }
- (BOOL)isExecuting    { return _executing; }
- (BOOL)isFinished     { return _finished;  }

- (void)start {
    _executing = YES;
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [NSThread sleepForTimeInterval:0.2];
        self->_executing = NO;
        self->_finished  = YES;
    });
}
@end
```

把它和一个普通的 `NSBlockOperation` 一起丢进 `maxConcurrentOperationCount = 1` 的队列：

```text
  [Broken 1] start，_executing = YES
  [Broken 1] 工作完成，_finished = YES（但队列不知道）
  t=0.5s  b1.isFinished=1  b2.isFinished=0  queue.operationCount=2
  t=1.0s  b1.isFinished=1  b2.isFinished=0  queue.operationCount=2
  t=1.5s  b1.isFinished=1  b2.isFinished=0  queue.operationCount=2
```

`b1.isFinished` 明明是 1。队列的 `operationCount` 却纹丝不动，后面那个 block 永远没跑起来。程序不崩、不报错、没有任何日志，就是不干活了。

差的是两行 `willChangeValueForKey:` / `didChangeValueForKey:`。

这一篇就从这里展开。NSOperation 的 API 面很小，难点全部集中在它那套 KVO 驱动的状态机上，而这套状态机是不看文档就一定会写错的东西。GCD 那些坑至少会当场死锁给你看，这个不会。

前置知识在 [[iOS GCD：队列不是线程，以及死锁的准确边界]] 和 [[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]]。

---

## 一、队列到底在听哪一个 key path

Apple 文档在 `isFinished` 那一页写得相当明确：

> An operation object does not clear a dependency until the value at the `isFinished` key path changes to `true`. Similarly, an operation queue does not dequeue an operation until the `isFinished` property contains the value `true`. Thus, marking operations as finished is critical to keeping queues from backing up with in-progress or cancelled operations.

归档的 Concurrency Programming Guide 说得更狠：

> Even if an operation is canceled, you should always notify KVO observers that your operation is now finished with its work. ... Failing to generate a finish notification can therefore prevent the execution of other operations in your application.

文档同时要求你重写并通知 `isExecuting` 和 `isFinished` 两个属性。那么少发哪一个会出事？两个都必须发吗？这个问题文档没答，我做了个 2×2 的矩阵：一个自定义并发 Operation，四种通知组合，每组配一个依赖它的后继 op，1.5 秒超时判定队列是否排空。

```text
maxConc=-1  两个都不发        op._finished=1  队列排空=0  后继执行=0  残留 operationCount=2
maxConc=-1  只发 isExecuting  op._finished=1  队列排空=0  后继执行=0  残留 operationCount=2
maxConc=-1  只发 isFinished   op._finished=1  队列排空=1  后继执行=1  残留 operationCount=0
maxConc=-1  两个都发          op._finished=1  队列排空=1  后继执行=1  残留 operationCount=0

maxConc= 1  两个都不发        op._finished=1  队列排空=0  后继执行=0  残留 operationCount=2
maxConc= 1  只发 isExecuting  op._finished=1  队列排空=0  后继执行=0  残留 operationCount=2
maxConc= 1  只发 isFinished   op._finished=1  队列排空=1  后继执行=1  残留 operationCount=0
maxConc= 1  两个都发          op._finished=1  队列排空=1  后继执行=1  残留 operationCount=0
```

**`isFinished` 的 KVO 通知是队列唯一在听的那个信号，少了它队列就永远不会把这个 operation 出队。**

`isExecuting` 的通知在这组实验里完全不影响结果。我又单独测了一次并发限流：`maxConcurrentOperationCount = 2`，六个自定义 op，发不发 `isExecuting` 通知两种都跑，实测并发峰值都是 2。所以队列的限流计数也不依赖它。

那 `isExecuting` 还要不要发？要。它是对外的可观测状态，进度条、日志、上层业务的 KVO 观察都指着它；`NSOperationQueue` 现在不读，不代表以后不读。我自己的做法是照发不误，但心里清楚：排查"队列卡住"这类问题时，只需要盯 `isFinished` 一个 key path 就够。

### 精确一点：卡住的到底是什么

网上讲这个 bug 的文章基本都写成"后续 operation 全部卡住"。这句话只在并发上限为 1 的时候成立。

无上限的队列上，我加了一个不发通知的 op、一个和它毫无关系的 block op：

```text
  无关的 op 跑了吗 = 1（跑了）
  stuck._finished = 1，但 queue.operationCount = 1（永远降不到 0）
```

无关的 operation 照常执行。真正被卡死的是三样东西：队列的排空（`waitUntilAllOperationsAreFinished` 永不返回）、依赖这个 op 的所有后继、以及并发名额（上限为 1 时名额只有一个，于是表现为整条队列停摆）。

这个区别在排查现场很有用。App 里 NSOperationQueue 通常不止一个，"某个功能不响应但别的都正常"恰恰是这个 bug 的典型症状，而不是排除它的理由。

### 正确的写法

```objc
@implementation TWAsyncOperation {
    os_unfair_lock _lock;
    BOOL _rawExecuting;
    BOOL _rawFinished;
}

- (instancetype)init {
    if ((self = [super init])) { _lock = OS_UNFAIR_LOCK_INIT; }
    return self;
}

- (BOOL)isAsynchronous { return YES; }

- (BOOL)isExecuting {
    os_unfair_lock_lock(&_lock); BOOL v = _rawExecuting; os_unfair_lock_unlock(&_lock); return v;
}
- (BOOL)isFinished {
    os_unfair_lock_lock(&_lock); BOOL v = _rawFinished; os_unfair_lock_unlock(&_lock); return v;
}

- (void)start {
    if (self.isCancelled) { [self finish]; return; }

    [self willChangeValueForKey:@"isExecuting"];
    os_unfair_lock_lock(&_lock); _rawExecuting = YES; os_unfair_lock_unlock(&_lock);
    [self didChangeValueForKey:@"isExecuting"];

    [self execute];                  // 子类重写这里，不要重写 start / main
}

- (void)finish {
    os_unfair_lock_lock(&_lock);
    BOOL wasExecuting = _rawExecuting, alreadyDone = _rawFinished;
    os_unfair_lock_unlock(&_lock);
    if (alreadyDone) return;         // 幂等，重复调用无害

    if (wasExecuting) [self willChangeValueForKey:@"isExecuting"];
    [self willChangeValueForKey:@"isFinished"];
    os_unfair_lock_lock(&_lock);
    _rawExecuting = NO; _rawFinished = YES;
    os_unfair_lock_unlock(&_lock);
    [self didChangeValueForKey:@"isFinished"];
    if (wasExecuting) [self didChangeValueForKey:@"isExecuting"];
}
@end
```

几个决定值得说明。

`finish` 做成幂等的。异步回调最常见的事故就是成功路径和超时路径都调一次收尾，第二次 `didChangeValueForKey:@"isFinished"` 会让队列对一个已经出队的对象再记一次账。加个 `alreadyDone` 早退，比在每个调用点小心翼翼便宜得多。

`start` 里判 `isCancelled` 之后直接 `finish`，不走 `execute`。这是文档规定的：`start()` 必须自己处理"进来时已经被取消"的情况。

`start` 里一次 `super` 都不调。这条 Apple 在三个地方重复了三遍：

> A（Subclassing Notes）：At no time in your `start()` method should you ever call `super`.
>
> A（start() 独立页）：Your custom implementation must not call `super` at any time.
>
> C（Table 2-2）：Your implementation must not call super at any time.

原因不难想：`NSOperation` 默认的 `start()` 会更新执行状态、调 `main()`、然后把自己标记完成。你重写 `start` 就是为了接管这套流程，调 `super` 等于让两套状态机同时跑。

我把这个基类拿去跑了个像样点的场景：四个模拟异步网络请求（`dispatch_after` 0.1 秒后回调）扔进串行队列。

```text
=== 串行队列上的四个异步 op ===
  4 个各 0.1s 的异步请求，串行跑完耗时 0.43s
=== 同样四个，max=4 ===
  耗时 0.11s
```

0.43 秒和 0.11 秒，说明串行队列是真的等到上一个的异步回调回来才启动下一个。这正是自定义并发 Operation 存在的全部意义：把一个"函数已经返回但活儿没干完"的任务，重新变成队列能调度的单位。

### 反面：在 start 之前就置 finished

有一种写法流传得很广：在 `cancel` 里直接把 `isFinished` 置成 `YES`，理由是"取消了就赶紧结束，别占着队列"。

我按这个思路写了一版，队列挂起、入队、`cancel`、恢复：

```text
  EagerFinishOp.cancel：直接置 isFinished=YES（队列还没 start 过它）
2026-07-27 01:52:15.896 exp5[29094:15007613] *** EagerFinishOp 0x600003e10000
    went isFinished=YES without being started by the queue it is in
```

Foundation 直接打了告警。这条日志在 swift-corelibs-foundation 的 `Operation.swift` 里能找到原文，但那份是给 Linux / Windows 用的重写实现；上面这行来自 iOS 模拟器上 Apple 的闭源 Foundation，文案一字不差。两套实现在这个点上是对齐的。

告警本身不致命，队列这次也排空了。但它说明的问题是真的：operation 提前从依赖图里逃逸，队列的记账和你的状态不一致。取消的正确姿势是只置 `isCancelled` 标志（`cancel` 的默认实现已经做了），把 finish 留给 `start()`。

---

## 二、"底层是 GCD"这句话现在可以直接看栈

NSHipster 2014 年那篇说 NSOperationQueue 用 GCD 实现，"though that's a private implementation detail"。这句话今天过时了，Apple 自己在两页文档里都写明了：

> A（Operation 页）: An operation queue executes its operations either directly, by running them on secondary threads, or indirectly using the `libdispatch` library (also known as Grand Central Dispatch).
>
> B（OperationQueue 页）: Operation queues use the `Dispatch` framework to initiate the execution of their operations. As a result, queues always invoke operations on a separate thread, regardless of whether the operation is synchronous or asynchronous.

在 block 里 `backtrace()` 一下就能看到全貌。队列名设成 `MyProbeQueue`：

```text
dispatch label: MyProbeQueue (QOS: UNSPECIFIED)
thread: <NSThread: 0x60000170c000>{number = 2, name = (null)}
isMainThread: 0
0   exp2                 __main_block_invoke + 336
1   Foundation           __NSINDEXSET_IS_CALLING_OUT_TO_A_BOOL_BLOCK__ + 16
2   Foundation           -[NSBlockOperation main] + 100
3   Foundation           __NSOPERATION_IS_INVOKING_MAIN__ + 12
4   Foundation           -[NSOperation start] + 620
5   Foundation           __NSOPERATIONQUEUE_IS_STARTING_AN_OPERATION__ + 12
6   Foundation           __NSOQSchedule_f + 168
7   libdispatch.dylib    _dispatch_block_async_invoke2 + 104
8   libdispatch.dylib    _dispatch_client_callout + 16
9   libdispatch.dylib    _dispatch_continuation_pop + 804
10  libdispatch.dylib    _dispatch_async_redirect_invoke + 864
11  libdispatch.dylib    _dispatch_root_queue_drain + 364
12  libdispatch.dylib    _dispatch_worker_thread2 + 232
13  libsystem_pthread    _pthread_wqthread + 228
```

`__NSOQSchedule_f` 被 `_dispatch_block_async_invoke2` 拉起，线程来自 libdispatch 的全局 worker 池。`dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL)` 返回的正是我给 `NSOperationQueue` 设的 `name`，说明队列在内部建了一条同名的 dispatch queue。

那几个全大写的符号是 Apple 故意留的路标，让崩溃栈能一眼看出走到了哪一层。

### 一个把我坑了十分钟的符号

第 1 帧写着 `__NSINDEXSET_IS_CALLING_OUT_TO_A_BOOL_BLOCK__`。这里跑的是 `NSBlockOperation`，跟 `NSIndexSet` 半点关系没有。

我最初怀疑是 `backtrace_symbols` 越界读到了隔壁函数。查了一下不是。把模拟器 runtime 里的 Foundation 二进制拉出来 `nm -a`：

```text
00000000006b3730 t ___NSBACKGROUNDACTIVITYSCHEDULER_IS_CALLING_OUT_TO_SCHEDULED_BLOCK__
00000000006b3730 t ___NSBLOCKOPERATION_IS_CALLING_OUT_TO_A_BLOCK__
00000000006b3730 t ___NSINDEXSET_IS_CALLING_OUT_TO_A_BLOCK__
00000000006b3730 t ___NSINDEXSET_IS_CALLING_OUT_TO_A_BOOL_BLOCK__
00000000006b3730 t ___NSINDEXSET_IS_CALLING_OUT_TO_A_RANGE_BLOCK__
00000000006b3730 t ___NSXPCCONNECTION_IS_CALLING_OUT_TO_ERROR_BLOCK__
```

六个名字，同一个地址。反汇编看一眼就明白了：

```asm
00000000006b3730	stp	x29, x30, [sp, #-0x10]!
00000000006b3734	mov	x29, sp
00000000006b3738	ldr	x8, [x0, #0x10]      ; 取 block 的 invoke 指针
00000000006b373c	blr	x8                   ; 调它
00000000006b3740	ldp	x29, x30, [sp], #0x10
00000000006b3744	ret
```

六个"路标"的机器码完全一样，都是"取出 block 的 invoke 字段然后跳过去"，被链接器的 identical code folding 合并成了一份。地址只剩一个，`backtrace_symbols` 按地址反查时只能从这一堆同名地址的符号里随便挑一个，挑中的未必是当前真正走的那条路径。

所以崩溃日志里这一层的符号名不可信。好消息是 NSOperation 自己那三个路标我逐个查过：`__NSOPERATION_IS_INVOKING_MAIN__`、`__NSOPERATIONQUEUE_IS_STARTING_AN_OPERATION__`、`__NSOQSchedule_f`，各自地址上只有一个符号，没被折叠。看栈判断"这是不是 NSOperationQueue 调出来的"仍然靠得住。

这也是一次很典型的"先怀疑仪器"。我差点写成"NSBlockOperation 内部复用了 NSIndexSet 的枚举机制"，那就纯属编故事了。

### 两个池子是同一个

如果 NSOperationQueue 真的架在 libdispatch 上，它的线程上限就该和 GCD 全局队列一样。测法很直接：200 个各阻塞 0.3 秒的任务，分别喂给不限并发的 NSOperationQueue 和 GCD 全局并发队列，记并发峰值和用到的线程 ID 数。八核 Apple Silicon，跑七轮：

```text
NSOperationQueue  峰值 63 / 64 / 64 / 64 / 64 / 64 / 68     不同线程数 63~64
GCD 全局并发队列   峰值 64 / 64 / 64 / 64 / 64 / 66 / 68     不同线程数 64~65
```

两边都顶在 64 上，这是 libdispatch worker 线程的老熟人数字。偶尔冒出的 66、68 是线程 ID 被内核回收复用造成的计数偏差，不影响结论。

再回答一个常见误解：`NSOperationQueue` 不会因为它是"高级 API"就帮你避免线程爆炸。你不设 `maxConcurrentOperationCount`，它和裸用全局队列一样能把 64 条线程全占满。真要限流，就得自己写那个数字。

---

## 三、加进队列之后，isAsynchronous 就没人看了

这一条我认为是中文资料里错得最普遍的。流传的说法是"想让 operation 并发执行，就要重写 `isAsynchronous` 返回 YES"。

Apple 的原话是反的：

> A: When you add an operation to an operation queue, the queue ignores the value of the `isAsynchronous` property and always calls the `start()` method from a separate thread. Therefore, if you always run operations by adding them to an operation queue, there is no reason to make them asynchronous.
>
> C: The ability to define concurrent operations is only necessary in cases where you need to execute the operation asynchronously without adding it to an operation queue.

队列本来就在别的线程上调 `start()`。你写不写 `isAsynchronous` 它都并发。

**自定义并发 Operation 的必要性来自另一件事：默认实现在 `main()` 返回的那一刻就认为这个 operation 结束了。** 只要你的活儿是异步的（网络请求、`dispatch_after`、任何 delegate 回调），`main()` 一定会先返回。

```objc
@implementation NaiveOp
- (void)main {
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [NSThread sleepForTimeInterval:0.3];
        NSLog(@"后台工作真正结束");
    });
}
@end
```

```text
  [Naive] main 进入
  [Naive] main 返回
  队列已排空，a.isFinished=1（后台工作还没跑完）
  [Naive] 后台工作真正结束（此时 op 早已 finished）
```

队列排空了，活儿还没干完。依赖这个 op 的后继会提前触发，`maxConcurrentOperationCount` 的限流形同虚设，`completionBlock` 拿到的是半成品。这个 bug 比第一节那个隐蔽得多，因为它不卡死，只是偶尔出错。

所以判断标准很简单：任务在 `main()` 里能同步做完，就老老实实写 `main()`；做不完，才需要接管 `start()` 那一整套状态机。`isAsynchronous` 在整件事里只是个声明性的标记。

---

## 四、cancel 是加速依赖链

先看两段原文，它们描述的是同一件事的两个方向：

> A: The dependencies supported by `NSOperation` make no distinction about whether a dependent operation finished successfully or unsuccessfully. (In other words, canceling an operation similarly marks it as finished.) It is up to you to determine whether an operation with dependencies should proceed in cases where its dependent operations were cancelled.
>
> B: Canceling an operation causes the operation to ignore any dependencies it may have. This behavior makes it possible for the queue to invoke the operation's `start()` method as soon as possible.

**cancel 不切断依赖链，它加速依赖链。** 被取消的 op 会尽快走到 finished，后继因此提前 ready。

三态快照（队列先挂起，B 依赖 A，取消 A）：

```text
[t0] 入队、未启动
  A   ready=1 executing=0 cancelled=0 finished=0
  B   ready=0 executing=0 cancelled=0 finished=0
[t1] 刚 cancel A（队列仍挂起）
  A   ready=1 executing=0 cancelled=1 finished=0
  B   ready=0 executing=0 cancelled=0 finished=0
[t2] 队列跑完
  A   ready=1 executing=0 cancelled=1 finished=1
  B   ready=1 executing=0 cancelled=0 finished=1
  B 是否真的执行过 = 1
```

B 跑了。A 的 `main` 一次都没执行。

关键在 t1 那一行：A 已经 `cancelled=1`，但 `finished` 还是 0，B 也还没 ready。得等队列真的调过 A 的 `start()`、把它推到 finished，B 才解锁。这解释了为什么"取消整条链"这种需求不能靠 `cancelAllOperations` 一把梭——你取消的是执行，不是编排。想让后继跳过，只能在后继里自己查前驱状态。

反向的那条（cancel 让自己忽略依赖）也测得到。me 依赖 dep，取消 me：

```text
  cancel 前 me  ready=0 executing=0 cancelled=0 finished=0
  cancel 后 me  ready=1 executing=0 cancelled=1 finished=0
```

`isReady` 立刻从 0 翻到 1，dep 一步没动。

### cancel 不会打断已经在跑的代码

```objc
NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
    for (int i = 0; i < 10; i++) { [NSThread sleepForTimeInterval:0.05]; loops++; }
}];
[q addOperation:op];
[NSThread sleepForTimeInterval:0.12];
[op cancel];
```

```text
  cancel 已发出，此刻 isCancelled=1 isExecuting=1
  不检查 isCancelled 的循环跑满 10 次
  最终 loops=10（cancel 一次也没少跑）  finished=1
```

一次都没少。文档说得很清楚：

> This method does not force your operation code to stop. Instead, it updates the object's internal flags to reflect the change in state.

CPG 还解释了为什么不做成强制中断：

> If an operation were terminated outright, there might not be a way to reclaim resources that had been allocated.

改成每次迭代查一下：

```text
  第 3 次迭代检测到 cancel，提前退出
  loops=3 finished=1 cancelled=1
```

CPG 给的建议是在任何实际工作开始之前、循环每次迭代至少一次、以及任何容易中止的点上检查，并且明说 `isCancelled` 本身很轻量，频繁调用没有显著开销。我自己的阈值是：只要循环体单次耗时可能超过几十毫秒，就在循环顶部加一句。

`completionBlock` 在被取消时照样执行。这一点和文档一致（"a finished (or canceled) operation is still given a chance to execute its completion block"），也意味着 completionBlock 里必须自己判断 `isCancelled`，否则会拿着空结果往下走。

### 文档在这里写错了一句

Apple 的 `cancel()` 页面有这么一句：

> If you cancel an operation that is not in a queue, this method immediately marks the object as finished.

实测不成立。三种 operation 类型全试了一遍：

```text
  BlockOperation cancel 后          ready=1 executing=0 cancelled=1 finished=0
  BlockOperation 0.2s 后            ready=1 executing=0 cancelled=1 finished=0
  BlockOperation 手动 start 之后    ready=1 executing=0 cancelled=1 finished=1
  NSOperation 子类 cancel 后        ready=1 executing=0 cancelled=1 finished=0
  NSOperation 子类 手动 start 之后  ready=1 executing=0 cancelled=1 finished=1
  NSInvocationOperation cancel 后   ready=1 executing=0 cancelled=1 finished=0
```

**取消一个不在队列里的 operation，`isFinished` 保持 0，直到有人调用 `start()`。** 等了 0.2 秒也没变，不是时序问题。

在队列里的情况一样。队列挂起时取消一个排队中的 op，KVO 打出来的顺序是：

```text
  B.isCancelled=1  B.isReady=1  |cancel 返回|  |resume 前|  B.isFinished=1
```

`isCancelled` 和 `isReady` 是 `cancel` 同步给的，`isFinished` 要等队列恢复之后才来。三轮结果完全一致。

准确的表述应该是：cancel 只置标志并让 op 忽略依赖，`isFinished` 永远由 `start()` 负责推进。这跟第一节那条"不要在 cancel 里 finish"是同一个设计的两面。文档那句话大概是把"队列里的 op 被取消后很快会 finished"简化成了"immediately marks as finished"，但它把因果讲反了。

取消一个已经完成的 operation 什么都不会发生，连 `isCancelled` 都不会变成 1：

```text
  完成后 cancel  ready=1 executing=0 cancelled=0 finished=1
```

---

## 五、maxConcurrentOperationCount = 1 不是串行队列

这个说法在中文博客里几乎是标准答案。Apple 的原文其实早就否掉了：

> C: Passing a value of 1 to this method causes the queue to execute only one operation at a time. Although only one operation at a time may execute, the order of execution is still based on other factors, such as the readiness of each operation and its assigned priority. Thus, a serialized operation queue does not offer quite the same behavior as a serial dispatch queue in Grand Central Dispatch does.

八个等价 operation，`max = 1`，顺序确实是 FIFO：

```text
  0 1 2 3 4 5 6 7
```

给第 5、6 个设高优先级，第 1 个设 `VeryLow`，其余不动：

```text
  5 6 0 2 3 4 7 1
```

三轮跑下来一模一样。对照组是串行 dispatch queue，用 `dispatch_block_create_with_qos_class` 给第 5 个提到 `USER_INTERACTIVE`：

```text
  0 1 2 3 4 5 6 7
```

QoS 影响调度优先级，不影响出队顺序。串行 dispatch queue 的 FIFO 是语义保证，NSOperationQueue 的"看起来 FIFO"只是同优先级同时 ready 时的观察结果。文档对此的措辞是 "don't rely on queue semantics to ensure a specific execution order"。

**要顺序就用依赖，不要用 `max = 1` 加祈祷。**

### 优先级不能替代依赖

CPG 里这句话讲得很到位：

> Priority levels are not a substitute for dependencies. Priorities determine the order in which an operation queue starts executing only those operations that are currently ready.

一个实验就够：低优先级的 op 先入队，等它跑起来 0.1 秒后再加一个 `VeryHigh` 的。

```text
  low high
```

优先级只在"当前所有 ready 的 operation"这个集合里排序。已经开跑的，谁也插不了队。

### 一个 BlockOperation 里的多个 block 仍然并发

这个坑很值得单独拿出来。`max = 1` 的队列上，放一个 `NSBlockOperation`，往里塞六个 block：

```text
  6 个 block：并发峰值=6 用到线程数=6
```

六个全在并发跑。文档写得明白：

> The `BlockOperation` class ... manages the concurrent execution of one or more blocks. ... When executing more than one block, the operation itself is considered finished only when all blocks have finished executing.

CPG 补了实现细节：这些 block 被提交到默认优先级的并发 dispatch queue。

`maxConcurrentOperationCount` 限制的是 operation 之间，管不到一个 BlockOperation 内部。把队列当串行用、又图省事往一个 BlockOperation 里塞多个 block，得到的是一个伪装成串行的并发执行。

补一句：operation 开跑之后就不能再加 block 了。

```text
  *** -[NSBlockOperation addExecutionBlock:]: blocks cannot be added after
      the operation has started executing or finished
```

---

## 六、状态机的其余部分

### 通知的先后顺序

给一条 A→B 的依赖链装上 KVO 观察，跑五轮，顺序完全稳定：

```text
A.isExecuting=1  B.isReady=1  A.isExecuting=0  A.isFinished=1  B.isExecuting=1  B.isExecuting=0  B.isFinished=1
```

有意思的是 `B.isReady=1` 排在 `A.isFinished=1` 前面。乍看像是"A 还没完成，B 就 ready 了"。我在 B 的 isReady 观察者里回读了 A 的状态：

```text
  收到 B.isReady=1 的那一刻：A.isFinished=1 A.isExecuting=0
```

值早就变了，晚到的只是 A 自己那份通知。也就是说 Foundation 是先落状态、再解锁后继、最后才广播给 A 的外部观察者。做时序判断时读属性值，别读通知顺序。

另外注意 A 从入队到执行的整个过程里没有任何 `isReady` 通知。它入队时就是 ready 的，KVO 不发无变化的通知。想靠观察 `isReady` 来做"什么时候轮到我"的逻辑，会漏掉一大类情况。

### 重写 isReady 要并进 super

nsprogrammer 那篇 2021 年的文章里有一条结论是"永远不要 override `isReady`，否则依赖图失效"。这跟 Apple 文档直接冲突：

> If you want to use custom conditions to define the readiness of your operation object, reimplement this property... your custom implementation must get the default property value from `super` and incorporate that readiness value into the new value of the property.

失效的不是重写本身，是漏掉 `super`。测一下就清楚了。

```objc
// 错误
- (BOOL)isReady { return self.gate; }
// 正确
- (BOOL)isReady { return [super isReady] && self.gate; }
```

```text
=== 重写 isReady 不并进 super：依赖被绕过 ===
  入队前：bad.isReady=1（依赖还没跑，却已经 ready）
    前驱开始
    BadReadyOp 执行（前驱是否已完成，它不关心）
    前驱结束

=== 并进 super 的版本 ===
  入队前：good.isReady=0
    前驱开始
    前驱结束
    GoodReadyOp 执行
```

不过那篇文章的工程建议我是认同的：能用一个 operation 当依赖表达的条件，就别去改 `isReady`。自定义 readiness 意味着你要自己负责在条件满足时发 `isReady` 的 KVO 通知，忘了发，op 就永远躺在队列里。

```text
  isReady=0 operationCount=1
  （手动 gate = YES 之后）
  打开闸门后 operationCount=0
```

### 循环依赖不会冻结程序

`addDependency:` 那页写着：

> It is a programmer error to create any circular dependencies among a set of operations. Doing so can cause a `deadlock` among the operations and may freeze your program.

CPG 的版本更准确：

> Doing so is a programmer error that will prevent the affected operations from ever running.

实测站在 CPG 这边。x 和 y 互相依赖，入队，主线程照常往下跑：

```text
  0.5s 后主线程还活着。x.ready=0 y.ready=0 operationCount=2
  别的队列照常工作 ran=1
```

没有 deadlock，没有 freeze。那两个 operation 的 `isReady` 永远是 0，卡在队列里。危害是资源泄漏加功能静默失效，不是崩溃。这类问题在真机上的表现就是"某个页面的数据永远转圈"，查起来相当费劲。

时序上还有一条坑：

> You should always configure dependencies before running your operations or adding them to an operation queue. Dependencies added afterward may not prevent a given operation object from running.

"may not prevent"是很客气的说法。实测入队 0.05 秒后再 `addDependency:`，既不抛异常也不生效：

```text
  late 执行
  入队之后 addDependency：没有抛异常
  slow 完成
```

`late` 已经跑完了才轮到那行 `addDependency:`。这种代码在开发机上可能一直是对的，直到某天设备卡一下、顺序就变了。

### underlyingQueue 的三条约束，只有一条是强制的

文档给了三条限制，我逐条试了：

```text
  设成 main queue：没抛异常，label=com.apple.main-thread
  队列非空时设置：NSInvalidArgumentException:
      *** -[NSOperationQueue setUnderlyingQueue:]: operation queue must be empty
          in order to change underlying dispatch queue
  给 mainQueue 设 underlyingQueue：没抛异常
```

只有"非空时设置"会抛异常。设成主队列是契约性禁令，运行时不拦；给 `NSOperationQueue.mainQueue` 赋值也不抛，但读回来还是 `com.apple.main-thread`，赋值被静默忽略了。很多博客写"设成 main queue 会崩溃"，实测不会。

有个连带效果值得知道：

```text
  underlyingQueue=串行 + max=8：实测并发峰值=1（底层队列说了算）
```

底层是串行 dispatch queue 时，`maxConcurrentOperationCount = 8` 就是一句空话。这倒是个可用的技巧。想要一个真·串行的 operation queue，它比设 `max = 1` 靠谱。

几个能直接读到的数字，一并记在这里：

```text
defaultMaxConcurrentOperationCount = -1
new queue maxConcurrentOperationCount = -1
mainQueue.maxConcurrentOperationCount = 1
mainQueue.underlyingQueue label = com.apple.main-thread
新建队列 underlyingQueue = (null)
```

`NSOperationQueueDefaultMaxConcurrentOperationCount` 在头文件里就是个常量：

```objc
static const NSInteger NSOperationQueueDefaultMaxConcurrentOperationCount = -1;
```

文档说"The operation queue determines this number dynamically based on current system conditions"，容易让人以为打印它能看到系统给了几个并发名额。打印出来永远是 -1。真正的动态限流发生在 libdispatch 那一层。

### waitUntilFinished 的自等

文档：

> An operation object must never call this method on itself and should avoid calling it on any operations submitted to the same operation queue as itself. Doing so can cause the operation to deadlock.

`max = 1` 的队列上，op1 里等一个排在它后面的 op：

```text
  op1 开始等 later（同一队列，max=1）
  1s 后 returned=0 later.isFinished=0 operationCount=2
```

死锁成立。op1 占着唯一的并发名额等 later，later 等 op1 让出名额。跟 GCD 里 `dispatch_sync` 到当前串行队列是同一类问题，区别是这里不会 trap，只是静静地挂住。挂住比崩溃难查得多。

---

## 七、2026 年还该不该用它

社区里"Apple 已经废弃 GCD 和 OperationQueue"的说法传得很广。我查了头文件，`NSOperation` / `NSBlockOperation` / `NSOperationQueue` 三个类全平台 `deprecated: false`，一个弃用标记都没有。这是社区推论，不是 Apple 原话。

Apple 的真实态度全写在 API 标注里。整个 `NSOperation.h` 包在 `NS_HEADER_AUDIT_BEGIN(nullability, sendability)` 里，三个类都标了 `NS_SWIFT_SENDABLE`，说明 Apple 主动为 Swift 6 严格并发做了适配。同时：

```objc
- (void)waitUntilFinished
    NS_SWIFT_UNAVAILABLE_FROM_ASYNC("Use completionBlock or a dependent Operation instead");
- (void)waitUntilAllOperationsAreFinished
    NS_SWIFT_UNAVAILABLE_FROM_ASYNC("Use addBarrierBlock or a dependent Operations instead");
- (void)addOperationWithBlock:(void (NS_SWIFT_SENDABLE ^)(void))block NS_SWIFT_DISABLE_ASYNC;
```

阻塞式等待在 async 上下文里编译不过，并且直接给了替代方案。准确表述是：类不弃用、Sendable 已适配，但两套并发模型的等待原语不通用。

`addBarrierBlock:` 和 `progress` 是 macOS 10.15 / iOS 13 才加进来的，Apple 显然还在往里加东西。barrier 挺好用：

```text
  a3 a2 a1 a0 |barrier| b1 b0 b2
```

前四个乱序完成，barrier 等它们全干完才跑，之后的三个再放开。

另外 `operations` 和 `operationCount` 已经在头文件里标了 `API_DEPRECATED`（网页文档没标）：

```objc
API_DEPRECATED("access to operations is inherently a race condition, it should not be used. "
               "For barrier style behaviors please use addBarrierBlock: instead", ...)
```

我这篇里到处在用 `operationCount` 做观测，那是实验代码。生产代码里别读它。

### 能不能用 async/await 实现 NSOperation

nsprogrammer 那篇文章原本写着 "You can implement your `NSOperation` using `async await` without issue!"，后来作者自己划掉加了 Update：

> I actually spent a good amount of time on this and can't figure out a way to implement `NSOperation` using `async await`… It is simple enough to use `NSOperation` from async contexts, but I could not get the KVO inside `NSOperation` to work using the new concurrency features of Swift 🤷‍♂️.

这段自打脸配上 `NS_SWIFT_UNAVAILABLE_FROM_ASYNC` 一起看，指向同一个结论：NSOperation 的状态机是 KVO 驱动的，KVO 是同步通知模型，跟 actor 隔离和结构化并发缝不到一起。

### 我的选型判断

新代码优先 Swift Concurrency。`TaskGroup` 能表达大部分并发编排，取消是结构化传播的，不用手写状态机。

仍然选 NSOperation 的场景，我认为只剩三类：需要复杂依赖图（一个 `addDependency:` 顶一堆手写同步）、需要运行时改并发度和优先级、以及大量存量 ObjC 代码里要塞一个可取消的异步任务。前两类 GCD 做起来相当难受，这是 NSOperation 至今没被替代的核心价值。

至于自定义并发 Operation——如果你的代码库里已经有一个写对了的基类，继续用；如果没有，我的判断是不该为了这一个功能新写一个。第一节那套东西看着不复杂，但线程安全、幂等、取消时序三样凑齐才算写对，而写错了不报错。

---

## 八、几个已经不准的说法

- "KVO 清单里有 `isConcurrent`"。CPG 那份是 2012 年的，现行清单是 `isCancelled` / `isAsynchronous` / `isExecuting` / `isFinished` / `isReady` / `dependencies` / `queuePriority` / `completionBlock`。头文件里 `concurrent` 属性的注释写着 "To be deprecated; use and override 'asynchronous' below"。
- "要让 operation 并发，得重写 `isAsynchronous` 返回 YES"。加进队列后这个值被忽略，队列一直是在别的线程调 `start()`。
- "NSOperationQueue 底层是 GCD，但那是私有实现细节"。2014 年成立，现在 Apple 两页文档都白纸黑字写了 `libdispatch`。
- "`defaultMaxConcurrentOperationCount` 是系统动态算出来的数字"。头文件里是常量 `-1`，打印永远是 -1。
- "`underlyingQueue` 设成 main queue 会崩溃"。实测不抛异常。真正强制的只有"队列非空时不能改"。
- "取消一个不在队列里的 operation 会立即把它标记为 finished"。这是 Apple 自己文档里的话，实测三种 operation 类型全都保持 `isFinished = 0`，直到有人调 `start()`。
- "循环依赖会 freeze your program"。夸张了。主线程照常跑，其他队列照常工作，只是那几个 op 永远 ready 不了。
- "永远不要 override `isReady`"。与文档冲突。文档要求的是并进 `super` 的值，不是禁止重写。
- "Apple 已经废弃了 GCD 和 OperationQueue"。三个类全平台 `deprecated: false`。被标 deprecated 的是 `threadPriority`（换 `qualityOfService`）和 `operations` / `operationCount`。

---

## 总结

`isFinished` 的 KVO 通知是 NSOperationQueue 唯一在听的信号。少发它，队列的 `operationCount` 永远降不到 0；上限为 1 的队列会整条停摆，不限并发的队列则是排空和后继被卡住。`isExecuting` 的通知实测不影响队列的任何行为，但该发还是要发。

自定义并发 Operation 存在的理由是"`main()` 一返回队列就认为结束了"。`isAsynchronous` 在队列场景下是个被忽略的标记。

cancel 只置标志、让 op 忽略自己的依赖，`isFinished` 永远由 `start()` 推进。它加速依赖链而非切断，后继照跑。Apple 文档里"cancel a non-queued operation immediately marks it as finished"这句实测不成立。

`maxConcurrentOperationCount = 1` 和串行 dispatch queue 不等价：优先级会重排出队顺序，一个 BlockOperation 内部的多个 block 照样并发。

这篇里所有推翻既有说法的结论，都是几十行程序跑出来的。NSOperation 的行为面全部可观测——状态属性能读、KVO 能挂、调用栈能抓、异常文案能捕获。碰到"文档这么说但我不确定"的时候，写个实验比读第十篇博客快。

下一篇 [[iOS 锁：从 OSSpinLock 的废弃说起]]。

## 参考资料

### 一手

- `$(xcrun --sdk iphonesimulator --show-sdk-path)/System/Library/Frameworks/Foundation.framework/Headers/NSOperation.h`：`NSOperationQueueDefaultMaxConcurrentOperationCount = -1`、`NS_SWIFT_SENDABLE`、`NS_SWIFT_UNAVAILABLE_FROM_ASYNC`、`operations` 的弃用文案，全部出自这里
- [Operation](https://developer.apple.com/documentation/foundation/operation)：`isFinished` 那段"critical to keeping queues from backing up"、`start()` 不许调 super、依赖不区分成败
- [OperationQueue](https://developer.apple.com/documentation/foundation/operationqueue)：明写用 `Dispatch` framework、"don't rely on queue semantics"
- [Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html)（归档，页脚 Updated: 2012-12-13，已停更）：串行 operation queue 与串行 dispatch queue 的差异、优先级不能替代依赖、循环依赖的准确描述。这三条现行文档都没有更好的版本

### 二手

- [NSHipster — NSOperation](https://nshipster.com/nsoperation/)：状态机的经典讲法；"private implementation detail" 那句已过时
- nsprogrammer《NSOperation》（2021-07）：结论有对有错，但作者自己加的那段 Update 很诚实，值得一读
- [swift-corelibs-foundation `Operation.swift`](https://github.com/swiftlang/swift-corelibs-foundation)：Linux / Windows 的重写实现，不是 iOS 上跑的代码。可以当"照着闭源复刻"的参考，"went isFinished=YES without being started" 这条诊断在两边一字不差

### 本地

- [[iOS GCD：队列不是线程，以及死锁的准确边界]]：worker 线程上限、串行队列的 FIFO 语义、死锁边界
- [[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]]：`willChangeValueForKey:` / `didChangeValueForKey:` 到底做了什么
- [[iOS 锁：从 OSSpinLock 的废弃说起]]

---

实验环境：Xcode 26.6（Apple clang 21），iOS 模拟器 iPhone 16 / iOS 18.3（arm64，8 核 Apple Silicon Mac），`clang -target arm64-apple-ios17.0-simulator`，ARC。符号表实验用的是模拟器 runtime 里的 `Foundation` 二进制（`nm -a` / `otool -tV`）。

判定"队列卡住"的实验一律用轮询 `operationCount` 加超时，不用 `waitUntilAllOperationsAreFinished`。理由很实际：卡住的时候它不返回，终端就废了。

> 待真机补测：第二节的线程上限数字（模拟器和真机的 libdispatch worker 上限可能不同），以及第四节那组 cancel 三态快照在 iOS 26.5 真机上是否一致。代码原样拿到真机跑即可。
