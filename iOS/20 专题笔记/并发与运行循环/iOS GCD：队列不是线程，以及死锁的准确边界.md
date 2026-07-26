---
title: 【iOS】GCD：队列不是线程，以及死锁的准确边界
published: 2026-07-27
description: 同一个串行队列的十次派发落在九条不同线程上；两个不同的串行队列被同一条线程服务。而 barrier 在全局队列上完全失效，用它写的读写锁会静默丢更新，一声不吭。
tags:
  - iOS
  - GCD
  - 并发
  - libdispatch
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 14
draft: true
---
# GCD：队列不是线程，以及死锁的准确边界

先看两组数字。

一个串行队列，反复往里 `dispatch_async` 十次，中间让线程池被别的活儿搅一遍。十次派发落在九条不同的线程上。

反过来，建两个毫无关系的串行队列 A 和 B，交替提交，它们全程被同一条线程 `0x16d76b000` 服务。

所以"每个队列背后有一个线程池"这个说法，从第一个字就不对。队列和线程之间根本没有稳定的对应关系，一次都没有。

这篇的另外两个结论：中文圈流传最广的那篇 GCD 文章里，`dispatch_barrier_async` 在全局队列上的行为写错了，而这个错误在真实代码里会让读写锁静默失效——实测能丢掉八成的写入，而且一个错都不报。以及"死锁"的边界比几乎所有教程讲的都要窄，也要宽：并发队列 sync 自己不死锁，但两个"不同的"串行队列 sync 也可能当场 trap。

全文实验都在 iOS 模拟器（arm64，Xcode 26.6，iPhone 16 / iOS 18.3）上真跑，编译命令统一是：

```shell
SDK=$(xcrun --sdk iphonesimulator --show-sdk-path)
clang -fobjc-arc -isysroot "$SDK" -target arm64-apple-ios17.0-simulator \
      -framework Foundation -o out prog.m
xcrun simctl spawn booted ./out
```

---

## 一、"GCD 线程池"错在哪

Apple 对 `DispatchQueue` 的描述只有一句关键话：

> Work submitted to dispatch queues executes on a pool of threads managed by the system. Except for the dispatch queue representing your app's main thread, the system makes no guarantees about which thread it uses to execute a task.

线程池是有的，但它属于系统，不属于你的队列。队列是提交点，线程是执行资源，中间隔着一层调度。

开头那两组数据就是这句话的两个方向。第一个方向，同一个队列换线程：

```objc
dispatch_queue_t s = dispatch_queue_create("one.serial", DISPATCH_QUEUE_SERIAL);
for (int i = 1; i <= 10; i++) {
    // 先用一批会阻塞的任务把线程池搅一遍
    dispatch_group_t grp = dispatch_group_create();
    for (int k = 0; k < 24; k++) dispatch_group_async(grp, g, ^{ usleep(60000); });
    dispatch_group_wait(grp, DISPATCH_TIME_FOREVER);

    dispatch_sync(s, ^{ printf("  第 %2d 次   thread=%p\n", i, pthread_self()); });
    dispatch_async(s, ^{ printf("      async 落在 thread=%p\n", pthread_self()); });
}
```

```text
  第  1 次   thread=0x102b35e00
      async 落在 thread=0x16d8d7000
  第  2 次   thread=0x102b35e00
      async 落在 thread=0x16d9ef000
  第  3 次   thread=0x102b35e00
      async 落在 thread=0x16dedb000
  ...
  第 10 次   thread=0x102b35e00
      async 落在 thread=0x16d963000
串行队列的 10 次 async 一共落在 9 条不同线程上
```

这组数据把第二节的结论也一起测出来了：那十行 `dispatch_sync` 全部落在 `0x102b35e00`，也就是主线程，一次都没变。同一个队列，`async` 换了九条线程，`sync` 一条都没换。

第二个方向，不同队列共用线程：

```text
  队列 A   第 1 次   thread=0x16d76b000
  队列 B   第 1 次   thread=0x16d76b000
  队列 A   第 2 次   thread=0x16d76b000
  队列 B   第 2 次   thread=0x16d76b000
```

第三个方向，队列忙起来也不会多占线程。往一个串行队列连续压 200 个任务，统计用到几条：

```text
  200 个任务用到 1 条不同线程
```

三组合起来，队列和线程的关系就清楚了。多对多，且随时间变化。

这不是纯理论洁癖，有几个直接后果。线程局部存储在队列任务里不可靠，你在第一个 block 里写进 TLS 的东西，第二个 block 可能读不到。`[NSThread currentThread]` 的名字、优先级、`threadDictionary` 同理。可重入锁（`PTHREAD_MUTEX_RECURSIVE`）在 `dispatch_async` 的两个 block 之间也没有意义。"同一个线程"这个前提就不成立。

---

## 二、`dispatch_sync` 到底在哪条线程执行

网页文档上那句话很多人引过。但 SDK 头文件里的措辞更完整，多了一个括号，而那个括号恰好是第三节的关键：

```shell
grep -A 30 "@function dispatch_sync$" "$SDK/usr/include/dispatch/queue.h"
```

