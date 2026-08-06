---
title: 【iOS】对象通信：delegate、通知、target-action 与 block 回调
published: 2026-07-27
description: UIKit 从来不数冒号来决定给 action 传几个参数，官方文档写的是"永远推两个"。另外，给 block 观察者传 queue 并不会让 post 异步返回，传 mainQueue 还能把主线程锁死。
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

Apple 的文档只有一句话："The control does not retain the object in the target parameter."。这句话排除了 strong，但没说剩下的是 weak 还是 `unsafe_unretained`。差别不小。前者在 target 死后置 nil，后者留一个野指针。我见过的中文资料两种说法都有，而且都说得很笃定。

这个问题不用开模拟器就能回答。UIKitCore 是 ARC 编译的。类的所有权信息就写在 `class_ro_t` 的 `ivarLayout` / `weakIvarLayout` 两个 nibble 串里，躺在磁盘上的二进制里。解析一遍：

```text
=== class_ro_t @ 0x2506700  (UIControlTargetAction) ===
  flags=0x194  instanceStart=8  instanceSize=41
  ivarLayout     = 0x1d3ecde  bytes: 01 00 ...
  weakIvarLayout = 0x1d3f29b  bytes: 11 00 ...

  ivar_list_t: entsize=32 count=5
    +8    _actionHandler   @"UIAction"  size=8
    +16   _target          @            size=8
    +24   _action          :            size=8
    +32   _eventMask       Q            size=8
    +40   _cancelled       B            size=1

  按 instanceStart=8 为起点解码 nibble 串：
    ivarLayout     命中偏移 = ['0x8']   → strong
    weakIvarLayout 命中偏移 = ['0x10']  → weak
```

`01` 是"跳过 0 个 word，接下来 1 个 word 是 strong"，落在 `+8` 的 `_actionHandler`。`11` 是"跳过 1 个 word，接下来 1 个 word 是 weak"，落在 `+16` 的 `_target`。**UIControl 存的是 zeroing weak。**

但这只是实现，不是契约。Apple 全篇没有写过 weak 或 zeroing 这两个词。对照 AppKit 就很刺眼，同一家公司在 `NSControl.h` 里把话说得明明白白：

```objc
@property (nullable, weak) id target; // Target is weak for zeroing-weak compatible objects
                                      // in apps linked on 10.10 or later. Otherwise the
                                      // behavior of this property is 'assign'.
```

AppKit 写了，UIKit 没写。所以正确的态度是：按"不 retain"写代码，别按"会置 nil"写代码。

这篇要讲的四种方式，罗列 API 的教程满大街都是。真正拉开差距的是上面这一层：每种方式的运行时基础是什么、持有关系往哪个方向走、忘了清理会发生什么。三个问题答完，选型是推出来的。不用背。

上一篇是 [[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]]，KVO 也是一种对象通信，和本篇四种放在一起看更清楚。

---

## 一、四种方式，运行时只有三种表示

先把名词拆开。所谓"把行为交给别人"，在 Objective-C 里只有三种做法：

1. 一个 SEL 加一个接收者。target-action 是这个，`addObserver:selector:name:object:` 也是这个。
2. 一个对象加一份编译期契约。delegate 是这个。调用时它仍然退化成第 1 种。
3. 一个函数指针加一份捕获的环境。block 回调是这个，`addObserverForName:...usingBlock:` 也是这个。

前两种最终都落到 `objc_msgSend(receiver, sel, ...)`，走 [[20 专题笔记/Runtime/Part 2 - 消息发送与转发]] 里那条完整的查找路径。第三种绕开消息发送，直接调 `Block_layout.invoke`。细节在 [[iOS Block 的结构：ABI、descriptor 与三种类型]]。

所以"哪种更快"有确定答案，我在第六节测了。但速度基本不该进入选型判据，那节真正有用的是另外两个数字。

---

## 二、delegate：一份编译期契约，加一次普通的消息发送

### protocol 在运行时是个对象

`@protocol` 编译出来是一个真对象。

```text
  @protocol(Feed)   = 0x104f909d8
  object_getClass   = Protocol
  它的类的父类      = NSObject
  objc_getProtocol("Feed") == @protocol(Feed) ? 是
```

它的类就叫 `Protocol`，父类是 `NSObject`。objc4 里对应的结构体是 `protocol_t`，继承自 `objc_object`。所以它有 isa，能收消息。

关键在于它的方法列表分了四份，`@required` / `@optional` 各自再分实例 / 类方法。`objc-runtime-new.mm` 里的取表函数就是直白的四选一：

```cpp
getProtocolMethodList(protocol_t *proto, bool required, bool instance)
{
    method_list_t **mlistp = nil;
    if (required) {
        if (instance) {
            mlistp = &proto->instanceMethods;
        } else {
            mlistp = &proto->classMethods;
        }
    } else {
        if (instance) {
            mlistp = &proto->optionalInstanceMethods;
        } else {
            mlistp = &proto->optionalClassMethods;
        }
    }
    ...
```

四个组合各调一次，四张表互不相通：

```text
  required=YES instance=YES → 1 条： feedDidLoad:(v24@0:8@16)
  required=YES instance=NO  → 1 条： feedIdentifier(@16@0:8)
  required=NO  instance=YES → 2 条： feed:didFailWithError:(v32@0:8@16@24) feedShouldRetry:(B24@0:8@16)
  required=NO  instance=NO  → 1 条： feedClassOptional(v16@0:8)
```

`Feed` 继承自 `Base`，而 `Base` 的两个方法一条都没出现在上面。这是 `protocol_copyMethodDescriptionList` 头文件里写明的：

> Methods in other protocols adopted by this protocol are not included.

同一个协议换 `protocol_getMethodDescription` 去问，父协议的方法就找得到了：

```text
  getMethodDescription(Feed, baseRequired)  → name=baseRequired types=v16@0:8
  getMethodDescription(Feed, baseOptional)  → name=baseOptional types=v16@0:8
```

它的 @note 正好相反："This function recursively searches any protocols that this protocol conforms to."。一个不递归一个递归，两个函数名字长得像，行为是两回事。

### 协议里的方法永远没有 IMP

这是 `@optional` 必须配 `respondsToSelector:` 的结构性原因，值得直接看证据。

clang 生成协议元数据时，方法条目的 IMP 位置写死是空指针（`CGObjCMac.cpp:7674`）：

```cpp
  if (forProtocol) {
    // Protocol methods have no implementation. So, this entry is always NULL.
    method.addNullPointer(ObjCTypes.Int8PtrProgramASTy);
  } else {
    llvm::Function *fn = GetMethodDefinition(MD);
    assert(fn && "no definition for method?");
```

编译器这么写的，运行时就该这么读得出来。照着 `protocol_t` 手写一份布局一致的 struct 强转过去，把方法表整个打出来：

