---
title: 【iOS】Block 的变量捕获与 __block
published: 2026-07-26
description: 同一行 &shared，在 Block 创建前打印是栈地址，创建后在同一个作用域里打印变成了堆地址。这个现象是理解 __forwarding 最短的路径。
tags:
  - iOS
  - Objective-C
  - Block
  - Memory
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 9
draft: true
---
# Block 的变量捕获与 __block

上一篇 [[iOS Block 的结构：ABI、descriptor 与三种类型]] 讲了 Block 的定长部分是 32 字节，捕获的变量按声明顺序追加在后面。这一篇讲"追加进去的到底是什么"——是值、是地址、还是别的东西。

结论会分成四种情况，取决于变量的存储期和有没有 `__block`。但比结论更值得看的是一个现象：**同一行 `&shared`，在 Block 创建之前打印是栈地址，创建之后在同一个作用域里打印，变成了堆地址。** 这是理解 `__forwarding` 最短的一条路。

---

## 一、捕获矩阵

先跑一遍。四种变量，看 Block 里能不能改：

```objc
int gGlobal = 100;        // 全局
static int sStatic = 200; // 文件内 static

int autoVar = 1;          // 局部 auto
__block int blockVar = 1; // 局部 auto + __block
static int localStatic = 1; // 函数内 static

void (^b)(void) = ^{
    // autoVar = 2;   ← 这一行编译不过
    blockVar    = 2;
    localStatic = 2;
    gGlobal     = 2;
    sStatic     = 2;
};
b();
```

```text
  auto（只读捕获）    改不了，编译期就拦住：Variable is not assignable
  __block auto        改后 = 2
  函数内 static       改后 = 2
  全局变量            改后 = 2
  文件内 static       改后 = 2
```

四种能改，一种不能。原因不在"Block 有多大权限"，而在编译器往 Block 结构体里塞了什么。

**全局变量和文件内 static** 根本没被捕获。它们的地址在编译期就确定了，Block 的 invoke 函数直接按符号访问，跟普通函数访问全局变量没有任何区别。所以能改，改了外面立刻可见。

**函数内的 static** 也一样。它虽然写在函数里，但存储期是静态的，地址同样在编译期确定。编译器捕获的是它的地址，不是值。

**普通 auto 变量** 被拷贝了一份值进 Block 结构体。invoke 函数里访问的是这份副本，所以改它没有意义——改了也影响不到外面那个。编译器干脆直接拒绝，报 `Variable is not assignable`。

**`__block` 修饰的 auto 变量** 是特殊处理，下面单独讲。

---

## 二、auto 变量捕获的是"创建那一刻的值"

这条容易被含糊过去，测一下就清楚：

```objc
int plain = 5;
void (^blk2)(void) = ^{ NSLog(@"  Block 内 plain = %d（只读副本）", plain); };
plain = 99;              // Block 创建之后再改
blk2();
```

```text
  Block 内 plain = 5（只读副本）
  Block 外 plain = 99
```

Block 里看到的是 5，不是 99。

值是在**创建 Block 的那一刻**被拷进结构体的，之后外面怎么改都跟它无关。这跟"闭包捕获了环境"那种模糊说法不一样——它捕获的不是环境，是一个快照。

上一篇那组 size 数据（无捕获 32 字节、捕获一个 `int` 变 36 字节）说的就是这件事：那 4 个字节就是这份副本的存放处。

---

## 三、`__block` 做了什么

`__block` 修饰之后，编译器不再把值直接塞进 Block 结构体，而是先把变量包进一个独立的结构体：

```c
struct Block_byref {
    void * __ptrauth_objc_isa_pointer isa;
    struct Block_byref *forwarding;
    volatile int32_t flags;
    uint32_t size;
    // 变量本体接在后面
};
```

Block 结构体里存的是指向这个 `Block_byref` 的指针。关键在第二个字段 `forwarding`——它是一个指向"自己"的指针。

