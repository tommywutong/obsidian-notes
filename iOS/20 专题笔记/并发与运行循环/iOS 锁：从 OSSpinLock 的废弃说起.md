---
title: 【iOS】锁：从 OSSpinLock 的废弃说起
published: 2026-07-27
description: OSSpinLock 被废弃十年后，我实测它依然是最快的锁，每次 7.00 ns，比 os_unfair_lock 还快 14%。它被废弃跟快慢无关。一秒的窗口里，等它的高优先级线程一次锁都没抢到。
tags:
  - iOS
  - 并发
  - 锁
  - Objective-C
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 16
draft: true
---
# 锁：从 OSSpinLock 的废弃说起

先说一个我自己没料到的实测结果。

`OSSpinLock` 从 iOS 10 起就被标记为废弃，中文圈的标准解释是"它不安全"。可我把它和现役的六把锁放在一起，无竞争的情况下跑两百万次加解锁，七轮取最小值，它是全场最快的：每次往返 7.00 ns，比官方指定的替代品 `os_unfair_lock`（8.11 ns）还快 14%。跑两遍，两遍都是这个名次。

所以"不安全"到底指什么？我用 16 条高优先级线程去抢一把被低优先级线程持有的锁，测每条线程等了多久。一秒的窗口，四轮：

```text
              最坏单次等待
OSSpinLock       1049 ms / 1097 ms / 1057 ms / 1099 ms
os_unfair_lock   18.7 ms /  12.1 ms /  16.5 ms /  13.1 ms
```

四轮里每一轮，都有至少一条高优先级线程从窗口开始一直等到窗口结束，一次锁都没拿到。而同样的场景换成 `os_unfair_lock`，最坏也只等 18.7 ms。

这就是废弃的理由，跟快慢无关。上一篇 [[iOS GCD：队列不是线程，以及死锁的准确边界]] 讲了互斥的一半（队列），这一篇讲另一半。

---

## 一、优先级反转，准确地说是什么

`OSSpinLockDeprecated.h` 开头那段话已经把话说得很明白了：

> These interfaces should no longer be used, particularily in situations where threads of differing priorities may contend on the same spinlock.

关键词是"不同优先级的线程"。单看这句还不够，得知道自旋锁在等待时干了什么。

自旋锁的等待就是一个死循环：反复读那个锁变量，直到它变成"未上锁"。这个循环要占着 CPU 才能跑。于是当一条低优先级线程正持有锁、而高优先级线程在自旋等待时，系统看到的是：高优先级线程一直是就绪的、一直有活干，低优先级线程可以往后排。可低优先级线程不被调度，就永远走不到 `unlock` 那一行。高优先级线程于是继续自旋。

两边就这么僵住。高的等低的放锁，低的等高的让出 CPU。

这个机制在苹果自家的平台上被放大了两次。第一次是 QoS：iOS/macOS 的线程优先级由 QoS 决定，`QOS_CLASS_BACKGROUND` 和 `QOS_CLASS_USER_INTERACTIVE` 之间的差距很大。第二次是 Apple Silicon 的大小核：`BACKGROUND` 的线程会被安排到能效核上跑。一条被限制在能效核上的线程，和一群霸占着性能核疯狂自旋的线程抢同一把锁，结局是可以预料的。

上面那组 1049 ms 就是这么来的。测的时候我用一条 `QOS_CLASS_BACKGROUND` 线程反复"持锁 → 干几十微秒的活 → 解锁"，16 条 `QOS_CLASS_USER_INTERACTIVE` 线程同时抢同一把锁，窗口一秒。完整的四轮数据：

```text
              高优先级共抢到    平均等待     最坏单次等待   低优先级完成
OSSpinLock       17987169 次   0.0009 ms    1049.358 ms      1952 次
                      658 次  25.3339 ms    1097.857 ms      9661 次
                     1231 次  13.6169 ms    1057.163 ms     10125 次
                  2370919 次   0.0070 ms    1099.476 ms      8725 次

os_unfair_lock    7946352 次   0.0017 ms      18.723 ms       990 次
                  7930408 次   0.0016 ms      12.140 ms      1003 次
                  8374318 次   0.0015 ms      16.496 ms       594 次
                  7644557 次   0.0018 ms      13.111 ms      2054 次
```

`OSSpinLock` 那四行的吞吐在 658 次和 1798 万次之间跳，相差四个数量级。这种双峰分布是活锁的典型特征：要么顺利，要么整个窗口都卡死。`os_unfair_lock` 四轮全在 764 万到 837 万之间，抖动不到 10%。

平均值在这里几乎没有信息量。第一轮和第四轮 `OSSpinLock` 的平均等待是 0.0009 ms 和 0.0070 ms，比谁都漂亮。要看最坏值才能看见问题。UI 卡顿是尾延迟问题，不是平均值问题。

### 我没能复现的那一半

有一条我要说清楚：低优先级持锁者被彻底饿死这个更极端的场景，**我在 M1 Pro 上没测出来**。

我专门设计了一组：一条 `BACKGROUND` 线程拿着锁去算一段定量的浮点活，16 条 `USER_INTERACTIVE` 线程等锁，另外再拿 24 条 `USER_INITIATED` 线程把 8 个核全部压满。如果存在"等待者把优先级捐给持锁者"的机制，持锁者应该明显变快；如果 `OSSpinLock` 没有这个机制，它应该明显变慢。结果五轮下来，四种锁的持锁者都在 26 到 39 ms 之间，分不出来：