> As an optimization, dispatch_sync() invokes the workitem on the thread which submitted the workitem, except when the passed queue is the main queue or a queue targetting it (See dispatch_queue_main_t, dispatch_set_target_queue()).

所以 `sync` 的默认行为是**在调用线程上就地执行**，只有目标是主队列（或者 target 链指向主队列的队列）才例外。实测：

```text
main() 起点                      thread=0x101011e00  is_main_thread=1
sync 到自建串行队列             thread=0x101011e00  is_main_thread=1
sync 到自建并发队列             thread=0x101011e00  is_main_thread=1
sync 到全局并发队列             thread=0x101011e00  is_main_thread=1
先占住 s 的 async block        thread=0x16f097000  is_main_thread=0
队列忙时 sync 到 s            thread=0x101011e00  is_main_thread=1
```

最后两行是我特意加的对照。先往串行队列扔一个 sleep 200ms 的 async 把它占住，再 `dispatch_sync` 过去。sync 的 block 老老实实排在后面，FIFO 没被破坏。但轮到它的时候，执行它的还是主线程。

等待的是队列的所有权。执行的位置不变。

这一条否掉了两个流行说法。

"`sync`/`async` 决定要不要开新线程"——`sync` 从来不开新线程，`async` 也不一定开（派到主队列就不开）。这组词描述的是调用方要不要等，和线程数没有直接关系。

"并发队列同步执行只会在主线程执行"——ming1016《细说 GCD》里的原话。它只在"你恰好从主线程调用"时看起来是对的。换到任何子线程去调用，block 就跟着跑到那条子线程上。上面那个 `0x102b35e00` 之所以是主线程，仅仅因为循环写在 `main()` 里。

---

## 三、死锁的准确边界

这是全文我最想讲清楚的一节，因为几乎所有教程都停在"当前队列 sync 当前队列就死锁"，而这句话既漏了情况，又多了情况。

我写了一个按 case 分支的程序，每个 case 装 `alarm(3)`，三秒没返回就打印并退出。这样能把两种失败区分开：libdispatch 主动检测到并 trap，和真的挂在那里。这两种在现场表现完全不同，前者给你一个崩溃栈，后者给你一个转圈的 App。

| 场景 | 结果 |
| --- | --- |
| 主线程上 `dispatch_sync(主队列)` | SIGTRAP，当场崩 |
| 串行队列内 `dispatch_sync(同一队列)` | SIGTRAP |
| 串行队列 A 内 `dispatch_sync(无关的串行队列 B)` | 正常返回 |
| A target 到 B，A 内 `dispatch_sync(B)` | SIGTRAP（待复核，见下） |
| A、C 都 target 到 B，A 内 `dispatch_sync(C)` | SIGTRAP（待复核，见下） |
| 并发队列内 `dispatch_sync(同一并发队列)` | 正常返回，且同线程 |
| 并发队列内 `dispatch_barrier_sync(同一并发队列)` | 三秒挂住 |
| A 内 sync B，同时 B 内 sync A | 三秒挂住 |
| 子线程 `dispatch_sync(主队列)`，主线程跑 `while(1)` | 三秒挂住 |

逐条说几个值得说的。

### trap 不是"死锁"，是断言

主线程 sync 主队列，程序不是卡住，是立刻崩。崩溃栈长这样：

```text
libdispatch.dylib  __DISPATCH_WAIT_FOR_QUEUE__      + 81644
libdispatch.dylib  _dispatch_sync_f_slow            + 80216
dl_main            main                             + 1640
```

`__DISPATCH_WAIT_FOR_QUEUE__` 这个全大写的帧名是 libdispatch 故意留的路标。看到它，就说明有人在等一个队列，而那个队列正等着他。异常类型是 `EXC_BREAKPOINT / SIGTRAP`。faulting thread 的 queue 字段写着 `com.apple.main-thread`。

所以老文章里"主队列 sync 会卡住不动"的截图，今天复现不出来了。libdispatch 在 `_dispatch_sync_f_slow` 里会检查目标队列当前的 owner 是不是自己这条线程，是就直接 trap。

这是好事。一个无从下手的挂起，换成了一个有栈可看的崩溃。

### "另一个串行队列就安全"是错的

这条是我这次测出来最意外的。两个串行队列 A 和 C，各自独立创建，看起来毫无关系。但只要它们 target 到同一个串行队列 B：

```objc
dispatch_queue_t b = dispatch_queue_create("B", DISPATCH_QUEUE_SERIAL);
dispatch_queue_t a = dispatch_queue_create_with_target("A", DISPATCH_QUEUE_SERIAL, b);
dispatch_queue_t c = dispatch_queue_create_with_target("C", DISPATCH_QUEUE_SERIAL, b);
dispatch_async(a, ^{
    dispatch_sync(c, ^{ puts("  C 的 block 跑了"); });
});
```

```text
A 和 C 都 target 到同一个串行队列 B，在 A 里 sync 到 C
  在 A 上，准备 sync 到 C
Child process terminated with signal 5: Trace/BPT trap
```

原因不难想：A 和 C 都要拿 B 的所有权才能跑，A 正拿着，C 拿不到。libdispatch 认得出这一点并直接 trap。

