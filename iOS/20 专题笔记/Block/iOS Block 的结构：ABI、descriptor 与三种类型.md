---
title: 【iOS】Block 的结构：ABI、descriptor 与三种类型
published: 2026-07-26
description: 讲 Block 的中文教程八成建立在 clang -rewrite-objc 上，而这个命令在现在的 Xcode 里已经跑不起来了。换一套办法：直接读 Block_layout 的内存，顺便发现「ARC 下看不到栈 block」这句话其实是测量姿势的问题。
tags:
  - iOS
  - Objective-C
  - Block
  - Runtime
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 8
draft: true
---
# Block 的结构：ABI、descriptor 与三种类型

先说一件影响这篇怎么写的事。

中文圈讲 Block 底层的文章，八成以上是这个套路：写一段带 Block 的代码，跑 `clang -rewrite-objc` 把它改写成 C++，然后逐行解释生成的 `__main_block_impl_0` 结构体。这个套路很好用——直到你在今天的 Xcode 上试一次：

```text
$ clang -rewrite-objc simple.m -o simple.cpp
error: action RewriteObjC not compiled in
```

Apple 出货的 clang 已经把这个 frontend action 整个移除了。不是"输出可能不准"，是根本无法执行。那些教程里的转写代码仍然有参考价值（它确实反映了 Block 的大致形态），但你没法自己复现，也没法验证它在当前编译器上是否还成立。

所以这篇换一套办法：照着 libclosure 的头文件手写一份布局一致的结构体，把 Block 强转过去直接读内存；调用路径去看 LLVM IR；类型判断直接跑起来打印。这三样都能在你自己机器上复现。

而这个换法本身带来了一个意外——它让我发现"ARC 下看不到栈 Block"这句流传很广的话，其实是个测量姿势问题。这部分放在第五节。

---

## 一、Block 是个货真价实的对象

先确认最基础的一点：

```objc
NSLog(@"是 NSObject 吗 %d   superclass=%@",
      [blk isKindOfClass:[NSObject class]], [[blk class] superclass]);
```

```text
是 NSObject 吗 1   superclass=NSBlock
```

它有 isa，能响应 `class`、`isKindOfClass:`、`copy`、`retain`/`release`。继承链是 `__NSMallocBlock__ → NSBlock → NSObject`。

"Block 是特殊语法糖，不是真正的 OC 对象"这个说法可以直接丢掉。它是对象，只是它的 isa 由编译器和 libclosure 一起管，而不是走普通的类注册流程。

---

## 二、Block_layout：文档和实现不是一回事

Clang 的 Block ABI 文档给的结构体长这样：

```c
struct Block_literal_1 {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 {
        unsigned long int reserved;
        unsigned long int size;
        void (*copy_helper)(void *dst, void *src);   // IFF (1<<25)
        void (*dispose_helper)(void *src);            // IFF (1<<25)
        const char *signature;                        // IFF (1<<30)
    } *descriptor;
    // imported variables
};
```

看起来 descriptor 是一个固定形状的结构体，某些字段"有没有值"取决于 flags。

但 libclosure 的真实实现不是这样。它把 descriptor 拆成了三段，谁存在完全由 flags 决定，而且是**依次拼接在内存里**：

```c
struct Block_descriptor_1 {
    uintptr_t reserved;
    uintptr_t size;
};

struct Block_descriptor_2 {
    // requires BLOCK_HAS_COPY_DISPOSE
    BlockCopyFunction copy;
    BlockDisposeFunction dispose;
};

struct Block_descriptor_3 {
    // requires BLOCK_HAS_SIGNATURE
    const char *signature;
    const char *layout;
};

struct Block_layout {
    void * __ptrauth_objc_isa_pointer isa;
    volatile int32_t flags;
    int32_t reserved;
    BlockInvokeFunction invoke;
    struct Block_descriptor_1 *descriptor;
    // imported variables
};
```

这个差别不是学术问题。想在运行期读出一个 Block 的签名，你**不能**按文档那个结构体去取字段，必须按 flags 手算偏移。Aspects 这类 AOP 库就是这么干的：

```c
void *desc = layout->descriptor;
desc += 2 * sizeof(unsigned long int);          // 跳过 Block_descriptor_1
if (layout->flags & AspectBlockFlagsHasCopyDisposeHelpers) {
    desc += 2 * sizeof(void *);                  // 仅当存在时跳过 Block_descriptor_2
}
const char *signature = (*(const char **)desc);  // 此时才是 Block_descriptor_3.signature
```

如果 descriptor 真是文档里那种固定结构体，根本不需要这样手算。这段生产代码是拆分式布局最好的第三方实证。

---

