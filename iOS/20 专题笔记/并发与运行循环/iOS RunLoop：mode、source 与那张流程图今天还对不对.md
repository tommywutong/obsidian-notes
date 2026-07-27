---
title: 【iOS】RunLoop：mode、source 与那张流程图今天还对不对
published: 2026-07-27
description: 主线程 RunLoop 的 mode 不是五个，命令行工具里一个、UIKit 起来之后十三个。经典文章里那两个管自动释放池的 observer，今天在 observer 列表里一个都读不到；真正在起作用的是每次 callout 各自一个池。
tags:
  - iOS
  - RunLoop
  - CoreFoundation
  - 并发
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 17
draft: true
---
# RunLoop：mode、source 与那张流程图今天还对不对

先摆三个我实测出来的数字。

一个纯 Foundation 命令行工具，主线程 RunLoop 里只有 1 个 mode。同一台机器上把 UIKit 拉起来，跑一个真正的 `UIApplicationMain`。启动完成那一刻是 11 个，一秒半之后 13 个。中文圈流传最广的说法是"系统默认注册了 5 个 mode"，这个数在我手上没有一次对上。

第二个数字：0。把主线程 RunLoop 的每个 mode、每个 observer 全部打印出来。`order` 等于 `-2147483647` 或 `2147483647` 的，一个都没有。那正是所有讲"RunLoop 和自动释放池"的文章里被抄了十年的两个数。

第三个：5985852。一条子线程，`while(1) { [runLoop runMode:beforeDate:] }`，里面不加任何 source。一秒钟这个循环转了 598 万圈，CPU 占用 98%。

RunLoop 的资料密度极高，高到几乎所有文章都在转述同一个源头：ibireme 2015 年那篇《深入理解 RunLoop》。那篇写得确实好。我这篇里很多结构判断还是从它来的。但它写于十一年前，对着的是 CF-855.17 的源码。这篇不打算再讲一遍 RunLoop 是什么。我要做的是把那张流程图上的每一条，用今天的机器重新验一次，然后老老实实说哪几条已经不对了。

CFRunLoop 是开源的，这是这个话题最大的便利。但要先说清一件事：开源的那份和你机器上跑的那份差得很远。我拿到的 `CFRunLoop.c` 版权头写的是 2015。本机 `kCFCoreFoundationVersionNumber` 是 5026.5，`sample` 抓到的镜像版本是 `6.9 - 5026.5.4.1.402`。差了十一年。所以本文的规矩是：源码只用来解释机制。凡是结论，都要有一份今天跑出来的输出顶着。

---

## 一、先把主线程的 RunLoop 打出来

这是全文最有力的一手证据，也是任何人都能在三十秒内复现的：

```objc
NSLog(@"%@", (__bridge id)CFRunLoopGetMain());
```

它会把所有 mode、每个 mode 下的 source0 / source1 / observer / timer 清单原样吐出来。先看纯命令行工具（`clang -framework Foundation -framework CoreFoundation`）：

```text
<CFRunLoop 0x1049da800>{wakeup port = 0xc03, stopped = false, ignoreWakeUps = true,
current mode = (none),
common modes = {count = 1, entries => 2 : "kCFRunLoopDefaultMode"},
common mode items = (null),
modes = {count = 1, entries =>
	2 : <CFRunLoopMode>{name = kCFRunLoopDefaultMode, port set = 0xd03, ...
	sources0 = (null),
	sources1 = (null),
	observers = (null),
	timers = (null),
}}
```

一个 mode。四个清单全空。没有 UIKit 的进程里，RunLoop 就是这么干净。

### UIKit 起来之后

要看真实 App 的样子，不用开模拟器。Mac Catalyst 编出来的是原生 macOS 二进制，但链接和加载的是真正的 `UIKitCore`：

```shell
SDK=$(xcrun --sdk macosx --show-sdk-path)
clang -fobjc-arc -target arm64-apple-ios17.0-macabi -isysroot "$SDK" \
      -iframework "$SDK/System/iOSSupport/System/Library/Frameworks" \
      -framework Foundation -framework UIKit -o out prog.m
```

在 `application:didFinishLaunchingWithOptions:` 里数一遍 mode：

```text
[before UIApplicationMain] mode count = 3
    NSModalPanelRunLoopMode / NSEventTrackingRunLoopMode / kCFRunLoopDefaultMode

[didFinishLaunching]       mode count = 11
    kCFRunLoopDefaultMode / dockmsg-mode / NSEventTrackingRunLoopMode
    com.apple.accessibilityServerIPC / CoreDragMode / UITrackingRunLoopMode
    AppleEventReplies / NSGraphicsRunLoopMode / NSModalPanelRunLoopMode
    __kCFPasteboardPrivateMode / kCFRunLoopCommonModes

[1.5 秒后]                 mode count = 13
    上面 11 个，加 com.apple.run-loop-mode.view-bridge.blocks
    和 IMKClient_com.apple.inputmethodkit.launcher
```

`UITrackingRunLoopMode` 确实在。这条老结论成立。但 mode 的数量是运行期长出来的，不是启动时固定的一张表。1.5 秒里多了两个。输入法一初始化，又多一个。

`common modes` 这一栏更有信息量，它只有 4 个：

```text
common modes = {count = 4, entries =>
	0 : "NSModalPanelRunLoopMode"
	2 : "kCFRunLoopDefaultMode"
	3 : "UITrackingRunLoopMode"
	5 : "NSEventTrackingRunLoopMode"
}
common mode items = {count = 30, ...}
```

