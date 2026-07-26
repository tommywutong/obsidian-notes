---
title: 【iOS】对象通信：delegate、通知、target-action 与 block 回调
published: 2026-07-27
description: UIControl 的 target 到底是 weak 还是 unsafe_unretained，我用两条互相独立的证据在两套 UIKit 上各测了一遍。另外，给 block 观察者传 queue 参数并不会让 post 异步返回，传 mainQueue 还能把主线程锁死。
tags:
  - iOS
  - Objective-C
  - Runtime
  - Foundation
  - 设计模式
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 12
draft: true
---
# 对象通信：delegate、通知、target-action 与 block 回调

`[button addTarget:self action:@selector(tap:) forControlEvents:UIControlEventTouchUpInside]` 这一行谁都写过。问题是：button 到底怎么持有 self？

Apple 的文档只写了一句 "The control does not retain the object in the target parameter"。这句话排除了 strong，但没说剩下的是 weak 还是 `unsafe_unretained`。这两个答案的差别不小：前者在 target 死后置 nil，后者留一个野指针。我见过的中文资料里两种说法都有，而且都说得很笃定。

所以我测了。两条互不相干的路子，两套 UIKit：

```text
=== UIControlTargetAction ===  （iOS 26.5 SDK / iOS 18.3 与 26.5 运行时都是这个结果）
  ivarLayout      01
  weakIvarLayout  11
  +8    slot=0   _actionHandler   @"UIAction"   strong
  +16   slot=1   _target          @             weak
```

```text
  注册后 allTargets = {( <Sink: 0x600000004290> )}
  Sink dealloc
  释放后 allTargets = {( <null> )}  count=1
  sendActions 返回，没崩。
```

**UIControl 存的是 weak。** 一边是 ivar 布局元数据，一边是 target 死后集合里出现 `NSNull` 且不崩，两条证据指向同一个答案。

这篇要讲的四种通信方式，教程满大街都是，罗列 API 谁都会。真正拉开差距的是上面这一层：每种方式的运行时基础是什么、持有关系往哪个方向走、忘了清理会发生什么。这三个问题答完，选型是推出来的。

顺带提前说一条最容易踩的：iOS 14 的 `addAction:forControlEvents:` 和上面那个 `_target` 不是一回事，`_actionHandler` 是 strong。从 target-action 迁到 UIAction 的时候，所有权语义悄悄翻了个面。第三节有实验。

上一篇是 [[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]]，KVO 也是一种对象通信，它和本篇的四种放在一起看会更清楚。

---

## 一、四种方式，其实只有三种运行时表示

先把名词拆开。所谓"把行为交给别人"，在 Objective-C 里只有三种做法：

1. 一个 SEL 加一个接收者。target-action 是这个，`addObserver:selector:name:object:` 也是这个。
2. 一个对象加一份编译期契约。delegate 是这个。调用时它仍然退化成第 1 种。
3. 一个函数指针加一份捕获的环境。block 回调是这个，`addObserverForName:...usingBlock:` 也是这个。

前两种最终都落到 `objc_msgSend(receiver, sel, ...)`，走的是 [[Runtime/Part 2 - 消息发送与转发]] 里那条完整的查找路径。第三种绕开了消息发送，直接调 `Block_layout.invoke`，细节在 [[iOS Block 的结构：ABI、descriptor 与三种类型]]。

所以"哪种更快"这个问题是有确定答案的，我在第六节测了。但速度基本不该进入选型判据，那一节真正有用的是另外两个数字。

---

## 二、delegate：一份编译期契约，加一次普通的消息发送

### protocol 在运行时是什么

`@protocol` 编译出来是一个真对象。

```text
  @protocol(Feed)  = 0x1009345c8
  object_getClass  = Protocol
  isa 指向的类的父类 = NSObject
  objc_getProtocol("Feed") == @protocol(Feed) ? 是
```

它的类就叫 `Protocol`，父类是 `NSObject`。objc4 里对应的结构体是 `protocol_t`，继承自 `objc_object`，所以它有 isa，能收消息。关键是它的方法列表分了四份：

```c
struct protocol_t : objc_object {
    const char *mangledName;
    struct protocol_list_t *protocols;
    method_list_t *instanceMethods;
    method_list_t *classMethods;
    method_list_t *optionalInstanceMethods;
    method_list_t *optionalClassMethods;
    property_list_t *instanceProperties;
    ...
};
```

`@required` 和 `@optional` 是两张互不相通的表。实测：

```text
  feedDidLoad:              required表=命中  optional表=无
  feed:didFailWithError:    required表=无    optional表=命中
  feedShouldRetry:          required表=无    optional表=命中

  required 实例方法 1 个：feedDidLoad:
  optional 实例方法 2 个：feed:didFailWithError:(v32@0:8@16@24) feedShouldRetry:(B24@0:8@16)
```

注意第二行里的类型编码。protocol 里存的只有方法描述（selector 加类型编码），一个 IMP 都没有。`@optional` 的全部实现就是"放进另一张表"，没有任何运行时机制去检查它有没有被实现。

### conformsToProtocol: 和 respondsToSelector: 查的是两件事

这两个方法经常被混着用，但它们的定义域是正交的。我写了两个类：`Lazy` 声明遵守协议、只实现 required；`Duck` 什么都不声明、三个方法全实现。

```text
  Lazy 声明遵守，只实现了 required：
    conformsToProtocol:            1
    respondsToSelector: required   1
    respondsToSelector: optional   0
  Duck 没声明遵守，三个方法全实现：
    conformsToProtocol:            0
    respondsToSelector: optional   1
```

`conformsToProtocol:` 查的是编译期写在 `@interface` 尖括号里的那份声明，跟有没有实现毫无关系。`respondsToSelector:` 查的是方法表，跟有没有声明遵守毫无关系。