## 三、捕获的变量直接追加在结构体后面

`Block_layout` 的定长部分是 isa(8) + flags(4) + reserved(4) + invoke(8) + descriptor(8) = 32 字节。注释里那句 `// imported variables` 说的是捕获的变量按声明顺序接在后面。

读一下 `descriptor->size` 就能验证：

```text
无捕获        size=32
捕获 int      size=36
捕获对象      size=40
```

32 是空的定长部分，加一个 `int` 变 36，加一个对象指针变 40。捕获的变量确实就是普通字段，顺序追加。

顺带一提，这也解释了为什么 Block 能"记住"外部变量的值：它在创建的那一刻把值拷进了自己的结构体里。这跟"闭包捕获环境"的抽象说法比起来，实在得多。

---

## 四、flags 里有什么

libclosure 的完整枚举比 Clang 文档多几位：

```c
enum {
    BLOCK_DEALLOCATING        = (0x0001),      // runtime
    BLOCK_REFCOUNT_MASK       = (0xfffe),      // runtime，低 16 位是引用计数
    BLOCK_INLINE_LAYOUT_STRING = (1 << 21),    // 文档未收录
    BLOCK_SMALL_DESCRIPTOR    = (1 << 22),     // 文档未收录
    BLOCK_IS_NOESCAPE         = (1 << 23),
    BLOCK_NEEDS_FREE          = (1 << 24),     // runtime，是否堆分配
    BLOCK_HAS_COPY_DISPOSE    = (1 << 25),
    BLOCK_HAS_CTOR            = (1 << 26),
    BLOCK_IS_GC               = (1 << 27),     // GC 时代遗留
    BLOCK_IS_GLOBAL           = (1 << 28),
    BLOCK_USE_STRET           = (1 << 29),
    BLOCK_HAS_SIGNATURE       = (1 << 30),
    BLOCK_HAS_EXTENDED_LAYOUT = (1 << 31),     // 文档未收录
};
```

低 16 位是引用计数，这一点很多文章只字不提，导致读者以为 flags 纯粹是类型标记。实测：

```text
无捕获     class=__NSGlobalBlock__  flags=0x50000000  refcount=0
                IS_GLOBAL HAS_SIGNATURE
捕获 int   class=__NSMallocBlock__  flags=0xc1000006  refcount=3
                NEEDS_FREE HAS_SIGNATURE HAS_EXTENDED_LAYOUT
捕获对象   class=__NSMallocBlock__  flags=0xc3000006  refcount=3
                HAS_COPY_DISPOSE NEEDS_FREE HAS_SIGNATURE HAS_EXTENDED_LAYOUT
```

三行的差别很说明问题。

全局 Block 的 refcount 是 0，而且没有 `NEEDS_FREE`——它住在数据段里，不需要引用计数，也不需要释放。

捕获 `int` 的 Block 有 `NEEDS_FREE`（在堆上）但**没有** `HAS_COPY_DISPOSE`。因为一个 `int` 不需要在 Block 被拷贝时做任何额外处理，直接跟着结构体一起 memcpy 就行。

捕获对象的 Block 多了 `HAS_COPY_DISPOSE`。这一位的意思是"descriptor 里有 copy 和 dispose 两个辅助函数"，它们负责在 Block 从栈搬到堆时 retain 捕获的对象、在 Block 销毁时 release 它。**捕获对象要多付的成本，就编码在这一位里。**

`BLOCK_USE_STRET`（1<<29）在 Clang 文档里叫 `BLOCK_HAS_STRET`，而文档自己都说它 "was usually set and was always ignored by the runtime - it had been a transitional marker"。别拿它判断返回值类型。

---

## 五、三种类型，以及一个测量假象

### 先看常见的做法

```objc
void (^blk1)(void) = ^{ NSLog(@"无捕获"); };
void (^blk2)(void) = ^{ NSLog(@"%d", g); };      // g 是全局变量
int age = 10;
void (^blk3)(void) = ^{ NSLog(@"%d", age); };    // 捕获 auto 变量
void (^blk3copy)(void) = [blk3 copy];
```

同一份代码，两种编译模式：

```text
--- ARC ---
无捕获        __NSGlobalBlock__
捕获全局变量  __NSGlobalBlock__
捕获auto变量  __NSMallocBlock__
再 copy 一次  __NSMallocBlock__

--- MRC ---
无捕获        __NSGlobalBlock__
捕获全局变量  __NSGlobalBlock__
捕获auto变量  __NSStackBlock__
再 copy 一次  __NSMallocBlock__
```

差别只在第三行。中文教程里被反复传抄的那句"访问了 auto 变量的 block 就是 `__NSStackBlock__`"，只在 MRC 下成立；ARC 下它已经是堆 Block 了。