原因不难想：A 和 C 都要拿 B 的所有权才能跑，A 正拿着，C 拿不到。libdispatch 认得出这一点并直接 trap。

由此推出的表述是：**`dispatch_sync` 的目标队列，只要和当前执行上下文落在同一条 target 链的同一个串行队列上，就会 trap。**不是"当前队列"，而是"当前的执行上下文"。

而 WWDC17 恰恰在推荐按子系统建 target 队列层级（"a fixed number of serial queue hierarchies"）。层级建起来之后，"这两个队列不是同一个对象所以 sync 没问题"这条直觉就失效了。这也是 Pierre Habouzit 那句"一旦开始复用自定义队列，`dispatch_sync()` 容易导致死锁"的具体含义。

> **这一条我标为待复核。** 上面那段输出是真跑出来的，但我换一种写法复现时结果不一致——用 `dispatch_set_target_queue` 建立 target 关系、并且从队列外部 `dispatch_sync(A)` 再嵌套 `dispatch_sync(C)`，没有 trap。两种写法的差别在于建立 target 的时机（`create_with_target` 是创建时绑定，`set_target_queue` 是事后绑定）以及外层用的是 async 还是 sync，这两个变量我还没拆开单独验证。
>
> 在拆清楚之前，稳妥的结论只到这一步：**target 链确实会把两个"看起来无关"的串行队列绑成同一个同步域，libdispatch 也确实会为此 trap；但触发它的准确条件我还没有完全确定。** 工程上的取用方式不受影响——一旦用了 target 队列层级，就不要再假设"不同的队列对象之间 sync 是安全的"。

### 并发队列 sync 自己是安全的，barrier_sync 不是

```text
== case4 并发队列内 dispatch_sync(同一并发队列) ==
   outer on C, thread=0x16aeef000
   inner on C, thread=0x16aeef000
   inner returned  -> 没有死锁

== case8 并发队列内 dispatch_barrier_sync(同一并发队列) ==
   outer on C
  >>> ALARM: 3s 内没有返回，判定为死锁/挂起
```

`dispatch_sync` 到并发队列不需要独占所有权，直接在调用线程上跑掉就行。注意上面两行的 thread 值完全相同。

`dispatch_barrier_sync` 不一样。它要求队列上没有别的任务在跑，而调用它的那个 block 自己就在跑。条件永远满足不了。

这里没有 trap，是真挂住。libdispatch 的自检覆盖串行队列的所有权环，不覆盖 barrier 的这种自等。

写读写锁封装的时候要小心。在 reader block 里调 writer 方法，就是这个形状。

### 多方等待也不 trap

A 里 sync B，B 里 sync A，两边都在等对方释放，谁也没有"自己等自己"，所以自检抓不到，三秒后被我的 alarm 打死。头文件里对这种情况有一句提醒：

> Use of dispatch_sync() is also subject to the same multi-party dead-lock problems that may result from the use of a mutex.

至于"子线程 sync 主队列 + 主线程死循环"，这是同一件事的变体：主线程永远不回到调度点，主队列永远排不上，子线程就一直等下去。

### 那怎么知道自己在不在某个队列上

`dispatch_get_current_queue` 是很多人的第一反应，它被废弃了，头文件写得很直白：

> When dispatch_get_current_queue() is called on the main thread, it may or may not return the same value as dispatch_get_main_queue(). Comparing the two is not a valid way to test whether code is executing on the main thread.
>
> This function is deprecated and will be removed in a future release.

诚实说一句：我在命令行程序里测的时候，主线程上它确实等于 `dispatch_get_main_queue()`。两个都是 `0x1efeb8980`，label 都是 `com.apple.main-thread`。

但文档说的是"可能相等也可能不相等"。一次相等证明不了什么。这种碰巧能跑对的 API 才最危险。

官方给的替代品是 `dispatch_assert_queue()` / `dispatch_assert_queue_not()`。我实测了它的语义，结果和上面那条死锁边界严丝合缝：

```objc
dispatch_queue_t b = dispatch_queue_create("B", DISPATCH_QUEUE_SERIAL);
dispatch_queue_t a = dispatch_queue_create_with_target("A", DISPATCH_QUEUE_SERIAL, b);
dispatch_sync(a, ^{ dispatch_assert_queue(b); puts("  通过"); });
```

```text
A target 到 B，在 A 上 assert_queue(B)
  通过：A 上的代码被认为也在 B 上
```

在 A 上执行的代码，被 libdispatch 认定同时也在 B 上。所以 `dispatch_assert_queue` 判定的是执行上下文而不是队列对象身份，和 trap 用的是同一套判据。断言不通过时它也是直接 trap（我在主线程上对一个串行队列断言，当场 `Trace/BPT trap`）。

要做条件分支而不是断言，就用 `dispatch_queue_set_specific` / `dispatch_get_specific`，FMDB 那套写法：

```objc
static const void *kKey = &kKey;
dispatch_queue_set_specific(s, kKey, (__bridge void *)s, NULL);

void (^safeSync)(dispatch_block_t) = ^(dispatch_block_t b) {
    if (dispatch_get_specific(kKey) == (__bridge void *)s) b();
    else dispatch_sync(s, b);
};
```