结论落到工程上就一句：**调用 `@optional` 方法之前必须自己 `respondsToSelector:`**，因为编译器和运行时都不会替你兜底。这也是 delegate 模式里那一大坨 `if ([self.delegate respondsToSelector:...])` 的来源。

这句话其实还说轻了。`@required` 一样不保证方法存在：

```text
  Sloppy conformsToProtocol:@protocol(P) = 1
  Sloppy respondsToSelector:@selector(mustImplement) = 0
```

`Sloppy` 声明了遵守，`mustImplement` 标着 `@required`，它就是没写。编译器只给一条 `warning: method 'mustImplement' in protocol 'P' not implemented [-Wprotocol]`，是警告不是错误。一个开着 `-Wno-protocol` 或者警告淹在几千条里的工程，这东西能一路活到线上。框架边界上对关键回调补一句 `respondsToSelector:` 或 `NSAssert` 成本极低，我自己的分界是：`@optional` 一律查，`@required` 在对外的库代码里查、自家模块内部不查。

### 两个 conformsToProtocol 走的不是同一条路

还有一组我没料到的数据。同一个对象，方法版和 C 函数版给出相反的答案：

```text
  LiarSub 实例（继承来的声明，自己啥也没写）    = 1
  class_conformsToProtocol(LiarSub 类本身)      = 0  (不查继承链)
  class_conformsToProtocol(OnlyRequired 类)     = 1
```

`LiarSub` 继承自一个声明了协议的类，自己什么都没写。答案在 `NSObject.mm` 里，方法版自己爬了一遍继承链：

```objc
- (BOOL)conformsToProtocol:(Protocol *)protocol {
    if (!protocol) return NO;
    for (Class tcls = [self class]; tcls; tcls = tcls->getSuperclass()) {
        if (class_conformsToProtocol(tcls, protocol)) return YES;
    }
    return NO;
}
```

`class_conformsToProtocol` 只看传进去的那一个类，不往上走。父协议那一层则由 `protocol_conformsToProtocol_nolock` 递归处理，而且它比的是名字不是指针：

```cpp
    if (0 == strcmp(self->mangledName, other->mangledName)) {
        return YES;
    }
```

用 `strcmp` 是因为同一个协议可能在多个镜像里各有一份副本，指针不等但语义相同。写运行时反射工具时这个区别会咬人：想问"这个类自己声明了吗"用 C 函数，想问"这个对象能不能当 delegate 用"用方法版。

### 一个我没料到的：respondsToSelector: 会触发动态方法解析

我原本以为 `respondsToSelector:` 就是查表返回。写了个类在 `+resolveInstanceMethod:` 里动态添加 optional 方法，结果是：

```text
  调 respondsToSelector:@selector(feedShouldRetry:)
     +resolveInstanceMethod: 被触发，sel = feedShouldRetry:
  结果 = 1
     动态解析出来的 IMP 执行了
```

`class_respondsToSelector` 走的是带 `LOOKUP_RESOLVER` 的查找路径，查不到会先给类一次动态添加方法的机会。这意味着一个实现了动态方法解析的 delegate，可以在被问到 optional 方法时当场"长出"这个方法来。

这条我没在任何 delegate 教程里见过。它有个现实后果：如果你的 delegate 类（或它的父类）里写了 `+resolveInstanceMethod:`，那么每次 `respondsToSelector:` 未命中都会调进去一次，而 delegate 回调往往在滚动、绘制这类高频路径上。

### delegate 为什么用 weak

流行答案是"防止循环引用"。我在 [[iOS 属性关键字：从所有权推导，而不是从类型名猜]] 里说过这个说法把因果讲反了，这里再补一次证据。

真正的理由是所有权方向：`UITableView` 不该拥有它的 delegate，因为那通常是一个更上层、活得更久的对象。这是一条关于"谁是谁的一部分"的判断，跟有没有环无关。三种写法各跑一遍：

```text
========== 1. delegate 是 weak：delegate 死后再回调 ==========
  delegate 活着时：
     weakDelegate   = 0x600000004040
     [VC loaderDidFinish:] name=活着
  VC dealloc
  delegate 已析构：
     weakDelegate   = 0x0
     调 [weakDelegate loaderDidFinish:] ... 返回了

========== 2. 同一个场景换成 unsafe_unretained ==========
  VC dealloc
  delegate 已析构，现在用野指针发消息：
     unsafeDelegate = 0x600000004040（野指针）
     调 [unsafeDelegate loaderDidFinish:] ...
Child process terminated with signal 11: Segmentation fault
```

开着 `NSZombieEnabled` 再跑一次，第二段变成经典的那行：

```text
*** -[VC loaderDidFinish:]: message sent to deallocated instance 0x600000004020
```

第三种写法（strong）不崩也不泄漏，只是把 VC 的寿命绑到了 Loader 身上。没成环就不是泄漏，只是所有权设计错了。

`[nil someMethod]` 安全返回，是整个 delegate 模式能这么写的前提。换成 Swift 的 `delegate?.method()` 是同一件事的语法版本。

### delegate 能做的、别人做不到的两件事

能拿返回值。 `tableView:heightForRowAtIndexPath:` 要一个 CGFloat 回来，这个只有 delegate（和 target-action 的同步调用）能做。通知的 post 没有返回值，block 回调理论上可以但工程上很少这么用。

能拦截。 `shouldChangeTextInRange:` 返回 NO 就阻止了那次编辑。这类"先问后做"的语义要求调用方同步等一个答案，一对多的通知在语义上就不成立：五个观察者返回五个答案，听谁的。

一对一、可回传、可拦截，这三条是 delegate 的全部差异化能力。

---

## 三、target-action：字符串驱动，编译期什么都不检查

### 从一次点击到 objc_msgSend

`UIControl` 收到触摸后调 `sendAction:to:forEvent:`，它把 action 转交给 `UIApplication` 的 `sendAction:to:from:forEvent:`。这中间那次转交才是关键：`to` 传 nil 时，消息会沿响应者链找第一个能处理它的对象。