刚创建时，`Block_byref` 在栈上，`forwarding` 指向它自己。等 Block 被 copy 到堆上，runtime 会把整个 `Block_byref` 也搬到堆上，然后**把栈上那份的 `forwarding` 改成指向堆上的新副本**。

而编译器改写过所有对这个变量的访问，一律走 `byref->forwarding->变量`。于是不管你从哪里访问、不管 Block 现在在栈还是在堆，最终读写的都是同一份数据。

### 一个能看见的证据

道理讲完了不如看一眼。`__block` 变量在 Block 创建前后各打印一次地址：

```objc
__block int shared = 5;
NSLog(@"  1. 刚声明，Block 还没出现     &shared = %p", &shared);

void (^blk)(void) = ^{
    NSLog(@"  3. Block 内部                 &shared = %p", &shared);
    shared = 7;
};

NSLog(@"  2. Block 已创建并赋给 strong  &shared = %p", &shared);
blk();
NSLog(@"  4. Block 改成 7 之后，外面读到 shared = %d", shared);
```

```text
  1. 刚声明，Block 还没出现     &shared = 0x16d6160c0
  2. Block 已创建并赋给 strong  &shared = 0x60000020c0f8  ← 外面看到的地址也变了
  3. Block 内部                 &shared = 0x60000020c0f8
  4. Block 改成 7 之后，外面读到 shared = 7
```

第 1 行是栈地址（`0x16d6...` 落在栈区），第 2 行变成了堆地址（`0x6000...` 是分配器管理的区域）。**中间只隔了一句 Block 的声明和赋值。**

同一个源码表达式 `&shared`，前后取到两个不同的地址。因为这两行 `&shared` 都被编译器改写成了 `byref->forwarding->shared` 的取址——第一次 forwarding 还指向栈上的自己，第二次已经被改指向堆上的副本了。

第 3 行和第 2 行相同，说明块内块外访问的确实是同一份。第 4 行确认了修改可见。

这就是 `__block` 为什么能让变量"在块内外共享"：**它把变量从栈上搬走了，并且留下一个转发指针让所有旧的访问路径也跟着走过去。**

---

## 四、捕获对象时会持有它

上面讨论的都是标量。捕获对象要多一层：Block 会持有它。

```objc
__weak id weakProbe = nil;
void (^hold)(void) = nil;
@autoreleasepool {
    NSObject *o = [NSObject new];
    weakProbe = o;
    hold = ^{ (void)o; };      // 捕获 o
}   // o 的局部强引用在这里失效
NSLog(@"  离开作用域后 weak = %@", weakProbe);
hold = nil;
NSLog(@"  block 置 nil 后  weak = %@", weakProbe);
```

```text
  离开作用域后 weak = <NSObject: 0x600000014000>
  block 置 nil 后  weak = (null)
```

`o` 的局部强引用已经随作用域结束而消失，但 weak 探针还能看到对象——说明 Block 接管了这份所有权。等 Block 自己被释放，对象才跟着走。

这个持有关系是靠上一篇讲的 `BLOCK_HAS_COPY_DISPOSE` 那一位实现的：置位表示 descriptor 里有 copy 和 dispose 两个辅助函数，Block 从栈搬到堆时调 copy 去 retain 捕获的对象，Block 销毁时调 dispose 去 release。上一篇实测过，捕获 `int` 的 Block 不置这一位，捕获对象的才置。

所以那一位不只是个标记，它对应着实打实的两个函数和一次引用计数操作。

这也是循环引用的全部来源：Block 持有对象，对象又持有 Block。下一篇 [[iOS Block 循环引用与 weak-strong dance]] 专门讲这个。

---

## 五、几个说法需要辨析

**"Block 捕获 auto 变量是值捕获，捕获 `__block` 变量是引用捕获。"** 结论上没错，但"引用捕获"这个说法会让人以为编译器传了个 C 意义上的指针。实际上是把变量整个搬进了一个带 `forwarding` 的结构体，而 `forwarding` 的存在恰恰说明它不是简单的取地址——简单取地址的话，Block 搬到堆上之后那个地址就悬空了。

