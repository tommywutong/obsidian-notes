---
title: 【iOS】Block 的变量捕获与 __block
published: 2026-07-26
description: 同一行 __block id，ARC 编出来的 byref layout 是 STRONG，MRC 编出来是 UNRETAINED。所有权语义的差别就固化在这 4 个 bit 里——这也是「__block 能打破循环引用」为什么变成了过期知识。
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

上一篇 [[iOS Block 的结构：ABI、descriptor 与三种类型]] 讲了 Block 的定长部分是 32 字节，捕获的变量按声明顺序追加在后面。这一篇讲追加进去的到底是什么。

答案分四种情况，其中三种一句话就能讲完，第四种（`__block`）值得单开两节。

---

## 一、捕获矩阵

四种变量，看 Block 里能不能改：

```objc
int gGlobal = 100;          // 全局
static int sStatic = 200;   // 文件内 static

int autoVar = 1;            // 局部 auto
__block int blockVar = 1;   // 局部 auto + __block
static int localStatic = 1; // 函数内 static

void (^b)(void) = ^{
    // autoVar = 2;   ← 这一行编译不过
    blockVar    = 2;
    localStatic = 2;
    gGlobal     = 2;
    sStatic     = 2;
};
```

```text
  auto（只读捕获）    改不了，编译期就拦住：Variable is not assignable
  __block auto        改后 = 2
  函数内 static       改后 = 2
  全局变量            改后 = 2
  文件内 static       改后 = 2
```

四种能改，一种不能。差别在编译器往那 32 字节后面塞了什么。读一下 `descriptor->size` 就一目了然：

```text
  空              size=32  flags=0x50000000  IS_GLOBAL=1
  只碰全局        size=32  flags=0x50000000  IS_GLOBAL=1
  只碰文件static  size=32  flags=0x50000000  IS_GLOBAL=1
  只碰函数static  size=32  flags=0x50000000  IS_GLOBAL=1
  捕获 auto       size=36  flags=0xc1000002  IS_GLOBAL=0
```

前四行的 size 都是 32，也就是纯定长部分，一个字节都没多。而且 flags 里都带着 `BLOCK_IS_GLOBAL`——**只碰全局或 static 变量的 Block 是全局 Block，连堆都不上。**

所以全局变量、文件内 static、函数内 static 三者一视同仁：**都不被捕获**。它们的地址在编译期就确定了，invoke 函数按符号直接访问，跟普通函数访问全局变量没有区别。能改，改了外面立刻可见。

这里要纠正我自己第一版写错的一条。我当时写"函数内 static 编译器捕获的是它的地址"——错的，它压根不进结构体。这个说法的来源是老的 `-rewrite-objc` 产物，那个改写工具确实把函数内 static 重写成了指针。但上一篇说过，rewriter 走的是一套独立的教学用逻辑，不是真实 codegen 的旁路输出，两者结论可以不一致。这就是一例。

只有普通 auto 变量真的被拷进了结构体（32 → 36，多的 4 字节就是那个 `int`）。invoke 函数访问的是这份副本，改它影响不到外面，所以编译器直接拒绝。

---

## 二、auto 变量捕获的是创建那一刻的值

```objc
int plain = 5;
void (^blk2)(void) = ^{ NSLog(@"  Block 内 plain = %d", plain); };
plain = 99;              // Block 创建之后再改
blk2();
```

```text
  Block 内 plain = 5
  Block 外 plain = 99
```

值是在创建 Block 的那一刻被拷进结构体的，之后外面怎么改都跟它无关。

---

## 三、同一行 &shared，前后两个地址

先看输出，再解释。`__block` 变量在 Block 创建前后各打印一次地址：

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

第 1 行是栈地址（`0x16d6...` 落在栈区），第 2 行变成了堆地址（`0x6000...` 是分配器管理的区域）。中间只隔了一句 Block 的声明和赋值，而且这两行 `&shared` 写在同一个作用域里、字面完全相同。