这条链可以直接测。三层视图嵌套，只有最外层实现了 `save:`：

```text
  从 leaf 出发找 save: 的 target：
     canPerformAction:save: 问到了 Root
  → Root (0x105904640)
  从 leaf 出发找一个没人实现的 selector：
  → nil

  nextResponder 链：Leaf → Middle → Root → nil
```

`targetForAction:withSender:` 的默认实现就是"我能处理吗？能就返回 self，不能就问 nextResponder"。这就是 nil-targeted action，也是 iOS 上"剪切/拷贝/粘贴"这类菜单命令的分发方式。响应者链本身留到第五周那篇讲。

命令行进程里 `[UIApplication sharedApplication]` 是 nil，所以我没法在这里跑通 UIControl 的完整发送路径。上面测的是响应链查找这一段，它不依赖 UIApplication。

### action 方法的三种签名

target-action 允许三种写法：无参、`(id)sender`、`(id)sender forEvent:(UIEvent *)`。运行时靠参数个数分辨：

```text
  zero                 numberOfArguments=2  types=v16@0:8
  one:                 numberOfArguments=3  types=v24@0:8@16
  two:forEvent:        numberOfArguments=4  types=v32@0:8@16@24
```

`method_getNumberOfArguments` 把隐含的 `self` 和 `_cmd` 也算进去，所以是 2/3/4。UIKit 拿到这个数字就知道该按哪种原型去调。

### selector 是唯一化的字符串

```text
  @selector=0x1f5e0688d  sel_registerName=0x1f5e0688d  NSSelectorFromString=0x1f5e0688d  全等？是
  made == @selector(save:) ? 是
     [Root save:] 被调用
```

三条路拿到的是同一个指针，运行时里每个方法名只有一份。所以从字符串拼出来的 selector 照样能发消息。这是 target-action 灵活性的来源，也是它全部危险的来源：编译器不检查 target 有没有这个方法，也不检查参数个数对不对。

后者的表现比想象中隐蔽：

```text
  把只接受 (id) 的方法当成 (id, UIEvent*) 发：
     一参版被调用，sender=Recv
```

多传的参数落在寄存器里没人读，什么都不会发生。反过来少传，被调方读到的就是上一次调用留在寄存器里的垃圾。这类 bug 没有编译期信号，只有运行时的诡异值。

Swift 里 `#selector(foo)` 至少会检查方法存在且标了 `@objc`，比字符串好一些，但参数个数仍然不检查。

### 陈旧的 target-action 记录不会被回收

回到开头那个 weak 结论，还有半句没说完。target 死了以后，注册记录本身还在：

```text
  三个 target 都死了，_targetActions.count = 3
    UIControlTargetAction  _target = 0x0
    UIControlTargetAction  _target = 0x0
    UIControlTargetAction  _target = 0x0
  allTargets = {( <null> )}
  actionsForTarget:nil = ( "go:", "go:", "go:" )
```

`allTargets` 返回一个装着 `NSNull` 的集合，这是 Apple 文档里写明的行为（"The set may include NSNull to indicate at least one nil target"）。但 `_targetActions` 数组本身没有缩短。一个长期存活的 control 反复换 target 而不 `removeTarget:`，条目会一直堆。量级很小，但这是个真实的积累。

这个"弱引用置了 nil，注册记录留着"的模式，等下在通知那节还会原样出现一次。

### iOS 14 的 addAction: 把所有权翻了过来

`_actionHandler` 那一行的 strong 不是笔误。同一个 `UIControl`，两种注册方式，所有权方向相反：

```text
---------- 3. UIAction（iOS 14+）的所有权对比 ----------
  UIAction 版：出作用域后 weak 探针 = 0x600000014060（非 nil 说明被 UIControl 间接强持有）
```

`addTarget:action:` 不持有 target；`addAction:` 持有 UIAction，UIAction 持有 handler block，block 强捕获里面用到的一切。写成这样就是一个环：

```objc
[self.button addAction:[UIAction actionWithHandler:^(UIAction *a) {
    [self reload];        // self 被 block 捕获 → UIAction → button → self
}] forControlEvents:UIControlEventTouchUpInside];
```

view controller 持有 button，button 持有 action，action 的 block 持有 view controller。经典三角。老写法里 `self` 传给 `addTarget:` 是安全的，同一个人换成新 API 会下意识觉得也安全。我的判断是：这是从 target-action 迁移到 UIAction 时最容易埋的雷，规则和 [[iOS Block 循环引用与 weak-strong dance]] 里那套完全一样，该 weakSelf 就得 weakSelf。

`UIBarButtonItem` 的 `_target` 我一并测了，也是 weak；它的 `_primaryAction`（UIAction）是 strong。同一个规律。

---

## 四、通知：唯一的一对多

### NSNotificationCenter 里没有注册表

先看它自己有什么：

```text
  NSNotificationCenter 的 ivar（共 1 个）：
    +8    _impl            ^{__CFNotificationCenter=}
  [nc description] = <CFNotificationCenter 0x60000020c000 [0x1e3b3b680]>
```

一个 ivar，指向一个 CoreFoundation 结构体。再确认一下这个指针是什么：

```text
  defaultCenter->_impl               = 0x600000205320
  CFNotificationCenterGetLocalCenter = 0x600000205320
  两者相同？是
```

`NSNotificationCenter` 是 `CFNotificationCenter` 本地中心的一层壳，增删查发全部转手给 CoreFoundation。所以在 Foundation 里怎么翻都找不到那张观察者表，它根本不在 Foundation 里。

顺便说一句：CoreFoundation 虽然开源，但 `CFNotificationCenter.c` 从来没在 Apple 的开源发布里出现过。网上"从 CoreFoundation 源码看 NSNotificationCenter 实现"的文章，讲的多半是 GNUstep 的 `NSNotificationCenter.m`。GNUstep 用的是"wildcard 裸链表 + nameless 表 + named 两级表"三张表，派发时先 wildcard 后 named。这套结构在 iOS 上不成立，下面的实验会当场证伪它。