**"`__block` 是为了让 Block 能修改外部变量。"** 这是效果，不是设计目的。真正要解决的问题是**生命周期**：Block 可能活得比创建它的栈帧更久，而栈上的变量会随栈帧一起消失。`__block` 把变量搬到堆上跟着 Block 走，"能修改"只是搬家之后的自然结果。理解成生命周期问题，才能解释 `forwarding` 为什么存在。

**"全局变量和 static 变量会被 Block 捕获。"** 不会。它们的地址编译期确定，Block 直接按符号访问，结构体里不占空间。你可以用上一篇那个读 `descriptor->size` 的办法自己验证——一个只访问全局变量的 Block，size 仍然是 32。

**"`__block` 修饰对象可以打破循环引用。"** MRC 时代成立（那时 `__block` 修饰的对象不会被 retain），ARC 下不成立——ARC 里 `__block` 修饰的对象照样被持有。ARC 下打破环要用 `__weak`。这是新旧语境混淆里最危险的一条，因为它会让人写出自以为安全的泄漏代码。

**"Block 里改了 `__block` 变量，外面要等 Block 执行完才能看到。"** 没有这回事。块内块外访问的是同一块内存，改完立刻可见，上面第 4 行输出就是证据。

---

## 总结

变量能不能在 Block 里改，取决于编译器往 Block 结构体里塞了什么：全局和 static 变量压根不进结构体（按符号访问，所以能改）；普通 auto 变量进的是一份创建时刻的值副本（所以不让改）；`__block` 变量进的是一个指向 `Block_byref` 结构体的指针（所以能改，而且块内外共享）。

`__block` 真正解决的是生命周期问题而不是可写性。变量被搬进一个带 `forwarding` 指针的结构体，Block 从栈搬到堆时这个结构体跟着搬，栈上那份的 `forwarding` 被改指向堆上的新家。所有访问都走 `forwarding`，于是无论 Block 在哪，读写的都是同一份数据。这一点用 `&shared` 在 Block 创建前后各打印一次就能看见——地址会变。

捕获对象时 Block 会持有它，靠的是 `BLOCK_HAS_COPY_DISPOSE` 那一位对应的两个辅助函数。这个持有关系就是循环引用的来源。

最后提醒一次那条最危险的旧知识：**ARC 下 `__block` 修饰对象不能打破循环引用**，那是 MRC 时代的结论。

## 参考资料

### 规范与源码

- [Clang — Block Implementation Specification](https://clang.llvm.org/docs/Block-ABI-Apple.html)：`__block` storage 与 forwarding pointer 的定义
- [Apple — Blocks and Variables](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Blocks/Articles/bxVariables.html)：按自动变量、静态变量、全局变量、`__block` 分类讲的官方版本
- [apple-oss-distributions/libclosure](https://github.com/apple-oss-distributions/libclosure)：`Block_private.h` 里的 `Block_byref` 定义，`runtime.cpp` 里是 `_Block_object_assign` / `_Block_object_dispose`

### 拓展

- [halfrost — 深入研究 Block 捕获外部变量和 __block 实现原理](https://halfrost.com/ios_block/)：捕获规则和 `__forwarding` 讲得系统
- [Desgard — 浅谈 block（2）：截获变量方式](https://github.com/Desgard/iOS-Source-Probe/tree/master/Objective-C/Runtime)：含 `-rewrite-objc` 的真实产物，注意那个命令在现在的 Xcode 上已经无法执行

### 本地

- [[iOS Block 的结构：ABI、descriptor 与三种类型]]
- [[iOS Block 循环引用与 weak-strong dance]]
- [[iOS 属性关键字：从所有权推导，而不是从类型名猜]]

---

实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -fobjc-arc -target arm64-apple-ios17.0-simulator`。所有输出块都是真实运行结果。

第三节那个地址实验对时机敏感：第 2 行的打印必须在 Block 已经被赋值给 strong 变量之后，否则 byref 还在栈上，你看到的会是两个相同的栈地址。这不是实验失败，是 Block 还没被 copy 到堆——上一篇讲过，ARC 只在特定上下文才做这次搬迁。