`kCFRunLoopDefaultMode` 和 `UITrackingRunLoopMode` 都被标记成了 common。所以"加到 `NSRunLoopCommonModes` 的 timer 在滚动时也走"这句话在 iOS 上成立。它有一个前提。那个前提在别的场合经常不满足，第五节展开。

边界要说清楚。`NSModalPanelRunLoopMode`、`NSEventTrackingRunLoopMode`、`NSGraphicsRunLoopMode`、`dockmsg-mode`、`CoreDragMode` 这几个是 AppKit 的。它们出现在这里，是因为 Catalyst 把 UIKit 架在 AppKit 上跑。所以这份清单能安全外推到 iOS 的只有两条：数量是动态的；`UITrackingRunLoopMode` 是 common mode。

> 待真机补测：同一段 `NSLog(@"%@", CFRunLoopGetMain())` 放进一个 iOS 真机 App 的 `viewDidAppear:` 里跑一次，记录 mode 总数、common modes 的成员，以及 `UIInitializationRunLoopMode` / `GSEventReceiveRunLoopMode` 是否出现。这两个 mode 在我这台机器上都没进 mode 列表。

### observer 清单里没有池

`kCFRunLoopDefaultMode` 下的 observer 一共 6 个，原样抄在这里：

```text
observers = (
  {activities = 0x1,  order = 0,       callout = UC::LoopTapCFRunLoop::onEntry},
  {activities = 0x21, order = 0,       callout = __CFRunLoopObserverWithHandlerPerform},
  {activities = 0x44, order = 0,       callout = __CFRunLoopObserverWithHandlerPerform},
  {activities = 0xa0, order = 1999000, callout = _beforeCACommitHandler},
  {activities = 0xa0, order = 2000000, callout = CA::Transaction::observer_callback},
  {activities = 0xa0, order = 2001000, callout = _afterCACommitHandler}
)
```

`CA::Transaction::observer_callback` 挂在 `order = 2000000`、`activities = 0xa0`（BeforeWaiting | Exit）上。这条和 2015 年那篇写的一模一样，十一年没变。界面更新走"即将休眠时提交 CATransaction"这条链路，今天依然如此。

但 `_wrapRunLoopWithAutoreleasePoolHandler` 不在。`order` 为 `±2147483647` 的 observer 也不在。这两个数字被抄了十年，我在自己能读到的所有 mode 的 observer 列表里都找不到它们。第九节会用行为实验交代这件事的另一半。

---

## 二、一轮循环的真实顺序

`CFRunLoopActivity` 一共六个值，实测打印出来是：

```text
Entry=0x1  BeforeTimers=0x2  BeforeSources=0x4
BeforeWaiting=0x20  AfterWaiting=0x40  Exit=0x80
kCFRunLoopAllActivities = 0xfffffff
```

`0x8` 和 `0x10` 这两个 bit 没有对应的 activity。注册一个 `kCFRunLoopAllActivities` 的 observer，配一个 0.4 秒的重复 timer，跑三轮后 `CFRunLoopStop`：

```text
--- CFRunLoopRun() 开始 ---
 1  0x01  kCFRunLoopEntry
 2  0x02  kCFRunLoopBeforeTimers
 3  0x04  kCFRunLoopBeforeSources
 4  0x20  kCFRunLoopBeforeWaiting
 5  0x40  kCFRunLoopAfterWaiting
    >>> timer fire #1
 6  0x02  kCFRunLoopBeforeTimers
 7  0x04  kCFRunLoopBeforeSources
 8  0x20  kCFRunLoopBeforeWaiting
 9  0x40  kCFRunLoopAfterWaiting
    >>> timer fire #2
10  0x02  kCFRunLoopBeforeTimers
11  0x04  kCFRunLoopBeforeSources
12  0x20  kCFRunLoopBeforeWaiting
13  0x40  kCFRunLoopAfterWaiting
    >>> timer fire #3
14  0x80  kCFRunLoopExit
```

几条值得记住的：

`Entry` 和 `Exit` 各只有一次，它们属于 `CFRunLoopRunSpecific` 这一次调用，不属于每一轮迭代。开源 CF 里这一段写得很直白：

```c
if (currentMode->_observerMask & kCFRunLoopEntry) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
if (currentMode->_observerMask & kCFRunLoopExit)  __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
```

`do { } while (0 == retVal)` 那个循环整个在 `__CFRunLoopRun` 里面，Entry / Exit 在它外面。所以卡顿监控里用 `Entry` 当"一轮开始"是错的，一轮的开始是 `BeforeTimers`。

timer 的回调落在 `AfterWaiting` 之后、下一次 `BeforeTimers` 之前。`BeforeTimers` 的字面意思是"即将处理 timer"。但它触发的时候，这一轮的 timer 还没处理。真正被处理的是上一次睡醒之后到期的那批。这个命名很骗人，我第一次画时序图就画反了。

`Exit` 前面没有再来一次 `BeforeTimers`。`CFRunLoopStop` 是在第三次 timer 回调里调的，循环直接判定退出，不补任何 observer。

---

## 三、source0 和 source1，用实验分清

定义谁都会背：source0 要手动唤醒，source1 由内核唤醒。但背下来和知道它意味着什么是两回事。我造了三组对照。主线程跑 `CFRunLoopRunInMode(default, 2.0)`，子线程在 0.6 秒时动手。