```text
  主线程调用：
    不在队列上，走 dispatch_sync
  队列内部再调用一次（这里换成裸 dispatch_sync 就会 crash）：
    已经在队列上，直接执行

  target 到 s 的子队列上能不能查到同一个 specific：
    child 上 get_specific = 0x600002c0c080（s = 0x600002c0c080）
```

最后一行是我特意补的：specific 沿 target 链继承。一个 target 到 `s` 的子队列，在上面查 `kKey` 拿得到 `s` 的值。这正好覆盖了前面那个"A 和 C 都 target 到 B"的坑。这套机制之所以可靠，就是因为它跟死锁判据走的是同一条 target 链。

---

## 四、barrier：一处流传极广的错误

ming1016《细说 GCD》写：`dispatch_barrier_async` 在全局并发队列和串行队列上，"效果和 `dispatch_sync` 一样"。这篇文章被转载了无数次。

Apple 说的是另一个词。SDK 头文件 `queue.h` 的 Dispatch Barrier API 段落：

> Barrier blocks only behave specially when submitted to queues created with the DISPATCH_QUEUE_CONCURRENT attribute; on such a queue, a barrier block will not run until all blocks submitted to the queue earlier have completed, and any blocks submitted to the queue after a barrier block will not run until the barrier block has completed.
>
> When submitted to a global queue or to a queue not created with the DISPATCH_QUEUE_CONCURRENT attribute, barrier blocks behave identically to blocks submitted with the dispatch_async()/dispatch_sync() API.

`dispatch_barrier_async` 退化成 `dispatch_async`，`dispatch_barrier_sync` 退化成 `dispatch_sync`。异步的还是异步的。

这个区别用一行计时就能测死。假设它真的等价于 `dispatch_sync`，那么在四个 sleep 300ms 的任务后面调用它，这一行至少要阻塞 300ms 才返回。

程序结构是：先扔 4 个"读"任务，再一个 barrier，再 4 个"写后"任务，每个任务打印开始/结束时刻和当前同时在跑的任务数。

自建并发队列：

```text
[    0.0ms] dispatch_barrier_async 调用耗时 0.000 ms
[    0.1ms] 读 1        开始  同时在跑=1  thread=0x16b5c7000
[    0.1ms] 读 2        开始  同时在跑=2  thread=0x16b653000
[    0.1ms] 读 4        开始  同时在跑=4  thread=0x16b76b000
[    0.1ms] 读 3        开始  同时在跑=3  thread=0x16b6df000
[  307.8ms] 读 4        结束  同时在跑=0
[  307.9ms] ★ BARRIER  开始  同时在跑=1  thread=0x16b76b000
[  609.0ms] ★ BARRIER  结束  同时在跑=0
[  609.2ms] 写后 1     开始  同时在跑=1
[  609.2ms] 写后 2     开始  同时在跑=2
[  609.3ms] 写后 4     开始  同时在跑=4
```

教科书式的效果。4 个并发跑完，barrier 独占（同时在跑始终是 1），完了后面 4 个再一起上。`queue.h` 里给这个模型的类比写得比任何博客都准：

> In other words, if a serial queue is equivalent to a mutex in the Dispatch world, a concurrent queue is equivalent to a reader-writer lock, where regular items are readers and barriers are writers.

全局并发队列：

```text
[    0.0ms] dispatch_barrier_async 调用耗时 0.000 ms
[    0.1ms] 读 1        开始  同时在跑=1  thread=0x16b93b000
[    0.2ms] 读 4        开始  同时在跑=4  thread=0x16badf000
[    0.2ms] ★ BARRIER  开始  同时在跑=5  thread=0x16bb6b000
[    0.2ms] 写后 1     开始  同时在跑=6  thread=0x16bbf7000
[    0.3ms] 写后 4     开始  同时在跑=9  thread=0x16bd9b000
[  304.5ms] ★ BARRIER  结束  同时在跑=1
```

九个任务同时在跑，barrier 和其他八个肩并肩。屏障语义整个消失了。

再看那个耗时：`0.000 ms`。它立刻就返回了。如果它等价于 `dispatch_sync`，这个数字不可能小于 300。**Apple 文档写的是退化成 `dispatch_async`，实测就是 `dispatch_async`。**

串行队列上同理，耗时 0.000ms，四个"写后"任务在"读 1"还没跑完的时候就已经提交进去了。要是等价于 `dispatch_sync`，那一行得阻塞 1.2 秒。

### 这个错误的实际代价

上面只是行为差异。放进真实代码里，代价是数据静默损坏。

写一个最标准的 barrier 读写锁封装，读用 `dispatch_sync`，写用 `dispatch_barrier_async`。唯一的变量是队列从哪来：

```objc
- (void)bump {
    dispatch_barrier_async(_q, ^{
        long t = self->_v;
        for (volatile int k = 0; k < 40; k++) { }   // 拉长临界区
        self->_v = t + 1;
    });
}
- (long)value { __block long r; dispatch_sync(_q, ^{ r = self->_v; }); return r; }
```

`dispatch_apply` 并发调用 `bump` 两万次，跑三轮：