```text
（无锁，无等待者）  最快 28.0 ms   最慢 84.5 ms
OSSpinLock          最快 28.1 ms   最慢 38.0 ms
os_unfair_lock      最快 26.4 ms   最慢 39.2 ms
pthread_mutex       最快 26.4 ms   最慢 32.0 ms
dispatch_semaphore  最快 27.5 ms   最慢 32.6 ms
```

同一组里高优先级线程的等待时间倒是稳定分出了名次，五轮中每一轮 `OSSpinLock` 都是最差的（94.2 ~ 108.4 ms），`pthread_mutex` 最稳（81.2 ~ 85.0 ms）。但持锁者本身没被饿死。

macOS 的调度器不会让一条 `BACKGROUND` 线程完全拿不到 CPU，8 个核也确实宽裕。所以我只能说：在这台机器、这个系统版本上，我复现出来的是高优先级等待者被饿死，不是低优先级持锁者被饿死。真机核少、功耗墙更紧，情况可能更糟，但那是我测不到的部分。

> 待真机补测：同一组实验在 iPhone 15 / iOS 26.5 上跑一遍，确认低优先级持锁者是否会被饿死。这一节的结论目前只在 8 核 M1 Pro 上成立。

### 一个我自己撞上的现场

写这组实验的时候我踩了个坑，坑本身比实验还能说明问题。

为了让所有线程同时开跑，我写了一个起跑闸：每条线程就位后把计数加一，然后自旋等一个 `atomic_int` 变成 1。主线程看到所有人就位就把它置 1。

程序挂住了，十分钟没有任何输出。`sample` 抓下来是这样：

```text
    315 Thread_15550544
    +     315 low_thread  (in inversion4) + 40
    315 Thread_15550545
    +     315 hi_thread  (in inversion4) + 80,76
    ...（16 条 hi_thread 全部停在同样两条指令上）
```

16 条 `USER_INTERACTIVE` 线程卡在自旋闸的那两条指令上，把 CPU 全吃掉了。那条 `BACKGROUND` 线程连闸口都没走到，计数永远差 1，主线程永远不会开闸。

我为了测优先级反转，先亲手写了一个优先级反转。而且它不涉及任何一把锁，光是"高优先级线程忙等一个低优先级线程要设置的变量"就够了。把起跑闸换成 `dispatch_semaphore`（阻塞式等待，等待的线程会让出 CPU），同一份代码立刻就跑通了。

自旋等待的危险不在锁里，在自旋本身。

---

## 二、os_unfair_lock 不是自旋锁

名字里带 lock 不带 spin，这是有意的。`os/lock.h` 对它的第一句描述是：

> Low-level lock that allows waiters to block efficiently on contention.

阻塞，不是自旋。等不到锁的线程会进内核睡下去，把 CPU 让出来。真正关键的是下面这句：

> The values stored in the lock should be considered opaque and implementation defined, they contain thread ownership information that the system may use to attempt to resolve priority inversions.

锁变量里存着持有者的身份。等待者进内核睡觉的时候，把"我在等谁"这条信息一并交给了内核。内核于是有机会把等待者的优先级临时借给持有者，让它快点跑完临界区。`OSSpinLock` 是一个裸的 `int32_t`，里面除了"锁没锁"什么都没有，内核根本不知道该去救谁。

这就是"有单一已知持有者"的价值。WWDC17 Session 706 讲串行队列时用了同一个说法：

> Primitives with a single known owner have this power. Things like serial queues and OS unfair lock.

所有权信息不是为了防止你写错代码，是为了让内核能解开优先级环。这一点是理解这一整篇的钥匙：后面 `dispatch_semaphore` 为什么不该当锁用，答案也在这里。

### unfair 是什么意思

头文件专门解释了这个名字：

> The name 'unfair' indicates that there is no attempt at enforcing acquisition fairness, e.g. an unlocker can potentially immediately reacquire the lock before a woken up waiter gets an opportunity to attempt to acquire the lock. This is often advantageous for performance reasons, but also makes starvation of waiters a possibility.

解锁的线程可以立刻再把锁抢回去，不用排队。这在读写一个共享状态的循环里是巨大的性能优势，省掉了唤醒和上下文切换。代价写在最后半句：等待者有被饿死的可能。

我的数据能看到这个代价。第一节那组里，`os_unfair_lock` 那四轮的低优先级线程只完成了 990 / 1003 / 594 / 2054 次，而 `pthread_mutex` 是 1506 / 1470 / 1831 / 1290 次。`os_unfair_lock` 给高优先级等待者的延迟更低更稳，但对低优先级线程更狠。这是一个明确的取舍，不是免费的。

### trylock 不能写重试循环

这条藏在 `os_unfair_lock_trylock` 的文档里，很少有人提：

> It is invalid to surround this function with a retry loop, if this function returns false, the program must be able to proceed without having acquired the lock, or it must call os_unfair_lock_lock() directly (a retry loop around os_unfair_lock_trylock() amounts to an inefficient implementation of os_unfair_lock_lock() that hides the lock waiter from the system and prevents resolution of priority inversions).

`while (!os_unfair_lock_trylock(&l)) {}` 会把一把设计良好的锁重新变成 `OSSpinLock`。等待者藏起来了，内核看不见，优先级反转又回来了。见过有人这么写"避免阻塞"，效果正好相反。

### iOS 18 之后它可以自旋了

`os/lock.h` 里新增了一个标志（macOS 15 / iOS 18 起）：

> `OS_UNFAIR_LOCK_FLAG_ADAPTIVE_SPIN`
> This flag allows the caller of os_unfair_lock_lock_with_flags API to spin temporarily before blocking, particularly useful when the holder of the lock is on core. This should only be used for locks where the protected critical section is always extremely short.