### 同步、同线程、按注册顺序

三个最基本的性质，一起测掉。

派发是同步的：

```text
  [1] post 之前
     ↳ A 收到 LabNote，线程=主线程(0x600001704080)
     ↳ B 收到 LabNote，线程=主线程(0x600001704080)
  [2] post 之后
```

回调跑在 post 的那个线程上，跟观察者在哪注册的没关系：

```text
  注册发生在：主线程(0x600001704080)
  post 发生在：非主线程(0x6000017080c0)
     ↳ A 收到 LabNote，线程=非主线程(0x6000017080c0)
     ↳ B 收到 LabNote，线程=非主线程(0x6000017080c0)
```

顺序按注册先后，而且通配观察者和具名观察者是混在一起排的：

```text
  注册顺序 1..5，实际调用顺序：
    1-通配(name=nil,obj=nil)
    2-具名(obj=nil)
    3-只按对象
    4-具名+对象
    5-通配
```

这一条直接推翻了"先发 wildcard 再发 named"的说法。不过要说清楚：Apple 的 Foundation Release Notes 早在 10.3 那次重构时就写过 "you should not have been depending on one observer receiving a notification before another anyway"。实现行为是注册序，契约是无序。依赖它就是依赖一个 Apple 随时能改的实现细节。

重入没有任何保护：

```text
     进入第 1 层  进入第 2 层  进入第 3 层
     离开第 3 层  离开第 2 层  离开第 1 层
```

递归 post 就是普通的递归调用，深了就爆栈。崩溃栈里会看到 post 相关的帧反复重复，这个形状本身就是识别通知重入的特征。

同步派发还有一个后果，我觉得比上面几条都更该知道：一个观察者抛异常，排在它后面的观察者收不到通知。

```text
    观察者 A 执行
    观察者 B 抛异常
    异常传到了 post 的调用方：LabException
  （如果没看到「观察者 C 执行」，说明异常打断了整条链）
```

C 没有执行，异常一路传回 post 的调用方。三个观察者可能分属三个互不相识的模块，其中一个的 bug 会让另外两个静默失效，而崩溃栈指向的是发通知的那一方。通知常被当成解耦手段，这条说明耦合只是被藏起来了，并没有消失。

### post 期间增删观察者：快照，加一次逐项校验

这两个行为容易被合并成一句"会先拷一份数组"，但那个模型解释不了全部现象。

在回调里新增观察者，本轮不生效：

```text
     第一个观察者执行，此时注册第二个
  后注册的观察者本轮被调用？否
```

在回调里移除一个还没轮到的观察者，它会被跳过：

```text
  A 在回调里移除了 C。实际执行：A,B
```

纯快照模型会预测 `A,B,C`。实际少了 C，说明派发循环在调用每一项之前还会再验一次这条注册是否仍然有效。**"先取快照排好序，再逐项校验"才是解释所有这类问题的模型。**

### queue: 参数是四种方式里最容易踩的一处

`addObserverForName:object:queue:usingBlock:` 的 queue 传 nil 时，官方文档写得很清楚："When nil, the block runs synchronously on the posting thread."

传 `mainQueue` 呢？文档没说。我测的结果是它取决于 post 发生在哪。

主线程 post：

```text
  [1] 主线程即将 post
  [2] queue=nil 的 block
  [3] queue=mainQueue 的 block
  [4] post 已返回
  [5] 跑了一轮 runloop 之后
```

两个 block 都在 `[4]` 之前跑完了，`mainQueue` 那个是同步内联执行的。

后台线程 post 时，`queue=nil` 的 block 留在后台线程，`queue=mainQueue` 的跑到了主线程上：

```text
     queue=nil   的 block 在：非主线程(0x6000017080c0)
     queue=main  的 block 在：主线程(0x600001704080)
```

到这里很容易就写下"非 nil 的 queue 就是异步派发"。我一开始就是这么记的，直到量了一次耗时。

block 里 sleep 300 毫秒，post 从后台线程发出，测 post 这一句本身花了多久：

```text
=== queue 非 nil 时 post 阻不阻塞（重复 3 次看稳定性）===
  第 1 次：post 调用耗时 311 ms（block 里 sleep 300ms）
  第 2 次：post 调用耗时 305 ms（block 里 sleep 300ms）
  第 3 次：post 调用耗时 305 ms（block 里 sleep 300ms）
```

block 换到别的队列上执行了，post 的调用方却被原地按住，一直等到 block 跑完才返回。异步的只是执行位置，不是调用时序。

这个组合会死锁。观察者注册在 `mainQueue`，主线程去等后台线程的 post 完成：

```text
=== 死锁复现：观察者注册在 mainQueue，主线程阻塞等 post 完成 ===
  主线程开始 wait（最多 3 秒）
  后台线程 post 前
  wait 返回 49  （非 0 = 超时 = 死锁成立）

=== 对照：同样的代码，把 queue 换成 nil ===
    block 执行了，thread=background
  wait 返回 0  （0 = 正常完成）
```

block 在主队列上排队等主线程，主线程在等 post 返回，post 在等 block。只改 `queue:` 一个参数，一边正常一边挂死。这是我写这篇时第一个跑挂的程序，当时两分钟超时被杀，我还以为是模拟器的问题。

post 恰好就发生在注册的那个 queue 上时不死锁，实测耗时 0.0 ms，说明这条路径上有"已经在目标队列"的短路。

所以 `queue:` 回答的是"回调在哪执行"，它完全不回答"post 会不会卡住调用方"。想要真正的异步，得在 block 里自己 `dispatch_async`，或者用下面那个 `NSNotificationQueue`。我自己的做法是：只要 block 里要碰 UI，就不信任 queue 参数，直接在 block 里显式 `dispatch_async(dispatch_get_main_queue(), ...)`，或者干脆用 iOS 26 的 `MainActorMessage`（第八节）。