第一组，只 `CFRunLoopSourceSignal`，不唤醒：

```text
  0.000  observer: Entry / BeforeTimers / BeforeSources / BeforeWaiting
  0.603  子线程: CFRunLoopSourceSignal(source0)  —— 只 signal，不唤醒
  2.001  observer: AfterWaiting
  2.001  observer: Exit
  2.001  CFRunLoopRunInMode 返回 3 (TimedOut)
```

source0 被标记成待处理了。RunLoop 一动不动，睡到 2 秒超时。回调一次都没跑。这就是"手动唤醒"四个字的实际后果：**忘了调 `CFRunLoopWakeUp`，你的事件不是晚到，是根本不到。**

第二组，`Signal` 之后补一次 `WakeUp`：

```text
  0.602  子线程: CFRunLoopSourceSignal + CFRunLoopWakeUp
  0.602  observer: AfterWaiting
  0.602  observer: BeforeTimers
  0.602  observer: BeforeSources
  0.602          [source0 回调执行]
  0.602  observer: BeforeTimers
  0.602  observer: BeforeSources
  0.602  observer: BeforeWaiting
```

醒了，但 source0 的回调没有当场执行，它等到了下一轮的 `BeforeSources` 之后。而且处理完之后又空转了一轮 `BeforeTimers` / `BeforeSources` 才重新睡下去。这个多出来的一轮在源码里有出处：

```c
Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
...
Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
...
if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
```

处理过 source0 的那一轮 `poll` 为真，直接跳过 `BeforeWaiting` 和真正的休眠，用零超时再探一次端口。所以你会看到 `BeforeWaiting` / `AfterWaiting` 成对消失的迭代，那不是漏打印。

第三组，source1。用 `CFMachPortCreate` 造一个，子线程直接 `mach_msg` 往那个 port 发一条消息，全程不碰任何 `CFRunLoop*` API：

```text
  0.605  子线程: 往 mach port 发一条消息（不碰 RunLoop 的任何 API）
         mach_msg send kr = 0x0
  0.605  observer: AfterWaiting
  0.605          [source1 回调执行 (CFMachPort)]
  0.605  observer: BeforeTimers
  0.605  observer: BeforeSources
  0.605  observer: BeforeWaiting
```

回调在 `AfterWaiting` 之后立刻执行，插在 `BeforeTimers` 前面，同一轮解决。两者的位置差别是结构性的。source1 就是把线程从 `mach_msg` 里捞出来的那条消息本身，处理它是唤醒流程的一部分。source0 只是一个标记位。它得等循环重新走到检查它的地方。

### 还有第三条路和第五类 item

`dispatch_async(dispatch_get_main_queue(), ^{})` 走的既不是 source0 也不是 source1：

```text
  0.420  observer: AfterWaiting
  0.420      <<< dispatch_async 到主队列的 block 执行
  0.420  observer: BeforeTimers
```

位置和 source1 一样，机制是单独的。CF 给主队列留了一个专用端口，收到消息就调 libdispatch 的回调：

```c
static void __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(void *msg) {
    _dispatch_main_queue_callback_4CF(msg);
}
```

这个函数名在崩溃栈里出现得很多，看到它就知道这一帧是主队列的 block，不是 UI 事件。[[iOS GCD：队列不是线程，以及死锁的准确边界]] 里有个实验：主队列必须有 RunLoop 才回到主线程。底下就是这条通路。

还有第五类 item。`CFRunLoopPerformBlock` 排进去的 block，教科书上的 source / timer / observer 三件套装不下它：

```text
  0.839      (子线程: PerformBlock 已排入，故意不 WakeUp)
  1.470      (子线程: 现在 CFRunLoopWakeUp)
  1.470  observer: AfterWaiting
  1.470      <<< CFRunLoopPerformBlock 的 block 执行
```

排进去了，不唤醒就不跑，行为和 source0 一样。它存在 `rl->_blocks_head` 这条独立链表上，`CFRunLoopCopyAllModes` 那份打印里看不到它。

---

## 四、睡着的时候在干什么

"RunLoop 不是 `while(1)` 空转"这句话每篇文章都写，但很少有人给证据。抓一个空闲进程的主线程栈就够了：

```shell
./e4 &            # 进 CFRunLoopRun()，什么都不做
sample $(pgrep -n e4) 2 -f e4.sample
```

```text
    1746 Thread_16396840   DispatchQueue_1: com.apple.main-thread  (serial)
      1746 start  (in dyld)
        1746 main  (in e4)
          1746 CFRunLoopRun  (in CoreFoundation)
            1746 _CFRunLoopRunSpecificWithOptions  (in CoreFoundation)
              1746 __CFRunLoopRun  (in CoreFoundation)
                1746 __CFRunLoopServiceMachPort  (in CoreFoundation)
                  1746 mach_msg  (in libsystem_kernel.dylib)
                    1746 mach_msg_overwrite
                      1746 mach_msg2_internal
                        1746 mach_msg2_trap
```

两秒采了 1746 个样本。1746 个全在 `mach_msg2_trap`。这是一次陷入内核的接收调用，线程被内核挂起，不占 CPU，直到有消息投递到端口集上才被唤醒。

对应的源码就是 `__CFRunLoopServiceMachPort`：

