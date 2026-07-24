# GCD
## 2026-05-14 23:33 GCD 同步执行主队列死锁分析
**来源**: 博客　**confidence**: 0.90

- 在主线程中调用同步执行 + 主队列会导致死锁：任务与当前任务互相等待。
- 在其他线程中调用同步执行 + 主队列不会死锁，任务在主线程顺序执行。
- 异步执行 + 主队列只在主线程中顺序执行任务，不开启新线程。
- 主队列是串行队列，任务按顺序执行。
- 关键 API：dispatch_sync、dispatch_async、dispatch_get_main_queue。

<details><summary>原文</summary>

4.5 同步执行 + 主队列
同步执行 + 主队列 在不同线程中调用结果也是不一样，在主线程中调用会发生死锁问题，而在其他线程中调用则不会。

4.5.1 在主线程中调用 「同步执行 + 主队列」
互相等待卡住不可行
/**
* 同步执行 + 主队列
* 特点(主线程调用)：互等卡主不执行。
* 特点(其他线程调用)：不会开启新线程，执行完一个任务，再执行下一个任务。
*/
- (void)syncMain {
NSLog(@"currentThread---%@",[NSThread currentThread]); // 打印当前线程
NSLog(@"syncMain---begin");
dispatch_queue_t queue = dispatch_get_main_queue();
dispatch_sync(queue, ^{
// 追加任务 1
[NSThread sleepForTimeInterval:2]; // 模拟耗时操作
NSLog(@"1---%@",[NSThread currentThread]); // 打印当前线程
});
dispatch_sync(queue, ^{
// 追加任务 2
[NSThread sleepForTimeInterval:2]; // 模拟耗时操作
NSLog(@"2---%@",[NSThread currentThread]); // 打印当前线程
});
dispatch_sync(queue, ^{
// 追加任务 3
[NSThread sleepForTimeInterval:2]; // 模拟耗时操作
NSLog(@"3---%@",[NSThread currentThread]); // 打印当前线程
});
NSLog(@"syncMain---end");
}
输出结果
2019-08-08 14:43:58.062376+0800 YSC-GCD-demo[17371:4213562] currentThread---<NSThread: 0x6000026e2940>{number = 1, name = main}
2019-08-08 14:43:58.062518+0800 YSC-GCD-demo[17371:4213562] syncMain---begin
(lldb)

在主线程中使用 同步执行 + 主队列 可以惊奇的发现：

追加到主线程的任务 1、任务 2、任务 3 都不再执行了，而且 syncMain---end 也没有打印，在 XCode 9 及以上版本上还会直接报崩溃。这是为什么呢？
这是因为我们在主线程中执行 syncMain 方法，相当于把 syncMain 任务放到了主线程的队列中。而 同步执行 会等待当前队列中的任务执行完毕，才会接着执行。那么当我们把 任务 1 追加到主队列中，任务 1 就在等待主线程处理完 syncMain 任务。而syncMain 任务需要等待 任务 1 执行完毕，才能接着执行。

那么，现在的情况就是 syncMain 任务和 任务 1 都在等对方执行完毕。这样大家互相等待，所以就卡住了，所以我们的任务执行不了，而且 syncMain---end 也没有打印。

要是如果不在主线程中调用，而在其他线程中调用会如何呢？

4.5.2 在其他线程中调用「同步执行 + 主队列」
不会开启新线程，执行完一个任务，再执行下一个任务
// 使用 NSThread 的 detachNewThreadSelector 方法会创建线程，并自动启动线程执行 selector 任务
[NSThread detachNewThreadSelector:@selector(syncMain) toTarget:self withObject:nil];
输出结果：
2019-08-08 14:51:38.137978+0800 YSC-GCD-demo[17482:4237818] currentThread---<NSThread: 0x600001dd6c00>{number = 3, name = (null)}
2019-08-08 14:51:38.138159+0800 YSC-GCD-demo[17482:4237818] syncMain---begin
2019-08-08 14:51:40.149065+0800 YSC-GCD-demo[17482:4237594] 1---<NSThread: 0x600001d8d380>{number = 1, name = main}
2019-08-08 14:51:42.151104+0800 YSC-GCD-demo[17482:4237594] 2---<NSThread: 0x600001d8d380>{number = 1, name = main}
2019-08-08 14:51:44.152583+0800 YSC-GCD-demo[17482:4237594] 3---<NSThread: 0x600001d8d380>{number = 1, name = main}
2019-08-08 14:51:44.152767+0800 YSC-GCD-demo[17482:4237818] syncMain---end