配合 `os_unfair_lock_lock_with_flags` 使用。绕了一圈，自旋又回来了，但这次是有条件的：先自旋一小会儿，不成再阻塞，而且全程保留所有权信息。"持有者正在核上跑"这个前提是自旋唯一说得通的场景。适用条件写得很死。临界区必须极短。

### OSSpinLock 今天还剩什么

有件事值得一看。`OSSpinLockDeprecated.h` 里有三套实现，取决于宏怎么定义。默认那套是真实的独立函数；定义 `OSSPINLOCK_USE_INLINED=1` 会得到一组内联转发：

```c
OSSPINLOCK_INLINE
void
OSSpinLockLock(volatile OSSpinLock *__lock)
{
	os_unfair_lock_t lock = (os_unfair_lock_t)__lock;
	return os_unfair_lock_lock(lock);
}
```

还有第三套，把 `OSSpinLockLock` 宏替换成 `_os_nospin_lock_lock`。名字里写着 nospin。

我查了一下这三个符号在当前系统上的实际去向：

```text
OSSpinLockLock         -> 0x18e13b6d8  (/usr/lib/system/libsystem_platform.dylib)
_os_nospin_lock_lock   -> 0x18e13965c  (/usr/lib/system/libsystem_platform.dylib)
os_unfair_lock_lock    -> 0x18e138be8  (/usr/lib/system/libsystem_platform.dylib)
```

三个不同的地址，三份独立的实现都还在。所以默认编译出来的 `OSSpinLockLock` 调的确实是老代码，第一节那组 1049 ms 也确实是老代码跑出来的。苹果没有偷偷把它换掉，只是给了你换的开关。

还有一点，`OSSpinLockLock` 的文档写的是"Although the lock operation spins, it employs various strategies to back off if the lock is held"，它并不是最朴素的死循环，有退避策略。退避解决不了优先级反转，因为问题不在于自旋消耗了多少 CPU，在于内核不知道该提谁的优先级。

---

## 三、重测那张流传极广的性能图

ibireme 2016 年那篇《不再安全的 OSSpinLock》里有一张锁性能对比图，十年来被引用了无数次。那是 2016 年的 iPhone 数据。我在 macOS 上重测了一遍。

方法：每种锁两百万次 `lock` / 临界区自增一次 / `unlock`，各种锁交错跑同一轮，七轮取每种锁的最小值。整个程序跑两遍确认名次稳定。

```text
=== 无竞争单线程，每次 lock+unlock 平均耗时（macOS arm64 原生，M1 Pro）===
 1. 裸自增（基线）           0.32 ns
 2. atomic_fetch_add(relaxed) 2.15 ns
 3. OSSpinLock（已废弃）      7.00 ns
 4. os_unfair_lock            8.11 ns
 5. pthread_mutex             8.75 ns
 6. NSCondition               8.76 ns
 7. NSLock                    9.03 ns
 8. dispatch_sync（串行队列）  9.30 ns
 9. dispatch_semaphore        9.77 ns
10. pthread_mutex（递归）     10.51 ns
11. pthread_rwlock（读）      11.93 ns
12. pthread_rwlock（写）      13.11 ns
13. NSRecursiveLock          15.46 ns
14. @synchronized            21.13 ns
15. NSConditionLock          34.13 ns
```

第二遍跑出来只有第 5、6 名互换（8.80 / 8.82），其余名次一字不差。5 到 9 名挤在 8.75 到 9.77 之间，这个区间内部的名次我不认为有意义，应该当成一档看。

几条和那张老图不一样的地方：

`NSLock` 只比 `pthread_mutex` 慢 3%。它就是 `pthread_mutex` 加一层 ObjC 消息发送，这层的成本在今天已经小到测不太出来了。"`NSLock` 比 `pthread_mutex` 慢很多"这个说法可以退休了。

`dispatch_semaphore` 排在第 9，落在 `NSLock` 后面。老图里它是仅次于 `OSSpinLock` 的第二名，常被推荐成"最快的锁"。今天它不是。

`dispatch_sync` 到串行队列是 9.30 ns，比 `os_unfair_lock` 只贵 15%。这个数字和本系列 GCD 那篇在模拟器上测到的 11.4 ns 对得上（模拟器整体偏慢）。"`dispatch_sync` 比锁慢很多"这句话，无竞争的情况下不成立。

### 那个 2.4 倍的差异，值得单独讲

同一台机器上，另一次独立的运行给出了一组不同的数字。那次是编成 iOS 模拟器二进制跑的（`clang -target arm64-apple-ios17.0-simulator`），不是 macOS 原生：

```text
OSSpinLock          8.09      pthread_mutex(递归)   11.27
os_unfair_lock      9.11      dispatch_sync         13.29
NSCondition         9.42      pthread_rwlock(读)    13.56
NSLock              9.79      pthread_rwlock(写)    14.02
pthread_mutex      10.11      NSRecursiveLock       17.13
dispatch_semaphore 10.89      NSConditionLock       40.64
                              @synchronized         51.15
```

把两组放在一起看，绝大多数锁在两个环境下差 10% 到 15%，名次几乎完全一致。只有一个例外：

```text
                 macOS 原生    iOS 模拟器
@synchronized      21.13 ns      51.15 ns      差 2.4 倍
NSConditionLock    34.13 ns      40.64 ns      差 1.2 倍
```

`@synchronized` 差了 2.4 倍，而且它和 `NSConditionLock` 的名次在两个环境里是反的。在模拟器上 `@synchronized` 是全场最慢，在 macOS 原生上它排第 14，比 `NSConditionLock` 快得多。

"@synchronized 是最慢的锁"这句几乎所有中文资料都写过的话，取决于你在哪测。