```c
for (;;) {		/* In that sleep of death what nightmares may come ... */
    mach_msg_header_t *msg = (mach_msg_header_t *)*buffer;
    ...
    if (TIMEOUT_INFINITY == timeout) { CFRUNLOOP_SLEEP(); } else { CFRUNLOOP_POLL(); }
    ret = mach_msg(msg, MACH_RCV_MSG|...|((TIMEOUT_INFINITY != timeout) ? MACH_RCV_TIMEOUT : 0)|...,
                   0, msg->msgh_size, port, timeout, MACH_PORT_NULL);
```

那句哈姆雷特是 Apple 工程师留的。调用方传进来的 timeout 是 `poll ? 0 : TIMEOUT_INFINITY`。上一节"处理过 source0 就不真睡"，实现就在这一处。

再看一眼栈里的 `_CFRunLoopRunSpecificWithOptions`。开源那份 CF 里没有这个函数，只有 `CFRunLoopRunSpecific`。后者今天依然存在，`dlsym` 取得到（本机 `0x18e1ecf90`），但 `CFRunLoopRun` 走的是前者（`0x18e2c01c4`）。凡是从 2015 年那份源码抄函数名去猜今天行为的，先看一眼自己的崩溃栈里到底是哪个符号。

---

## 五、mode 切换，以及 CommonModes 的真实语义

经典例子是 `NSTimer` 加在 `NSDefaultRunLoopMode` 上，滚动 tableView 时停走。macOS 上没有滚动。但 mode 切换可以手工复现：自建一个 `MyPrivateMode`，往里塞一个空 source0 让它别立刻退出，再 `CFRunLoopRunInMode` 切过去一秒。

0.3 秒间隔的重复 timer，总共跑 3 秒，理论 10 次。

A：timer 加在 `NSDefaultRunLoopMode`。

```text
  0.00  跑 default mode 1.0 秒
  0.30  timer 第 1 次触发   当前 mode = kCFRunLoopDefaultMode
  0.60  timer 第 2 次触发   当前 mode = kCFRunLoopDefaultMode
  0.90  timer 第 3 次触发   当前 mode = kCFRunLoopDefaultMode
  1.00  切到 MyPrivateMode 跑 1.0 秒
  2.01  切回 default mode 1.0 秒
  2.01  timer 第 4 次触发
  ...
  3.01  结束，timer 一共触发 8 次
```

1.0 到 2.0 秒之间一次都没有。现象复现了。

B：timer 加在 `NSRunLoopCommonModes`。

```text
  1.00  切到 MyPrivateMode 跑 1.0 秒
  2.01  切回 default mode 1.0 秒
  2.01  timer 第 4 次触发
  ...
  3.01  结束，timer 一共触发 8 次
```

一模一样。改成 CommonModes 完全没有帮助。

这是我在这篇里最想强调的一点。`NSRunLoopCommonModes` 不是"所有 mode"的意思，它是"所有被打上 Common 标记的 mode"。`MyPrivateMode` 没有这个标记，所以同步不到它。加一行：

```objc
CFRunLoopAddCommonMode(CFRunLoopGetCurrent(), CFSTR("MyPrivateMode"));
```

C：先标记，再用 CommonModes。

```text
  1.00  切到 MyPrivateMode 跑 1.0 秒
  1.20  timer 第 4 次触发   当前 mode = MyPrivateMode
  1.50  timer 第 5 次触发   当前 mode = MyPrivateMode
  1.81  timer 第 6 次触发   当前 mode = MyPrivateMode
  2.01  切回 default mode 1.0 秒
  ...
  3.01  结束，timer 一共触发 10 次
```

10 次，一次不少。

iOS 上 B 之所以能"work"，是因为 UIKit 早就替你把 `UITrackingRunLoopMode` 加进了 common modes。第一节那份 dump 里能读到。这个前提一旦不成立，`NSRunLoopCommonModes` 就是个空操作。不报错，不打日志。我见过有人在自建的常驻线程上用 CommonModes 注册 timer，然后花两天找"为什么 timer 偶尔不走"。

这个字符串本身的地位也得说清。它不是 mode，把它当 mode 用会被 CF 当场拒绝：

```text
invalid mode 'kCFRunLoopCommonModes' provided to CFRunLoopRunSpecific -
break on _CFRunLoopError_RunCalledWithInvalidMode to debug.
```

`CFRunLoopRunInMode(kCFRunLoopCommonModes, 0.2, false)` 直接返回 `kCFRunLoopRunFinished`，什么都没跑。同步是双向的。先用 CommonModes 加 source，再 `CFRunLoopAddCommonMode` 一个新 mode，那个 source 会自动出现在新 mode 里。`CFRunLoopContainsSource` 查得到。

有意思的是第一节 Catalyst 那份 dump 里，`modes` 集合中真的有一项名字就叫 `kCFRunLoopCommonModes`。说明"伪 mode"只是约定，系统框架里有代码把它当真 mode 名用了。这条我没查出是谁干的。

---

## 六、timer：tolerance 的上限，以及错过不补

`tolerance` 的文档只有一句含糊话：

> The system may put a maximum value of the tolerance.

没给数字。那就自己量一遍。对不同 interval 的重复 timer 统一设 `tolerance = 1e6`，再读回来：

