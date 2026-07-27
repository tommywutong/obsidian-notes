---
title: 【iOS】Block 循环引用与 weak-strong dance
published: 2026-07-26
description: weakSelf 和 strongSelf 解决的是两个不同的问题，不是一个操作的两个步骤。搞混了就解释不了「为什么 strongSelf 不会造成新的环」。
tags:
  - iOS
  - Objective-C
  - Block
  - Memory
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 10
draft: true
---
# Block 循环引用与 weak-strong dance

`weakSelf` 断的是环。`strongSelf` 防的是另一件事——block 执行到一半，对象没了。

这两件事经常被打包成一个叫 "weak-strong dance" 的模板，于是变成了"一个操作的两个步骤"。搞混的代价是：你解释不了为什么加了 `strongSelf` 不会重新造成循环引用。

前两篇讲清楚了 Block 持有对象的机制（[[iOS Block 的结构：ABI、descriptor 与三种类型|BLOCK_HAS_COPY_DISPOSE]]）和 `__block` 的搬家机制（[[iOS Block 的变量捕获与 __block]]）。环是这两件事的直接推论：

```text
Node ──handler──▶ Block
  ▲                 │
  └────捕获 n───────┘
```

两个箭头都是强引用，谁也释放不了谁。这一篇不再展开原理，只关心怎么断。

---

## 一、六种写法

用一个 weak 探针就能判断有没有泄漏：对象离开作用域后探针还非 nil，就是泄漏了。

| 写法 | 结果 | 强引用从哪来 |
| --- | --- | --- |
| ① 直接用 `self` | 泄漏 | Block copy 时 helper 里 `objc_storeStrong` |
| ② `weakSelf` | 释放 | 捕获的是 `__weak`，不 retain |
| ③ `__block` 修饰对象 | 泄漏 | byref helper 里 `objc_storeStrong`，layout 是 `STRONG` |
| ④ `__block` + 执行后置 nil | 释放 | 同 ③，靠执行时手动打断 |
| ⑤ 只捕获需要的值 | 释放 | 压根没捕获对象 |
| ⑥ `__block __unsafe_unretained` | 释放 | layout 是 `UNRETAINED`，不生成 helper |

实测输出：

```text
① block 里直接用 self          还活着（泄漏）
② 改用 weakSelf                已释放
③ 用 __block 修饰对象          还活着（泄漏）
④ __block + 调用后手动置 nil    已释放
⑤ 只捕获需要的值               已释放
⑥ __block __unsafe_unretained  已释放
```

### 第 ③ 和第 ⑥ 行放在一起才有意思

流行的说法是"MRC 下 `__block` 能断环，ARC 下不能"。前半句对，后半句不准确——**ARC 下 `__block` 默认跟着 strong，但你可以显式改。**

上一篇量过 byref 的 layout 字段：

```text
ARC  __block id                     layout=STRONG      HAS_COPY_DISPOSE=1
ARC  __block __unsafe_unretained    layout=UNRETAINED  HAS_COPY_DISPOSE=0
MRC  __block id                     layout=UNRETAINED  HAS_COPY_DISPOSE=1
```

MRC 那行的 layout 就是 `UNRETAINED`——所以老经验成立。ARC 把默认值改成了 `STRONG`，于是同一句 `__block id` 换了个语义。而你只要写上 `__unsafe_unretained`，layout 又回到 `UNRETAINED`，helper 一个都不生成，断环能力就回来了。

所以变的不是"`__block` 这个关键字的能力"，是**所有权修饰符的默认值**。libclosure 一行没改，改的是编译器给 byref 生成什么 helper。

顺带说，`__block __weak` 也能断环，而且比 `__unsafe_unretained` 安全（对象没了会置 nil 而不是留悬垂指针）。但如果只是为了断环，直接用 `weakSelf` 更直白，没必要绕 `__block`。

### 第 ④ 种的代价

```objc
__block Node *blockN = n;
n.handler = ^{ NSLog(@"%@", blockN.name); blockN = nil; };
n.handler();   // 必须真的执行
```

它确实能解开，但**环的解除依赖 block 被执行**。如果这是个可能永远不会触发的回调（网络失败、用户没点、页面提前关掉），环就永远留着。而且这个依赖不写在方法签名上，也不写在属性声明上，你得读进 block 内部才知道。