我第一次跑这个实验时两次打印是同一个栈地址，以为 forwarding 根本没生效。问题在于 Block 还没被 copy 到堆——第 2 行的打印必须在 Block 已经赋给 strong 变量之后。上一篇讲过，ARC 只在特定上下文才做这次搬迁。

### Block_byref 与 forwarding

`__block` 修饰之后，编译器不再把值直接塞进 Block 结构体，而是先把变量包进一个独立的结构体：

```c
struct Block_byref {
    void * __ptrauth_objc_isa_pointer isa;
    struct Block_byref *forwarding;
    volatile int32_t flags;     // 含引用计数与 layout
    uint32_t size;
};

struct Block_byref_2 {          // 仅当 flags & BLOCK_BYREF_HAS_COPY_DISPOSE
    BlockByrefKeepFunction    byref_keep;
    BlockByrefDestroyFunction byref_destroy;
};

struct Block_byref_3 {          // 仅当 layout == BLOCK_BYREF_LAYOUT_EXTENDED
    const char *layout;
};
```

跟 Block 的 descriptor 一样，这也是三段拼接，后两段是否存在由 flags 决定。变量本体接在最后一段之后。实测能看出来：`__block int` 的 byref size 是 32（没有第二段），ARC 下 `__block id` 是 48（有第二段）。

Block 结构体里存的是指向这个 `Block_byref` 的指针。关键在第二个字段 `forwarding`——一个指向"自己"的指针。

`_Block_byref_copy` 里那两行把机制说完了：

```c
copy->forwarding = copy;  // 堆上那份指向自己
src->forwarding  = copy;  // 栈上那份改指向堆上副本
```

第一行保证 forwarding 永远只跳一层，不会跳成链。第二行是关键：栈上那份不删，但它的 forwarding 被改指向了新家。

而编译器把初始化之后的每一次读写都改写成了 `byref->forwarding->变量`。于是读写永远落在同一份数据上，跟调用方在哪无关。

（说"所有访问"就说满了。声明处的初始化是直接写字段的，不过 forwarding；销毁时 `_Block_object_dispose` 传的也是栈上那份。`runtime.cpp` 里还留着一句注释吐槽这事：`// dereference the forwarding pointer since the compiler isn't doing this anymore (ever?)`。）

画出来是这样：

```text
Block 创建前                      Block copy 到堆之后

  栈                                栈
┌──────────────┐                 ┌──────────────┐
│ Block_byref  │                 │ Block_byref  │
│ forwarding ──┼──┐              │ forwarding ──┼────┐
│ shared = 5   │  │              │ shared = 5   │    │  （残留，不再被读）
└──────────────┘  │              └──────────────┘    │
       ▲──────────┘                                   │
                                   堆                 ▼
                                 ┌──────────────┐   ┌──────────────┐
                                 │ Block        │   │ Block_byref  │
                                 │ ...          │   │ forwarding ──┼──┐
                                 │ byref 指针 ──┼──▶│ shared = 5   │  │
                                 └──────────────┘   └──────────────┘  │
                                                           ▲──────────┘
```

第 1 行的 `&shared` 走的是左图那条自环，拿到栈地址；第 2 行走的是右图，栈上那份的 forwarding 已经指向堆，于是拿到堆地址。

所以 `__block` 真正解决的是**生命周期**问题：Block 可能活得比创建它的栈帧更久，而栈上的变量会随栈帧消失。它把变量搬到堆上跟着 Block 走。"能修改"只是搬家之后的自然结果——理解成生命周期问题，才能解释 forwarding 为什么存在。

### 多个 Block 可以共享同一个 byref

```objc
__block int n = 1;
void (^a)(void) = ^{ n++; };
void (^b)(void) = ^{ NSLog(@"%d", n); };
a();  b();   // 打印 2
```

byref 有自己的引用计数（在 flags 低位），独立于任何一个 Block 存在。这正是它必须是个独立结构体、而不能塞进某个 Block 内部的原因。

---

## 四、`__block` 修饰对象：ARC 和 MRC 编出来的不是一回事

这是本篇最值得看的一组数据。