```text
interval=0.05    读回=0.0250      cap/interval=0.5000
interval=0.10    读回=0.0500      cap/interval=0.5000
interval=0.50    读回=0.2500      cap/interval=0.5000
interval=1.00    读回=0.5000      cap/interval=0.5000
interval=2.00    读回=1.0000      cap/interval=0.5000
interval=10.00   读回=5.0000      cap/interval=0.5000
```

上限是 interval 的一半。六个量级完全一致。`repeats:NO` 的 timer 不截断（读回 1000000），负数也不截断（读回 -5）。所以文档那句 "may put a maximum value" 在重复 timer 上是硬性的 50%。写 `timer.tolerance = 100` 和写 `timer.tolerance = interval/2`，效果一样。

至于 tolerance 对实际抖动的影响，我没测出来。100ms 间隔跑 25 次，`tolerance` 分别设 0 / 0.05 / 1.0：

```text
tolerance=0.000   平均 99.94ms  最小 96.68ms  最大 102.86ms  抖动跨度 6.19ms
tolerance=0.050   平均 99.96ms  最小 98.06ms  最大 102.16ms  抖动跨度 4.10ms
tolerance=1.000   平均 100.02ms 最小 97.70ms  最大 102.22ms  抖动跨度 4.52ms
重跑 tolerance=0     抖动跨度 3.61ms
重跑 tolerance=1.0   抖动跨度 2.72ms
```

跨度在 2.7 到 6.2 之间乱跳，和 tolerance 没有关系。我的判断是这台机器太闲。没有别的定时器可以合并，省电优化就没有发挥余地。所以结论只能写成"在空载 macOS 上测不出差别"，不能写成"tolerance 没用"。这个实验只扫了 tolerance 一个自变量，系统负载那个维度我一次都没动。

平均值稳在 100ms 是有出处的，头文件写得很清楚：

> For repeating timers, the next fire date is calculated from the original fire date regardless of tolerance applied at individual fire times, to avoid drift.

### 错过的不补

0.1 秒的重复 timer，在第三次回调里 `usleep(1000000)` 把主线程堵一秒：

```text
   0.105  第  1 次触发
   0.204  第  2 次触发
   0.305  第  3 次触发
   0.305  >>> 在回调里 sleep(1) 阻塞主线程，期间理论上应该错过 10 次
   1.308  >>> 阻塞结束
   1.404  第  4 次触发
   1.505  第  5 次触发
   ...
   2.004  第 10 次触发
总共触发 10 次
```

不阻塞是 20 次。实际 10 次。堵住的一秒里错过了大约 10 次，一次都没补。连"立刻补一次"都没有。阻塞在 1.308 结束，下一次触发是 1.404，正好落在原本的 0.1 秒网格上。

上一节 A 组也是同一个现象，1.0 到 2.0 秒错过 3 次，回到 default mode 时只在 2.01 补了一次。补的那一次其实是"到期了就触发"，不是补偿。

用 `NSTimer` 做倒计时或者计费的，这条要记住：**它保证的是"不早于"，从来没保证过次数。** 需要准次数就自己记时间戳算差值。

---

## 七、"mode 为空就退出"有两个例外

这条规则本身是对的，源码在 `__CFRunLoopModeIsEmpty` 里：

```c
if (NULL != rlm->_sources0 && 0 < CFSetGetCount(rlm->_sources0)) return false;
if (NULL != rlm->_sources1 && 0 < CFSetGetCount(rlm->_sources1)) return false;
if (NULL != rlm->_timers   && 0 < CFArrayGetCount(rlm->_timers)) return false;
struct _block_item *item = rl->_blocks_head;
while (item) { ... if (doit) return false; }
return true;
```

数的是 source0、source1、timer、block 四类。**observer 不在这个列表里。** 光加 observer 不足以让一个 mode 活着。

它前面还有一段更容易被忽略：

```c
Boolean libdispatchQSafe = pthread_main_np() && (...);
if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name))
    return false; // represents the libdispatch main queue
```

主线程 + 这个 mode 是 common mode，直接判为非空，理由是它代表 libdispatch 的主队列。

两条一起测：

```text
[主线程]
  自建空 mode（非 common）                    返回 Finished，耗时 0.0000 秒
  自建 mode，只加 observer                    返回 Finished，耗时 0.0000 秒
  default mode（主线程，是 common mode，空）   返回 TimedOut，耗时 0.5035 秒
[子线程]
  空 mode（什么都不加）                       返回 Finished，耗时 0.0000 秒
  只加一个 observer 的 mode                   返回 Finished，耗时 0.0000 秒
  default mode（子线程，空）                  返回 Finished，耗时 0.0000 秒
```

同样是 `CFRunLoopRunInMode(kCFRunLoopDefaultMode, 0.5, false)`，主线程老老实实睡满 0.5 秒，子线程 0 秒返回。所以 `main.m` 里的 `CFRunLoopRun()` 从来不用担心"没有 source 会不会直接返回"。换到子线程，这个担心是真的。

---

## 八、常驻线程：把常见的那个错误犯一遍

常驻线程的标准写法长这样：

```objc
NSRunLoop *rl = [NSRunLoop currentRunLoop];
[rl addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
while (!self.cancelled) {
    [rl runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
}
```

那行 `addPort:` 经常被当成可有可无的仪式省掉。省掉的后果我量了一下，两组各跑一秒墙钟：

