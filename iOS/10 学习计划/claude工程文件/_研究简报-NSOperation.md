---
title: NSOperation 研究简报（给写作 agent 用）
draft: true
---
# NSOperation / NSOperationQueue 研究简报

信源编号：
**A** 现行文档 `developer.apple.com/documentation/foundation/operation`｜**B** 同上 `/operationqueue`｜**C** Concurrency Programming Guide（归档，页脚 Updated: 2012-12-13，已停更）｜**D** 本机 SDK 头文件 `NSOperation.h`（一手，最权威）｜**E** NSHipster（约 2014）｜**F** nsprogrammer（2021-07）｜**G** swift-corelibs-foundation `Operation.swift`（**⚠️ 是 Linux/Windows 重写实现，不是 Apple 平台跑的代码**）｜**H** 本机在 Apple 闭源 Foundation 上实测

---

## 一、最值得写的五个高信息量点

### 1. NSOperationQueue 就是 GCD 之上的一层 —— 有调用栈实证

实测拿到的栈（H）：

```
dispatch label: MyProbeQueue (QOS: UNSPECIFIED)
thread: <NSThread: 0x1029c1580>{number = 2, name = (null)}
  1   Foundation        __NSBLOCKOPERATION_IS_CALLING_OUT_TO_A_BLOCK__ + 24
  2   Foundation        -[NSBlockOperation main] + 88
  3   Foundation        __NSOPERATION_IS_INVOKING_MAIN__ + 16
  4   Foundation        -[NSOperation start] + 640
  5   Foundation        __NSOPERATIONQUEUE_IS_STARTING_AN_OPERATION__ + 16
  6   Foundation        __NSOQSchedule_f + 164
  7   libdispatch.dylib _dispatch_block_async_invoke2 + 148
  8   libdispatch.dylib _dispatch_client_callout + 16
  9   libdispatch.dylib _dispatch_continuation_pop + 596
  10  libdispatch.dylib _dispatch_async_redirect_invoke + 580
  11  libdispatch.dylib _dispatch_root_queue_drain + 360
  12  libdispatch.dylib _dispatch_worker_thread2 + 184
  13  libsystem_pthread _pthread_wqthread + 232
```

要点：`__NSOQSchedule_f` 被 `_dispatch_block_async_invoke2` 拉起，线程来自 libdispatch 的全局 worker 池；`dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL)` 返回的正是我给 `NSOperationQueue` 设的 `name`。那几个全大写符号是 Apple 为了让崩溃栈可读故意留的路标。

官方也已明说（不再是私有细节）：

> A（Operation 页）: An operation queue executes its operations either directly, by running them on secondary threads, or indirectly using the **`libdispatch`** library (also known as Grand Central Dispatch).
>
> B（OperationQueue 页）: **Operation queues use the `Dispatch` framework** to initiate the execution of their operations. As a result, queues always invoke operations on a separate thread, regardless of whether the operation is synchronous or asynchronous.

NSHipster 那句 "though that's a private implementation detail" 已过时。

### 2. 加入队列后 `isAsynchronous` 被忽略 —— 大多数中文博客讲反了

三处官方原文互相印证：

> A: **When you add an operation to an operation queue, the queue ignores the value of the `isAsynchronous` property and always calls the `start()` method from a separate thread. Therefore, if you always run operations by adding them to an operation queue, there is no reason to make them asynchronous.**
>
> C: The ability to define concurrent operations is **only necessary in cases where you need to execute the operation asynchronously without adding it to an operation queue.**

**自定义 async operation 的真正必要性来自「`main()` 一返回队列就认为 op 结束了」，不是「队列需要 isAsynchronous 才并发」。**

### 3. cancel 是加速依赖链，不是切断它

> A: The dependencies supported by `NSOperation` **make no distinction about whether a dependent operation finished successfully or unsuccessfully. (In other words, canceling an operation similarly marks it as finished.)** It is up to you to determine whether an operation with dependencies should proceed in cases where its dependent operations were cancelled.
>
> B: **Canceling an operation causes the operation to ignore any dependencies it may have.** This behavior makes it possible for the queue to invoke the operation's `start()` method as soon as possible.

实测（H）：A 被 cancel，B 依赖 A：

```
after cancel A: A.isCancelled=1 A.isReady=1 A.isFinished=0 B.isReady=0
B ran (successor of cancelled A)
-> B.isFinished=1 (successor DID run despite dep cancelled)
```

注意 `A.isReady` 立刻变 1，但 `A.isFinished` 仍是 0——必须等队列真的 call `start()` 把它推到 finished，B 才 ready。

### 4. `defaultMaxConcurrentOperationCount` 恒为 -1，文档描述具误导性

文档只说 "The operation queue determines this number dynamically based on current system conditions"。但头文件 D 第 103 行：