在其他线程中使用 同步执行 + 主队列 可看到：

所有任务都是在主线程（非当前线程）中执行的，没有开启新的线程（所有放在主队列中的任务，都会放到主线程中执行）。
所有任务都在打印的 syncConcurrent---begin 和 syncConcurrent---end 之间执行（同步任务 需要等待队列的任务执行结束）。
任务是按顺序执行的（主队列是 串行队列，每次只有一个任务被执行，任务一个接一个按顺序执行）。
为什么现在就不会卡住了呢？

因为syncMain 任务 放到了其他线程里，而 任务 1、任务 2、任务3 都在追加到主队列中，这三个任务都会在主线程中执行。syncMain 任务 在其他线程中执行到追加 任务 1 到主队列中，因为主队列现在没有正在执行的任务，所以，会直接执行主队列的 任务1，等 任务1 执行完毕，再接着执行 任务 2、任务 3。所以这里不会卡住线程，也就不会造成死锁问题。

4.6 异步执行 + 主队列
只在主线程中执行任务，执行完一个任务，再执行下一个任务。
/**
* 异步执行 + 主队列
* 特点：只在主线程中执行任务，执行完一个任务，再执行下一个任务
*/
- (void)asyncMain {
NSLog(@"currentThread---%@",[NSThread currentThread]); // 打印当前线程
NSLog(@"asyncMain---begin");
dispatch_queue_t queue = dispatch_get_main_queue();
dispatch_async(queue, ^{
// 追加任务 1
[NSThread sleepForTimeInterval:2]; // 模拟耗时操作
NSLog(@"1---%@",[NSThread currentThread]); // 打印当前线程
});
dispatch_async(queue, ^{
// 追加任务 2
[NSThread sleepForTimeInterval:2]; // 模拟耗时操作
NSLog(@"2---%@",[NSThread currentThread]); // 打印当前线程
});
dispatch_async(queue, ^{
// 追加任务 3
[NSThread sleepForTimeInterval:2]; // 模拟耗时操作
NSLog(@"3---%@",[NSThread currentThread]); // 打印当前线程
});
NSLog(@"asyncMain---end");
}
输出结果：
2019-08-08 14:53:27.023091+0800 YSC-GCD-demo[17521:4243690] currentThread---<NSThread: 0x6000022a1380>{number = 1, name = main}
2019-08-08 14:53:27.023247+0800 YSC-GCD-demo[17521:4243690] asyncMain---begin
2019-08-08 14:53:27.023399+0800 YSC-GCD-demo[17521:4243690] asyncMain---end
2019-08-08 14:53:29.035565+0800 YSC-GCD-demo[17521:4243690] 1---<NSThread: 0x6000022a1380>{number = 1, name = main}
2019-08-08 14:53:31.036565+0800 YSC-GCD-demo[17521:4243690] 2---<NSThread: 0x6000022a1380>{number = 1, name = main}
2019-08-08 14:53:33.037092+0800 YSC-GCD-demo[17521:4243690] 3---<NSThread: 0x6000022a1380>{number = 1, name = main}

在 异步执行 + 主队列 可以看到：

所有任务都是在当前线程（主线程）中执行的，并没有开启新的线程（虽然 异步执行 具备开启线程的能力，但因为是主队列，所以所有任务都在主线程中）。
所有任务是在打印的 syncConcurrent---begin 和 syncConcurrent---end 之后才开始执行的（异步执行不会做任何等待，可以继续执行任务）。
任务是按顺序执行的（因为主队列是 串行队列，每次只有一个任务被执行，任务一个接一个按顺序执行）。
弄懂了难理解、绕来绕去的「不同队列」+「不同任务」使用区别之后，我们来学习一个简单的东西：5. GCD 线程间的通信。

</details>

---

## 2026-05-15 11:02 GCD同步执行主队列死锁分析
**来源**: 博客　**confidence**: 0.90