```text
[什么都不加]     1 秒内 runMode: 返回了 5985852 次，进程 CPU 时间 0.981 秒（占墙钟 98%）
[加了 NSMachPort] 1 秒内 runMode: 返回了 19 次，进程 CPU 时间 0.002 秒（占墙钟 0%）
再跑一遍：
[什么都不加]     6080593 次，0.991 秒（99%）
[加了 NSMachPort] 19 次，0.002 秒（0%）
```

598 万次对 19 次。98% CPU 对 0%。传 `[NSDate distantFuture]` 也一样。mode 空的时候 `runMode:` 直接返回 `NO`，一微秒都不等，实测 0.000010 秒。上一节那条规则的后果就是这么直接。

这个 bug 的表现是 App 待机时一条后台线程把一个核吃满。用 Time Profiler 看，`CFRunLoopRunSpecific` 占了 99%，很容易被误读成"RunLoop 本身很贵"。它不贵。是你让它空转了六百万次。

### performSelector:afterDelay: 为什么会静默失效

同一件事的另一个面孔。三条 `pthread`：

```text
[A] 线程启动，不跑 RunLoop
[A] performSelector:afterDelay: 已调用，睡 1 秒等它
[A] 1 秒过去了，上面那行 >>> 出现了吗？

[B] 线程启动
[B] performSelector:afterDelay: 已调用，现在跑 RunLoop 1 秒
        >>> hello: 被调用了 (B)
[B] RunLoop 结束
```

A 里什么都没发生。没有崩溃，没有警告，没有日志。`performSelector:withObject:afterDelay:` 的实现是往当前线程的 RunLoop 装一个 timer。没有 RunLoop 在跑，那个 timer 就永远不会到期。`NSTimer`、`NSURLConnection` 的 delegate 回调、`NSObject` 的 `performSelector:onThread:` 都是同一类。

这条也解释了常驻线程存在的理由。一条 `pthread` 跑完函数体就退出了。想让它留下来接活，就得有一个不返回的循环，而这个循环还得在没活的时候不烧 CPU。RunLoop 就是这两个要求的现成解。

---

## 九、和自动释放池的接缝

这一节只讲 RunLoop 这一侧。池自己的结构（`AutoreleasePoolPage`、哨兵、页链表、每页装多少）在 [[iOS AutoreleasePool：哨兵、页链表与 RunLoop 的关系]] 里。

流传的说法是：主线程有两个 observer，一个在 `Entry` 时 push 池，另一个在 `BeforeWaiting` 时 pop 再 push、在 `Exit` 时 pop。第一节已经说了，这两个 observer 我在 observer 列表里读不到。那么现在的行为到底是什么？

设计一个探针。一个会在 `dealloc` 时打日志的 `Canary`，在 `Entry` 和 `AfterWaiting` 两个 observer 回调里各 autorelease 一个，timer 回调里也放一个。看它们分别死在哪。

```text
--- CFRunLoopRun() ---
  observer: Entry          ← autorelease 一个 Canary(Entry)
            !!!! Canary(Entry) dealloc
  observer: BeforeTimers
  observer: BeforeSources
  observer: BeforeWaiting
  observer: AfterWaiting   ← autorelease 一个 Canary(AfterWaiting)
            !!!! Canary(AfterWaiting) dealloc
  [timer 回调 #1] autorelease 一个 Canary(timer)
  [timer 回调 #1] 即将返回
            !!!! Canary(timer) dealloc
  observer: BeforeTimers
  ...
```

每一个对象都在它所在的那次 callout 返回时就被释放了。一个都活不到 `BeforeWaiting`。同时用 `_objc_autoreleasePoolPrint` 数池的层数：整个 `CFRunLoopRun` 期间稳定停在 2 层，外层那个 `@autoreleasepool` 占一层。`BeforeWaiting` 前后没有任何变化。

结论很清楚。今天的 CoreFoundation 是**每一次 callout 各自套一个自动释放池**，push 和 pop 都发生在回调函数的调用点上。observer 根本看不见它。这和"进入时 push、休眠前 pop 再 push"是两种机制。

我又把同一个探针放进 Catalyst 上真正由 UIKit 驱动的主线程 RunLoop 里跑了一次。`Entry` 里 autorelease 的 Canary 同样在 callout 返回时就 dealloc。行为一致。

开源那份 CF 的表现更说明问题：整个 `CFRunLoop.c` 里 grep 不到一个 `autorelease`、`objc_autoreleasePoolPush` 或者 `_CFAutorelease`。一个都没有。这套 per-callout 的池是 2015 年之后加进去的。那批经典文章成文时它还不存在。

关于那两个 observer，我把手上的证据摊开，不下超出证据的结论：

- `_wrapRunLoopWithAutoreleasePoolHandler` 这个符号在本机 dyld 共享缓存里还在，`dyld_shared_cache_arm64e.04.dyldlinkedit` 里有 `__wrapRunLoopWithAutoreleasePoolHandler` 和它的 `.cold.1`，`.01` 里还有完整的函数原型字符串 `void _wrapRunLoopWithAutoreleasePoolHandler(CFRunLoopObserverRef, CFRunLoopActivity, void *)`。函数没被删。
- 但它没有出现在主线程 RunLoop 任何一个 mode 的 observer 列表里（Catalyst，13 个 mode 全查过）。
- 这个实验只扫了"平台"这一个自变量的一个取值：macOS / Catalyst。iOS 真机上 UIKit 是否仍然注册它，我没有测过，不知道。