```text
protocol_t 内存直读：mangledName=Wire size=96 flags=0x0
  instanceMethods          count=2 entsize=12 (small/相对偏移格式)
      req1     types=v16@0:8      impOffset=0    imp=0x0 ← NULL
      req2     types=v16@0:8      impOffset=0    imp=0x0 ← NULL
  classMethods             (NULL)
  optionalInstanceMethods  count=3 entsize=12 (small/相对偏移格式)
      opt1     types=v16@0:8      impOffset=0    imp=0x0 ← NULL
      opt2     types=v16@0:8      impOffset=0    imp=0x0 ← NULL
      opt3     types=v16@0:8      impOffset=0    imp=0x0 ← NULL
  optionalClassMethods     (NULL)

对照：真实类 Impl 的方法表里 imp 是有值的
      req1     imp=0x100e84908
      req2     imp=0x100e8491c
      opt1     imp=0x100e84930
```

**协议里存的只是方法描述，一个 IMP 都没有。** `@optional` 的全部实现就是"放进另一张表"，没有任何运行时机制去检查它有没有被实现。

这个实验我踩了一个坑。第一版直接按 `{SEL, char*, IMP}` 三个指针去读，当场段错误。现在的方法列表默认是 small 格式，entsize 是 12 不是 24，三个字段都是 int32 相对偏移。不判断 `0x80000000` 这个标志位，就会把偏移量当指针解引用。

### conformsToProtocol: 和 respondsToSelector: 查的是两件事

这两个方法经常被混着用，它们的定义域是正交的。`Lazy` 声明遵守协议、只实现 required；`Duck` 什么都不声明、三个方法全实现：

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

`@required` 一样不保证方法存在：

```text
  Sloppy 声明遵守 Strict，required 的 mustImplement 一行没写：
    conformsToProtocol:            1
    respondsToSelector:            0
```

编译器只给一条 `warning: method 'mustImplement' in protocol 'Strict' not implemented [-Wprotocol]`。是警告，不是错误。开着 `-Wno-protocol` 的工程，或者警告淹在几千条里的工程，这东西能一路活到线上。

我自己的分界线：`@optional` 一律查；`@required` 在对外的库代码里查，自家模块内部不查。

### 两个 conformsToProtocol 走的不是同一条路

同一个对象，方法版和 C 函数版给出相反的答案：

```text
  [LiarSub 实例 conformsToProtocol:]            = 1
  class_conformsToProtocol([LiarSub class])     = 0
  class_conformsToProtocol([LiarBase class])    = 1
```

`LiarSub` 继承自一个声明了协议的类，自己什么都没写。答案在 `NSObject.mm` 里，继承链是方法版自己用 for 循环爬的：

```objc
- (BOOL)conformsToProtocol:(Protocol *)protocol {
    if (!protocol) return NO;
    for (Class tcls = [self class]; tcls; tcls = tcls->getSuperclass()) {
        if (class_conformsToProtocol(tcls, protocol)) return YES;
    }
    return NO;
}
```

这是最容易记反的一处：走继承链的是 NSObject 这一层的循环，`class_conformsToProtocol` 自己只看传进去的那一个类。

父协议是另一条正交的路，由 `protocol_conformsToProtocol_nolock` 递归处理，而且比的是名字：

```cpp
    if (0 == strcmp(self->mangledName, other->mangledName)) {
        return YES;
    }
```

用 `strcmp` 是因为同一个协议可能在多个镜像里各有一份副本。指针不等，语义相同。实测能对上。`class_conformsToProtocol([LiarBase class], @protocol(Base))` 返回 1。`Base` 是 `Feed` 的父协议，而 `LiarBase` 只声明了 `Feed`。

写运行时反射工具时这个区别会咬人。想问"这个类自己声明了吗"用 C 函数，想问"这个对象能不能当 delegate 用"用方法版。

### 一个我没料到的：respondsToSelector: 会触发动态方法解析

我原本以为它就是查表返回。写了个类在 `+resolveInstanceMethod:` 里动态添加 optional 方法：

```text
  调 respondsToSelector:@selector(feedShouldRetry:)
     +resolveInstanceMethod: 被触发，sel = feedShouldRetry:
  结果 = 1
     动态解析出来的 IMP 执行了
```

`class_respondsToSelector` 走的是带 `LOOKUP_RESOLVER` 的查找路径，查不到会先给类一次动态添加方法的机会。一个实现了动态方法解析的 delegate，可以在被问到 optional 方法时当场"长出"这个方法来。

这条我没在任何 delegate 教程里见过。现实后果是这样。如果你的 delegate 类（或它的父类）里写了 `+resolveInstanceMethod:`，每次 `respondsToSelector:` 未命中都会调进去一次。而 delegate 回调往往在滚动、绘制这类高频路径上。

### delegate 为什么用 weak