```objc
static const NSInteger NSOperationQueueDefaultMaxConcurrentOperationCount = -1;
```

实测（H）：

```
defaultMaxConcurrentOperationCount = -1
new queue default maxConcurrentOperationCount = -1
mainQueue max = 1, underlyingQueue label = com.apple.main-thread
```

**任何"打印它可以看到系统给你几个并发"的说法都是错的，永远打印 -1。** 开源实现里 `-1` 被翻译成 `Int32.max`（不限制），真正的限流交给 GCD。

### 5. Apple 对 Swift Concurrency 时代的态度只体现在 API 标注上

头文件 D：

```objc
- (void)waitUntilFinished
    NS_SWIFT_UNAVAILABLE_FROM_ASYNC("Use completionBlock or a dependent Operation instead");
- (void)waitUntilAllOperationsAreFinished
    NS_SWIFT_UNAVAILABLE_FROM_ASYNC("Use addBarrierBlock or a dependent Operations instead");
- (void)addOperationWithBlock:(void (NS_SWIFT_SENDABLE ^)(void))block NS_SWIFT_DISABLE_ASYNC;
```

同时：`NSOperation` / `NSBlockOperation` / `NSOperationQueue` 全部标了 `NS_SWIFT_SENDABLE`，整个头文件包在 `NS_HEADER_AUDIT_BEGIN(nullability, sendability)` 里——Apple 主动为 Swift 6 严格并发做了适配。而 `addBarrierBlock:` 和 `progress` 是 macOS 10.15 / iOS 13 才加的新 API。

**类本身全平台 `deprecated: false`。"Apple 已废弃 GCD 和 OperationQueue"是社区推论，不是 Apple 原话。** 准确表述：**类不弃用、Sendable 已适配，但阻塞式等待在 async 上下文里编译不过，并给了替代方案。**

---

## 二、状态机与 KVO 合规

### KVO key path 清单（A，现行）

> `isCancelled` / `isAsynchronous` / `isExecuting` / `isFinished` / `isReady` / `dependencies` / `queuePriority` / `completionBlock`

⚠️ CPG（C）那份把第二条写成 `isConcurrent`，已过时。头文件印证：`@property (readonly, getter=isConcurrent) BOOL concurrent; // To be deprecated; use and override 'asynchronous' below`

### 关键原文

> **isFinished（A）**：An operation object **does not clear a dependency until the value at the `isFinished` key path changes to `true`. Similarly, an operation queue does not dequeue an operation until the `isFinished` property contains the value `true`. Thus, marking operations as finished is critical to keeping queues from backing up** with in-progress or cancelled operations.
>
> **C 说得更狠**：Even if an operation is canceled, you should always notify KVO observers that your operation is now finished with its work. ... **Failing to generate a finish notification can therefore prevent the execution of other operations in your application.**
>
> **isCancelled（A）**：Support for cancellation is voluntary but encouraged and **your own code should not have to send KVO notifications for this key path.**

→ `isCancelled` 是唯一一个官方明说"不用自己发 KVO 通知"的；`isExecuting` / `isFinished` 必须自己发。

> **isReady 独立页**：If you want to use custom conditions to define the readiness of your operation object, reimplement this property... **your custom implementation must get the default property value from `super` and incorporate that readiness value into the new value of the property.**

### 自定义并发 Operation 必须重写（A）

> `start()` / `isAsynchronous` / `isExecuting` / `isFinished`

### `start()` 里绝对不能调 super —— 三处原文

> A（Subclassing Notes）：**At no time in your `start()` method should you ever call `super`.**
>
> A（start() 独立页）：**Your custom implementation must not call `super` at any time.**
>
> C（Table 2-2）：**Your implementation must not call super at any time.**

`main()` 同样：**In your implementation, do not invoke `super`.** 而且 `main()` 会自动包在 NSOperation 提供的 autorelease pool 里。

### 默认 start() 做了什么

> The default implementation of this method updates the execution state of the operation and calls the receiver's `main()` method. ... **if the receiver was cancelled or is already finished, this method simply returns without calling `main()`.** ... **If the operation is currently executing or is not ready to execute, this method throws an `NSInvalidArgumentException` exception.**

实测异常文案（H）：

```
start when not ready: NSInvalidArgumentException:
*** -[NSBlockOperation start]: receiver is not yet ready to execute
```

---

## 三、maxConcurrentOperationCount = 1 不等于串行队列

**C 的原文（唯一一处，很重要）**：

> Passing a value of 1 to this method causes the queue to execute only one operation at a time. **Although only one operation at a time may execute, the order of execution is still based on other factors, such as the readiness of each operation and its assigned priority. Thus, a serialized operation queue does not offer quite the same behavior as a serial dispatch queue in Grand Central Dispatch does.** If the execution order of your operation objects is important to you, you should use dependencies to establish that order before adding your operations to a queue.