> 待真机补测：iOS 真机 App 里 `NSLog(@"%@", CFRunLoopGetMain())`，在 `kCFRunLoopDefaultMode` 的 observers 数组里搜 `order = -2147483647` 和 `order = 2147483647` 两项，以及 callout 名里含 `AutoreleasePool` 的项。同时把上面那个 Canary 探针原样跑一遍，看在 `Entry` 里 autorelease 的对象是死在 callout 返回时还是死在 `BeforeWaiting`。这两个观察合起来能唯一确定 iOS 上用的是哪套机制。

不管是哪套，对写业务代码的人来说结论不变。主线程上 autorelease 的临时对象活不过当前这一轮，甚至活不过当前这一次回调。子线程上没有 RunLoop 的时候没人替你排空，长循环里要自己套 `@autoreleasepool`。

### 一个我踩进去的坑

第一版探针我是这么写的：

```objc
Canary *c = [Canary new];
objc_autorelease(c);     // ARC 下
```

跑起来直接 SIGSEGV。原因是 ARC 在作用域结束时还会给 `c` 补一次 `release`，对象当场归零，池里留了个野指针，等池排空时炸。第二版改成 MRC 编译，用 `[[[Canary alloc] init] autorelease]`。又炸了一次。这回是 `Canary` 的 `tag` 字段挂着同一个池里的 autoreleased 字符串，`dealloc` 里去读的时候它已经没了。

两次都是仪器的问题，不是被观察对象的问题。观察自动释放池的时候，探针本身必须完全不依赖池。否则你测的是自己造的现象。这条我在写 MRC 那篇的时候被提醒过一次，这次还是又踩了一遍。

---

## 十、2015 年那篇经典文章，今天哪些不成立了

先说清立场。ibireme 那篇的骨架至今是我见过最好的。mode / source / timer / observer 的关系、休眠靠 mach_msg、CATransaction 挂在 BeforeWaiting 上，这些今天全都站得住。下面列的是我实测对不上的部分，每一条都注明我实际扫了哪个自变量。

「主线程 RunLoop 有 5 个预置 mode。」 我这里读到的是：命令行工具 1 个，UIKit App 启动完 11 个，跑一会儿 13 个。`UIInitializationRunLoopMode` 在本机 dyld 共享缓存里一次都搜不到。`GSEventReceiveRunLoopMode` 作为符号还在，`dlsym` 能取到，值就是同名字符串，但它没进 mode 列表。自变量只扫了平台的一个取值：macOS / Catalyst，iOS 真机没测。不过哪怕真机上真是 5 个，"预置"这个词也已经不准，因为数量会在运行期增长。

「两个 observer 管自动释放池，order 是 ±2147483647，activities 是 0x1 和 0xa0。」 这两项在我能读到的所有 observer 列表里都没有。实际生效的是每次 callout 各自一个池。函数符号还在共享缓存里。自变量同上，只有一个平台取值。

「`_UIApplicationHandleEventQueue` 是事件回调的入口。」 这个名字在本机共享缓存里出现 0 次，精确匹配和模糊匹配都是 0。存在的是 `_UIApplicationHandleEvent`，以及一个 source0 的 callout `__eventFetcherSourceCallback`（第一节的 dump 里 `order = -2`）。拿这个函数名讲"触摸事件怎么进 App"的文章，符号名要换了。

这三条 `strings` 结论共用同一个自变量限制：搜的是 macOS 的 dyld 共享缓存，UIKitCore 取的是 `/System/iOSSupport` 下那一份。iOS 设备上的缓存我没搜过。

「`CFRunLoopRunSpecific` 是 run 的实现。」 今天的栈里是 `_CFRunLoopRunSpecificWithOptions`。`CFRunLoopRunSpecific` 这个符号还在，`dlsym` 取得到，但 `CFRunLoopRun` 不走它。你在崩溃日志里看到的不是它那一帧。

「一个 mode 里一个 item 都没有，RunLoop 直接退出。」 有两个例外。observer 不算 item，只加 observer 的 mode 照样立刻退出。主线程上的 common mode 永远判为非空，因为它代表 libdispatch 主队列。这条在源码和运行时都对得上，是老文章漏讲不是讲错。

「把 timer 加到 `NSRunLoopCommonModes` 就能在滚动时继续走。」 只在目标 mode 已经被标记成 Common 时成立。UIKit 替你标记了 `UITrackingRunLoopMode`，所以在 App 里成立；换到自建 mode 上就是空操作，不报错。

「`kCFRunLoopCommonModes` 是伪 mode，不是真的 mode。」 语义上对，`CFRunLoopRunInMode` 传它会被 CF 明确拒绝并打日志。但 Catalyst 那份 dump 的 `modes` 集合里，确实有一项名字就叫 `kCFRunLoopCommonModes`。说明系统里有代码把它当真 mode 名注册过。这条我只观察到，没查出来源。

最后一条不算"不成立"。那篇文章自己就提醒过，只是后来抄它的人普遍丢掉了：它引的是 CF-855.17 的源码。我今天能下到的最新开源版本，时间戳还是 2015 年。本机运行的是 6.9 - 5026.5.4.1.402。凡是从这份源码得到的函数名、字段名、代码顺序，都要当成"十一年前的样子"来读。

---

## 总结

主线程 RunLoop 的 mode 数量是运行期长出来的，不是一张固定的表。想知道自己 App 里有几个，`NSLog(@"%@", CFRunLoopGetMain())` 一行就够，比读十篇文章都准。