我一开始怀疑是测量方法不同，把两份代码逐行对过：临界区都是一次 `g_sink++`，锁对象都是一个复用的 `[NSObject new]`，都是七轮取最小，都开了 ARC。唯一的差别就是编译目标。第三个数据点可以佐证：本系列 GCD 那篇在模拟器上测到的 `@synchronized` 是 49.1 ns，和这里的 51.15 对得上，两次独立的模拟器测量互相印证。

结论是 `objc_sync_enter` / `objc_sync_exit` 这条路径在模拟器上明显更贵，而 pthread 和 dispatch 那些原语在两个环境下基本一致。模拟器跑的是 iOS 的 libobjc，macOS 跑的是 macOS 的 libobjc，两份实现不一样。

对写文章的人来说，教训比数字重要：这类锁性能排行榜的**绝对值和名次都不可移植**，看到一张图先问它在什么平台上测的。 ibireme 那张图是 2016 年的 iPhone，我这张是 2026 年的 M1 Pro macOS 原生，两张都对，都只对自己那一栏。

还有一个更小的差异也值得记一笔：那组数据里"原子自增"是 8.29 ns，我这边是 2.15 ns，差近四倍。原因是内存序不同，他们用的是 `__ATOMIC_SEQ_CST`，我用的是 `memory_order_relaxed`。两个都叫"原子自增"，成本差四倍。报数字的时候不写内存序，这个数字就没有意义。

---

## 四、@synchronized：它到底做了什么

`@synchronized (obj) { ... }` 编译后展开成 `objc_sync_enter(obj)` 和 `objc_sync_exit(obj)`，外面套一层异常清理，保证从 block 里 `return` 或者抛异常时也能解锁。SDK 里 `objc/objc-sync.h` 的声明是这样的：

```c
/** 
 * Begin synchronizing on 'obj'.  
 * Allocates recursive pthread_mutex associated with 'obj' if needed.
 * ...
 * @return OBJC_SYNC_SUCCESS once lock is acquired.  
 */
OBJC_EXPORT int
objc_sync_enter(id _Nonnull obj);

enum {
    OBJC_SYNC_SUCCESS                 = 0,
    OBJC_SYNC_NOT_OWNING_THREAD_ERROR = -1
};
```

两个细节。它是递归锁（"recursive"），后面会验证。返回值有两种，而 `@synchronized` 语法把返回值整个丢掉了。