- 在主线程使用dispatch_sync提交任务到主队列会导致死锁，因为主队列是串行队列且当前任务阻塞。
- dispatch_sync语义是阻塞当前线程直到任务完成，但主队列正忙无法执行新任务，导致循环等待。
- 死锁的三个关键事实：主队列串行、当前任务占用队列、dispatch_sync阻塞线程。
- 子线程同步执行主队列任务不死锁，因为阻塞线程与目标队列线程不同。
- 异步执行主队列任务不死锁，因为dispatch_async不阻塞当前线程，无循环依赖。

<details><summary>原文</summary>

我们假设代码运行在主线程中,大致是这样:
objc复制- (void)syncMain {    NSLog(@"syncMain---begin");        dispatch_sync(dispatch_get_main_queue(), ^{        NSLog(@"任务 1");    });        NSLog(@"syncMain---end");}

执行后你会看到只打印了 syncMain---begin,然后程序就卡死(Xcode 9 及以上版本会直接抛出 EXC_BAD_INSTRUCTION 崩溃,提示 Thread 1: EXC_BAD_INSTRUCTION,这是因为系统检测到了主队列死锁,主动进行了断言)。
要理解为什么卡死,需要先记住三个关键事实:

主队列(Main Queue)是一个串行队列,它里面的任务只会在主线程上被取出执行,而且一次只执行一个。
当前正在执行 syncMain 这段代码的,本身就是主队列上的一个任务——也就是说,主队列正"忙"着跑 syncMain,在 syncMain 没结束之前,主队列不会去取下一个任务。
​dispatch_sync 的语义是:把 block 提交到目标队列,然后阻塞当前线程,直到这个 block 在目标队列上执行完毕,才返回。

把这三点叠加起来,死锁就一目了然了。
才返回。

把这三点叠加起来,死锁就一目了然了。三、为什么换成「其他线程 + 同步执行 + 主队列」就不死锁?​
对照着看就能明白。如果你在子线程中调用 syncMain,情况变成:

阻塞等待的是子线程,不是主线程;
主线程此时是空闲的,主队列也没有"正在执行的前置任务";
所以「任务 1」可以立刻被主线程取出执行,执行完后子线程的 dispatch_sync 返回,继续走下去。

关键差异在于:阻塞的线程和目标队列所依赖的线程不是同一条。死锁的充分条件是"阻塞者"和"被阻塞者所等待的执行者"重合,在主线程调用主队列同步,正好踩中这一点。
四、为什么换成「主线程 + 异步执行 + 主队列」也不死锁?​
因为 dispatch_async 不阻塞当前线程——它把「任务 1」扔进主队列之后就立即返回。于是 syncMain 可以顺利跑完,主线程结束当前任务后再回头从主队列里取「任务 1」执行。没有等待,就没有循环依赖。

</details>

---

## 2026-05-15 11:06 GCD 死锁原理解析
**来源**: 未知　**confidence**: 0.80

- dispatch_sync 投递到当前队列会死锁。
- 死锁原理：dispatch_sync 阻塞当前线程等待 block 执行完，但 block 在队列尾部，当前线程等待自己执行下一个任务，互相等待。
- dispatch_async 异步执行，不阻塞调用线程，所以不会死锁。
- 关键 API：dispatch_sync 和 dispatch_async。
- 解决方案：避免在当前队列使用 dispatch_sync 同步调用。

<details><summary>原文</summary>

GCD 的 dispatch_sync 投递到当前队列会死锁。原理是 dispatch_sync 会阻塞当前线程等待 block 执行完，但 block 又排在当前队列尾部，当前线程在等自己执行下一个任务，互相等待形成死锁。dispatch_async 则不会阻塞调用线程，所以不会死锁。

</details>

---

## 2026-05-15 11:07 GCD 中 dispatch_barrier_async 的读写隔离应用
**来源**: 自述　**confidence**: 0.90

- dispatch_barrier_async 用于并发队列中的读写隔离
- 提交后会等待之前的任务完成，自己执行时独占队列
- 执行完之后队列才继续接受其他 block
- 常用于实现读写锁模式

<details><summary>原文</summary>

测试 GCD 第二条：dispatch_barrier_async 用于并发队列里的读写隔离，提交后会等待之前的任务完成，自己执行时独占队列，执行完之后队列才继续接受其他 block。常用于实现读写锁模式。

</details>

---

## 2026-05-15 12:04 dispatch_group_notify：非阻塞完成回调详解
**来源**: 博客　**confidence**: 0.90