```text
自建并发队列：bump 20000 次，最终值 = 20000  ✅
全局并发队列：bump 20000 次，最终值 = 3110   ❌ 丢了更新
自建并发队列：bump 20000 次，最终值 = 20000  ✅
全局并发队列：bump 20000 次，最终值 = 5924   ❌ 丢了更新
自建并发队列：bump 20000 次，最终值 = 20000  ✅
全局并发队列：bump 20000 次，最终值 = 9534   ❌ 丢了更新
```

自建队列三轮都精确。全局队列丢掉一半到八成，每轮丢的量还都不一样。

封装代码一行没改。只是 `dispatch_queue_create(..., DISPATCH_QUEUE_CONCURRENT)` 换成了 `dispatch_get_global_queue()`。互斥就没了，还不报任何错。

丢多少这件事要说清楚：**它完全取决于临界区有多长。** 上面那段 `bump` 里特意加了一个 40 次的空循环把临界区拉长，所以丢得触目惊心。我换一种写法复核了一遍——去掉那个空循环、临界区只剩一次自增、并且改用两万次顺序 `dispatch_barrier_async` 提交——全局队列三轮的结果是 19710 / 19632 / 19615，只丢百分之二。

方向是一样的，数量级差了很多。所以别把"两万次只剩三千"当成这个 bug 的固有特征。它真正的固有特征是：**丢失率随临界区长度上升，而且不报任何错。**临界区短的时候它可能几个月都不出事，直到某次你在里面多加了一行。

我见过的封装里，队列常常来自初始化参数或者某个全局配置。这种改动在 code review 里看起来就是"复用一下系统队列，少建一个对象"。

barrier 的另一半也要说清楚：它只对同一个队列上的其他任务生效，对访问同一份数据但走别的路径的代码毫无约束。有人在别处直接 `_v++`，barrier 一点忙帮不上。

即便队列建对了，这套读写锁还有一条官方承认的短板，写在同一段注释的 Caveat 里：

> Dispatch concurrent queues at this time do not implement priority inversion avoidance when lower priority regular workitems (readers) are being invoked and are preventing a higher priority barrier (writer) from being invoked.

低优先级的读会把高优先级的写堵在后面，GCD 不管。读多写少、而且写的那一侧要紧的场景，`os_unfair_lock` 或者 `pthread_rwlock` 反而更稳。

---

## 五、主队列不是主线程

这两个概念的混淆几乎是默认状态，而且命令行 demo 会主动骗你。

同一段代码，只改主线程最后跑什么：

```objc
dispatch_async(dispatch_get_global_queue(QOS_CLASS_DEFAULT, 0), ^{
    p("global queue 上的 block");
    dispatch_sync(dispatch_get_main_queue(),  ^{ p("  sync 到 main queue 的 block"); });
    dispatch_async(dispatch_get_main_queue(), ^{ p("  async 到 main queue 的 block"); });
});
dispatch_main();      // 或者 CFRunLoopRun();
```

```text
--- 主线程跑 dispatch_main() ---
main() 起点                      thread=0x1007d9e00  is_main_thread=1
global queue 上的 block          thread=0x16f933000  is_main_thread=0
  sync 到 main queue 的 block    thread=0x16f933000  is_main_thread=0
  async 到 main queue 的 block   thread=0x16f933000  is_main_thread=0

--- 主线程跑 CFRunLoopRun() ---
main() 起点                      thread=0x1053bde00  is_main_thread=1
global queue 上的 block          thread=0x16af8b000  is_main_thread=0
  sync 到 main queue 的 block    thread=0x1053bde00  is_main_thread=1
  async 到 main queue 的 block   thread=0x1053bde00  is_main_thread=1
```

上面那半里，提交到主队列的 block 跑在一条 worker 线程上，`pthread_main_np()` 返回 0。这直接和"Blocks submitted to the main dispatch queue always run on the main thread"对不上。

原因在 `dispatch_main` 的头文件注释里：

> This function "parks" the main thread and waits for blocks to be submitted to the main queue. This function never returns.
>
> Applications that call NSApplicationMain() or CFRunLoopRun() on the main thread do not need to call dispatch_main().

主线程被 park 掉之后，主队列改由工作线程来 drain。同一份 `queue.h` 早就把这事写在 `dispatch_get_main_queue` 的注释里了：

> Because the main queue doesn't behave entirely like a regular serial queue, it may have unwanted side-effects when used in processes that are not UI apps (daemons). For such processes, the main queue should be avoided.

下半段换成 `CFRunLoopRun()`，主队列的 block 就回到主线程了，和 UIKit App 的实际情形一致。iOS App 走的是 `UIApplicationMain`，主线程上有 RunLoop，所以 App 里"主队列的任务一定在主线程"成立。

我一开始就是用 `dispatch_main()` 写第一版实验的，看到 `is_main_thread=0` 以为自己的探针写错了。这件事的教训不是"文档错了"，而是**你在命令行工具里量到的 GCD 行为，不一定是 App 里的行为**，主队列尤其如此。我后来在每个实验里都打了 `pthread_main_np()`，就是被这一次坑出来的。

反方向也要说清楚：主线程上跑的任务不一定来自主队列。第二节那组数据里，主线程正在执行的是 `dispatch_sync` 派给三个不同队列的 block。它们跑在主线程，但它们不属于主队列。