B 的对应表述：

> If all of the queued operations have the same `queuePriority` and the `isReady` property returns `true`, the queue invokes them in the order you added them. **However, don't rely on queue semantics to ensure a specific execution order** because changes in the readiness of an operation can change the resulting execution order.

实测（H）：max=1、8 个等价 op、无优先级差异 → 严格 FIFO `0 1 2 3 4 5 6 7`。但这只印证了"same priority + all ready ⇒ 按添加顺序"，不构成保证。

**依赖 vs 优先级（C）**：

> **Priority levels are not a substitute for dependencies.** Priorities determine the order in which an operation queue starts executing only those operations that are currently ready. ... Priority levels apply only to operations in the same operation queue.

---

## 四、cancel 的语义

> **This method does not force your operation code to stop.** Instead, it updates the object's internal flags to reflect the change in state. **If the operation has already finished executing, this method has no effect.**
>
> **If you cancel an operation that is not in a queue, this method immediately marks the object as finished.**
>
> C: **If an operation were terminated outright, there might not be a way to reclaim resources that had been allocated.** As a result, operation objects are expected to check for cancellation events and to exit gracefully.

建议检查 `isCancelled` 的位置（C）：任何实际工作之前；循环每次迭代至少一次；任何容易中止的点。**`isCancelled` 本身很轻量，频繁调用没有显著开销。**

`cancelAllOperations()`：

> **Canceling the operations does not automatically remove them from the queue or stop those that are currently executing.** ... In both cases, a finished (or canceled) operation is still given a chance to execute its completion block before it is removed from the queue.

---

## 五、循环依赖

> A（addDependency 页）：**It is a programmer error to create any circular dependencies among a set of operations. Doing so can cause a `deadlock` among the operations and may freeze your program.**
>
> C：One thing that is not acceptable is to create circular dependencies between operations. Doing so is a programmer error that **will prevent the affected operations from ever running.**

⚠️ **技术上 C 更准确**——循环依赖不会 hang 住调用线程，只是那几个 op 永远 `isReady == false` 卡在队列里。"freeze your program"是夸张说法。建议两句都引，并指出后者更精确。

时序陷阱（C）：

> **You should always configure dependencies before running your operations or adding them to an operation queue. Dependencies added afterward may not prevent a given operation object from running.**

`dependencies` 属性：**Operations are not removed from this dependency list as they finish executing.**

---

## 六、NSBlockOperation 的多个 block 是并发的

> The `BlockOperation` class ... **manages the concurrent execution of one or more blocks.** ... When executing more than one block, the operation itself is considered finished only when all blocks have finished executing.
>
> C: When it comes time to execute an `NSBlockOperation` object, the object submits all of its blocks to the **default-priority, concurrent dispatch queue**.

实测（H）：一个 `NSBlockOperation` 加 6 个各 sleep 120ms 的 block：

```
BlockOperation 6 blocks: overlapping=1 distinctThreads=6
```

异常文案：

```
*** -[NSBlockOperation addExecutionBlock:]: blocks cannot be added after the operation has started executing or finished
```

📌 **反直觉点**：把 `maxConcurrentOperationCount = 1` 当串行用，然后往**一个** `BlockOperation` 里塞多个 block —— 这些 block 仍然并发。串行限制的是 operation 之间，不是一个 BlockOperation 内部的 blocks 之间。

---

## 七、waitUntilFinished 的风险

> **An operation object must never call this method on itself and should avoid calling it on any operations submitted to the same operation queue as itself. Doing so can cause the operation to deadlock.**

更硬的信号在头文件里（见第一节第 5 点）。

顺带：`operations` / `operationCount` 已在头文件里标 `API_DEPRECATED`（网页文档没标）：

```objc
API_DEPRECATED("access to operations is inherently a race condition, it should not be used. "
               "For barrier style behaviors please use addBarrierBlock: instead", ...)
```

---

## 八、underlyingQueue 的约束：文档写了 3 条，实测只强制 1 条

实测（H）：

```
set underlyingQueue=main -> NO exception, label=com.apple.main-thread
set underlyingQueue on non-empty -> EXCEPTION NSInvalidArgumentException:
    *** -[NSOperationQueue setUnderlyingQueue:]: operation queue must be empty in order to change underlying dispatch queue
```

- "非空时设置 → 抛异常"：**强制执行**
- "must not be `dispatch_get_main_queue()`"：**运行时不阻止**，是契约性禁令不是崩溃。很多博客写成"会崩溃"，错的
- `OperationQueue.main.underlyingQueue` 只读