对象和锁之间的映射由运行时维护：一张按对象地址分片的全局表，加上线程本地的缓存。上一篇 [[iOS 内存管理：从 MRC、ARC 到属性关键字#属性关键字：把所有权写进 API|属性关键字：从所有权推导，而不是从类型名猜]] 里讲 `atomic` 时提过同一个数据结构 `StripedMap`，`PropertyLocks` 用的就是它，真机上只有 8 个分片、Mac 和模拟器上是 64 个。`@synchronized` 的对象表是同一个套路。

### 一个我猜错了的实验

按 objc4 公开源码的写法，那张全局表的每个分片是一条链表，查找要线性遍历。我据此做了个预测：用来当锁的不同对象越多，链表越长，`@synchronized` 应该越慢。

测出来完全不是这样。让单线程轮流锁 N 个不同的对象，N 从 1 涨到 32768：

```text
不同对象数     每次耗时      相对 1 个对象
1              20.77 ns      1.00x
8              20.36 ns      0.98x
64             20.66 ns      0.99x
512            19.33 ns      0.93x
2048           19.49 ns      0.94x
8192           19.38 ns      0.93x
32768          19.30 ns      0.93x
```

一条平线，甚至还微微下降。

我不甘心，又做了个更尖锐的对照。链表是头插的，所以最早插入的对象应该在链表最末端，最晚插入的应该在最前端。往表里放两万个对象，然后分别锁这两个：

```text
锁 objs[0]      （理论上在链表最末端）  20.51 ns
锁 objs[19999]  （理论上在链表最前端）  20.67 ns
锁一个刚建的新对象                     20.75 ns
os_unfair_lock 参照                     7.94 ns
```

位置完全不影响。三个数字在 1% 以内。

所以热路径上没有发生链表遍历。至于当前发布的 libobjc（这台机器上是 951.7）用的是什么查找结构，我从公开源码推不出来，也没有拆它。实测结论是：**`@synchronized` 的开销和锁对象的数量无关**，从 1 个到 32768 个都是 20 ns 上下。 机制我没搞清楚，就不编一个。

拆开量一下成本：手写 `objc_sync_enter` + `objc_sync_exit` 一对是 17.75 ns，`@synchronized` 是 21.13 ns，差的三点几纳秒是异常清理那层。嵌套两层是 44.92 ns，第二层比第一层还贵。

### 它是递归锁

三层嵌套同一个对象：

```objc
@synchronized (o) {
    @synchronized (o) {
        @synchronized (o) { }
    }
}
```

```text
[synchronized]
  进入第 1 层
  进入第 2 层（同一个对象，没卡住）
  进入第 3 层
  三层全部退出，正常返回
```

递归是要付钱的。`NSRecursiveLock` 15.46 ns 对 `NSLock` 9.03 ns，`pthread_mutex` 递归型 10.51 ns 对默认型 8.75 ns，都是 1.2 到 1.7 倍。递归锁要在锁里记住持有者是谁、进了几层，这些簿记跑不掉。

### @synchronized(nil) 什么都不锁

传 nil 进去会怎样？先看返回值：

```text
objc_sync_enter(nil) 返回 0，objc_sync_exit(nil) 返回 0（OBJC_SYNC_SUCCESS = 0）
```

两个都返回成功。block 体照常执行，不崩，不报错。那互斥呢？4 条线程各做 20 万次 `@synchronized (nil) { shared++; }`：

```text
期望 800000，实得 587185
```

丢了 26%。`@synchronized (nil)` 是一句什么都不做的空语句，但它长得和一句正经的加锁一模一样，而且返回"成功"。

这是这一整篇里我认为最危险的一条。写成 `@synchronized (self.delegate)` 或者 `@synchronized (_cachedThing)`，只要那个东西某个时刻是 nil，互斥就在那一刻静默消失。没有日志、没有断言、没有崩溃。

还有，头文件把参数标注成了 `id _Nonnull obj`，所以传 nil 编译器会给一条 `-Wnonnull` 警告。注解和实现对不上，实现里明确处理了 nil。

### 锁对象被换掉，互斥一样会消失

比 nil 更隐蔽的一种。`@synchronized` 锁的是求值那一刻拿到的那个对象，不是那个变量。

```objc
@synchronized (box.items) {   // items 是个 atomic 属性，随时可能被换成新数组
    ...
}
```

4 条线程各做 6 万次，同时另有一条线程不停把 `box.items` 换成新的数组。我在临界区里放了个计数器，进入时加一，如果发现值不是 1 就说明有别人和我同时在里面：

```text
锁对象用的是：box.items（会被换掉）
→ 共进入临界区 240000 次，其中 154 次发现有别的线程同时在里面

锁对象用的是：一个专门的、不会被替换的对象
→ 共进入临界区 240000 次，其中 0 次发现有别的线程同时在里面
```

154 次 / 24 万次，大约万分之六。这种概率的 bug 在测试环境基本不可能撞上，上线之后变成每天几十条来源不明的崩溃。

我自己的做法很简单：**锁对象一律用一个专门的、在 `init` 里就建好的实例变量**，不用 `self`，不用任何属性。 用 `self` 的问题是外部代码也能拿到 `self` 去 `@synchronized`，你的锁范围就不再由你控制了。

### objc_sync_exit 的返回值被吃掉了

直接调底层函数能看到 `@synchronized` 语法藏起来的东西：

```text
没 enter 直接 exit：返回 -1（OBJC_SYNC_NOT_OWNING_THREAD_ERROR = -1）
别的线程持有，本线程 exit：返回 -1
```

运行时是知道你在解一把不属于你的锁的，它把这件事报告给了调用方。`@synchronized` 语法把这个返回值丢了。这也是为什么用 `@synchronized` 写错所有权不会崩，而 `os_unfair_lock` 会当场崩。后者选择了 abort，前者选择了返回错误码，然后语法糖把错误码扔了。

---

## 五、谁锁的必须谁解

这条规则在不同的锁上执行力天差地别。我建了三把锁，主线程锁上，然后开一条真正的新线程去解：

```text
[os_unfair_lock]
  主线程 tid=0x1fa339d80  已持有
  子线程 tid=0x16ceaf000，它从没 lock 过，现在去 unlock
  → 进程崩溃

[pthread_mutex（默认类型）]
  子线程 unlock 返回 0（成功），没崩

[NSLock]
  子线程 unlock 返回了，没崩
```

`os_unfair_lock` 的崩溃信息很直白：

```text
exception: EXC_BREAKPOINT
asi: "BUG IN CLIENT OF LIBPLATFORM: Unlock of an os_unfair_lock not owned by current thread"
    libsystem_platform.dylib  _os_unfair_lock_unowned_abort
    libsystem_platform.dylib  _os_unfair_lock_unlock_slow
```

头文件里承诺过这件事：

> This lock must be unlocked from the same thread that locked it, attempts to unlock from a different thread will cause an assertion aborting the process.

`pthread_mutex` 默认类型和 `NSLock` 都是静默放行。`NSLock` 的文档说这种行为未定义，实测就是不报错。所以"`NSLock` 保证了所有权"这个说法只对了一半：它内部确实是 `pthread_mutex`，但默认类型的 `pthread_mutex` 本来就不检查所有权。

### 我第一次测错了

这个实验我第一版写成了这样：

```objc
os_unfair_lock_lock(&l);
dispatch_sync(dispatch_get_global_queue(0, 0), ^{
    os_unfair_lock_unlock(&l);      // 想在"另一条线程"上解锁
});
```

跑出来不崩。我差一点就写下"头文件说会崩，实测不崩"。

问题出在仪器上。`dispatch_sync` 会尽量在调用线程上就地执行 block，这是上一篇 [[iOS GCD：队列不是线程，以及死锁的准确边界]] 的核心结论之一。所以那个 block 压根就在主线程上跑，锁的持有者和解锁者是同一条线程，当然不崩。换成 `pthread_create` 之后立刻就崩了。

规范里那条"先怀疑仪器，再怀疑结论"，这次又应验了一遍。跨篇的结论在这里救了我：如果没写过上一篇，我大概率会把这个错误结论发出去。

---

## 六、重入：哪些锁能，哪些锁会当场挂掉

同一条线程连续加锁两次，六种锁六种反应。每个 case 单独一个进程，程序内部用 `alarm(3)` 自杀，三秒不返回就判定为挂住。

```text
NSLock                第一次成功，第二次挂住            → 死锁
pthread_mutex（默认）  第一次返回 0，第二次挂住          → 死锁
dispatch_semaphore    第一次 wait 通过，第二次挂住      → 死锁
NSRecursiveLock       三层全部通过，配平退出            → 可重入
@synchronized         三层全部通过                     → 可重入
pthread_mutex（ERRORCHECK 型）  第二次返回 11（Resource deadlock avoided）
os_unfair_lock        第二次直接崩
```

`os_unfair_lock` 那条的崩溃信息：

```text
exception: EXC_BREAKPOINT
asi: "BUG IN CLIENT OF LIBPLATFORM: Trying to recursively lock an os_unfair_lock"
    libsystem_platform.dylib  _os_unfair_lock_recursive_abort
    libsystem_platform.dylib  _os_unfair_lock_lock_slow
```

libplatform 有一个专门的 abort 函数来处理这种情况。它知道持有者是谁，发现持有者就是你自己，于是不让你等，直接终止进程。这算是所有权信息的一个副作用：能诊断出自死锁的前提是知道持有者。

有个细节值得单说，`os_unfair_lock_trylock` 在自己已经持有时不会崩：

```text
assert_owner 通过：当前线程就是持有者
同一线程再 trylock：失败（返回 false）
```

返回 false，不 abort。所以想探测"我是不是已经持有这把锁"，`trylock` 是安全的，`lock` 会要你的命。

`pthread_mutex` 有四种类型，默认的那种在 Darwin 上重入是直接挂住，不返回 `EDEADLK`。想要它报错得显式设成 `PTHREAD_MUTEX_ERRORCHECK`。`NSLock` 底下是默认类型，`NSRecursiveLock` 底下是 `PTHREAD_MUTEX_RECURSIVE`，两者的区别就在这里。

### 递归锁不是"更安全的锁"

需要递归通常说明设计有问题。一个方法拿着锁去调另一个也要拿同一把锁的方法，意味着"临界区"的边界已经模糊了。递归锁让代码不崩，但没解决那个模糊。

我自己的阈值是这样：默认写 `os_unfair_lock`，让重入直接崩掉。那个崩溃告诉我的信息，比一个能跑通的递归锁多得多。 只有在改不动的老代码里，才用 `NSRecursiveLock` 兜底。

---

## 七、把 dispatch_semaphore 当锁用的三个问题

`dispatch_semaphore_create(1)` 然后 wait / signal，这是很常见的写法。它能工作，性能也不差（9.77 ns，第 9 名）。但它不是锁，把它当锁用有三个具体的问题。

没有所有权。 信号量不记谁 wait 的。A 线程 wait、B 线程 signal 完全合法，这本来就是信号量的设计意图（它是用来做通知和限流的）。代价是内核不知道该把优先级捐给谁，第一节讲的那套机制在这里没有着力点。WWDC21 Session 10254 讲得更狠，说信号量在 Swift Concurrency 下是不安全的，因为它"hide dependency information from the Swift runtime"。同一个道理。

不支持递归。 上一节测过了，第二次 wait 直接挂住，而且没有任何诊断信息，就是安静地卡住。

争用时的吞吐塌得很厉害。 第一节那组数据里最刺眼的一列其实是它：

```text
              一秒窗口内高优先级线程抢到的次数
os_unfair_lock   7946352 / 7930408 / 8374318 / 7644557
pthread_mutex    9940194 / 11945407 / 10863226 / 11673059
dispatch_semaphore  52574 / 54812 / 92425 / 83241
```

差了两个数量级。平均等待时间也从其他锁的 0.001 ms 级跳到 0.17 ~ 0.31 ms。原因是每次争用都要走一趟内核，而 `os_unfair_lock` 和 `pthread_mutex` 在争用不激烈时能在用户态解决大部分情况。

还有一个和锁无关但很容易踩的坑。信号量析构时如果当前值小于初始值，会直接崩：

```text
asi: "BUG IN CLIENT OF LIBDISPATCH: Semaphore object deallocated while in use (current value < original value)"
    libdispatch.dylib  _dispatch_semaphore_dispose
```

我是在写另一个实验时撞上的：wait 了没配平 signal，函数返回，ARC 释放信号量，当场崩溃。用信号量做超时控制的代码很容易漏掉某条返回路径上的 signal。

它真正该用的地方是限流和等待：控制并发数、等一个异步结果、`dispatch_group` 的手工版本。保护一段共享状态用锁。

---

## 八、读写分离什么时候才划算

读多写少的场景有两个标准方案：`pthread_rwlock`（读共享、写独占），和自建并发队列上的 `dispatch_barrier`（读用 `dispatch_sync`，写用 `dispatch_barrier_sync`）。

上一篇已经讲过 barrier 的一个大坑：它只在 `DISPATCH_QUEUE_CONCURRENT` 的自建队列上有屏障语义，放到全局队列上会静默退化成普通 `dispatch_async`。这里默认已经建对了队列。

我关心的是另一个问题：读写分离到底能带来多少收益。6 条读线程 + 1 条写线程，横轴是临界区长度（一次读多少个 long），窗口 1.2 秒，四轮取最大吞吐：

```text
临界区长度   os_unfair_lock   pthread_rwlock   dispatch_barrier   rwlock/unfair
1              27,979,783        8,265,018        11,378,586         0.30x
8              32,692,492        7,693,795         9,617,266         0.24x
64             21,023,131        8,390,836         6,374,067         0.40x
512             5,925,499        7,345,901         5,259,403         1.24x
4096              446,614        1,807,401         1,829,447         4.05x
```

短临界区上读写锁是负收益，慢三到四倍。交叉点在 64 和 512 之间。到 4096 个元素时才有 4 倍的领先。

道理不难理解：读写锁要维护读者计数，这份簿记本身比一次互斥加解锁还贵。只有临界区里真的有活干、多个读线程并行执行那段活省下的时间超过了簿记成本，分离才开始赚钱。

我之前写过一版实验，临界区是一次 `NSDictionary` 取值，三种方案的吞吐差在 4% 以内，完全分不出来。因为循环里 `[NSString stringWithFormat:]` 的开销把锁的差异整个淹掉了。那组数据没有意义，是这一版把临界区长度当自变量扫了一遍才看出规律。

实用的判断是：读操作是一次字典取值、一次属性读取这种量级，直接用 `os_unfair_lock`，读写分离只会更慢。 要遍历一个大集合、要做序列化、要算点什么，再考虑分离。

---

## 九、死锁的四个必要条件

教科书上的四条，放在 iOS 的语境里：

互斥。 资源同一时刻只能被一个线程占有。这是锁的定义，去不掉。

持有并等待。 拿着一把锁的同时去要另一把。

不可剥夺。 锁只能由持有者主动释放，系统不会抢走。`os_unfair_lock` 和 `pthread_mutex` 都是这样。

循环等待。 存在一个环：A 等 B 手里的锁，B 等 A 手里的锁。

四条同时满足才会死锁，破坏任何一条就够了。实际工程里最好破坏的是最后一条，办法是给所有锁定一个全局顺序，任何线程都按这个顺序加锁。

实测一下。两条线程，一条按 A→B 的顺序，一条按 B→A：

```text
[abba]
  线程1 按 A→B 的顺序，线程2 按 B→A 的顺序
  >>> 3 秒未返回，判定死锁
```

两条线程都改成 A→B：

```text
[ordered]
  两条线程都按 A→B 的固定顺序
  两条线程都跑完了，没死锁
```

顺序一致，环就构不成。这个手法在 objc4 里有现成的例子：`SideTable` 的双锁操作会先比较两张表的地址，永远先锁地址小的那个。本系列 weak 那篇里贴过那段代码。

上一篇讲的 `dispatch_sync` 死锁是同一件事的一个特例：向当前所在的串行队列同步派发，等于自己等自己，环长度为 1。区别是 libdispatch 能自检出这种环并当场 trap，而锁的环没人替你检查，就是安静地挂住。

第一节那个自旋起跑闸的活锁不属于这四条。所有线程都在跑，没有一条在等待中阻塞，只是永远推进不了。死锁会挂住，活锁会烧 CPU，Instruments 上表现完全不同。

---

## 十、所以该用哪个

这一篇测下来，我自己的选择顺序是这样：

保护一段共享状态，默认 `os_unfair_lock`。它最快（8.11 ns）、争用下尾延迟最稳（12 ~ 18 ms）、所有权错误会当场崩而不是静默出错。缺点是 Swift 里要小心 `&` 会拷贝锁内存，头文件明说了这点，该用 `OSAllocatedUnfairLock`。

需要 ObjC 层面的便利、或者要配合 `NSCondition` 做等待通知，用 `NSLock` / `NSCondition`。它们只比 `pthread_mutex` 慢 3%，这层封装今天基本是免费的。

需要跨越 wait / signal 的异步等待，或者要限制并发数，用 `dispatch_semaphore`。别拿它当互斥锁。

要切换执行上下文（回主线程、进后台），用队列。只是为了保护一个变量而建一个串行队列，是把一个 8 ns 的问题变成一个需要考虑 target 链和死锁边界的问题。

`@synchronized` 我基本不用。21 ns 的开销我能接受，不能接受的是它的两个失效模式：传 nil 静默失效，锁对象被替换静默失效。两种都不报错。它唯一的优势是写起来短，以及自动配平（异常路径上也会解锁），改老代码的时候能少改几行。

`OSSpinLock` 不要用。

---

## 十一、几个已经不准的说法

- 「`OSSpinLock` 被废弃是因为它慢/有 bug。」 它现在依然是最快的（7.00 ns，比 `os_unfair_lock` 快 14%），实现也没有 bug。废弃是因为自旋等待不向内核暴露持有者信息，不同优先级的线程争用同一把锁时会反转。实测一秒窗口里高优先级线程可以一次都抢不到。
- 「`os_unfair_lock` 是自旋锁。」 头文件第一句就是 "allows waiters to block efficiently on contention"。等待者会阻塞让出 CPU。iOS 18 起有一个可选的 `OS_UNFAIR_LOCK_FLAG_ADAPTIVE_SPIN` 标志能让它先自旋一下，但那是要显式开的。
- 「`@synchronized` 是最慢的锁。」 取决于平台。macOS 原生 21.13 ns，排第 14，比 `NSConditionLock`（34.13 ns）快得多；iOS 模拟器上 51.15 ns，才是最慢。两次独立的模拟器测量都在 49 ~ 51 ns，两次 macOS 测量都在 21 ns 上下。
- 「`NSLock` 比 `pthread_mutex` 慢很多。」 实测 9.03 对 8.75，差 3%。
- 「`dispatch_semaphore` 是最快的锁。」 老图里它是第二名，实测排第 9（9.77 ns），落在 `NSLock` 后面。争用时更差，吞吐比 `os_unfair_lock` 低两个数量级。
- 「`@synchronized(nil)` 会崩溃/会抛异常。」 什么都不会发生，`objc_sync_enter(nil)` 返回 `OBJC_SYNC_SUCCESS`，block 体照常执行，但没有任何互斥。4 线程 20 万次自增丢了 26%。
- 「`NSLock` 保证了谁锁谁解。」 实测另一条线程 unlock 直接成功，不报错。真会 abort 的只有 `os_unfair_lock`。
- 「读写锁在读多写少时一定更快。」 临界区短的时候是负收益，实测慢 3 到 4 倍。交叉点在临界区大约几百个元素的量级。
- 「`@synchronized` 用的锁对象越多越慢。」 从 1 个到 32768 个，实测一直是 20 ns 上下，链表位置也不影响。
- 「用 `trylock` 循环重试可以避免阻塞。」 头文件明说这等于一个低效版的 `lock`，还会把等待者藏起来、阻止优先级反转的解决。
- 「`dispatch_sync` 比锁慢很多，别拿它当锁用。」 无竞争时 9.30 ns，只比 `os_unfair_lock` 贵 15%。该用锁替代它的理由是死锁边界跟着 target 链走、review 时看不出来，跟性能无关。

---

## 总结

`OSSpinLock` 被废弃，不是因为它慢，它到今天还是最快的那一把。是因为自旋等待不告诉内核"我在等谁"，于是内核没法解开优先级环。一秒的窗口，四轮，每一轮都有高优先级线程从头等到尾。

`os_unfair_lock` 的锁变量里存着持有者身份，等待者阻塞时把这条信息交给内核。代价是它明确放弃了公平性，等待者有被饿死的可能，头文件写在明面上。

无竞争时锁与锁之间的差距比传说中小得多，除了 `NSConditionLock` 和模拟器上的 `@synchronized`，其余十几种都挤在 7 到 16 ns。真正拉开差距的是争用下的尾延迟，那里 `OSSpinLock` 比 `os_unfair_lock` 差六十倍以上。选锁应该看第二组数字。

`@synchronized` 有两个静默失效模式，都不报错：锁对象是 nil，以及锁对象在两次进入之间被换掉。后者实测万分之六的概率，测试环境撞不上。

最后一条方法论，这一篇撞了三次。第一次是自旋起跑闸把 `BACKGROUND` 线程饿死，实验挂了十分钟；第二次是用 `dispatch_sync` 冒充"另一条线程"，差点写出错误结论；第三次是临界区太短，读写锁三种方案的差异被 `stringWithFormat:` 淹掉。**结果不符合预期的时候，先查仪器。** 这三次里有两次是仪器坏了，一次是结论真的和源码对不上。那一次我选择只写实测数字，不补机制。

## 参考资料

### 一手

- `$(xcrun --show-sdk-path)/usr/include/os/lock.h` —— 本文关于 `os_unfair_lock` 的全部引文都出自这里：所有权信息与优先级反转、unfair 的含义、`trylock` 不能写重试循环、`OS_UNFAIR_LOCK_FLAG_ADAPTIVE_SPIN`
- `.../usr/include/libkern/OSSpinLockDeprecated.h` —— 废弃理由的原始措辞、三套实现分支、`_os_nospin_lock_lock`
- `.../usr/include/objc/objc-sync.h` —— `objc_sync_enter` / `objc_sync_exit` 的声明与返回值枚举
- WWDC17 Session 706《Modernizing Grand Central Dispatch Usage》："Primitives with a single known owner have this power. Things like serial queues and OS unfair lock."
- WWDC21 Session 10254《Swift concurrency: Behind the scenes》：信号量隐藏依赖信息、在 Swift Concurrency 下不安全

### 二手

- ibireme《不再安全的 OSSpinLock》（2016）：中文圈关于优先级反转最重要的一篇，那张性能对比图的出处。机制部分今天依然成立；图里的数字是 2016 年的 iPhone，第三节重测了一遍
- Pierre Habouzit《Making efficient use of the libdispatch (GCD)》：`os_unfair_lock` 通常是系统中最快的锁、小于 1ms 的任务用锁优于 `dispatch_async`

### 本地

- [[iOS GCD：队列不是线程，以及死锁的准确边界]]：`dispatch_sync` 在调用线程就地执行、barrier 在全局队列上失效、死锁的 target 链判据
- [[iOS 内存管理：从 MRC、ARC 到属性关键字#属性关键字：把所有权写进 API|属性关键字：从所有权推导，而不是从类型名猜]]：`atomic` 保护了什么，`PropertyLocks` 的 `StripedMap` 分片（真机 8 个，Mac 和模拟器 64 个）
- [[iOS weak 的实现：SideTable 与置 nil 的时机]]：按地址排序加锁避免环形等待的实例

---

实验环境：macOS 26.5.2（25F84），Apple M1 Pro（8 核，6 性能 + 2 能效），Apple clang 21.0.0。全部实验编成 macOS 原生二进制运行，没有使用模拟器：

```shell
clang -fobjc-arc -O2 -framework Foundation -o out prog.m && ./out
```

第三节里那组模拟器数字（`@synchronized` 51.15 ns 等）来自同一台机器上的另一次独立运行，编译目标是 `arm64-apple-ios17.0-simulator`，不是我这次跑的，文中已标明出处与用途。它只用来说明"同一个数字在两个平台上差 2.4 倍"这件事本身。

死锁与崩溃类实验一律在程序内部用 `alarm(3)` 加信号处理函数自杀，这样"挂住"和"当场 trap"能分开统计。崩溃信息取自 `~/Library/Logs/DiagnosticReports/` 下的 `.ips` 文件，`asi` 字段里是 libplatform / libdispatch 打出的原始诊断字符串。

性能类实验一律采用交错轮次加取最小值：同一轮里每种锁各跑一次，跑满 5 到 7 轮，取每种锁的最小值，整个程序再重跑一遍确认名次稳定。第三节的 15 项排名两遍只有相邻两项互换。争用类实验的抖动大得多，所以我给的是每一轮的原始数据而不是平均值。

> 待真机补测：第一节那组优先级反转数据在 iPhone 15 / iOS 26.5 上复现。真机核数更少、功耗墙更紧，低优先级持锁者被饿死的场景（我在 M1 Pro 上没能复现的那一半）可能在真机上出现。第八节读写分离的交叉点也依赖核数，真机上会更靠左。