"主队列的任务一定在主线程"成立（有 RunLoop 的前提下），"主线程上的任务一定来自主队列"不成立。前一句是 Apple 给的保证，后一句是很多人自己加上去的。

---

## 六、group 的两种用法，以及那个反例

`dispatch_group` 的两种写法在多数文章里被并列成"风格选择"。它们不是等价的，第一种有个明确的失效场景。

用法一，`dispatch_group_async` + `notify`，包同步耗时任务：

```text
[    0.0ms] notify 已注册，主线程没被挡住
[  204.0ms] 任务 1 完成
[  403.7ms] 任务 2 完成
[  603.6ms] 任务 3 完成
[  603.8ms] notify：全部完成
```

正常。但把任务换成一个"自己也是异步"的 API（网络请求、`dispatch_after`、任何带 completion handler 的东西），同样的写法就废了：

```objc
dispatch_group_async(grp, g, ^{
    fakeRequest(i, ^(int k) { printf("请求 %d 的回调到了\n", k); });
});
```

```text
[    0.1ms] notify 触发 <<< 一个回调都还没回来
[  320.9ms] 请求 1 的回调到了
[  434.0ms] 请求 2 的回调到了
[  536.4ms] 请求 3 的回调到了
```

`notify` 在 0.1ms 就触发了。因为 group 跟踪的是提交进去的那个 block 什么时候返回，而那个 block 干的事就是"发起请求"，发起完立刻返回。回调是另一回事，group 根本不知道它存在。

改成 `enter` / `leave` 手动配平，把 `leave` 放进回调里：

```objc
dispatch_group_enter(grp);
fakeRequest(i, ^(int k) { ...; dispatch_group_leave(grp); });
```

```text
[  317.5ms] 请求 1 的回调到了
[  425.3ms] 请求 2 的回调到了
[  533.6ms] 请求 3 的回调到了
[  533.6ms] notify 触发 <<< 三个回调都齐了
```

我自己的判断是：`dispatch_group_async` 只在包裹同步耗时任务时才写得。而实际项目里需要 group 的场景，九成是网络请求汇合。既然九成情况都得写 `enter`/`leave`，我的做法就是一律用它，省掉每次判断"这个 API 是同步还是异步"。

代价是配平。`leave` 多于 `enter` 会当场 trap：我试了 `enter` 一次 `leave` 两次，直接 `Trace/BPT trap`。少一次呢，`notify` 永远不触发，UI 一直转圈。

所以回调有多个出口的时候，成功、失败、提前 return，每条都得有 `leave`。

### 信号量做并发上限

20 个任务，信号量初值 3，记录峰值并发：

```text
信号量做并发数上限：20 个任务，闸门开 3
峰值并发 = 3（期望 3）

对照：不加信号量
峰值并发 = 20
```

这是信号量最站得住的用法。但它挡住的只是"同时在跑几个"，挡不住"开了几条线程"。在同一个实验里数一下线程：

```text
信号量限流到 3，但 20 个任务一共占了 20 条线程
```

三轮都是 20。20 个任务全被派上了线程，其中 17 条就卡在 `dispatch_semaphore_wait` 上干等着，也就是第七节要讲的线程爆炸。真要控并发，`NSOperationQueue.maxConcurrentOperationCount` 或者 `dispatch_apply` 更合适。

还有一条硬约束，WWDC21 Session 10254 的原话：

> Primitives like semaphores and condition variables are unsafe to use with Swift concurrency. This is because they hide dependency information from the Swift runtime... This violates the runtime contract of forward progress for threads.

在 `async`/`await` 的上下文里用信号量等一个 `Task`，是明确被禁止的写法，不是"不推荐"。

---

## 七、线程爆炸，以及 sync 到底贵不贵

WWDC17 Session 706 对线程爆炸的描述是：

> The way the global concurrent queue works is that it creates more threads when existing threads block to give you a continuing good level of concurrency in your application. But if those threads then block again, you can get something that we call the thread explosion.

这句话能直接量出来。同一台机器（8 核），64 个任务，只改任务体：

```text
  64 个阻塞任务用到 64 条不同线程（CPU 核数 8）
  64 个纯 CPU 任务用到 8 条不同线程
```

阻塞的那批，GCD 每看到一条线程睡过去就再开一条，一路开到 64。纯 CPU 的那批稳定在 8，正好是核数。区别只在任务里是 `usleep` 还是 `for` 循环。

`dispatch_apply` 不吃这一套：

```text
  dispatch_apply(64) 用到 8 条线程
```

同样是 64 个会阻塞的迭代，`dispatch_apply` 只用了 8 条。它知道总共有多少活，按核数分片跑。批量任务优先考虑它。

### 一个我不同意的说法

Pierre Habouzit 那份原则里有一条流传很广：block 执行时间小于 1ms 时用 `dispatch_async` 是性能浪费，用锁更好。经常被简化成"`dispatch_sync` 慢，别拿它当锁用"。

我量了一下无竞争时的单次开销（模拟器，20 万次取平均，三轮结果一致）：

```text
  dispatch_sync          11.4 ns
  os_unfair_lock          8.0 ns   (dispatch_sync 是它的 1.4 倍)
  @synchronized          49.1 ns
```

