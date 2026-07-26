---
title: GCD 研究简报（给写作 agent 用）
draft: true
---
# GCD 研究简报

来源分级：🅐 一手官方（Apple 文档 / 头文件 / WWDC）｜🅑 二手博客｜⚠️ 已过时

---

## 一、必须写进文章的官方原文（逐字，可直接引用）

### 1. barrier 在非自建并发队列上的行为 —— 中文圈流传的一处硬错误

Apple `dispatch_barrier_async` Discussion 原文：

> Calls to this function always return immediately after the block is submitted and never wait for the block to be invoked. When the barrier block reaches the front of a private concurrent queue, it is not executed immediately. Instead, the queue waits until its currently executing blocks finish executing. At that point, the barrier block executes by itself. Any blocks submitted after the barrier block are not executed until the barrier block completes.
>
> **The queue you specify should be a concurrent queue that you create yourself using the `dispatch_queue_create` function. If the queue you pass to this function is a serial queue or one of the global concurrent queues, this function behaves like the `dispatch_async` function.**

出处：https://developer.apple.com/documentation/dispatch/dispatch_barrier_async

**ming1016（戴铭）2016 年那篇《细说 GCD》里写的是"在全局并发队列和串行队列上，效果和 `dispatch_sync` 一样"——错了，Apple 说的是 `dispatch_async`。** 这条被大量转载。写文章时把它当成"读一手文档"的教学切入点。

### 2. sync 在哪个线程执行 —— "主队列 vs 主线程"的官方裁决

`DispatchQueue.sync(execute:)` 原文：

> This function submits a block to the specified dispatch queue for synchronous execution. Unlike `dispatch_async`, this function does not return until the block has finished. **Calling this function and targeting the current queue results in deadlock.**
>
> **As a performance optimization, this function executes blocks on the current thread whenever possible, with one exception: Blocks submitted to the main dispatch queue always run on the main thread.**

出处：https://developer.apple.com/documentation/dispatch/dispatchqueue/sync(execute:)-3segw

这段直接否定了两个流行说法：
- "并发队列同步执行就只会在主线程执行"（ming1016）—— 错，是在**调用线程**
- "sync/async 表示是否开启新线程" —— 不准确，`sync` 尽量复用调用线程

### 3. 队列与线程的关系

`DispatchQueue` 官方原文：

> Work submitted to dispatch queues executes on a pool of threads managed by the system. **Except for the dispatch queue representing your app's main thread, the system makes no guarantees about which thread it uses to execute a task.**

计划书开头点名要纠正的"GCD 线程池"说法，依据就在这里。

### 4. dispatch_get_main_queue（libdispatch 头文件）

> Returns the default queue that is bound to the main thread.
>
> In order to invoke blocks submitted to the main queue, the application must call `dispatch_main()`, `NSApplicationMain()`, or use a CFRunLoop on the main thread.
>
> **Because the main queue doesn't behave entirely like a regular serial queue, it may have unwanted side-effects when used in processes that are not UI apps (daemons).**

### 5. dispatch_get_current_queue 为什么被废弃（头文件）

> **When `dispatch_get_current_queue()` is called on the main thread, it may or may not return the same value as `dispatch_get_main_queue()`. Comparing the two is not a valid way to test whether code is executing on the main thread** (see `dispatch_assert_queue()` and `dispatch_assert_queue_not()`).
>
> **This function is deprecated and will be removed in a future release.**

```c
API_DEPRECATED("unsupported interface", macos(10.6,10.9), ios(4.0,6.0))
```

官方替代品：`dispatch_assert_queue()` / `dispatch_assert_queue_not()`，或 FMDB 式的 `dispatch_queue_set_specific` / `dispatch_get_specific`。

### 6. QoS 的一个大坑（头文件，三篇中文博客都没提）

> **The QOS class and relative priority set this way on a queue have no effect on blocks that are submitted synchronously to a queue (via `dispatch_sync()`, `dispatch_barrier_sync()`).**