### iOS 9 那条"不用移除"的准确边界

原文出自 Foundation Release Notes for OS X v10.11 and iOS 9：

> In OS X 10.11 and iOS 9.0 NSNotificationCenter and NSDistributedNotificationCenter will no longer send notifications to registered observers that may be deallocated. If the observer is able to be stored as a zeroing-weak reference the underlying storage will store the observer as a zeroing weak reference... This means that observers are not required to un-register in their deallocation method. The next notification that would be routed to that observer will detect the zeroed reference and automatically un-register the observer... Block based observers via the -[NSNotificationCenter addObserverForName:object:queue:usingBlock] method still need to be un-registered when no longer in use since the system still holds a strong reference to these observers. CFNotificationCenterAddObserver does not conform to this behavior since the observer may not be an object.

这段话里有四个限定条件，通常只有第一个被引用：

1. 只对 selector 版观察者成立；
2. block 版的 token 被系统强持有，必须移除；
3. 对象若不支持弱引用（自定义 retain/release、`allowsWeakReference` 返回 NO），退化成非 weak 的置零引用，dealloc 期间收到通知的旧行为保留；
4. `CFNotificationCenterAddObserver` 完全不适用。

selector 版的部分我验了。关键是要排除"只是碰巧没崩"的可能，所以开着 zombie 跑：

```text
NSZombieEnabled = YES
     Ghost 0x60000000c050 dealloc
观察者已析构（没有 removeObserver:），post 之前
post 之后：没有 'message sent to deallocated instance'，也没有回调。
```

如果通知中心存的是 `unsafe_unretained`，僵尸对象会在这里拦下一条消息并打印。什么都没有，说明存的是会置 nil 的弱引用。另一条旁证：注册之后丢掉所有外部强引用，对象立刻 dealloc，说明中心也没有 retain 它。

block 版是另一回事：

```text
  token 类型 = __NSObserver
  token 与 Payload 都已出作用域。weak 探针 = 0x600000004030
  再 post 一次 Leaky：
     block 观察者仍在执行，捕获的 Payload = 0x600000004030
```

token 丢了，block 照跑，捕获的对象还活着。`__NSObserver` 里存着 `nc`、`_token`、`_block` 三个字段，中心攥着它不放，你不 `removeObserver:` 就没人放。

这里和上一篇有个可以对账的地方：KVO 的观察者变成置零引用是 iOS 11（macOS 10.13）的事，通知中心是 iOS 9。两次改动的性质一样，时间差了两年。

### 那条 dealloc 竞态，实际上被 weak 的语义挡住了

一个直觉上的隐患：`clearDeallocating` 在整条 `-dealloc` 跑完之后才清弱引用槽位，那么在 dealloc 执行期间，另一个线程 post 一条通知，会不会打进一个半析构的对象？

```text
  [dealloc] 开始，睡 300ms
  [另一线程] dealloc 进行中读 __weak → 0x0
  [另一线程] 此刻 post 通知：
  [另一线程] post 返回，没崩
  [dealloc] 结束
```

读不到。`objc_loadWeakRetained` 走 `rootTryRetain()`，对象一旦进入 deallocating 状态就返回 nil，用不着等槽位清零。这条和 [[iOS weak 的实现：SideTable 与置 nil 的时机]] 里的结论是同一个机制。

所以对可弱引用的观察者，这条竞态不成立。Release Notes 那句 "If an object can be weakly referenced notifications will no longer be sent to the observer during deallocation" 说的就是这个。但它紧接着的下半句同样是官方原文：不可弱引用的观察者保留旧行为。那才是真正还没消失的雷区。

### 不移除 selector 观察者的实际代价

Release Notes 说"下一次本该发给它的时候会一并注销"。这句话的言外之意是：如果那个通知名再也没被 post 过，记录就一直在。我拿 `malloc_zone_statistics` 量了一下，三次跑完全一致：

```text
基线 malloc in-use              =    0.059 MB
同名 100000 条死记录（未 post）  =    0.459 MB  (+0.401 MB, 每条 4.2 字节)
  post 一次之后                 =    0.459 MB  (+0.401 MB)
每条一个不同 name（100000 条）   =    1.048 MB  (+0.588 MB, 每条 6.2 字节)
对照：同样 100000 条但 add+remove=    1.048 MB  (+0.000 MB, 每条 0.0 字节)
```

十万条从不移除的注册记录，残留 0.4 MB，每条约 4 字节。配平的 add/remove 是精确的 0。post 一次之后残留没有回落。

4 字节远小于一条完整注册记录该有的大小，所以大部分内容确实被回收了，剩下的更像是哈希表扩容之后不再收缩的那部分（这一句是我的推测，不是实测结论）。

结论我倾向于说得温和一点：iOS 9 的改动解决的是崩溃，泄漏只是被压到了很小的量级，没有归零。`removeObserver:` 从"正确性要求"降级成了"卫生习惯"，但它没有变成多余代码。

这里得提一个我自己犯的错。我最早用 `phys_footprint` 量同一件事，两次跑都得到 +1.5 MB 左右，差点写进正文。再跑三次全变成 +0.05 MB。`phys_footprint` 是页粒度的进程级汇总，受同进程里其他分配的干扰，根本不适合量这种几百 KB 的差异。换成 `malloc_zone_statistics` 之后三次结果连小数点后三位都一样。**两次采样支持的结论不算结论。**

### 真正异步的那个是 NSNotificationQueue

```text
  连续 enqueue 5 次（NSPostWhenIdle + 按名字合并）：
  enqueue 全部返回，此刻 fired=0（异步，还没发）
  合并后的通知到达
  跑完 runloop，fired=1
```