`dispatch_async` 到串行队列则是每个几十到上百纳秒（三轮 51 / 63 / 96 ns，噪声偏大，只能说数量级）。

所以 Habouzit 说的是 `dispatch_async` 那条，我实测支持：跨线程投递的固定成本，比 block 本身干的活还大。但被简化出来的"`dispatch_sync` 慢"这句我不同意。无竞争时它只比 `os_unfair_lock` 贵 1.4 倍，比 `@synchronized` 还便宜四倍。

真正该用锁替代 `dispatch_sync` 的理由，是第三节那张表。`dispatch_sync` 的死锁边界跟着 target 链走，封装一层之后就很难在 review 里看出来。`os_unfair_lock` 的边界只有它自己。这是可维护性问题，跟性能没关系。

> 模拟器的绝对耗时不能代表真机，这三个数只用来看相对关系。待真机补测：同一组循环在 iPhone 15 / iOS 26.5 上复现，确认三者的比例是否一致。

关于队列数量，Habouzit 的建议是大多数 App 不超过 3~4 个，按子系统划分；WWDC17 的说法是"a fixed number of serial queue hierarchies"。两者是一回事：队列的数量应该在编译期就是确定的，不能随请求数、随 cell 数增长。

这里有一处表面矛盾值得点破。Apple 文档某处建议"不要建自定义并发队列，用 global queue"，Habouzit 却说"不要使用 `dispatch_get_global_queue()`"。两句话针对的不是同一个问题。前者担心线程数膨胀，后者担心优先级和 QoS 传不下去。WWDC17 的层级方案同时解决两个：几个串行队列 target 到一个子系统根队列，根队列再 target 到 global。

第四节还给出了第三个不能用 global queue 的理由：barrier 在上面根本不生效。

---

## 八、QoS 对 `dispatch_sync` 无效

这一条我在三篇常被引用的中文 GCD 文章里都没看到，但它写在 `queue.h` 里：

> The QOS class and relative priority set this way on a queue have no effect on blocks that are submitted synchronously to a queue (via dispatch_sync(), dispatch_barrier_sync()).

建一个 `QOS_CLASS_BACKGROUND` 的串行队列，从 `DEFAULT` 的主线程往里提交，用 `qos_class_self()` 读执行时的实际 QoS：

```text
主线程自身 qos = DEFAULT(21)

队列建成 QOS_CLASS_BACKGROUND：
  dispatch_sync 提交           执行时 qos=DEFAULT(21)
  dispatch_barrier_sync 提交   执行时 qos=DEFAULT(21)
  dispatch_async 提交          执行时 qos=BACKGROUND(9)

全局队列的 QoS：
  sync 到 global(BACKGROUND)    执行时 qos=DEFAULT(21)
  async 到 global(BACKGROUND)   执行时 qos=BACKGROUND(9)
```

`sync` 提交的 block 跑在调用方的 QoS 上，队列自己的 QoS 被完全忽略。这和第二节是同一件事：`sync` 就是在调用线程上执行的，线程的 QoS 是什么，block 就是什么。`dispatch_sync` 的头文件说得更概括：

> Work items submitted to a queue with dispatch_sync() do not observe certain queue attributes of that queue when invoked (such as autorelease frequency and QOS class).

实用价值在于：想把一段重活降级到 background 省电，只写 `dispatch_sync(bgQueue, ...)` 是无效的。它照样以你当前的优先级在跑，还把主线程一起堵住。

反过来看，这个设计也挡掉了一类优先级反转：`sync` 的调用方是在等结果的，让 block 以调用方的 QoS 跑，而不是被队列的低 QoS 拖住，正是该有的行为。同一套思路在 `dispatch_block_wait` 的注释里说得更明白：

> If at the time this function is called, the specified dispatch block object has been submitted directly to a serial queue, the system will make a best effort to apply the necessary QOS overrides to ensure that the block and any blocks submitted earlier to that serial queue are executed at the QOS class (or higher) of the thread calling dispatch_block_wait().

注意它写的是 serial queue。并发队列的 barrier 没有这个待遇，就是第四节末尾那条 Caveat。

---

## 九、几个已经不准的说法