source0 和 source1 的区别不止在"谁来唤醒"。source0 被 signal 之后如果没人 `CFRunLoopWakeUp`，事件根本不会到。醒了之后它还要等到下一轮的 `BeforeSources` 才处理。source1 是唤醒这件事本身，回调紧跟 `AfterWaiting`，同一轮结束战斗。

`NSRunLoopCommonModes` 是"所有打了 Common 标记的 mode"，不是"所有 mode"。目标 mode 没标记的时候它是空操作，不报错不打日志。

`tolerance` 在重复 timer 上被硬性截断到 interval 的一半。头文件只说 "may put a maximum value"，没给数。而错过的触发一次都不补，`NSTimer` 保证的只有"不早于"。

自动释放池这一侧，我实测到的是每次 callout 各自一个池。`Entry` 里 autorelease 的对象活不到 `BeforeTimers`。那两个 order 为 ±2147483647 的 observer 我读不到，但函数符号还在共享缓存里。iOS 真机上是什么样，我没测过。这一条留着。

方法论还是老一套。这篇里所有"网上说的不对"，没有一条来自读更多文章。`CFRunLoopGetMain()` 打一遍，`sample` 抓一次栈，`strings` 翻一遍共享缓存，把每个说法编成一个能跑的 case。而且每下一条否定结论之前，我都逼自己回答一次：这个实验有几个自变量，我扫了几个。第九节和第十节里那几处"只扫了平台的一个取值"，就是这么留下来的。

下一篇 [[iOS AutoreleasePool：哨兵、页链表与 RunLoop 的关系]] 接着讲池自己那一半。

## 参考资料

### 一手

- [apple-oss-distributions/CF — CFRunLoop.c](https://github.com/apple-oss-distributions/CF/blob/main/CFRunLoop.c)：本文所有源码引文出处。注意版权头是 2015，和你机器上跑的版本差了十一年
- `$(xcrun --sdk macosx --show-sdk-path)/System/Library/Frameworks/Foundation.framework/Headers/NSTimer.h`：tolerance 的完整措辞、"不 drift"的承诺
- [Threading Programming Guide — Run Loops](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)：Anatomy 一节那张官方时序图，本文第二节是拿它逐条对的
- [CFRunLoop](https://developer.apple.com/documentation/corefoundation/cfrunloop)：activity 常量与 mode 相关 API 的定义

### 二手

- [ibireme — 深入理解 RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)：中文圈几乎所有 RunLoop 文章的源头，结构判断至今可用；第十节列的是对不上的部分
- [Dive into CFRunLoop](https://suelan.github.io/2021/02/13/20210213-dive-into-runloop-ios/)：跟着源码走读，比多数中文教程新
- [macOS CFRunLoop Internals](https://meldstudio.co/blog/macos-cfrunloop-internals-scheduling-high-precision-timers-and-recurring-tasks/)：高精度定时和 mach port 那一段写得细

### 本地

- [[iOS GCD：队列不是线程，以及死锁的准确边界]]：主队列必须有 RunLoop 才回到主线程，底下走的就是本文第三节那个 dispatchPort
- [[iOS NSOperation：状态机、依赖与自定义并发 Operation]]
- [[iOS AutoreleasePool：哨兵、页链表与 RunLoop 的关系]]
- [[RunLoop 延迟加载图片]]：`performSelector:withObject:afterDelay:inModes:` 只传 `NSDefaultRunLoopMode`，让图片赋值等到滚动停下来。这个技巧成立的前提正是第五节讲的 mode 隔离
- [[RunLoop 与 AutoReleasePool]]：早期笔记，本文是它的替代版本

---

实验环境：macOS 26.5.2（25F84），Apple Silicon（arm64），Xcode 26.6 / Apple clang 21.0.0，`kCFCoreFoundationVersionNumber = 5026.5`，运行时 CoreFoundation 镜像版本 `6.9 - 5026.5.4.1.402`。

绝大多数实验是原生 macOS 二进制：

```shell
clang -fobjc-arc -framework Foundation -framework CoreFoundation -o out prog.m && ./out
```

第九节的 Canary 探针用 `-fno-objc-arc` 编，避免 ARC 在探针里插 retain/release。第一节和第九节的 UIKit 部分用 Mac Catalyst target 编（`-target arm64-apple-ios17.0-macabi`），打成 `.app` bundle、`codesign -s -` 之后用 `open -W` 启动。全程没有 boot 任何模拟器。

还有个坑。Catalyst 上 `UIApplicationMain` 第一次跑会 SIGTRAP，报 "More than one NSApplication instance was created"。解法是在调 `UIApplicationMain` 之前先取一次 `[NSApplication sharedApplication]`。macabi target 下不能直接链 AppKit，用 `NSClassFromString` 加 KVC 拿。这个细节我没在别处见人写过，记在这里。

> 待真机补测：第一节的 mode 清单、第九节的 autorelease 池行为、以及 `UITrackingRunLoopMode` 在真实滚动时的切换，都需要在 iOS 真机上复现一次。复现方法：把 `NSLog(@"%@", CFRunLoopGetMain())` 和 Canary 探针原样放进一个真机 App 的 `viewDidAppear:`，滚动一个 tableView 的同时用 `CFRunLoopCopyCurrentMode` 打当前 mode。