从 Block 结构体里掏出 byref 指针，读它的 flags 高 4 位（layout 字段）：

```text
--- ARC ---
  __block id                    flags=0x33000004 size=48  layout=STRONG      HAS_COPY_DISPOSE=1
  __block __weak id             flags=0x43000004 size=48  layout=WEAK        HAS_COPY_DISPOSE=1
  __block __unsafe_unretained   flags=0x51000004 size=32  layout=UNRETAINED  HAS_COPY_DISPOSE=0
  __block int                   flags=0x21000004 size=32  layout=NON_OBJECT  HAS_COPY_DISPOSE=0

--- MRC ---
  __block id                    flags=0x52000000 size=48  layout=UNRETAINED  HAS_COPY_DISPOSE=1
  __block int                   flags=0x20000000 size=32  layout=NON_OBJECT  HAS_COPY_DISPOSE=0
```

对比两边的第一行：**同一句 `__block id`，ARC 编出 `STRONG`，MRC 编出 `UNRETAINED`。**

这就是"`__block` 能打破循环引用"从正确变成过期的全部原因。MRC 下 byref 不持有对象，用 `__block` 修饰确实能断环；ARC 改成了 `STRONG`，照样持有，断不了。语义差别固化在这 4 个 bit 里，不是文档里一句模糊的约定。

ARC 下 `byref_keep` 实际做的事比"retain 一下"更细致：

```llvm
define internal void @__Block_byref_object_copy_(ptr %0, ptr %1) {
  call void @llvm.objc.storeStrong(ptr %6, ptr %9)    ; 堆上那份 = 栈上那份
  call void @llvm.objc.storeStrong(ptr %8, ptr null)  ; 栈上那份 = nil
}
```

是一次 **move** 而不是单纯 retain——所有权从栈 byref 挪到堆 byref。这解释了为什么栈上那份残留不会导致重复释放。

编译器自己也认这个语义。ARC 下写一个自引用的 `__block` Block：

```objc
__block void (^loop)(void);
loop = ^{ (void)loop; };
```

会直接报 `-Warc-retain-cycles`，note 原文是 `block will be retained by the captured object`。同样的代码在 MRC 下零告警。

---

## 五、`BLOCK_HAS_COPY_DISPOSE` 不等于"捕获了对象"

上一篇说这一位表示"捕获了需要 retain/release 的东西"，那是在那一篇的语境里的简化。放到这一篇要说准一点：它表示 **descriptor 里有 copy 和 dispose 两个辅助函数**，至于这两个函数做什么，分三种情况。

第四节那张表里，`__block int` 也置了这一位——一个对象都没有。它要处理的是 byref 从栈搬到堆这件事。

捕获 `__weak` 变量同样置位，但 copy helper 调的是 `objc_copyWeak` 而不是 retain。这个调用的成本在 [[iOS weak 的实现：SideTable 与置 nil 的时机]] 里量过：一次完整的 `loadWeakRetained` 加一次 `initWeak`，两次加锁两次查表。

只有 ARC 加强捕获对象这一格，copy helper 里才是 `objc_storeStrong`。

顺带一提，ARC 下强捕获对象走的是编译器生成的 copy helper，`_Block_object_assign` 一次都不会被调到；同一份代码换成 `-fno-objc-arc`，它就被调了两次。`runtime.cpp` 里有句注释正好对上：`// Note this is MRC unretained __block only. ARC retained __block is handled by the copy helper directly.`

---

## 六、捕获对象时 Block 会持有它

```objc
__weak id weakProbe = nil;
void (^hold)(void) = nil;
@autoreleasepool {
    NSObject *o = [NSObject new];
    weakProbe = o;
    hold = ^{ (void)o; };
}   // o 的局部强引用在这里失效
```

```text
  离开作用域后 weak = <NSObject: 0x600000014000>
  block 置 nil 后  weak = (null)
```

局部强引用已经消失，weak 探针还能看到对象，说明 Block 接管了这份所有权。这个持有关系就是循环引用的来源，[[iOS Block 循环引用与 weak-strong dance]] 专门讲。