五条合并成一条，而且要等 runloop 空闲才发。`NSNotificationQueue` 本身不派发，它是 NotificationCenter 前面的一个缓冲区，最后仍然走同步那条路。它绑定 runloop 和线程，所以在没有 runloop 的线程上 `NSPostWhenIdle` 永远不会被 flush，这是它最常见的坑。新代码基本没有理由用它。

---

## 五、block 回调：唯一默认强持有的那个

block 的运行时结构在 [[iOS Block 的结构：ABI、descriptor 与三种类型]]，捕获规则在 [[iOS Block 的变量捕获与 __block]]，环怎么成的在 [[iOS Block 循环引用与 weak-strong dance]]。这里只补它作为通信手段的那一面。

四种方式里，前三种的默认所有权都是"不持有回调方"：delegate 用 weak、UIControl 的 target 是 weak、通知中心对 selector 观察者是弱引用。block 反过来，默认强持有它捕获的一切，而且这个持有是隐式的，语法上看不见。

这带来两个后果。一是环，那是上一篇的内容。二是一个更隐蔽的问题：block 会把回调方的寿命延长到 block 自己被释放为止。一个网络请求的 completion 持有 view controller，用户已经退出界面了，controller 还得等请求回来才能死。这不是泄漏，Instruments 的 Leaks 也查不出来，但它是真实的行为差异。

block 回调适合的形状很窄：一次性、有明确起止、调用方就是关心结果的人。`fetchData:completion:` 是标准形态。一旦变成"多次触发的持续回调"，block 的所有权就开始变得难管，因为你得决定它什么时候被释放。这时候 delegate 反而更清爽。

判空是另一个老问题。`if (completion) completion();` 解决的是空指针，不解决属性在判断和调用之间被另一个线程改掉。真要并发安全，得先取到局部变量再调。

block 拿不到"多个回调方各自的意见"，也不适合拦截语义。所以我自己的阈值是：需要返回值或需要拦截，用 delegate；一次性异步结果，用 block；其余情况先想清楚是不是真的需要一对多。

---

## 六、四种方式的实际开销

三次运行，每种测 200000 次取平均、5 轮取最小值：

```text
  直接消息发送                       2.3 ns   （1.0×）
  delegate（unsafe 存取）             2.3 ns   （1.0×）
  delegate（weak 读 + responds）     18.1 ns   （8.1×）
  block 调用                          6.1 ns   （2.7×）
  通知 postNotificationName:        244.7 ns  （109.9×）
  通知 postNotification:（对象已建好） 193.5 ns  （86.9×）
```

三次里通知那两行在 190~310 ns 之间跳，其余各行稳定在小数点后一位。所以只有量级可信：通知比一次直接消息发送贵约两个数量级。

把 delegate 那 18 ns 拆开：

```text
  [u ping]  纯消息发送               2.32 ns
  id d = w;  只读一次 weak          14.01 ns
  [u respondsToSelector:]  只判断     4.03 ns
  读 weak + responds + 调用          25.60 ns
```

大头是读 weak，不是 `respondsToSelector:`。`objc_loadWeakRetained` 要查 SideTable、加锁、tryRetain、再 autorelease，比一次消息发送贵五六倍。这解释了为什么 UIKit 内部大量使用"把 delegate 读到局部变量再连着用"的写法。

这些数字来自模拟器，绝对值不能拿去做真机性能判断。它们能说明的只有相对量级和成本来源。

还有一个数字更有用：

```text
  0 个注册记录：每次 post 155.7 ns
  10 万条死记录：每次 post 157.2 ns
  倍数 = 1.0×
```

十万条陈旧注册记录对派发速度没有影响。所以"不移除观察者会拖慢通知"这个担心是不必要的，代价只在那 0.4 MB 内存上。

真正该记住的是：通知的开销集中在"每次 post 都要造一个 `NSNotification` 对象、查四个桶、排一次序"，不在派发本身。高频路径（每帧、每个 cell、每次滚动）不要用通知。这不是性能洁癖，200 ns 乘以每秒几千次就是可见的开销。

---

## 七、怎么选

我不喜欢"通知用于一对多、delegate 用于一对一"这种记法，它把结论当定义背。实际推导只要问三个问题。

**第一，通信是几对几？** 一个明确的对方，还是任意多个不认识的听众。这条最先排除掉一半选项。

**第二，调用方需要回应吗？** 需要返回值或需要拦截，只剩 delegate。这条是硬约束，不是偏好。

**第三，两边的生命周期谁长？** 回调方比发起方活得久（controller 之于 view），发起方就不该持有它，用 delegate / target-action / 通知。回调方的存在只是为了这一次调用（completion handler），用 block。

三个问题答完，基本没有歧义了。

| | delegate | target-action | 通知 | block 回调 |
| --- | --- | --- | --- | --- |
| 关系 | 一对一 | 一对一（可多注册） | 一对多 | 一对一 |
| 编译期检查 | 有（protocol） | 无（selector 字符串） | 无（通知名字符串） | 有（block 签名） |
| 能否拿返回值 | 能 | 能 | 不能 | 能但少用 |
| 能否拦截 | 能 | 不能 | 不能 | 不能 |
| 默认持有回调方 | 否（weak） | 否（weak） | 否（selector 版） | 是 |
| 忘了清理 | 无事（weak 置 nil） | 无事，条目残留 | selector 版无事、条目残留；block 版永久调用 | 环 / 寿命延长 |
| 调用方是否知道对方 | 知道类型 | 知道对象 | 完全不知道 | 知道 |
| 调用栈可追 | 是 | 是 | 是（同步路径） | 是 |

最后一行值得展开一句。通知的同步派发有个常被忽略的好处：在观察者里下断点，调用栈里直接能看到是谁 post 的。这是它相对 KVO 的调试优势。但 `queue:` 非 nil 的异步路径会把这个栈丢掉，新的 `AsyncMessage` 也一样。