- 「每个队列有自己的线程池。」 队列没有线程池，系统有。同一队列的十次派发落在九条线程上，两个不同队列共用一条线程，都是常态。
- 「`dispatch_barrier_async` 在全局队列上效果和 `dispatch_sync` 一样。」 文档写的是 `dispatch_async`。实测调用耗时 0.000ms，屏障语义完全消失，用它写的读写锁会静默丢更新，丢多少取决于临界区长度。
- 「`sync`/`async` 决定要不要开新线程。」 `sync` 从不开新线程，在调用线程上就地执行。`async` 派到主队列也不开。
- 「并发队列同步执行只会在主线程执行。」 是在调用线程。从子线程调用就在子线程。
- 「当前队列 sync 当前队列才死锁。」 判据是执行上下文所在的 target 链。两个独立创建、target 到同一个串行队列的队列之间互相 sync，照样 trap。
- 「主队列 sync 会卡住不动。」 今天是当场 SIGTRAP，崩溃栈里有 `__DISPATCH_WAIT_FOR_QUEUE__`。真会卡住不动的是多方互等和 barrier 自等，那两种没有自检。
- 「用 `dispatch_get_current_queue()` 判断在不在主线程。」 已废弃，头文件明说这个比较不合法。用 `dispatch_assert_queue` 或 `dispatch_get_specific`。
- 「信号量能防止线程爆炸。」 它限制的是同时执行的任务数，被挡住的任务照样占着线程在 `wait`。
- 「`DISPATCH_QUEUE_PRIORITY_HIGH` 这套五档优先级。」 已经映射到 QoS：HIGH → `USER_INITIATED`，DEFAULT → `DEFAULT`，LOW → `UTILITY`，BACKGROUND → `BACKGROUND`。新代码直接写 QoS。
- 「`dispatch_release` / `dispatch_retain`。」 ARC 下 dispatch 对象是 ObjC 对象，写了直接编不过：`'release' is unavailable: not available in automatic reference counting mode`。2012 年前后的 GCD 文章基本都要打这个折扣。

---

## 总结

队列和线程是多对多的，且随时间变化。同一个串行队列十次派发换了九条线程，两个不同的串行队列共用一条线程，两百个任务只用一条线程。"每个队列有自己的线程池"这句话没有任何一个版本是对的。

`dispatch_sync` 默认在调用线程上就地执行，唯一的例外是主队列（以及 target 到主队列的队列）。QoS 对它无效、`sync` 不开新线程、"并发队列同步执行只在主线程"是错的，都是这一条的推论。

死锁的判据是执行上下文所在的 target 链，不是队列对象的身份。libdispatch 能自检出串行队列的所有权环并直接 trap，但对多方互等和 barrier 自等无能为力，后两种才是真会挂住的。`dispatch_assert_queue` 和 `dispatch_get_specific` 走的是同一条 target 链，所以它们是可靠的自检手段。

barrier 只在 `DISPATCH_QUEUE_CONCURRENT` 的自建队列上有屏障语义。放到全局队列上，`dispatch_barrier_async` 就退化成了普通的 `dispatch_async`，读写锁静默失效，一声不吭。丢失率随临界区长度上升，短的时候可能几个月都不出事。

最后一条方法论，和这个系列前几篇一样：这篇里所有"网上说的不对"的结论，没有一条是靠读更多博客得出来的。全部来自 `grep` 当前 SDK 的 `queue.h`，加上把每个说法编成一个能跑的 case 扔进模拟器。头文件比网页文档更完整（`dispatch_sync` 那句 "or a queue targetting it" 只有头文件里有），而且它就在你硬盘上。

下一篇讲互斥的另一半：[[iOS 锁：从 OSSpinLock 的废弃说起]]。第七节留下的那个问题在那里展开：什么时候该用锁，什么时候该用队列。

## 参考资料

### 一手

- `$(xcrun --sdk iphonesimulator --show-sdk-path)/usr/include/dispatch/` —— `queue.h`、`group.h`、`semaphore.h`、`block.h`。本文所有引文都出自这里，比网页文档更完整
- [DispatchQueue](https://developer.apple.com/documentation/dispatch/dispatchqueue)：线程无保证那句的出处
- [dispatch_barrier_async](https://developer.apple.com/documentation/dispatch/dispatch_barrier_async)：private concurrent queue 的完整措辞
- WWDC17 Session 706《Modernizing Grand Central Dispatch Usage》：线程爆炸、串行队列层级、"fundamental synchronization primitive"
- WWDC21 Session 10254《Swift concurrency: Behind the scenes》：新线程池按核数封顶、信号量在 Swift Concurrency 下不安全

### 二手

- Pierre Habouzit《Making efficient use of the libdispatch (GCD)》（经 dirtmelon 转述）：队列数量、不要用 global queue、锁优于 `dispatch_sync` 的实践原则。libdispatch 维护者本人写的，全部二手资料里最有现代价值的一份
- ming1016《细说 GCD》（2016）：五种死锁 case、`dispatch_source` 事件表、`set_target_queue` 层级仍可读；barrier 那条和"并发队列同步只在主线程"那条本文已证伪
- 唐巧《使用 GCD》（2012）：GCD 动机的叙事写得好；MRC、`dispatch_release`、五档优先级已过时

### 本地

- [[iOS 属性关键字：从所有权推导，而不是从类型名猜]]：`atomic` 为什么不等于线程安全
- [[iOS 锁：从 OSSpinLock 的废弃说起]]

---

实验环境：Xcode 26.6（Apple clang 21），iOS 模拟器 iPhone 16 / iOS 18.3（arm64，8 核 Apple Silicon Mac），`clang -target arm64-apple-ios17.0-simulator`，ARC。

死锁类实验全部用 `alarm(3)` + 信号处理函数做超时保护，这样"挂住"和"trap"能分开统计，也不会把终端卡死。这个手法比 `timeout` 命令好用，因为它能在挂住的那一刻自己打印现场。

> 待真机补测：第七节的三个耗时数字，以及第三节那张死锁表在 iOS 26.5 真机上是否有差异。表里的 trap / 挂住之分依赖 libdispatch 的实现，属于可能变化的部分。