有意思的是，很多文章的行文本身就暴露了这个矛盾——先写下"访问了 auto 变量就是栈 block"这个结论，紧接着贴的 ARC 控制台输出却是 `__NSMallocBlock__`，然后作者切到 MRC 重跑一次才验证成功。这恰恰说明那个"结论"的默认语境是 MRC。

于是又流传出第二个说法：ARC 下根本看不到栈 Block。

### 这句话也不对，而且我先踩了坑

我一开始也是这么测的：写一个函数把 Block 传进去，在函数里读它的 flags。

```objc
static void show(const char *tag, id blk) { /* 读 flags */ }
static void withNoEscape(__attribute__((noescape)) void (^b)(void)) { show("带 noescape", b); }
```

结果全是 `__NSMallocBlock__`，`NEEDS_FREE` 置位，连标了 `noescape` 的参数也一样。当时我准备写"ARC 下确实看不到栈 Block"。

问题出在 `show` 的参数类型上。它是 `id`——而 ARC 在把一个 Block 传给 `id` 类型参数时，会插入一次 copy。**观察这个动作本身把栈 Block 搬到了堆上。**

把参数类型换成不触发 copy 的形式再测：

```objc
struct BL { void *isa; volatile int32_t flags; int32_t r; void (*invoke)(void*,...); void *d; };

// 关键：参数不是 id，ARC 不会在传参时插入 copy
static void show(const char *t, struct BL *p) { /* 读 flags */ }

static void withNE(__attribute__((noescape)) void (^b)(void)) { show("带 noescape", (__bridge struct BL *)b); }
static void without(void (^b)(void))                          { show("不带",        (__bridge struct BL *)b); }
```

```text
带 noescape   isa=__NSStackBlock__     flags=0xc0000000  NEEDS_FREE=0
不带          isa=__NSStackBlock__     flags=0xc0000000  NEEDS_FREE=0
赋给 strong   isa=__NSMallocBlock__    flags=0xc1000002  NEEDS_FREE=1
```

ARC 下的栈 Block 就在那儿，一直都在。

准确的说法是：**ARC 会在特定上下文把 Block 从栈搬到堆**——赋值给 `__strong` 变量、作为返回值、传给 `id` 类型参数、作为 GCD 或 `...UsingBlock:` 这类 API 的参数。而"把一个 Block 字面量直接当参数传给一个类型明确的 Block 参数"不在此列，所以它保持在栈上。

问题是日常写代码时，`void (^block)(void) = ^{...}` 这个最常见的写法恰好命中了第一条。所以"看不到"是真的常见，"看不见"不是。

我把这个坑完整写出来，是因为它跟这个系列前几篇撞见的错误是同一类：**测量手段干扰了被测对象**。上一篇讲 weak 时也有类似的事——你没法在不 retain 的前提下读一个 weak 变量。遇到"结果和预期不符"，先怀疑仪器。

顺带一个诚实的负面结果：`BLOCK_IS_NOESCAPE` 这一位在我所有测试里都没有置位过，带不带 `__attribute__((noescape))` 打出来的 flags 完全一样。我没有找到让它置位的写法，也不打算猜测机制——这一位在当前编译器上什么时候会被设，我不知道。

---

## 六、block() 是怎么调到 invoke 的

```objc
int callBlock(int (^blk)(int)) { return blk(42); }
```

`-O0` 的 IR：

```llvm
define i32 @callBlock(ptr noundef %0) #0 {
  ...
  %3 = load ptr, ptr %2, align 8
  %4 = getelementptr inbounds nuw %struct.__block_literal_generic, ptr %3, i32 0, i32 3
  %5 = load ptr, ptr %4, align 8
  %6 = call i32 %5(ptr noundef %3, i32 noundef 42)
  ...
}
```

字段下标从 0 数：`0=isa, 1=flags, 2=reserved, 3=invoke, 4=descriptor`。所以 `getelementptr ... i32 3` 取的正是 `invoke` 函数指针，然后间接调用它，**把 Block 自身作为第一个实参传进去**，原始参数依次追加。

翻译成 C 就是：

```c
((int (*)(void *, int))((struct Block_layout *)blk)->invoke)(blk, 42);
```

Block 自身当第一参数这件事，和 Objective-C 方法把 `self` 当第一参数是同一个套路。invoke 函数体里访问捕获的变量，靠的就是从这个指针往后偏移——所以捕获的变量必须紧跟在定长部分后面，第三节那组 size 数据也就有了解释。

---

## 七、几个说法需要辨析