我自己的默认值：能用 delegate 就用 delegate；跨模块、发送方不该知道接收方存在的，才用通知；UI 控件的点击继续用 target-action，除非已经全面转 UIAction；异步结果用 block。通知我会额外加一条自律：一个通知名只在一处 post，并且把 post 点写进注释。一对多的代价就是没人知道谁在听，这个代价必须靠约定去付。

---

## 八、Swift 里的演进

三代 API，三种默认行为，而且都和"通知在哪个线程、丢不丢"有关。

**Combine（iOS 13）**：`publisher(for:object:)`。它内部就是 `addObserver(forName:object:queue: nil)`，所以没有任何队列跳转，要切线程得自己 `.receive(on:)`。它的 `object` 参数是 `public let object: AnyObject?`，强持有。

**AsyncSequence（iOS 15）**：`notifications(named:object:)`。当前 SDK 里的签名已经不是老文章里那个样子了：

```swift
@available(macOS 12, iOS 15, tvOS 15, watchOS 8, *)
@preconcurrency public func notifications(
    named name: Foundation.Notification.Name,
    object: (any Swift.AnyObject & Swift.Sendable)? = nil
) -> Foundation.NotificationCenter.Notifications
```

`Notification` 本身的 Sendable 遵守是 `@available(*, unavailable)` 的，所以 Swift 6 下跨隔离域传它会直接报错。Apple 给的解法是先 `map` 出可发送的字段再用。

**类型化通知（iOS 26）**。这是我认为最值得关注的一次改动。在本机 SDK 的 swiftinterface 里能直接读到：

```swift
@available(macOS 26, iOS 26, tvOS 26, watchOS 26, visionOS 26, *)
extension Foundation.NotificationCenter {
  public protocol MainActorMessage : Swift.SendableMetatype {
    associatedtype Subject
    static var name: Foundation.Notification.Name { get }
    @MainActor static func makeMessage(_ notification: Foundation.Notification) -> Self?
    @MainActor static func makeNotification(_ message: Self) -> Foundation.Notification
  }
  public protocol AsyncMessage : Swift.Sendable { ... }
}
```

它把三件事从"运行时约定"提到了"类型系统"：

- **隔离域**。`MainActorMessage` 保证同步投递到主线程，`AsyncMessage` 明确表示异步、任意线程。第四节那个"queue: .main 有时同步有时异步"的坑，在这套 API 里从设计上就不存在了。
- **载荷类型**。不用再从 `userInfo` 里按字符串 key 取值再强转。
- **注销**。新的 `ObservationToken` 是 `deinit` 时自动注销的。老 block token 丢了等于永久泄漏，新 token 丢了等于注销，语义正好相反。

SDK 里已经带了一批内建类型，比如 `UndoManager.WillUndoChangeMessage`（MainActorMessage）、`ProcessInfo.ThermalStateDidChangeMessage`（AsyncMessage）、`Date.SystemClockDidChangeMessage`。

有一个迁移风险要提前知道：观察端桥接旧通知时用的是 `MainActor.assumeIsolated`，这是运行时断言。老代码从后台线程 post 一条恰好有 `MainActorMessage` 遵守的通知，新式观察者会当场崩。提案 SF-0011 自己承认了这一点，给的处理办法是去改那个 post 的调用方。

> 待真机补测：`MainActorMessage` 的同步投递保证、以及后台 post 触发 `assumeIsolated` 崩溃这两条，我这台机器上没有 iOS 26 真机。模拟器上写一个最小 Swift 工程即可复现，两个观察者一个新式一个旧式，分别从主线程和后台线程 post 同一个名字。

---

## 九、几个不准的说法

- **"UIControl 的 target 是 unsafe_unretained。"** 在我测的两套 UIKit（iOS 18.3 和 26.5 运行时）上是 weak，两条独立证据一致。这个说法在 ARC 之前的 UIKit 上大概是对的，今天不对了。
- **"delegate 用 weak 是为了防循环引用。"** 因果反了。理由是所有权方向，循环引用只是用错 strong 之后的一个后果。
- **"NSNotificationCenter 内部有三张表，先发 wildcard 再发 named。"** 那是 GNUstep。iOS 上通配和具名观察者按注册顺序混排，实验在第四节。
- **"通知是异步的。"** `postNotification:` 是同步的，返回前所有 selector 观察者都已执行完。真正异步的只有 `NSNotificationQueue`。
- **"`queue:` 参数非 nil 就是异步派发。"** 只有执行位置是异步的。实测 post 会一直等到 block 执行完毕才返回：block 里 sleep 300ms，post 这一句稳定花掉 305 ms。传 `mainQueue` 而主线程正好在等 post，就是死锁。
- **"一个观察者出错不影响别人。"** 同步派发是一条链，中间抛异常会让后面的观察者收不到，异常还会传给 post 的调用方。
- **"通知在主线程执行。"** 在 post 的那个线程执行。
- **"iOS 9 之后不用移除观察者。"** 只对 selector 版成立。block 版的 token 被强持有，不移除会一直被调用。
- **"post 期间移除观察者不会生效，因为已经取了快照。"** 会生效。快照之外还有一次逐项校验，实测被移除的那个不会被调用。
- **"不移除观察者会让通知中心越来越慢。"** 十万条死记录对 post 耗时没有可测影响；代价是约 0.4 MB 内存。
- **"respondsToSelector: 只是查表。"** 它会触发 `+resolveInstanceMethod:`。
- **"`conformsToProtocol:` 返回 YES 就说明方法都实现了。"** 它只查编译期的遵守声明。反过来，全部实现了但没声明遵守的类会返回 NO。

---

## 总结

四种方式在运行时只有三种表示：SEL 加接收者、对象加编译期契约、函数指针加捕获环境。前两种最终都是 `objc_msgSend`。

持有关系是选型的主线。delegate 的 weak、UIControl target 的 weak、通知中心对 selector 观察者的弱引用，三者是同一个设计判断：被通知的一方通常活得更久，不该被通知的发起方拥有。block 是唯一反过来的，它默认强持有捕获的一切，`addAction:` 把 target-action 也拖进了这个语义。