我的判断是这种写法基本不该用。它唯一还算合理的场景是"这个 block 一定只执行一次且一定会执行"，但满足这个条件的话 `weakSelf` 同样能解决，还不用背这个额外假设。

第 ⑤ 种最省事：block 里只需要某个属性的值，就在外面先取出来捕获那个值，压根别碰 `self`。能用这招的时候优先用。

### 编译器其实会帮你看着

`-Warc-retain-cycles` 默认开启：

```text
warning: capturing 'self' strongly in this block is likely to lead to a retain cycle
    self.h = ^{ (void)self->_n; };
                      ^~~~
note: block will be retained by the captured object
```

而且它比我预想的聪明——不光认 `self` 的直接形态，把对象作为函数参数传进来再赋值给它自己的 block 属性，照样报：

```text
warning: capturing 'n' strongly in this block is likely to lead to a retain cycle
```

但它只能看见一个函数体内能看全的持有关系。环绕过一层数据结构（存进数组、经第三方对象中转），或者跨越两个以上对象，它就无能为力了。它是第一道防线，不是全部。

---

## 二、weakSelf 之后剩下的问题

先给结论：

| | 解决的问题 | 存在时长 |
| --- | --- | --- |
| `weakSelf` | 断开 block 与对象之间的**永久**持有 | 与 block 同寿 |
| `strongSelf` | 保证对象在**这一次执行**期间不消失 | 一次调用 |

下面这个实验说明为什么第二行是必要的。

让 block 在后台队列里读两次 weak 变量，中间人为拉开一个窗口，期间主线程释放对象：

```objc
__weak Worker *weakW = w;
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"    第一次读 weakW.tag = %@", weakW.tag);
    dispatch_semaphore_signal(sem);
    usleep(50000);
    NSLog(@"    第二次读 weakW.tag = %@", weakW.tag);
});
```

```text
    第一次读 weakW.tag = A
    [dealloc] A
    第二次读 weakW.tag = (null)
```

同一个 block、同一个变量，前后两次读到不同的东西。

这就是 `__weak` 的语义：对象销毁，弱引用置 nil。不是什么并发边界情况。

问题在于 block 里的代码通常假设"我读到的这个对象在我这段逻辑里一直有效"。这个假设不成立时，轻则逻辑错乱——前半段基于对象状态做了判断，后半段对象没了走进了错误分支；重则静默失败。Objective-C 给 `nil` 发消息不崩，后半段的操作于是全变成空操作。你什么都看不到。

### strongSelf 把对象钉住

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    __strong Worker *strongW = weakW;
    if (!strongW) { return; }
    NSLog(@"    第一次读 strongW.tag = %@", strongW.tag);
    usleep(50000);
    NSLog(@"    第二次读 strongW.tag = %@", strongW.tag);
});
```

```text
    第一次读 strongW.tag = B
    第二次读 strongW.tag = B
    [dealloc] B
    block 结束后 weakW = 已释放