- dispatch_group_notify 函数在组任务完成后异步执行指定 block，不阻塞当前线程
- 行为时间线：组任务并发执行，完成后 block 被派发到指定队列，主线程可继续运行
- 核心特点：不阻塞当前线程、可指定回调队列（如主队列用于 UI）、支持多次调用 notify
- 典型使用场景：并行网络请求后刷新 UI、批量下载后合并处理等扇出-扇入模式

<details><summary>原文</summary>

二、dispatch_group_notify:非阻塞的"完成回调"​
函数原型与语义
objc复制dispatch_group_notify(dispatch_group_t group,                      dispatch_queue_t queue,                      dispatch_block_t block);

它的语义是:当 group 里的所有任务都完成时,把 block 异步派发到 queue 上去执行。这一行代码本身立即返回,不会让当前线程停留一秒。
行为时间线
假设你在主线程写下:
objc复制dispatch_group_t group = dispatch_group_create();
dispatch_group_async(group, globalQueue, ^{    // 任务 A:下载图片 1});dispatch_group_async(group, globalQueue, ^{    // 任务 B:下载图片 2});
dispatch_group_notify(group, dispatch_get_main_queue(), ^{    // 全部下载完后,回主线程刷新 UI    self.imageView1.image = img1;    self.imageView2.image = img2;});
NSLog(@"我先打印,不会被卡住");

执行顺序大致是:

主线程立刻打印"我先打印",因为 dispatch_group_notify 本身不阻塞;
任务 A、B 在子线程并发跑;
两个任务都完成后,notify 的 block 被自动派发到主队列;
主线程在下一个 runloop 周期里取出这个 block,执行 UI 刷新。

核心特点

不阻塞当前线程:这是它最大的优势,主线程可以继续响应用户交互、动画、滚动等。
可以指定回调队列:你可以让最终的"汇合任务"跑在主队列做 UI、跑在某个串行队列保证线程安全、或跑在并发队列做后续耗时处理。
可以被多次调用:对同一个 group 多次调用 notify 是安全的,每个 notify 块都会在组归零时被触发。
如果调用 notify 时组里已经没有任务了,block 会被立即派发执行,不会"死等"。

典型使用场景

并行发起多个网络请求,等全部回来后刷新 UI;
批量下载图片/文件后做合并处理;
多个独立的预加载任务都完成后再进入下一个页面;
任何"扇出-扇入(scatter-gather)"模式下的汇合点。

</details>

---

## 2026-05-15 12:10 GCD 信号量 (Dispatch Semaphore) 原理
**来源**: 未知　**confidence**: 0.90

- GCD 中的信号量为 Dispatch Semaphore，是一种基于计数的同步机制
- 类比过高速路收费站栏杆：计数控制访问权限，可通过时打开栏杆，不可通过时关闭
- 计数小于 0 时等待且不可通过；计数为 0 或大于 0 时，计数减 1 且不等待可通过

<details><summary>原文</summary>

[200~GCD 中的信号量是指 Dispatch Semaphore，是持有计数的信号。类似于过高速路收费站的栏杆。可以通过时，打开栏杆，不可以通过时，关闭栏杆。在 Dispatch Semaphore 中，使用计数来完成这个功能，计数小于 0 时等待，不可通过。计数为 0 或大于 0 时，计数减 1 且不等待，可通过。

</details>

---

## 2026-05-15 12:11 Dispatch Semaphore 的三个方法
**来源**: 未知　**confidence**: 0.70

- Dispatch Semaphore 提供了三个方法
- 用于线程同步
- 是 GCD 的一部分

<details><summary>原文</summary>

Dispatch Semaphore 提供了三个方法：

</details>

---

## 2026-05-15 12:12 dispatch_semaphore_create 函数简介
**来源**: 未知　**confidence**: 0.90

- dispatch_semaphore_create 用于创建信号量。
- 它初始化信号量的计数总量。
- 属于 GCD 并发编程 API。

<details><summary>原文</summary>

dispatch_semaphore_create：创建一个 Semaphore 并初始化信号的总量

</details>

---

## 2026-05-15 12:12 dispatch_semaphore_signal 信号发送机制
**来源**: 未知　**confidence**: 0.90

- dispatch_semaphore_signal 是发送信号的函数
- 该操作让信号总量加 1
- 用于线程同步和资源控制

<details><summary>原文</summary>

dispatch_semaphore_signal：发送一个信号，让信号总量加 1

</details>

---