出自 `dispatch_queue_attr_make_with_qos_class`。

### 7. dispatch_after 只是延后提交

> This function waits until the specified time and then **asynchronously adds** `block` to the specified `queue`.
>
> Passing `DISPATCH_TIME_NOW` as the `when` parameter is supported, but is not as optimal as calling `dispatch_async` instead. Passing `DISPATCH_TIME_FOREVER` is undefined.

### 8. 定时器精度（dispatch_source_set_timer）

> The `leeway` parameter is a hint from the application as to the amount of time, in nanoseconds, up to which the system can defer the timer to align with other system activity for improved system performance or power consumption. **Note that some latency is to be expected for all timers, even when a leeway value of zero is specified.**
>
> If the start time is `DISPATCH_TIME_NOW` or is created with `dispatch_time`, the timer is based on `mach_absolute_time`. Otherwise, if the start time of the timer is created with `dispatch_walltime`, the timer is based on `gettimeofday`(3).

`mach_absolute_time`（不含睡眠）vs `dispatch_walltime`（墙钟）语义不同。

### 9. 线程爆炸（WWDC17 Session 706《Modernizing Grand Central Dispatch Usage》）

> The way the global concurrent queue works is that it creates more threads when existing threads block to give you a continuing good level of concurrency in your application. But if those threads then block again, you can get something that we call **the thread explosion**.
>
> The main thing that is important here is to have a **fixed number of serial queue hierarchies** in your application.
>
> （谈串行队列）This is really our **fundamental synchronization primitive** in GCD. It provides you with mutual exclusion as well as FIFO ordering.
>
> Primitives with a single known owner have this power. Things like **serial queues and OS unfair lock**.

### 10. Swift Concurrency 的对比（WWDC21 Session 10254）

> The new thread pool will only spawn as many threads as there are CPU cores, thereby making sure not to overcommit the system.
>
> Unlike GCD's concurrent queues, which will spawn more threads when work items block, with Swift threads can always make forward progress.
>
> **Primitives like semaphores and condition variables are unsafe to use with Swift concurrency.** This is because they hide dependency information from the Swift runtime... **This violates the runtime contract of forward progress for threads.**
>
> Consider an iPhone with six CPU cores. If our news application has a hundred feed updates that need to be processed, this means that we have overcommitted the iPhone with 16 times more threads than cores. **This is the phenomenon we call thread explosion.**

---

## 二、Pierre Habouzit（Apple libdispatch 维护者）的实践原则

出自《Making efficient use of the libdispatch (GCD)》，经 dirtmelon 转述。这是全部二手资料里最有现代价值的一段：

- 只使用非常少、明确定义的 queue，按 App 子系统划分。**大多数 App 尽量不要超过 3~4 个 queue**
- **不要使用 `dispatch_get_global_queue()`**，它不能很好处理优先级，同时会导致线程爆炸
- 如果 block 执行时间小于 1ms，用 `dispatch_async()` 是性能浪费。**用锁保护共享状态是更好的选择**
- `os_unfair_lock` 通常是系统中最快的锁
- **串行队列比并行队列优化得更好**。只有需要性能改善时才用并行队列，否则是过早优化
- 一旦开始复用自定义队列，`dispatch_sync()` 容易导致死锁。**锁是更好的解决方案**，只有需要切换执行上下文时才用 `dispatch_async()`

注意：Apple 文档里有一段说"不要建自定义并发队列，用 global queue，串行队列 target 到 global queue"，与 Habouzit 的"不要用 global queue"表面冲突。前者针对"避免线程数膨胀"，后者针对"优先级/QoS 传递"。WWDC17 的 "queue hierarchy per subsystem" 是二者的调和答案。

---

## 三、死锁的五种情况（ming1016 归纳，可作为实验设计参考）