```

两次都读到 `B`。最后一行显示 block 结束后弱引用已释放，说明那个强引用是临时的。它只活在 block 这一次执行期间，随局部变量出作用域而消失，不会形成新的环。

这就是"为什么 strongSelf 不会造成循环引用"的答案：环的定义是永久互相持有，而 `strongSelf` 是个局部变量，block 一执行完就没了。对象只是多活了一次 block 执行的时间。

### 但这行 dealloc 发生在后台队列上

回头看那组输出的倒数第二行——`[dealloc] B` 打在 block 执行完之后，也就是**在 global queue 上**。

因为 `strongW` 出作用域时，如果它恰好是最后一个强引用，对象就在当前线程销毁。对 `UIView`、`UIViewController` 这类必须在主线程释放的对象，这会直接崩。

这是 weak-strong dance 一个真实存在的副作用，而且很隐蔽——它只在"主线程已经放手、后台 block 拿着最后一个引用"这个时序下发生。要防的话，让最后的释放回到主线程：

```objc
dispatch_async(dispatch_get_main_queue(), ^{ (void)strongW; });
```

### 什么时候可以不写 strongSelf

判据不是"block 里访问了几次 self"，而是"**`weakSelf` 这个变量被读取了几次**"。这两个说法不一样。

```objc
[weakSelf.a.b doSomething];        // 一次 weak 读，整条链在同一次 retain 保护下
[weakSelf doA]; [weakSelf doB];    // 两次 weak 读，中间可能变
```

单次读取有语言级保证。编译器插入的是 `objc_loadWeakRetained`，读出来的值被 retain 住，一直保到当前 full-expression 结束——这不是运气，是 [[iOS weak 的实现：SideTable 与置 nil 的时机|ARC 规范写明的语义]]。

不过"单次就安全"这个说法要收窄一点。它没有**状态不一致**问题，但仍然有**该做的事没做**问题。`[weakSelf reload]` 静默不执行无所谓；`[weakSelf.delegate didFinishWithResult:r]` 静默不执行就是丢了一次事件。要不要处理，取决于这个调用可不可丢。

判断"读了几次"本身也是负担，而且代码会演化——今天一次，明天有人加一行就变两次。所以一律写 dance 也是合理选择，成本两行。我自己的做法是：只要 block 里有分支或者多于一条语句，就写。

---

## 三、`if (block) block()` 的那个窗口

调用 block 不是"跳到 block 指针指向的地址"。它要先从 block 结构体偏移 16 处取出 `invoke` 函数指针，再跳过去。指针是 NULL 时，崩在"取"这一步：

```asm
ldr x0, [sp, #16]   ; x0 = block 指针 = NULL
ldr x8, [x0, #16]   ; x8 = block->invoke   ← 崩在这条
blr x8              ; 从来没执行到
```

lldb 里看到的是 `EXC_BAD_ACCESS (address=0x10)`。`0x10` 就是 16，正是 [[iOS Block 的结构：ABI、descriptor 与三种类型|Block_layout 里 invoke 的偏移]]。跳转从未发生。

所以判空是必要的，它解决"这个 block 属性从来没被赋值过"。

但下面这个写法在并发下还有问题：

```objc
if (self.completion) {
    self.completion(result);
}
```

不过问题不是很多文章说的"旧值被 release 导致悬垂指针"。ARC 已经在调用点插了 retain 和 release：

```llvm
%13 = call ptr @"objc_msgSend$completion"(...)
%14 = call ptr @llvm.objc.retainAutoreleasedReturnValue(ptr %13)
call void %17(ptr %14, ptr %16)              ; 调用 block
call void @llvm.objc.release(ptr %14)
```

调用期间 block 被释放这个窗口，ARC 早就关掉了。剩下的问题分两种：

| | 悬垂指针 | 判空过了、调用前变 nil |
| --- | --- | --- |
| `nonatomic` | 窗口极窄（裸 load 返回到 ARC retain 之间） | 有 |
| `atomic` | 无（getter 在锁内 retain + autorelease） | 有 |
| 取本地变量 | 无 | 无 |

真正剩下的是 TOCTOU：判空读了一次属性，调用又读了一次，两次之间属性可能被改成 `nil`，于是调用一个 NULL——回到本节开头那个 `0x10`。

```objc
void (^completion)(id) = self.completion;   // 只读一次
if (completion) {
    completion(result);
}
```

只读一次，检查和使用作用在同一个值上，窗口消失。这跟上一节 `strongSelf` 是同一个手法：把一个可能变化的引用先固定成不变的局部值。一个作用于 `self`，一个作用于 block 本身。

我的做法是一律取本地变量。不是因为竞态概率高（它很低），而是这个写法顺带把意图讲清楚了——`self.completion` 每写一次就是一次消息发送，取一次本地变量之后，读几次、读的是不是同一个值，都一目了然。两行变三行，没有理由不写。

---

## 四、几个说法需要辨析

> "`__block` 可以打破循环引用。"

MRC 下成立。ARC 下 `__block id` 默认是 `STRONG` layout，断不了；但写成 `__block __unsafe_unretained` 就又能了。变的是默认值，不是关键字的能力。

> "加了 strongSelf 又形成了循环引用。"

不会。环是永久互相持有，`strongSelf` 是局部变量，block 执行完就释放。实测 block 结束后弱引用即显示已释放。

> "weak-strong dance 是为了防循环引用。"

防环的是 `weakSelf` 那一半。`strongSelf` 防的是执行期间对象消失。把两者混为一谈，就解释不了为什么 `strongSelf` 不会造成新的环。

> "用了 weakSelf 就万事大吉。"

单次读取有 `objc_loadWeakRetained` 保底，多次读取会读到不一致的状态。而且 dance 本身也有副作用，见第二节末尾那个后台线程 dealloc。

> "`if (block) block();` 是线程安全的。"

不是，但原因是 TOCTOU 而不是悬垂指针。见第三节。

> "所有 block 属性都要担心循环引用。"

判据可以做得比"有没有被长寿对象持有"更硬：**翻 SDK 头文件看这个参数带不带 `NS_NOESCAPE`**。带的话编译器保证它不逃逸，写 `weakSelf` 纯属噪音。

但要注意别把"不成环"和"不持有"混为一谈。`[UIView animateWithDuration:animations:completion:]` 的参数**没有** `NS_NOESCAPE`——它不会造成永久环，但会持有你的 block 直到动画结束。对一个正在被 pop 的 VC 来说，这意味着 dealloc 被推迟，是能感知到的。

---

## 总结

六种写法里，`weakSelf` 是标准解法，`__block` 加显式所有权修饰符也能断环，只捕获值最省事，`__block` 加手动置 nil 把生命周期押在"block 一定会执行"上、不建议。

`weakSelf` 和 `strongSelf` 解决的是两个不同的问题：前者断开永久持有，后者保证单次执行期间对象不消失。实测同一个 block 里两次读 weak 变量能读到 `A` 和 `null`，这就是后者存在的理由；而 `strongSelf` 是局部变量、执行完即释放，所以不会造成新的环。代价是最后那次释放可能发生在后台线程上。

下一篇进入第三周：[[iOS Method Swizzling：正确姿势、+load 时机与那些坑]]。

## 参考资料

### 规范与源码

- [Clang — Block Implementation Specification](https://clang.llvm.org/docs/Block-ABI-Apple.html)：`Block_layout` 的字段顺序，第三节那个 `0x10` 从这里来
- [Clang ARC Specification](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)：`__weak` 读取的 full-expression 保活语义
- [apple-oss-distributions/libclosure — runtime.cpp](https://github.com/apple-oss-distributions/libclosure/blob/main/runtime.cpp)：`_Block_object_assign` 里那条分支的注释写明了"只服务 MRC unretained `__block`"
- [Apple — Working with Blocks](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html)

### 拓展

- [iOS 中的 block 是如何持有对象的](https://github.com/draveness/analyze/blob/master/contents/FBRetainCycleDetector/iOS%20%E4%B8%AD%E7%9A%84%20block%20%E6%98%AF%E5%A6%82%E4%BD%95%E6%8C%81%E6%9C%89%E5%AF%B9%E8%B1%A1%E7%9A%84.md)：FBRetainCycleDetector 判断 block 强捕获的办法很漂亮——伪造一个 block，把捕获槽填上重写了 `release` 的哨兵对象，然后调它的 dispose helper，谁被 release 了谁就是强捕获。等于把前两篇讲的 `BLOCK_HAS_COPY_DISPOSE` 机制反过来用
- [深入理解 weak-strong dance](https://luohs.github.io/2017/05/31/20170531/)

### 本地

- [[iOS Block 的结构：ABI、descriptor 与三种类型]]
- [[iOS Block 的变量捕获与 __block]]
- [[iOS weak 的实现：SideTable 与置 nil 的时机]]

---

实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -fobjc-arc -target arm64-apple-ios17.0-simulator`。所有输出块都是真实运行结果。

第二节那个实验用信号量和 `usleep` 人为拉开了时间窗口，目的是让一个真实存在但概率很低的竞态稳定复现。真实代码里这个窗口通常只有几微秒——这类 bug 的特征正是"线上偶现、本地永远复现不了"。

用 weak 探针判断泄漏很轻量，适合写进单元测试，但它只能判断"这一个对象有没有被释放"，查不出引用链。查复杂的环用 Memory Graph Debugger。

不过 Memory Graph 有个边界值得知道：它的泄漏徽标只标 **unreachable** 的对象。像 NSTimer、NotificationCenter 这类环，根是 RunLoop 或通知中心——都是进程级全局根，对象**可达**，Memory Graph 什么都不会报。那类"没释放但可达"的问题要用 Instruments 的 Allocations 做 generation 对比，或者对导出的 `.memgraph` 跑 `leaks --traceTree`（后者是唯一能进 CI 的）。