⚠️ 有意思的一致性：开源实现 G 的报错文案 `"operation queue must be empty in order to change underlying dispatch queue"` 与 Apple 闭源抛出的异常 reason **一字不差**，但**行为等级不同**：开源是 `fatalError`（不可捕获），闭源是可 `@catch` 的 ObjC 异常。这是"开源照着闭源复刻但不等于闭源"的最佳实例。

---

## 九、开源实现里的两个亮点（G，⚠️ 非 Apple 平台实现）

### 状态机不是 4 个 Bool，是一个有序枚举

```swift
enum __NSOperationState : UInt8 {
    case initialized = 0x00, enqueuing = 0x48, enqueued = 0x50, dispatching = 0x88
    case starting = 0xD8, executing = 0xE0, finishing = 0xF0, finished = 0xF4
}
open var isFinished: Bool { return __NSOperationState.finishing.rawValue <= _state.rawValue }
```

注意 `isFinished` 用的是 `>= finishing` 而不是 `== finished`——一进入 finishing 就对外报完成，finished 是收尾态。

### 一条诊断输出，精确描述了自定义 async Operation 最常见的致命 bug

```
*** <Class> <ptr> went isFinished=YES without being started by the queue it is in
```

即：在队列还没 call `start()` 之前（比如在 `cancel()` 里）就把 `isFinished` 置为 true。这会让 operation 提前从依赖图里逃逸，队列记账错乱。

`cancel()` 的实现只有 7 行，同时实现了官方文档里的三句话：

```swift
open func cancel() {
    if isFinished { return }                                    // 已完成则无效
    __atomicLoad.lock(); __isCancelled = true; __atomicLoad.unlock()
    if __NSOperationState.executing.rawValue <= _state.rawValue { return }   // 正在跑只置标志
    _lock(); __unfinishedDependencyCount = 0; _unlock()          // 依赖被忽略
    Operation.observeValue(forKeyPath: _NSOperationIsReady, ofObject: self)
}
```

---

## 十、过时与错误说法汇总

| 说法 | 出处 | 判定 |
| --- | --- | --- |
| KVO 清单含 `isConcurrent` | C (2012) | 过时，现行是 `isAsynchronous` |
| 并发 Operation 要重写 `isConcurrent` 返回 YES | C | 过时，改 `isAsynchronous` |
| 循环依赖 "may freeze your program" | A | 夸张，C 的"prevent from ever running"更准确 |
| "NSOperationQueue 底层是 GCD，但那是私有实现细节" | E (2014) | 已过时，现在 A/B 都白纸黑字写了 |
| "**永远不要 override `isReady`**，否则依赖图失效" | F (2021) | **与 Apple 文档直接冲突**。A 明说 "must get the default property value from `super` and incorporate"。不过 F 的结论建议（用其他 Operation 当依赖）工程上仍是好建议 |
| F 的 `cancel` 示例：不调 `[super cancel]`，直接在里面 `_finish` | F | **危险**。会触发"went isFinished=YES without being started"。应该只置 `isCancelled`，把 finish 留给 `start()` |
| `defaultMaxConcurrentOperationCount` 是动态数字 | Apple 属性页 | 误导，恒为 -1 |
| `underlyingQueue` 设成 main queue 会崩溃 | 大量博客 | 错，实测不抛异常 |
| `threadPriority` | C | 已弃用（macOS 10.10 / iOS 8），用 `qualityOfService` |
| "GCD 和 OperationQueue 已被 Apple 废弃" | 大量社区博客 | **无 Apple 依据**，全平台 `deprecated: false` |

---

## 十一、值得引用的一段"作者自打脸"

nsprogrammer（F）原文先写：

> ~~You can implement your `NSOperation` (`Operation` in Swift) using `async await` without issue!~~

然后自己加了 Update：

> ***Update:* I actually spent a good amount of time on this and can't figure out a way to implement `NSOperation` using `async await`… It is simple enough to use `NSOperation` from async contexts, but I could not get the KVO inside `NSOperation` to work using the new concurrency features of Swift 🤷‍♂️.**

配合头文件里的 `NS_SWIFT_UNAVAILABLE_FROM_ASYNC`，构成"这两套并发模型无法平滑缝合"的完整论据。

---

## 十二、建议主实验清单

1. 自定义并发 Operation：**不手动发 KVO 通知会怎样**（队列永远认为它没结束、后续 op 全卡住），再演示正确写法
2. `maxConcurrentOperationCount = 1` 与串行队列的行为差异
3. `cancel` 之后正在执行的 operation 不会停
4. `NSBlockOperation` 多个 block 的并发性与线程数
5. 依赖被 cancel 之后后继仍会执行（三态快照）
6. 用 KVO 观察 `isExecuting` / `isFinished` 的迁移时序
7. 抓一次 libdispatch 调用栈坐实"底层是 GCD"