---

## 七、几个说法需要辨析

**"Block 捕获 auto 变量是值捕获，捕获 `__block` 变量是引用捕获。"** 结论没错，但"引用捕获"会让人以为传的是个 C 指针。`forwarding` 的存在说明不是——简单取地址的话，Block 搬到堆上之后那个地址就悬空了。

**"全局变量和 static 变量会被 Block 捕获。"** 不会，三种都不会。第一节那组 size=32 就是证据，而且这类 Block 连堆都不上。

**"`__block` 修饰对象可以打破循环引用。"** MRC 下的结论。ARC 下 byref layout 是 `STRONG`，照样持有。第四节那两行对照是硬证据。

**"C 数组也能捕获。"** 不能，编译期就拦：`cannot refer to declaration with an array type inside block`。加 `__block` 也一样。结构体倒是可以正常按值捕获。

**"Block 里改了 `__block` 变量，外面要等 Block 执行完才能看到。"** 没有这回事。块内块外访问同一块内存，改完立刻可见。

---

## 总结

能不能在 Block 里改一个变量，取决于编译器往结构体里塞了什么。全局、文件内 static、函数内 static 三者都不进结构体，Block 按符号直接访问（这类 Block 甚至是全局 Block）；普通 auto 变量进的是一份创建时刻的值副本；`__block` 变量进的是一个指向 `Block_byref` 的指针。

`__block` 解决的是生命周期而不是可写性。变量被搬进一个带 `forwarding` 的结构体跟着 Block 走，栈上那份的 forwarding 被改指向堆上的新家，于是所有访问路径都跟着过去。

最值得记住的是 ARC 和 MRC 的语义差：同一行 `__block id`，byref layout 一个是 `STRONG` 一个是 `UNRETAINED`。"`__block` 能打破循环引用"这条老知识就死在这 4 个 bit 上。

## 参考资料

### 规范与源码

- [Clang — Block Implementation Specification](https://clang.llvm.org/docs/Block-ABI-Apple.html)：`__block` storage 与 forwarding pointer 的定义
- [Apple — Blocks and Variables](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Blocks/Articles/bxVariables.html)：按变量类别分的官方版本
- [apple-oss-distributions/libclosure](https://github.com/apple-oss-distributions/libclosure)：`Block_private.h` 里的 `Block_byref` 三段定义与 `BLOCK_BYREF_LAYOUT_*` 枚举，`runtime.cpp` 里是 `_Block_byref_copy` 和 `_Block_object_assign`

### 拓展

- [halfrost — 深入研究 Block 捕获外部变量和 __block 实现原理](https://halfrost.com/ios_block/)：捕获规则和 `__forwarding` 讲得系统
- [Desgard — 浅谈 block（2）：截获变量方式](https://github.com/Desgard/iOS-Source-Probe/tree/master/Objective-C/Runtime)：含 `-rewrite-objc` 的真实产物。注意那个命令在现在的 Xcode 上已经无法执行，而且第一节那个"函数内 static 被捕获地址"的说法就是从它的产物来的

### 本地

- [[iOS Block 的结构：ABI、descriptor 与三种类型]]
- [[iOS Block 循环引用与 weak-strong dance]]
- [[iOS weak 的实现：SideTable 与置 nil 的时机]]

---

实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -target arm64-apple-ios17.0-simulator`，ARC 与 MRC 两份分别构建。所有输出块都是真实运行结果。

第四节那组 byref flags 的读法：byref 指针就在 Block 定长部分（32 字节）之后，照着 `Block_private.h` 手写一份布局一致的结构体强转过去读即可。flags 的高 4 位是 layout，`BLOCK_BYREF_LAYOUT_STRONG` 是 3、`WEAK` 是 4、`UNRETAINED` 是 5、`NON_OBJECT` 是 2。

这些 flags 位是实现细节，可能随版本变化。稳定的是"ARC 与 MRC 对 `__block` 对象的所有权语义不同"这个事实本身——真要验证，换个 SDK 重跑一遍这段代码就行。
