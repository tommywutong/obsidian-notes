# RunLoop
Runloop，也叫事件循环，用于处理程序在运行时接收到的各种事件，比如触摸事件，定时器，UI刷新。其保证了程序的持续运行，程序启动时自动开启主线程runloop。同时在有事件需要处理的时候唤醒线程，无事件的时候让线程休眠，节省了CPU资源。

![[Pasted image 20260506180722.png]]

## RunLoop相关类和构成
```
CFRunLoopRef

CFRunLoopModeRef

CFRunLoopSourceRef

CFRunLoopTimerRef

CFRunLoopObserverRef
```

一个Runloop对应一条线程，一个runloop里面可以有多个CFRunLoopModeRef(模式）。

同一个时刻，RunLoop只能是在一个mode上面的运行。如果需要切换mode,只能是退出currentMode ,切换到指定的 mode 。

每一个mode又可以包含多个 source/timer/observer。不同 mode 里面的子元素，互不影响。

5个类的对应关系大概是：

![[Pasted image 20260506183342.png]]

1. **CFRunLoopModeRef

一个 run loop mode 是若干个 source、timer 和 observer 的集合。它能帮我们过滤掉一些不想要的事件。
**即一个 RunLoop 在某个 mode 下运行时，不会接收和处理其他 mode 的事件 。**

要保持一个 mode 活着，就必须往里面添加至少一个 source、timer 或 observer 。

这一点很容易理解，你想啊，如果一个 mode 里面什么东西都没有，那么他根本就没有活干，那 mode 活着还有什么意思。

苹果公开的 mode 有两个：kCFRunLoopDefaultMode (NSDefaultRunLoopMode) 和 UITrackingRunLoopMode。

前者是默认的模式，程序运行的大多时候都处于该 mode 下，后者是滑动 tableView 或 scrollerView 时为了界面流畅而用的 mode。

  CFRunLoop里面还有一个**伪mode**叫做 kCFRunLoopCommonModes，它不是一个真正的 mode，而是若干个 mode 的集合。

我们往 CommonModes 里面加入 任意的 source/timer/observer 。

只要加入到了 CommonModes 里面，就相当于添加到了它里面所有的 mode 中（当然，根据各自的情况，可能不仅仅只要默认的两个 mode ）。

我们可以通过 NSLog(@"%@", [NSRunLoop currentRunLoop]) 从打印结果看到 CommonMode 包含了上面的 DefaultMode 和 TrackingRunLoopMode。

  2. CFRunLoopSourceRef

source 是事件产生的地方（输入源），虽然官方文档在概念上把 source 分为三类：Port-Based Sources，Custom Input Sources，Cocoa Perform Selector Sources。

但在源码中 source 只有两个版本：**source0** 和 **source1**，它们的区别在于它们是怎么被标记 (signal) 的。

**source0** 是app内部的消息机制，使用时需要调用 CFRunLoopSourceSignal()来把这个 source 标记为待处理，然后掉用 CFRunLoopWakeUp() 来唤醒 RunLoop，让其处理这个事件。

**source1** 是基于 mach_ports 的，用于通过**内核和其他线程**互相发送消息。

iOS / OSX 都是基于 Mach 内核，Mach 的对象间的通信是通过消息在两个端口(port)之间传递来完成。

很多时候我们的 app 都是处于什么事都不干的状态，在空闲前指定用于唤醒的 mach port 端口，然后在空闲时被 mach_msg() 函数阻塞着并监听唤醒端口， mach_msg() 又会调用 mach_msg_trap() 函数从用户态切换到内核态，这样系统内核就将这个线程挂起，一直停留在 mac_msg_trap 状态。直到另一个线程向内核发送这个端口的 msg 后， trap 状态被唤醒， RunLoop 继续开始干活。

其实，总结下来，**事件产生的地方就是source(输入源)**, 运用发消息的机制，让事件可以唤醒休眠的runloop执行。

3. CFRunLoopTimerRef


4. CFRunLoop ObserverRef
他这就是一个观察者，他主要用途就是监听RunLoop的状态变化
```objc
/* Run Loop Observer Activities */

typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {

// 即将进入 loop  
kCFRunLoopEntry = (1UL << 0),

// 即将处理 timer  
kCFRunLoopBeforeTimers = (1UL << 1),

// 即将处理 source  
kCFRunLoopBeforeSources = (1UL << 2),

// 即将 sleep  
kCFRunLoopBeforeWaiting = (1UL << 5),

// 刚被唤醒，退出 sleep  
kCFRunLoopAfterWaiting = (1UL << 6),

// 即将退出  
kCFRunLoopExit = (1UL << 7),

// 全部的活动  
kCFRunLoopAllActivities = 0x0FFFFFFFU

};
```

![[Pasted image 20260506202236.png]]
































![[Pasted image 20260506105355.png]]
## 2026-05-14 23:29 RunLoop事件处理与模式切换机制
**来源**: 未知　**confidence**: 1.00

- RunLoop本质是do-while循环，处理Source0、Source1、Timer、Observer四类事件。
- CFRunLoopMode决定当前接收哪些事件源，影响事件处理行为。
- 主线程默认在NSDefaultRunLoopMode和UITrackingRunLoopMode之间切换。
- 模式切换导致滚动ScrollView时NSTimer暂停，解释了Runloop模式对任务调度的影响。

<details><summary>原文</summary>

RunLoop 本质是一个 do-while 循环，处理 Source0/Source1/Timer/Observer 四类事件。CFRunLoopMode 决定当前接收哪些 source，主线程默认在 NSDefaultRunLoopMode 和 UITrackingRunLoopMode 之间切换，这就是为什么滚动 ScrollView 时 NSTimer 会停。

</details>

---