1. 主线程直接 `dispatch_sync(dispatch_get_main_queue())` → 死锁
2. 主线程 `dispatch_sync(global_queue)` → 不死锁
3. `dispatch_async(serialQueue){ dispatch_sync(同一个 serialQueue){} }` → 死锁
4. `dispatch_async(global){ dispatch_sync(main){} }` → 不死锁
5. 子线程 `dispatch_sync(main)` + 主线程 `while(1)` 死循环 → 主队列永远得不到调度，卡死

总纲："当前串行队列里面同步执行当前串行队列就会死锁"。准确表述是**"向当前所在的串行队列同步派发"**，不是"在主队列 sync 就死锁"。

---

## 四、三篇常被引用的中文文章，可信度评估

| 文章 | 年份 | 可直接引用 | 已过时 |
| --- | --- | --- | --- |
| 唐巧《使用 GCD》 | 2012 | GCD 的动机叙事（三方法拆分 → 一个 block）、block 语义、`__block` | MRC/`dispatch_release`、`DISPATCH_QUEUE_PRIORITY_*`、同步网络请求、后台运行"10 分钟"（iOS 7 起大幅缩短） |
| ming1016《细说 GCD》 | 2016 | 五种死锁 case、dispatch_source 事件表、`dispatch_after` 只是延后提交、`set_specific` 防死锁、`set_target_queue` 层级、`dispatch_apply` 防线程爆炸 | **barrier 在 global/serial 上等价于 `dispatch_sync`（错，应为 `dispatch_async`）**、"并发队列同步执行只会在主线程"（错）、5 优先级队列模型、`OSSpinLock`、`OSAtomic`、`dispatch_release`、Swift 2 语法、`USEC_PER_SEC` 注释写成"毫秒"（应为微秒） |
| dirtmelon《Grand Central Dispatch》 | 无日期 | "GCD 注意点"整节（Habouzit 原则） | `OSSpinLock` 段落 |

---

## 五、Swift Concurrency 时代仍需 GCD 的场景

| 场景 | 是否仍需 GCD | 依据 |
| --- | --- | --- |
| 文件系统监听、UNIX signal、Mach port、内存压力、进程事件、fd 读写 | **必须**，无替代 | `DispatchSource` 官方文档 |
| 不依赖 RunLoop 的重复定时器 + leeway 省电对齐 + walltime 时钟 | **强烈建议** | `dispatch_source_set_timer` |
| 一次性延时（`asyncAfter`） | 应迁移到 `Task.sleep` | Quinn @ Apple DTS 在 Swift Forums 的回复 |
| 分钟级 CPU 密集 / 会阻塞的工作 | 建议留在 GCD，外面包 `withCheckedContinuation` | WWDC21 + Swift Forums |
| 保护少量共享可变状态 | 既不用队列也不用 actor，用 `os_unfair_lock` / `Mutex` | WWDC17 + Habouzit |
| 与既有串行队列共享执行上下文的遗留模块 | 用 `DispatchSerialQueue` 作为 actor 的 `SerialExecutor` | `DispatchSerialQueue` 已 conform `SerialExecutor` |
| `DispatchSemaphore` | **在 Swift Concurrency 上下文中禁止** | WWDC21 原文 |

---

## 六、建议主实验清单

1. **死锁的四种边界**：主队列 sync 主队列 / 串行队列 sync 同一队列 / 串行队列 sync 另一串行队列 / 并发队列 sync 自己 —— 分别验证，很多文章讲得含糊
2. **barrier 在自建并发队列 vs 全局队列上的行为差异** —— 这条能直接实测出 ming1016 那个错误
3. **队列不是线程**：在一个串行队列上打印 `pthread_self()`，观察多次执行是否在同一线程；多个队列观察线程复用
4. **主队列 vs 主线程**：`dispatch_get_main_queue()` 上的任务一定在主线程，但主线程上跑的不一定是主队列的任务
5. `dispatch_group` 两种用法对照、信号量做并发控制
6. QoS 对 `dispatch_sync` 无效这一条

注意：模拟器的**性能数字**不能用来下"优化有效"的结论，但**行为逻辑**的验证有效。