流行答案是"防止循环引用"。我在 [[iOS 内存管理：从 MRC、ARC 到属性关键字#第三部分：属性关键字：从所有权推导，而不是从类型名猜|属性关键字：从所有权推导，而不是从类型名猜]] 里说过这个说法把因果讲反了。

真正的理由是所有权方向。`UITableView` 不该拥有它的 delegate，因为那通常是一个更上层、活得更久的对象。这是一条关于"谁是谁的一部分"的判断。跟有没有环无关。三种写法各跑一遍：

```text
========== 1. delegate 是 weak：delegate 死后再回调 ==========
  delegate 活着时：
     weakDelegate   = 0x1012b82b0
     [VC loaderDidFinish:] name=活着
  VC dealloc
  delegate 已析构：
     weakDelegate   = 0x0
     调 [weakDelegate loaderDidFinish:] ... 返回了

========== 2. 换成 strong：不崩，但 VC 被 Loader 吊着 ==========
  出作用域（若无 dealloc 打印，说明 Loader 还攥着它）
  此刻 Loader 仍能回调：     [VC loaderDidFinish:] name=被 strong 持有
  VC dealloc
```

strong 那版不崩也不泄漏，只是把 VC 的寿命绑到了 Loader 身上。没成环就不是泄漏，只是所有权设计错了。

换成 `unsafe_unretained` 就是野指针。delegate 析构后再回调，直接 `Segmentation fault: 11`。开着 `NSZombieEnabled` 跑，会看到经典的 `message sent to deallocated instance`。

`[nil someMethod]` 安全返回，是整个 delegate 模式能这么写的前提。Swift 的 `delegate?.method()` 是同一件事的语法版本。

### delegate 能做、别人做不到的两件事

能拿返回值。`tableView:heightForRowAtIndexPath:` 要一个 CGFloat 回来。通知的 post 没有返回值，block 理论上可以但工程上很少这么用。

能拦截。`shouldChangeTextInRange:` 返回 NO 就阻止了那次编辑。这类"先问后做"的语义要求调用方同步等一个答案，一对多在语义上就不成立。五个观察者返回五个答案，听谁的。

一对一、可回传、可拦截。这三条是 delegate 的全部差异化能力。

---

## 三、target-action：字符串驱动，编译期什么都不检查

### selector 是唯一化的字符串

```text
  @selector=0x1002511dc  sel_registerName=0x1002511dc  NSSelectorFromString=0x1002511dc  全等？是
  从字符串拼出来的 selector 照样能发消息：
     one: 被调用
```

三条路拿到同一个指针，运行时里每个方法名只有一份。这是 target-action 灵活性的来源，也是它全部危险的来源。编译器不检查 target 有没有这个方法，也不检查参数个数对不对。

### UIKit 怎么决定传几个参数：流行说法是错的

中文资料里几乎统一的说法是"UIKit 数方法名里的冒号"，或者"查 `methodSignature.numberOfArguments`"，然后按 0/1/2 个参数分别调用。这个说法我不同意。

我找不到任何一手证据支持它。官方文档的措辞是反过来的，`sendAction(_:to:from:for:)` 的 Discussion 写着：

> **By default, this method pushes two parameters when calling the target.** These last two parameters are optional for the receiver because it is up to the caller (usually a UIControl object) to remove any parameters it added. This design enables the action selector to be one of the following:
> - `-(void)action`
> - `-(void)action:(id)sender`
> - `-(void)action:(id)sender forEvent:(UIEvent *)event`

"永远推两个参数"，然后说这三种签名都能用。**UIKit 从来没有数过冒号，它每次都按两个参数发，少声明几个参数能跑是调用约定的事。**

`objc_msgSend(target, action, sender, event)` 一律按四个实参发（含 self 和 _cmd）。arm64 上它们躺在 x0~x3。callee 声明几个参数，就只读前几个寄存器，多出来的没人管。统一按两个参数发三次，三种签名全部正常返回：

```text
=== 2. 永远按「两个参数」发，callee 声明少了照样跑 ===
  统一用 objc_msgSend(target, action, sender, event) 发三次：
     two:forEvent: 被调用，sender=Sender event=Event
     one: 被调用，sender 指针=0x10075a780
     zero 被调用
  三次都正常返回，没有任何运行时检查参数个数
```

反过来少传，被调方读到的就是上一次调用留在寄存器里的残留：

```text
=== 3. 反过来：少传，被调方读到的是寄存器里的残留 ===
  先用一次正常调用把 x2 填上已知值：
     one: 被调用，sender 指针=0x10075a780
  再按「无参」原型去调 one:（不传 sender）：
     one: 被调用，sender 指针=0x120a8
  ↑ 打印出来的 sender 不是我传的，是上一次调用留在寄存器里的
```

`0x120a8` 是垃圾值。这类 bug 没有编译期信号，只有运行时的诡异值。

那句 "remove any parameters it added" 是 i386 caller-cleanup 年代留下的措辞。今天的 arm64 上没有对应动作。

这个实验还有个插曲值得说。上面这段我最早用 ARC 编，一跑就崩。原因是 ARC 会给传进来的 `id` 参数插一次 retain，于是 `objc_retain` 作用在寄存器垃圾上当场爆掉。换成 `-fno-objc-arc` 才看到干净的残留值。观察手段又一次改变了被观察对象。

所以"数冒号"只能写成一条建议。自己实现 target-action 派发时，用 `methodSignature.numberOfArguments` 决定传几个参数是个好做法。它不是 UIKit 的实现。

### 四种 target，四种持有语义

把它们并排放，比单独记每一个有用得多。

`UIControl` 前面已经解析过了。同一套 nibble 解码接着跑 `UIGestureRecognizerTarget`：

```text
=== class_ro_t @ 0x255aef8  (UIGestureRecognizerTarget) ===
  ivarLayout     = 0x0        (NULL)
  weakIvarLayout = 0x1d3ecde  bytes: 01 00 ...

  ivar_list_t: entsize=32 count=2
    +8    _target          @            size=8
    +16   _action          :            size=8

    ivarLayout     命中偏移 = []       → 没有 strong 的对象 ivar
    weakIvarLayout 命中偏移 = ['0x8']  → weak
```

`UIGestureRecognizer` 的 target 也是 weak。这一条我认为值得单独指出来，因为 Apple 对手势识别器的 target 持有语义只字未提。`addTarget(_:action:)` 的 Discussion 和两个参数说明我都通读过，没有任何一句提到 retain 或不 retain。UIControl 至少还写了"does not retain"。所以这里的处境比 UIControl 更糟：没有文档，只有实现。

正因为这条结论只有实现撑着，我不放心让它只依赖一条证据链。上面整套推导是手工解析磁盘上的 `class_ro_t`——要自己算字段偏移、自己解 chained fixups 的指针高位、自己解 nibble，任何一步算错都会得到一个看起来很像样的错误答案。所以我换了条完全独立的路重验：把程序编成 Mac Catalyst target。它产出的是原生 macOS 二进制，直接 `./out` 就跑，但链接和加载的是真正的 `UIKitCore`，于是可以绕开所有手工解析，直接问 runtime：

```shell
SDK=$(xcrun --sdk macosx --show-sdk-path)
clang -fobjc-arc -target arm64-apple-ios17.0-macabi -isysroot "$SDK" \
      -iframework "$SDK/System/iOSSupport/System/Library/Frameworks" \
      -framework Foundation -framework UIKit -o out prog.m && ./out
```

```text
UIGestureRecognizerTarget    ivarLayout=(null)  weakIvarLayout=01
      ivar[0] _target   off=8    type=@
      ivar[1] _action   off=16   type=:
UIControlTargetAction        ivarLayout=01      weakIvarLayout=11
      ivar[0] _actionHandler  off=8   type=@"UIAction"
      ivar[1] _target         off=16  type=@
      ivar[2] _action         off=24  type=:
```

`class_getIvarLayout` 和 `class_getWeakIvarLayout` 吐出来的字节和手工解析的结果逐字相同。两条互不相干的路径落到同一个答案上，这条结论我就敢写了。

顺带说一句，这个 Catalyst 的口子比它在这里的用处大得多。一直以为要验 UIKit 的东西就得开模拟器，其实 `UIKitCore` 在 macOS 上有一份能原生加载的，`-target arm64-apple-ios*-macabi` 编出来就能跑。代价是桥接层可能改变某些行为，所以它适合验"UIKitCore 里那份代码怎么写的"这类问题，不适合验触摸输入、屏幕 scale 这类和平台强绑定的东西。

`NSTimer` 是反过来的那个，文档写得很明确：

> The timer maintains a strong reference to this object until it (the timer) is invalidated.

实测对得上：

```text
========== 3. NSTimer：强持有 target 直到 invalidate ==========
  target 已出作用域，weak 探针 = 0x1012be670  （非 nil 说明 timer 强持有）
     TimerTarget dealloc
  invalidate 之后，weak 探针 = 0x0
```

`userInfo` 同样是强引用。这就是那个人人踩过的坑：`self` 持有 timer，timer 持有 `self`，不 `invalidate` 谁也走不掉。而 `invalidate` 必须在 timer 所在的那个 runloop 线程上调。

`NSInvocation` 是第四种，它默认连参数都不 retain：

```text
========== 4. NSInvocation 默认不 retain 参数 ==========
  setArgument: 之后（未 retainArguments），arg 还活着 = 0x1012be690
     Arg dealloc
  arg 出作用域后 weak 探针 = 0x0  （nil 说明 NSInvocation 没 retain 它）

  对照：调用 retainArguments 之后
  arg2 出作用域后 weak 探针 = 0x1012be690  （非 nil 说明被 retain 了）
  argumentsRetained = 1
```

一个 invocation 存起来延后再 `invoke`，中间没调 `retainArguments`，参数就已经是野指针了。

四种放一起，规律就清楚了。只要这个对象代表一次持续存在的调用意图，它就倾向于持有：timer 会一直响，invocation 会被延后执行。只要它只代表一次注册关系，它就不持有：control 和手势的 target 都是这一类。

### iOS 14 的 addAction: 把所有权翻了过来

回头看第一段那个 `_actionHandler` 是 strong，不是笔误。同一个 `UIControl`，两种注册方式，所有权方向相反。

顺着链再解析一层，`UIAction` 自己怎么拿它的 handler：

```text
=== class_ro_t @ 0x25d2588  (UIAction) ===
  weakIvarLayout = 0x0  (NULL)
    ...
    +144  _handler         @?           → strong
```

`UIAction._handler` 是 strong，而且 `UIAction` 整个类一个 weak ivar 都没有。于是这条链每一环都是强的：

```objc
[self.button addAction:[UIAction actionWithHandler:^(UIAction *a) {
    [self reload];        // self 被 block 捕获 → UIAction → button → self
}] forControlEvents:UIControlEventTouchUpInside];
```

view controller 持有 button。button 的 `UIControlTargetAction` 持有 UIAction。UIAction 持有 handler block，block 又强捕获 `self`。经典三角。

老写法里 `self` 传给 `addTarget:` 是安全的，同一个人换成新 API 会下意识觉得也安全。我的判断：这是从 target-action 迁到 UIAction 时最容易埋的雷。规则和 [[iOS Block 循环引用与 weak-strong dance]] 里那套完全一样。该 weakSelf 就得 weakSelf。

`UIBarButtonItem` 一并解析了，同一个规律：`_target` 是 weak，`_primaryAction`（UIAction）是 strong。

### 陈旧的注册记录

target 死了以后，`_target` 槽位置零，但包着它的 `UIControlTargetAction` 对象仍然被 control 的数组强持有。`allTargets` 的文档写明了这一点：

> The set may include NSNull to indicate at least one nil target.

一个长期存活的 control 反复换 target 而不 `removeTarget:`，条目会一直堆。量级很小，但这是个真实的积累。这个"弱引用置了 nil、注册记录留着"的模式，下一节在通知那里会原样再出现一次。

> 待补测：`_targetActions` 数组在 target 析构后是否真的不缩短，需要在 iOS 上跑 UIKit 才能确认。上面是从 ivar 布局推出来的结构性结论加官方文档，不是我实测的计数。复现方法：注册三个 target 后全部释放，用 KVC 读 `_targetActions` 的 count，并调 `actionsForTarget:nil`。
>
> 同样待补测的还有 nil-target 沿响应者链分发（`targetForAction:withSender:` 的默认实现是"我能处理吗？能就返回 self，不能就问 nextResponder"）。命令行进程里 `[UIApplication sharedApplication]` 是 nil，这段没法在 macOS 上跑。响应者链本身留到第五周那篇。

---

## 四、通知：唯一的一对多

### NSNotificationCenter 里没有注册表

先看它自己有什么：

```text
  ivar 共 3 个： _impl(^{__CFNotificationCenter=})
                actorQueueManagerLock({os_unfair_lock_s=...})
                _actorQueueManager(@"_NotificationCenterActorQueueManager")
  [nc description] = <CFNotificationCenter 0x104fee000 [0x1fb9c8880]>
```

第一个 ivar 指向一个 CoreFoundation 结构体，确认一下它是什么：

```text
  defaultCenter->_impl               = 0x1022bdaf0
  CFNotificationCenterGetLocalCenter = 0x1022bdaf0
  两者相同？是
```

`NSNotificationCenter` 是 `CFNotificationCenter` 本地中心的一层壳，增删查发全部转手给 CoreFoundation。所以在 Foundation 里怎么翻都找不到那张观察者表。它根本不在 Foundation 里。

后面两个 ivar 是新的，服务于第八节要讲的类型化通知。旧文章里这个类只有一个 ivar，别照着旧截图对。

CoreFoundation 虽然开源，`CFNotificationCenter.c` 却从来没在 Apple 的开源发布里出现过。所以网上那些"从 CoreFoundation 源码看 NSNotificationCenter 实现"的文章，讲的多半是 GNUstep 的 `NSNotificationCenter.m`。GNUstep 用的是"wildcard 裸链表 + nameless 表 + named 两级表"三张表，派发时先 wildcard 后 named。这套结构在 Apple 平台上不成立，下面的实验会当场证伪它。

### 同步、同线程、按注册顺序

三个最基本的性质，一起测掉。

派发是同步的：

```text
  [1] post 之前
     ↳ A 收到 LabNote，线程=主线程
     ↳ B 收到 LabNote，线程=主线程
  [2] post 之后
```

回调跑在 post 的那个线程上，跟观察者在哪注册的没关系：

```text
  注册发生在：主线程
  post 发生在：非主线程
     ↳ A 收到 LabNote，线程=非主线程
     ↳ B 收到 LabNote，线程=非主线程
```

顺序按注册先后，通配观察者和具名观察者混在一起排：

```text
  注册顺序 1..5，实际调用顺序：
    1-通配(name=nil,obj=nil)
    2-具名(obj=nil)
    3-只按对象
    4-具名+对象
    5-通配
```

这直接推翻了"先发 wildcard 再发 named"的说法。但别急着用。Apple 的 Foundation Release Notes 在 10.3 那次重构时就写过一句话：

> you should not have been depending on one observer receiving a notification before another anyway

实现行为是注册序，契约是无序。依赖它就是依赖一个 Apple 随时能改的实现细节。

重入没有任何保护：

```text
     进入第 1 层  进入第 2 层  进入第 3 层  离开第 3 层  离开第 2 层  离开第 1 层
```

递归 post 就是普通的递归调用，深了就爆栈。崩溃栈里 post 相关的帧反复重复，这个形状本身就是识别通知重入的特征。

同步派发还有一个后果，我觉得比上面几条更该知道。一个观察者抛异常，排在它后面的观察者收不到通知：

```text
    观察者 A 执行
    观察者 B 抛异常
    异常传到了 post 的调用方：LabException
  （如果上面没有「观察者 C 执行」，说明异常打断了整条链）
```

C 没有执行，异常一路传回 post 的调用方。三个观察者可能分属三个互不相识的模块。其中一个的 bug 会让另外两个静默失效，而崩溃栈指向的是发通知的那一方。通知常被当成解耦手段。这条说明耦合只是被藏起来了。

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
  （纯快照模型会预测 A,B,C）
```

纯快照模型会预测 `A,B,C`。实际少了 C，说明派发循环在调用每一项之前还会再验一次这条注册是否仍然有效。**"先取快照排好序，再逐项校验"才是能解释所有这类问题的模型。**

### queue: 参数是四种方式里最容易踩的一处

queue 传 nil 时文档写得很清楚：

> When nil, the block runs synchronously on the posting thread.

传 `mainQueue` 呢？文档没说。实测取决于 post 发生在哪。

主线程 post，两个 block 都在 post 返回前跑完了：

```text
  [1] 主线程即将 post
  [2] queue=nil 的 block，线程=主线程
  [3] queue=mainQueue 的 block，线程=主线程
  [4] post 已返回
  [5] 跑了一轮 runloop 之后
```

后台线程 post 时，两个 block 分别落在不同线程：

```text
     queue=nil   的 block 在：非主线程
     queue=main  的 block 在：主线程
```

到这里很容易就写下"非 nil 的 queue 就是异步派发"。我一开始就是这么记的，直到量了一次耗时。block 里 sleep 300 毫秒，从后台线程 post，测 post 这一句本身花多久：

```text
  第 1 次：post 调用耗时 304 ms（block 里 sleep 300ms）
  第 2 次：post 调用耗时 305 ms（block 里 sleep 300ms）
  第 3 次：post 调用耗时 303 ms（block 里 sleep 300ms）
```

block 换到别的队列上执行了，post 的调用方却被原地按住，一直等到 block 跑完才返回。**`queue:` 只搬走了执行位置，没搬走等待。**

post 恰好发生在注册的那个 queue 上，也一样要等：

```text
  在目标 queue 上 post，耗时 302.2 ms
```

这条我一开始预期是 0，猜想实现里会有"已经在目标队列"的短路而直接跳过。跑出来是满额的 300 毫秒，三次一致（302 / 325 / 305）。短路是有的，省掉的是入队和线程切换，不是那次等待。

这个组合会死锁。观察者注册在 `mainQueue`，主线程去等后台线程的 post 完成：

```text
=== 死锁复现：观察者注册在 mainQueue，主线程阻塞等 post 完成 ===
  主线程开始 wait（最多 3 秒）
  后台线程 post 前
  wait 返回 49  （非 0 = 超时 = 死锁成立）

=== 对照：同样的代码，把 queue 换成 nil ===
    block 执行了，thread=非主线程
  wait 返回 0  （0 = 正常完成）
```

block 在主队列上排队等主线程，主线程在等 post 返回，post 在等 block。只改 `queue:` 一个参数，一边正常一边挂死。这是我写这篇时第一个跑挂的程序。

所以 `queue:` 回答的是"回调在哪执行"，它完全不回答"post 会不会卡住调用方"。想要真正的异步，得在 block 里自己 `dispatch_async`。我自己的做法：只要 block 里要碰 UI，就不信任 queue 参数。直接在 block 里显式 `dispatch_async(dispatch_get_main_queue(), ...)`。

### iOS 9 那条"不用移除"的准确边界

原文出自 Foundation Release Notes for OS X v10.11 and iOS 9：

> In OS X 10.11 and iOS 9.0 NSNotificationCenter and NSDistributedNotificationCenter will no longer send notifications to registered observers that may be deallocated. If the observer is able to be stored as a zeroing-weak reference the underlying storage will store the observer as a zeroing weak reference... This means that observers are not required to un-register in their deallocation method. The next notification that would be routed to that observer will detect the zeroed reference and automatically un-register the observer... Block based observers via the -[NSNotificationCenter addObserverForName:object:queue:usingBlock] method still need to be un-registered when no longer in use since the system still holds a strong reference to these observers. CFNotificationCenterAddObserver does not conform to this behavior since the observer may not be an object.

这段话里有四个限定条件，通常只有第一个被引用：

1. 只对 selector 版观察者成立；
2. block 版的 token 被系统强持有，必须移除；
3. 对象若不支持弱引用（自定义 retain/release、`allowsWeakReference` 返回 NO），退化成非 weak 的置零引用，dealloc 期间收到通知的旧行为保留；
4. `CFNotificationCenterAddObserver` 完全不适用。

selector 版我验了。关键是排除"只是碰巧没崩"的可能，所以开着 zombie 跑：

```text
  NSZombieEnabled = YES
     注册后立刻放手（中心若 retain，这里不会 dealloc）
     Ghost 0x104ff3c50 dealloc
  观察者已析构（没有 removeObserver:），post 之前
  post 之后：没有 'message sent to deallocated instance'，也没有回调
```

两条信息。注册之后丢掉所有外部强引用，对象立刻 dealloc，说明中心没有 retain 它。post 时僵尸对象没有拦下任何消息，说明存的不是 `unsafe_unretained`，而是会置 nil 的弱引用。

block 版是另一回事：

```text
  token 类型 = __NSObserver
  token 的 ivar： nc _token _block
  token 与 Payload 都已出作用域。weak 探针 = 0x104ff3c70
  再 post 一次 Leaky：
     block 观察者仍在执行，捕获的 Payload = 0x104ff3c70
```

token 丢了，block 照跑，捕获的对象还活着。`__NSObserver` 里存着 `nc`、`_token`、`_block` 三个字段。中心攥着它不放。你不 `removeObserver:` 就没人放。

和上一篇有个可以对账的地方：KVO 的观察者变成置零引用是 iOS 11（macOS 10.13）的事，通知中心是 iOS 9。两次改动性质一样，时间差了两年。

### 不移除 selector 观察者的实际代价

Release Notes 说"下一次本该发给它的时候会一并注销"。言外之意是：如果那个通知名再也没被 post 过，记录就一直在。用 `malloc_zone_statistics` 量，三次跑几乎一致：

```text
  基线 malloc in-use                =    0.119 MB
  同名 100000 条死记录（未 post）    =    0.546 MB  (+0.427 MB, 每条 4.5 字节)
    post 一次之后                   =    0.546 MB  (+0.427 MB)
  对照：同样 100000 条但 add+remove  =    0.546 MB  (+0.000 MB, 每条 0.0 字节)
```

十万条从不移除的注册记录，残留 0.43 MB，每条约 4.5 字节。配平的 add/remove 是精确的 0。post 一次之后残留没有回落。

4.5 字节远小于一条完整注册记录该有的大小。所以大部分内容确实被回收了，剩下的更像是哈希表扩容之后不再收缩的那部分。这一句是我的推测，不是实测结论。

结论我倾向于说得温和些：iOS 9 的改动解决的是崩溃，泄漏被压到了很小的量级但没有归零。`removeObserver:` 从"正确性要求"降级成了"卫生习惯"，它没有变成多余代码。

这里得提一个我自己犯的错。我最早用 `phys_footprint` 量同一件事，两次跑都得到 +1.5 MB 左右，差点写进正文。再跑三次全变成 +0.05 MB。`phys_footprint` 是页粒度的进程级汇总，受同进程里其他分配的干扰，根本不适合量几百 KB 的差异。换成 `malloc_zone_statistics` 之后三次结果连小数点后三位都对得上。两次采样支持的结论不算结论。

### 真正异步的那个是 NSNotificationQueue

`NSNotificationQueue` 本身不派发。它是 NotificationCenter 前面的一个缓冲区，按 `NSPostWhenIdle` / `NSPostASAP` / `NSPostNow` 三种时机把通知交出去，可以按名字或对象合并。最后仍然走同步那条路。

它绑定 runloop 和线程，所以在没有 runloop 的线程上 `NSPostWhenIdle` 永远不会被 flush。这是它最常见的坑。新代码基本没有理由用它。

---

## 五、block 回调：唯一默认强持有的那个

block 的运行时结构在 [[iOS Block 的结构：ABI、descriptor 与三种类型]]。捕获规则在 [[iOS Block 的变量捕获与 __block]]。环怎么成的在 [[iOS Block 循环引用与 weak-strong dance]]。这里只补它作为通信手段的那一面。

四种方式里，前三种的默认所有权都是"不持有回调方"。delegate 用 weak、UIControl 和手势的 target 是 weak、通知中心对 selector 观察者是弱引用。block 反过来，默认强持有它捕获的一切，而且这个持有是隐式的，语法上看不见。

这带来两个后果。一是环。二是一个更隐蔽的问题：block 会把回调方的寿命延长到 block 自己被释放为止。一个网络请求的 completion 持有 view controller。用户已经退出界面了，controller 还得等请求回来才能死。这不算泄漏，Instruments 的 Leaks 也查不出来。但它是真实的行为差异。

block 回调适合的形状很窄：一次性、有明确起止、调用方就是关心结果的人。`fetchData:completion:` 是标准形态。一旦变成"多次触发的持续回调"，block 的所有权就开始难管，因为你得决定它什么时候被释放。这时候 delegate 反而更清爽。

我很少用 block 做持续回调。

判空是另一个老问题。`if (completion) completion();` 解决的是空指针，不解决属性在判断和调用之间被另一个线程改掉。真要并发安全，得先取到局部变量再调。

我自己的阈值：需要返回值或需要拦截，用 delegate；一次性异步结果，用 block；其余情况先想清楚是不是真的需要一对多。

---

## 六、四种方式的实际开销

每种 20 万次一轮，取 5 轮最小值，三次运行结果稳定到小数点后一位：

```text
  直接消息发送                       2.56 ns
  delegate（读 weak + responds + 调用）
                                    26.5 ns
  block 调用                         0.32 ns
  通知 postNotificationName:       213.2 ns
  通知 postNotification:（对象已建好）
                                   156.9 ns
```

只有量级值得记：通知比一次直接消息发送贵约两个数量级，block 调用比消息发送还便宜（一次间接跳转，没有方法查找）。

把 delegate 那 26 ns 拆开：

```text
  [t ping] 纯消息发送               2.08 ns
  只读一次 weak                    16.48 ns
  只判断 respondsToSelector:        4.28 ns
  读 weak + responds + 调用        26.34 ns
```

大头是读 weak。`objc_loadWeakRetained` 要查 SideTable、加锁、tryRetain、再 autorelease。比一次消息发送贵七八倍。这解释了 UIKit 内部为什么大量使用"把 delegate 读到局部变量再连着用"的写法。

这里有个测量陷阱，我自己先掉进去了一次。最初我加了一行"unsafe_unretained 的 delegate"作对照，量出 13 ns，比直接发消息贵五倍，怎么想都不对。隔离出来才看清：

```text
  [t ping] 基线                                        2.57 ns
  [h->_ivar ping]  直接读 ivar                         2.59 ns
  [h.prop ping]    走 getter（ARC 会 retain 返回值）   13.25 ns
  只调 getter 不发消息                                  5.41 ns
```

直接读 ivar 和基线一模一样。慢的是走属性 getter：ARC 会给返回 `id` 的方法调用插一次 retain / release，那对配平就是这 10 ns。所以"unsafe delegate 是零成本的"这句话只在直接读 ivar 时成立。

还有一个数字更有用：

```text
  0 个注册记录：每次 post 145.8 ns
  10 万条死记录：每次 post 147.4 ns
  倍数 = 1.01×
```

十万条陈旧注册记录对派发速度没有影响。"不移除观察者会拖慢通知"这个担心是不必要的，代价只在那 0.43 MB 内存上。

通知的开销集中在"每次 post 都要造一个 `NSNotification` 对象、查表、排序"，不在派发本身。高频路径不要用通知。每帧、每个 cell、每次滚动，这些都算。200 ns 乘以每秒几千次就是可见的开销。

---

## 七、怎么选

我不喜欢"通知用于一对多、delegate 用于一对一"这种记法，它把结论当定义背。实际推导只要问三个问题。

第一，通信是几对几？一个明确的对方，还是任意多个不认识的听众。这条最先排除掉一半选项。

第二，调用方需要回应吗？需要返回值或需要拦截，只剩 delegate。这条是硬约束，不是偏好。

第三，两边的生命周期谁长？回调方比发起方活得久（controller 之于 view），发起方就不该持有它，用 delegate / target-action / 通知。回调方的存在只是为了这一次调用（completion handler），用 block。

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
| 单次开销 | ~26 ns | ~一次消息发送 | ~200 ns | ~0.3 ns |

最后展开一句表里放不下的。通知的同步派发有个常被忽略的好处：在观察者里下断点，调用栈里直接能看到是谁 post 的。这是它相对 KVO 的调试优势。但 `queue:` 非 nil 的异步路径会把这个栈丢掉，新的 `AsyncMessage` 也一样。

我自己的默认值是这样。能用 delegate 就用 delegate。跨模块、发送方不该知道接收方存在的，才用通知。UI 控件的点击继续用 target-action，除非已经全面转 UIAction。异步结果用 block。通知我会额外加一条自律：一个通知名只在一处 post，并且把 post 点写进注释。一对多的代价就是没人知道谁在听，这个代价必须靠约定去付。

---

## 八、Swift 里的演进

三代 API，三种默认行为，都和"通知在哪个线程、丢不丢"有关。

**Combine（iOS 13）**：`publisher(for:object:)`。内部就是 `addObserver(forName:object:queue: nil)`，没有任何队列跳转。要切线程得自己 `.receive(on:)`。

**AsyncSequence（iOS 15）**：`notifications(named:object:)`。本机 SDK 里的签名已经不是老文章里那个样子了：

```swift
@preconcurrency public func notifications(
    named name: Foundation.Notification.Name,
    object: (any Swift.AnyObject & Swift.Sendable)? = nil
) -> Foundation.NotificationCenter.Notifications
```

`object` 多了 `Sendable` 约束。而 `Notification` 本身的 Sendable 遵守是 `@available(*, unavailable)` 的。所以 Swift 6 下跨隔离域传它会直接报错。Apple 给的解法是先 `map` 出可发送的字段再用。

**类型化通知（iOS 26）**。这是我认为最值得关注的一次改动。本机 SDK 的 swiftinterface 里能直接读到：

```swift
@available(macOS 26, iOS 26, tvOS 26, watchOS 26, visionOS 26, *)
extension Foundation.NotificationCenter {
  public protocol MainActorMessage : Swift.SendableMetatype {
    associatedtype Subject
    static var name: Foundation.Notification.Name { get }
    @MainActor static func makeMessage(_ notification: Foundation.Notification) -> Self?
    @MainActor static func makeNotification(_ message: Self) -> Foundation.Notification
  }
  public protocol AsyncMessage : Swift.Sendable {
    associatedtype Subject
    static var name: Foundation.Notification.Name { get }
    static func makeMessage(_ notification: Foundation.Notification) -> Self?
    static func makeNotification(_ message: Self) -> Foundation.Notification
  }
}
```

它把三件事从"运行时约定"提到了"类型系统"：

- 隔离域。`MainActorMessage` 保证同步投递到主线程，`AsyncMessage` 明确表示异步、任意线程。第四节那个"queue: .main 有时同步有时异步"的坑，在这套 API 里从设计上就不存在了。
- 载荷类型。不用再从 `userInfo` 里按字符串 key 取值再强转。
- 注销。新的 `ObservationToken` 在 `deinit` 时自动注销。老 block token 丢了等于永久泄漏，新 token 丢了等于注销，语义正好相反。

第四节看到的 `_actorQueueManager` 那个新 ivar，就是这套东西在 Objective-C 侧留下的痕迹。

有一个迁移风险要提前知道。观察端桥接旧通知时用的是 `MainActor.assumeIsolated`，这是运行时断言。老代码从后台线程 post 一条恰好有 `MainActorMessage` 遵守的通知，新式观察者会当场崩。提案 SF-0011 自己承认了这一点。给的处理办法是去改那个 post 的调用方。

> 待真机补测：`MainActorMessage` 的同步投递保证、以及后台 post 触发 `assumeIsolated` 崩溃这两条。上面的协议定义是从本机 SDK 的 swiftinterface 抄的原文，行为本身我没跑。复现方法：写一个最小 Swift 工程，两个观察者一新一旧，分别从主线程和后台线程 post 同一个名字。

---

## 九、几个不准的说法

- **"UIKit 靠数冒号（或 `numberOfArguments`）决定给 action 传几个参数。"** 文档原文是 "pushes two parameters"。永远按两个发，callee 少声明也能跑，这是调用约定的事。数冒号只能当作自己实现派发时的建议。
- **"UIControl 的 target 是 unsafe_unretained。"** `weakIvarLayout` 解出来是 zeroing weak。但文档只写了 "does not retain"，weak 是实现不是契约。
- **"UIGestureRecognizer 的 target 语义和 UIControl 一样有文档保证。"** 实现上一样是 weak，但 Apple 对它的持有语义一个字都没写。
- **"delegate 用 weak 是为了防循环引用。"** 因果反了。理由是所有权方向，循环引用只是用错 strong 之后的一个后果。
- **"NSNotificationCenter 内部有三张表，先发 wildcard 再发 named。"** 那是 GNUstep。实测通配和具名观察者按注册顺序混排。
- **"通知是异步的。"** `postNotification:` 是同步的，返回前所有 selector 观察者都已执行完。
- **"`queue:` 参数非 nil 就是异步派发。"** 只有执行位置是异步的。block 里 sleep 300ms，post 这一句稳定花掉 303~305 ms。post 恰好发生在目标队列上也一样要等满。
- **"一个观察者出错不影响别人。"** 同步派发是一条链，中间抛异常会让后面的观察者收不到，异常还会传给 post 的调用方。
- **"通知在主线程执行。"** 在 post 的那个线程执行。
- **"iOS 9 之后不用移除观察者。"** 只对 selector 版成立。block 版的 token 被强持有，不移除会一直被调用。
- **"post 期间移除观察者不会生效，因为已经取了快照。"** 会生效。快照之外还有一次逐项校验。
- **"不移除观察者会让通知中心越来越慢。"** 十万条死记录对 post 耗时没有可测影响，代价是约 0.43 MB 内存。
- **"respondsToSelector: 只是查表。"** 它会触发 `+resolveInstanceMethod:`。
- **"`conformsToProtocol:` 返回 YES 就说明方法都实现了。"** 它只查编译期的遵守声明。反过来，全部实现了但没声明遵守的类会返回 NO。
- **"`class_conformsToProtocol` 会走继承链。"** 不走。继承链是 `-[NSObject conformsToProtocol:]` 里那个 for 循环走的；`class_conformsToProtocol` 只递归父协议。

---

## 总结

四种方式在运行时只有三种表示：SEL 加接收者、对象加编译期契约、函数指针加捕获环境。前两种最终都是 `objc_msgSend`。

持有关系是选型的主线。delegate 的 weak、UIControl 与手势 target 的 weak、通知中心对 selector 观察者的弱引用，都是同一个设计判断。被通知的一方通常活得更久，不该被发起方拥有。反过来的是那些代表"一次持续存在的调用意图"的对象，NSTimer 强持有 target 到 invalidate 为止。block 也在这一侧，`addAction:` 把 target-action 一并拖了进来。

"忘了清理"的后果各不相同，这比 API 差异更值得记。delegate 什么都不会发生。target-action 和 selector 版通知留下无害但不回收的条目。block 版通知观察者会被永久调用。block 回调则是成环或延长寿命。

通知那节最该带走的不是"同步还是异步"这个二分，而是同步派发牵出的两件事：一个观察者抛异常会掐断整条链，以及 `queue:` 只搬走了执行位置、没搬走等待。后者能直接把主线程锁死。

方法论上，这一篇有三次"先怀疑仪器"。协议方法表按老格式解析当场段错误。原因是现在默认走 small 相对偏移格式。参数残留实验在 ARC 下崩溃，因为 ARC 给入参插了一次 retain。`unsafe_unretained` delegate 量出 13 ns，因为走属性 getter 时 ARC 给返回值配了 retain / release。三次都是观察手段改变了被观察对象。结果和预期不符的时候，先查仪器。

下一篇 [[iOS GCD：队列不是线程，以及死锁的准确边界]]。

## 参考资料

### 官方

- [Foundation Release Notes for OS X v10.11 and iOS 9](https://developer.apple.com/library/archive/releasenotes/Foundation/RN-FoundationOlderNotes/index.html)：NotificationCenter 一节，第四节那四个限定条件的原文出处
- [UIApplication.sendAction(_:to:from:for:)](https://developer.apple.com/documentation/uikit/uiapplication/sendaction(_:to:from:for:))："pushes two parameters"，第三节纠正"数冒号"说法的依据
- [UIControl.addTarget(_:action:for:)](https://developer.apple.com/documentation/uikit/uicontrol/addtarget(_:action:for:))："does not retain the object in the target parameter"
- [Timer.init(timeInterval:target:selector:userInfo:repeats:)](https://developer.apple.com/documentation/foundation/timer)："maintains a strong reference to this object until it is invalidated"
- [NSInvocation.retainArguments()](https://developer.apple.com/documentation/foundation/nsinvocation/retainarguments())：默认不 retain 参数
- [addObserver(forName:object:queue:using:)](https://developer.apple.com/documentation/foundation/notificationcenter/addobserver(forname:object:queue:using:))："When nil, the block runs synchronously on the posting thread"
- [SF-0011 Concurrency-Safe Notifications](https://github.com/swiftlang/swift-foundation/blob/main/Proposals/0011-concurrency-safe-notifications.md)：iOS 26 类型化通知的提案。它在 swift-foundation 的 `Proposals/` 目录，不在 swift-evolution 里
- [apple-oss-distributions/objc4](https://github.com/apple-oss-distributions/objc4)：`NSObject.mm` 的 `conformsToProtocol:`；`objc-runtime-new.mm` 的 `getProtocolMethodList` 与 `protocol_conformsToProtocol_nolock`
- [llvm-project — CGObjCMac.cpp](https://github.com/llvm/llvm-project/blob/main/clang/lib/CodeGen/CGObjCMac.cpp)：第 7674 行，协议方法 IMP 恒为 NULL 的注释原文
- 本机 SDK：`AppKit/NSControl.h`（target 是 weak 的书面契约）、`Foundation.swiftmodule/arm64e-apple-macos.swiftinterface`（`MainActorMessage` / `AsyncMessage`）

### 经典

- [objc.io — Communication Patterns](https://www.objc.io/issues/7-foundation/communication-patterns/)：最早把四种方式按"是否需要返回值、观察者数量、耦合度"对比的一篇，结构至今可用
- [Cocoa Design Patterns](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CocoaFundamentals/CocoaDesignPatterns/CocoaDesignPatterns.html)：Delegation、Notifications、Target-Action 的官方定义
- [Ole Begemann — Do you still need to unregister notification observers?](https://oleb.net/blog/2018/01/notificationcenter-removeobserver/)：block 版必须移除的实测，也是当年抓出 Apple 文档自相矛盾的那篇
- [GNUstep NSNotificationCenter.m](https://github.com/gnustep/libs-base/blob/master/Source/NSNotificationCenter.m)：一份可读的完整实现。拿它理解概念可以，别把它当 Apple 的实现

### 本地

- [[20 专题笔记/Runtime/Part 2 - 消息发送与转发]]
- [[iOS 内存管理：从 MRC、ARC 到属性关键字#第三部分：属性关键字：从所有权推导，而不是从类型名猜|属性关键字：从所有权推导，而不是从类型名猜]]
- [[iOS Block 循环引用与 weak-strong dance]]
- [[iOS Block 的结构：ABI、descriptor 与三种类型]]
- [[iOS weak 的实现：SideTable 与置 nil 的时机]]
- [[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]]

---

实验环境：macOS 26.5.2（arm64，Apple Silicon），Apple clang 21.0.0。全部实验都是 macOS 原生二进制，没有启动任何模拟器。

```shell
clang -fobjc-arc  -framework Foundation -o out prog.m && ./out   # 默认
clang -fno-objc-arc -framework Foundation -o out prog.m          # 参数残留那组
clang -fobjc-arc -O1 -framework Foundation -o out prog.m         # 第六节的耗时
NSZombieEnabled=YES ./out                                        # 观察者不移除那组
```

UIKit 的部分没有跑，用的是静态二进制分析。直接解析磁盘上 iOS 26.5 模拟器运行时里的 `UIKitCore`（arm64 slice），按 `class_ro_t` 的字段偏移读出 `ivarLayout` / `weakIvarLayout`。再用 nibble 解码规则映射到 `ivar_list_t` 的偏移上：高 4 位是跳过几个 word，低 4 位是连续几个 word 命中。二进制用了 chained fixups，指针字段要先取低 36 位才是目标 vmaddr。四个类各解一遍：`UIControlTargetAction`、`UIGestureRecognizerTarget`、`UIBarButtonItem`、`UIAction`。

四个类里最关键的 `UIGestureRecognizerTarget` 和 `UIControlTargetAction` 另用 Mac Catalyst target（`-target arm64-apple-ios17.0-macabi`）编了一份原生二进制，直接调 `class_getIvarLayout` / `class_getWeakIvarLayout` 复核，结果与手工解析逐字一致。全程零模拟器：开工前和收工后各查一次 `xcrun simctl list devices booted`，两次都是空。

这个方法只对 ARC 编译的类有效。Foundation 里 `NSConcreteNotification`、`__NSObserver` 这些类的 `ivarLayout` 全是 NULL。解码器会把每个对象型 ivar 都判成 `unsafe_unretained`，那是没有信息，不是结论。UIKitCore 是 ARC 编译的，所以对上面四个类有效。

> 待真机补测：第六节那组耗时（macOS 与 iOS 的 objc 运行时相同，但缓存与调度不同，绝对值不可移植）；第三节末尾 `_targetActions` 是否真的不缩短；第八节 iOS 26 类型化通知的运行时行为。