**"Block 是语法糖，不是真正的对象。"** 它有 isa，`isKindOfClass:[NSObject class]` 返回 YES，继承链是 `__NSMallocBlock__ → NSBlock → NSObject`。

**"访问了 auto 变量的 Block 就是 `__NSStackBlock__`。"** 只在 MRC 下成立。ARC 下只要赋值给 strong 变量就已经是 `__NSMallocBlock__` 了。

**"ARC 下看不到栈 Block。"** 看得到，只是别用会触发 copy 的方式去观察。见第五节。

**"`copy` 一个 Block 就是深拷贝一份内存。"** 不一定。`_Block_copy` 的逻辑是：已经在堆上（`BLOCK_NEEDS_FREE`）就只把引用计数加一直接返回；是全局 Block 就原样返回；只有确实是栈 Block 时才 `malloc` + `memmove`。多数情况下 `copy` 只是一次引用计数递增。

**"MRC 下 Block 会自动持有捕获的对象。"** 反了。MRC 下捕获普通 `id` 是纯粹的指针拷贝，不 retain；ARC 才会自动 retain。这一点很多文章讲反或者语焉不详。

**"`clang -rewrite-objc` 能看到编译器真实生成的代码。"** 两个问题：它走的是一套独立的教学用改写逻辑，不是真实 codegen 的旁路输出，两者结论可能不一致；而且它在现代 Xcode 上已经无法执行。想验证真实行为，用 `-S -emit-llvm` 看 IR，或者直接跑起来打印。

---

## 总结

Block 是个对象，结构是"32 字节定长部分 + 捕获的变量顺序追加"，调用 `block(args)` 就是取出第 4 个字段 `invoke` 然后把 Block 自己当第一参数传进去。

三种类型的判断依据是 flags：`BLOCK_IS_GLOBAL` 表示在数据段，`BLOCK_NEEDS_FREE` 表示在堆上，两个都没有就在栈上。`BLOCK_HAS_COPY_DISPOSE` 单独标记"捕获了需要 retain/release 的东西"，捕获 `int` 不置位，捕获对象才置位。

关于三种类型的流行结论大多默认语境是 MRC。ARC 下你依然能观察到栈 Block，只要别用 `id` 参数或 strong 变量去接它——那个动作本身就会把它搬到堆上。

最后一条方法论，跟前几篇一样：这一篇里两个"网上说的不对"，一个是编译一次就发现的（`-rewrite-objc` 跑不了），一个是换个参数类型就发现的（测量假象）。**遇到结果和预期不符，先怀疑仪器，再怀疑结论。**

下一篇讲捕获的细节：[[iOS Block 的变量捕获与 __block]]。

## 参考资料

### 规范与源码

- [Clang — Block Implementation Specification](https://clang.llvm.org/docs/Block-ABI-Apple.html)：最权威，但 descriptor 部分是简化版，别照着它做指针运算
- [apple-oss-distributions/libclosure](https://github.com/apple-oss-distributions/libclosure)：`Block_private.h` 里是真实的三段式 descriptor 和完整 flags 枚举，`runtime.cpp` 里是 `_Block_copy` 的实现
- [Apple — Blocks Programming Topics](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html)：概念导论，底层细节没有

### 第三方实证

- [steipete/Aspects — Aspects.m](https://github.com/steipete/Aspects/blob/master/Aspects.m)：`aspect_blockMethodSignature` 是三段式 descriptor 在生产代码里的活证据

### 拓展

- [Desgard — 浅谈 block](https://github.com/Desgard/iOS-Source-Probe/tree/master/Objective-C/Runtime)：中文资料里质量较高的一份，含真实的 `-rewrite-objc` 产物，而且作者自己指出了这个工具的局限。注意那些转写代码今天已经无法自己复现
- [halfrost — 深入研究 Block 捕获外部变量和 __block 实现原理](https://halfrost.com/ios_block/)：捕获规则和 `__forwarding` 讲得系统

### 本地

- [[iOS 属性关键字：从所有权推导，而不是从类型名猜]]
- [[iOS weak 的实现：SideTable 与置 nil 的时机]]

---

实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -target arm64-apple-ios17.0-simulator`，ARC 与 MRC 两份分别构建。所有输出块都是真实运行结果。

复现方式：手写一份和 `Block_private.h` 布局一致的结构体，把 Block 用 `__bridge` 强转过去读字段。**注意别把中转函数的参数类型写成 `id`**，否则 ARC 会插入一次 copy，你测到的就不是原来那个 Block 了——第五节整节都在讲这个。

flags 的位定义和 Foundation 的内部类名（`__NSMallocBlock__` 等）属于实现细节，可能随版本变化。稳定的是"三种存储位置"和"捕获的变量顺序追加在结构体后面"这两层语义。