"忘了清理"的后果各不相同，这一条比 API 差异更值得记：delegate 什么都不会发生；target-action 和 selector 版通知会留下无害但不回收的条目；block 版通知观察者会被永久调用；block 回调则是成环或延长寿命。

通知那一节最该带走的不是"同步还是异步"这个二分，而是同步派发牵出的两件事：一个观察者抛异常会掐断整条链，以及 `queue:` 参数只搬走了执行位置、没搬走等待。后者能直接把主线程锁死。

选型问三个问题就够了：几对几、要不要回应、谁活得更久。不用背表。

方法论上，这一篇最有价值的两次经历都是关于仪器的。第一次是 ivar 布局解码器，我一开始把 layout 的起点当成对象首地址，于是"测出"`UIControl._targetActions` 是 unsafe_unretained，一个存着 NSMutableArray 的 ivar 显然不可能是这个答案。加一个自己写死答案的对照类校准之后，起点应该是父类的 `instanceSize`。第二次是内存测量，`phys_footprint` 给出的 +1.5 MB 完全不可复现，换成 `malloc_zone_statistics` 之后三次跑连小数点后三位都一样。**结果和预期不符的时候，先怀疑仪器。**

下一篇 [[iOS GCD：队列不是线程，以及死锁的准确边界]]。

## 参考资料

### 官方

- [Foundation Release Notes for OS X v10.11 and iOS 9](https://developer.apple.com/library/archive/releasenotes/Foundation/RN-FoundationOlderNotes/index.html)：NotificationCenter 一节，第四节引的四个限定条件的原文出处
- [addObserver(forName:object:queue:using:)](https://developer.apple.com/documentation/foundation/notificationcenter/addobserver(forname:object:queue:using:))："When nil, the block runs synchronously on the posting thread"，以及 token 与 block 被强持有的说明
- [removeObserver(_:)](https://developer.apple.com/documentation/foundation/notificationcenter/removeobserver(_:)-2yciv)：现行版本已经把豁免限定到 selector 版，网上流传的引文多半是 2018 年之前那版
- [UIControl.addTarget(_:action:for:)](https://developer.apple.com/documentation/uikit/uicontrol/addtarget(_:action:for:))："does not retain the object in the target parameter"
- [UIApplication.sendAction(_:to:from:for:)](https://developer.apple.com/documentation/uikit/uiapplication/sendaction(_:to:from:for:)) 与 [UIResponder.target(forAction:withSender:)](https://developer.apple.com/documentation/uikit/uiresponder/target(foraction:withsender:))：nil-target 与响应者链
- [NotificationQueue](https://developer.apple.com/documentation/foundation/notificationqueue)：三种 posting style 与 coalescing 的定义
- [SF-0011 Concurrency-Safe Notifications](https://github.com/swiftlang/swift-foundation/blob/main/Proposals/0011-concurrency-safe-notifications.md)：iOS 26 类型化通知的提案。注意它走的是 swift-foundation 的 `Proposals/` 目录，不在 swift-evolution 里
- [WWDC25 — What's new in Swift](https://developer.apple.com/videos/play/wwdc2025/245/)：`MainActorMessage` 同步、`AsyncMessage` 异步这句话的官方措辞
- [apple-oss-distributions/objc4](https://github.com/apple-oss-distributions/objc4)：`protocol_t` 在 `objc-runtime-new.h`；`class_respondsToSelector` 的解析器路径在 `objc-runtime-new.mm`

### 经典

- [objc.io — Communication Patterns](https://www.objc.io/issues/7-foundation/communication-patterns/)：最早把四种方式按"发送者是否需要返回值、观察者数量、耦合度"来对比的一篇，结构至今可用
- [Cocoa Design Patterns](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CocoaFundamentals/CocoaDesignPatterns/CocoaDesignPatterns.html)：Delegation、Notifications、Target-Action 的官方定义
- [Ole Begemann — Do you still need to unregister notification observers?](https://oleb.net/blog/2018/01/notificationcenter-removeobserver/)：block 版必须移除的实测，也是当年抓出 Apple 文档自相矛盾的那篇
- [GNUstep NSNotificationCenter.m](https://github.com/gnustep/libs-base/blob/master/Source/NSNotificationCenter.m)：一份可读的完整实现。拿它理解概念可以，别把它当 Apple 的实现

### 本地

- [[Runtime/Part 2 - 消息发送与转发]]
- [[iOS 属性关键字：从所有权推导，而不是从类型名猜]]
- [[iOS Block 循环引用与 weak-strong dance]]
- [[iOS Block 的结构：ABI、descriptor 与三种类型]]
- [[iOS weak 的实现：SideTable 与置 nil 的时机]]
- [[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]]

---

实验环境：Xcode 26.6（Apple clang 21），iPhoneSimulator 26.5 SDK，`clang -fobjc-arc -target arm64-apple-ios17.0-simulator`。程序通过 `xcrun simctl spawn` 在 iOS 18.3（iPhone 16）和 iOS 26.5（iPhone 17）两个模拟器上分别运行，UIControl 那组实验两边结果一致。性能实验用 `-O1` 构建。

ivar 所有权的判定用的是 `class_getIvarLayout` / `class_getWeakIvarLayout` 的 nibble 解码，起点取父类的 `class_getInstanceSize`。这个方法对 MRC 编译的类无效：Foundation 里的 `NSConcreteNotification`、`__NSObserver` 等类 `ivarLayout` 全是 NULL，解码器会把每个对象型 ivar 都判成 unsafe_unretained，那是没有信息、不是结论。UIKit 是 ARC 编译的，所以对 `UIControlTargetAction` 有效。

> 待真机补测：UIControl target 的 weak 判定、以及第六节那组耗时。程序原样拿到设备上跑即可，`class_getWeakIvarLayout` 在真机同样可用。
