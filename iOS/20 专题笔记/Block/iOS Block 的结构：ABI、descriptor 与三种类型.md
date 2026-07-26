---
title: 【iOS】Block 的结构：ABI、descriptor 与三种类型
published: 2026-07-26
description: 讲 Block 底层的中文教程八成建立在 clang -rewrite-objc 上，而这个命令在现在的 Xcode 里已经跑不起来了。换一套办法之后，顺手发现「ARC 下看不到栈 block」其实是测量姿势的问题。
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

中文圈讲 Block 底层的文章，八成以上是同一个套路：写一段带 Block 的代码，跑 `clang -rewrite-objc` 把它改写成 C++，然后逐行解释生成的 `__main_block_impl_0` 结构体。这个套路很好用——直到你在今天的 Xcode 上试一次：

```text
$ clang -rewrite-objc simple.m -o simple.cpp
error: action RewriteObjC not compiled in
```

就这样。不是版本问题，是这个 frontend action 被整个移除了。

那些教程里的转写代码仍有参考价值，但你没法自己复现，也没法验证它在当前编译器上是否还成立。所以这篇换一套办法：照着 libclosure 的头文件手写一份布局一致的结构体，把 Block 强转过去直接读内存；调用路径去看 LLVM IR；类型判断直接跑起来打印。三样都能在你自己机器上复现。

这个换法带来了一个意外——它让我发现"ARC 下看不到栈 Block"这句话，其实是个测量姿势问题。那部分放在第五节。

---

## 一、Block 确实是对象

```objc
NSLog(@"是 NSObject 吗 %d   superclass=%@",
      [blk isKindOfClass:[NSObject class]], [[blk class] superclass]);
```

```text
是 NSObject 吗 1   superclass=NSBlock
```

它有 isa，能响应 `class`、`isKindOfClass:`、`copy`、`retain`/`release`。继承链是三级：`__NSMallocBlock__ → NSBlock → NSObject`，`__NSGlobalBlock__` 也一样。

"Block 是特殊语法糖，不是真正的 OC 对象"这个说法可以丢掉。

---

## 二、Block_layout：文档和实现不是一回事

Clang 的 Block ABI 文档给的结构体是这样：

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

看起来 descriptor 是一个固定形状的结构体，某些字段有没有值取决于 flags。

libclosure 的真实实现不是这样。它把 descriptor 拆成三段，谁存在完全由 flags 决定，而且是依次拼接在内存里：

```c
struct Block_descriptor_1 { uintptr_t reserved; uintptr_t size; };

struct Block_descriptor_2 {          // requires BLOCK_HAS_COPY_DISPOSE
    BlockCopyFunction copy;
    BlockDisposeFunction dispose;
};

struct Block_descriptor_3 {          // requires BLOCK_HAS_SIGNATURE
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

这个差别不是学术问题。想在运行期读出一个 Block 的签名，你不能按文档那个结构体取字段，得按 flags 手算偏移。libclosure 自己就是这么干的：

```c
static struct Block_descriptor_3 * _Block_descriptor_3(struct Block_layout *aBlock) {
    uint8_t *desc = (uint8_t *)_Block_get_descriptor(aBlock);
    desc += sizeof(struct Block_descriptor_1);
    if (aBlock->flags & BLOCK_HAS_COPY_DISPOSE) desc += sizeof(struct Block_descriptor_2);
    return (struct Block_descriptor_3 *)desc;
}
```

Aspects 这类 AOP 库做的是同一件事，只是它不敢用 SPI，得自己把这段抄一遍：

```c
if (!(layout->flags & AspectBlockFlagsHasSignature)) return nil;   // 先判有没有签名
void *desc = layout->descriptor;
desc += 2 * sizeof(unsigned long int);
if (layout->flags & AspectBlockFlagsHasCopyDisposeHelpers) {
    desc += 2 * sizeof(void *);
}
const char *signature = (*(const char **)desc);
```

有意思的是 Aspects 自己也声明了一个扁平的 struct，六个字段包括 `layout`，却全程不用它取值、坚持手算偏移。作者显然知道那个扁平声明只是给编译器看的，不是内存里的真实形状。

顺带说，`Block_private.h` 其实直接 `BLOCK_EXPORT` 了 `_Block_signature`、`_Block_extended_layout`、`Block_size`、`_Block_use_stret` 这几个函数，今天在 libSystem 里 `extern` 一下就能调。上架代码不敢用 SPI 是另一回事。

---

## 三、捕获的变量直接追加在结构体后面

定长部分是 isa(8) + flags(4) + reserved(4) + invoke(8) + descriptor(8) = 32 字节。读 `descriptor->size` 验证：

```text
无捕获        size=32
捕获 int      size=36
捕获对象      size=40
```

加一个 `int` 变 36，加一个对象指针变 40。捕获的变量就是普通字段，顺序追加。

不过别以为是简单相加。同时捕获一个 `NSString *` 和一个 `int` 时，size 是 44——32 + 8 + 4，指针要先对齐到 8 字节边界。跟 [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer#ivar 重排|ivar 的对齐规则]]是同一套。

这也解释了 Block 为什么能"记住"外部变量的值：它在创建的那一刻把值拷进了自己的结构体。

---

## 四、flags 里有什么

libclosure 的完整枚举比 Clang 文档多几位，而且每一位都标了是编译期烧死的还是运行期改的：

```c
enum {
    BLOCK_DEALLOCATING         = (0x0001),   // runtime
    BLOCK_REFCOUNT_MASK        = (0xfffe),   // runtime
    BLOCK_INLINE_LAYOUT_STRING = (1 << 21),  // compiler（文档未收录）
    BLOCK_SMALL_DESCRIPTOR     = (1 << 22),  // compiler（文档未收录，且当前未启用）
    BLOCK_IS_NOESCAPE          = (1 << 23),  // compiler
    BLOCK_NEEDS_FREE           = (1 << 24),  // runtime
    BLOCK_HAS_COPY_DISPOSE     = (1 << 25),  // compiler
    BLOCK_HAS_CTOR             = (1 << 26),  // compiler: helpers have C++ code
    BLOCK_IS_GC                = (1 << 27),  // runtime
    BLOCK_IS_GLOBAL            = (1 << 28),  // compiler
    BLOCK_USE_STRET            = (1 << 29),  // compiler: undefined if !BLOCK_HAS_SIGNATURE
    BLOCK_HAS_SIGNATURE        = (1 << 30),  // compiler
    BLOCK_HAS_EXTENDED_LAYOUT  = (1u << 31), // compiler（文档未收录）
};
```

compiler / runtime 这一列是理解整个字段的关键：一个 32 位的 flags 里同时装着编译期的类型信息和运行期的引用计数，靠的就是把位分成两拨。

实测：

| Block | class | flags | refcount | 置位的标志 |
| --- | --- | --- | --- | --- |
| 无捕获 | `__NSGlobalBlock__` | `0x50000000` | 0 | IS_GLOBAL, HAS_SIGNATURE |
| 捕获 int | `__NSMallocBlock__` | `0xc1000006` | 3 | NEEDS_FREE, HAS_SIGNATURE, HAS_EXTENDED_LAYOUT |
| 捕获对象 | `__NSMallocBlock__` | `0xc3000006` | 3 | HAS_COPY_DISPOSE, NEEDS_FREE, HAS_SIGNATURE, HAS_EXTENDED_LAYOUT |

全局 Block 的 refcount 是 0，也没有 `NEEDS_FREE`。它住在 `__DATA_CONST,__const`——不是 `__TEXT`，也不是普通的 `__DATA`，因为 isa 要在启动时被 dyld 绑到 `_NSConcreteGlobalBlock`，绑完页面才转只读。`nm -m` 一眼能看到：

```text
(__DATA_CONST,__const) non-external ___block_literal_global
```

### 引用计数就藏在 flags 里

低位那两个常量值得单独算一遍。`BLOCK_REFCOUNT_MASK` 是 `0xfffe`，也就是 bit1 到 bit15，bit0 让给了 `BLOCK_DEALLOCATING`。而且计数是**左移一位存的**——`_Block_copy` 里初始化时写的是 `flags |= BLOCK_NEEDS_FREE | 2`，注释说这个 2 表示逻辑引用计数 1。

所以取值要 `(flags & 0xfffe) >> 1`。上表第二行 `0xc1000006`，低 16 位是 6，右移一位得 3。

腾出 bit0 是有理由的，`_Block_release` 里那次 CAS 要同时干两件事：

```c
if ((old_value & (BLOCK_REFCOUNT_MASK|BLOCK_DEALLOCATING)) == 2) {
    new_value = old_value - 1;   // 计数归零的同时置上 DEALLOCATING
    result = true;
}
```

一次原子操作把计数清零并标记正在析构，保证 `_Block_tryRetain` 不会复活一个已经在释放路上的 Block。这跟 [[iOS weak 的实现：SideTable 与置 nil 的时机|weak 那边靠 tryRetain 挡住正在析构的对象]]是同一个思路。

### HAS_COPY_DISPOSE 标的不是"捕获了对象"

上表第二第三行的差别值得看。捕获 `int` 的 Block 有 `NEEDS_FREE` 但没有 `HAS_COPY_DISPOSE`——一个 `int` 不需要在 Block 拷贝时做任何额外处理，跟着结构体一起 memcpy 就行。捕获对象的才置位。

这一位准确的含义是"descriptor 里有 copy 和 dispose 两个辅助函数"，至于这两个函数做什么，取决于捕获的是什么。这一点在 [[iOS Block 的变量捕获与 __block]] 里展开——`__block int` 也会置位，它要处理的是别的事。

### BLOCK_USE_STRET 要配对读

Clang 文档说这一位 "was usually set and was always ignored by the runtime - it had been a transitional marker"。很多文章引到这里就停了，得出"这一位没用"的结论。

文档下一句话是它现在跟 `(1<<30)` 配成了一对：`case (3<<29)` 才表示"stret 调用约定 + 有签名字段"。libclosure 的 SPI 也是这么判的：

```c
bool _Block_use_stret(void *aBlock) {
    int requiredFlags = BLOCK_HAS_SIGNATURE | BLOCK_USE_STRET;
    return (layout->flags & requiredFlags) == requiredFlags;
}
```

所以准确的说法是：单看这一位没意义，必须和 `BLOCK_HAS_SIGNATURE` 配对。前面那句引文说的是 10.6 那一代 ABI 的情况。

### BLOCK_SMALL_DESCRIPTOR 目前不会出现

源码里这一位整段被 `#if BLOCK_SMALL_DESCRIPTOR_SUPPORTED` 包着，而仓库里没有任何地方 define 这个宏。紧邻的注释：

```c
// Note: small block descriptors are not currently supported anywhere.
// Don't enable this without security review; see rdar://91727169
```

设计意图是把 descriptor 里的指针全换成 32 位相对偏移，省二进制体积和 rebase 开销——跟近年 Swift 和 Objective-C runtime 普遍改用相对指针是同一个趋势。但它没启用，所以你今天读到的 descriptor 一定是大描述符。

---

## 五、三种类型，以及一个测量假象

同一份代码，两种编译模式：

```objc
void (^blk1)(void) = ^{ NSLog(@"无捕获"); };
void (^blk2)(void) = ^{ NSLog(@"%d", g); };      // g 是全局变量
int age = 10;
void (^blk3)(void) = ^{ NSLog(@"%d", age); };    // 捕获 auto 变量
void (^blk3copy)(void) = [blk3 copy];
```

```text
--- ARC ---                      --- MRC ---
无捕获        __NSGlobalBlock__   无捕获        __NSGlobalBlock__
捕获全局变量  __NSGlobalBlock__   捕获全局变量  __NSGlobalBlock__
捕获auto变量  __NSMallocBlock__   捕获auto变量  __NSStackBlock__
再 copy 一次  __NSMallocBlock__   再 copy 一次  __NSMallocBlock__
```

差别只在第三行。中文教程里反复传抄的那句"访问了 auto 变量的 block 就是 `__NSStackBlock__`"，只在 MRC 下成立。

这个矛盾在很多文章的行文里是明摆着的：先写下"访问了 auto 变量就是栈 block"这个结论，紧接着贴的 ARC 控制台输出却是 `__NSMallocBlock__`，然后作者切到 MRC 重跑一次才验证成功。

于是又流传出第二个说法：ARC 下根本看不到栈 Block。

### 这句话也不对，而且我先踩了坑

我一开始也是这么测的：写一个函数把 Block 传进去，在函数里读它的 flags。

```objc
static void show(const char *tag, id blk) { /* 读 flags */ }
```

结果全是 `__NSMallocBlock__`，`NEEDS_FREE` 置位。当时我准备写"ARC 下确实看不到栈 Block"。

问题出在参数类型上。它是 `id`——ARC 规范对这种情况有明文规定：

> When a block pointer type is converted to a non-block pointer type (such as `id`), `Block_copy` is called. This is necessary because a block allocated on the stack won't get copied to the heap when the non-block pointer escapes.

观察这个动作本身把栈 Block 搬到了堆上。

把参数类型换成不触发 copy 的形式再测：

```objc
struct BL { void *isa; volatile int32_t flags; int32_t r; void (*invoke)(void*,...); void *d; };

static void show(const char *t, struct BL *p) { /* 读 flags */ }
static void without(void (^b)(void)) { show("不带", (__bridge struct BL *)b); }
```

```text
不带          isa=__NSStackBlock__     flags=0xc0000000  NEEDS_FREE=0
赋给 strong   isa=__NSMallocBlock__    flags=0xc1000002  NEEDS_FREE=1
```

ARC 下的栈 Block 就在那儿，一直都在。

为什么传给一个类型明确的 Block 参数不会触发 copy，ARC 规范里也写了，而且是一条显式的例外：

> **With the exception of retains done as part of initializing a `__strong` parameter variable** or reading a `__weak` variable, whenever these semantics call for retaining a value of block-pointer type, it has the effect of a `Block_copy`.

初始化 `__strong` 形参时的那次 retain 被排除在"等价于 Block_copy"之外，所以被调方拿到的仍是栈上那份。

ARC 会做这次搬迁的场合是：

- 赋值给 `__strong` 变量
- 作为函数返回值
- 转换成 `id` 或其他非 block 指针类型

还有一类堆化不是 ARC 干的。`dispatch_async` 这类 API 的形参本身就是 block 类型，调用点看到的仍是栈 block；真正的 `Block_copy` 是 libdispatch 在自己函数体里调的，属于运行期库行为。把它算进 ARC 的清单会和上面那条例外打架。

问题是日常写代码时，`void (^block)(void) = ^{...}` 这个最常见的写法恰好命中了第一条。所以"看不到"是真的常见，"看不见"不是。

### 一个我以为是自己测错的位

`BLOCK_IS_NOESCAPE` 在我所有测试里都没置位过，带不带 `__attribute__((noescape))` 打出来的 flags 完全一样。我一度打算就写"我不知道它什么时候会被设"。

答案在我自己列在参考资料第一位的那份文档里，enum 上方三行：

> Set to true on blocks that have captures (and thus are not true global blocks) but are known not to escape for various other reasons. For backward compatibility with old runtimes, whenever `BLOCK_IS_NOESCAPE` is set, `BLOCK_IS_GLOBAL` is set too. Copying a non-escaping block returns the original block and releasing such a block is a no-op.

触发条件是纯编译期的两步。Sema 那边要求**实参本身就是一个 block 字面量**（传一个已经存在的变量不算，我原来的写法天然命中不了），CodeGen 那边把 isa 设成 `_NSConcreteGlobalBlock`、置上 `BLOCK_IS_NOESCAPE | BLOCK_IS_GLOBAL`，并且不生成 copy/dispose helper。

但改成字面量直传还是测不到。把上游 clang 自带的回归测试 `clang/test/CodeGenObjC/noescape.m` 原样跑一遍就明白了：

```text
上游期望：  _NSConcreteGlobalBlock   flags = 0xD0800000   无 copy helper
Apple 实测：_NSConcreteStackBlock    flags = 0xC0000000   照常生成 copy helper
```

**Apple 出货的 clang 没有实现这条 codegen。** 不是我测错了，是这个优化在这个工具链上不存在。

顺着这条线还能把 `noescape` 的定位说准：它是一个**参数属性、编译期契约**，约束的是被调方不许让 block 逃逸出这次调用（文档里明确写着 `g1 = Block_copy(block); // Not OK either.`），违反就是未定义行为。`BLOCK_IS_NOESCAPE` 只是这个契约允许的一项优化的副产品，从来不是一个运行期状态位。契约在，优化不一定在。

---

## 六、block() 是怎么调到 invoke 的

```objc
int callBlock(int (^blk)(int)) { return blk(42); }
```

```llvm
%4 = getelementptr inbounds nuw %struct.__block_literal_generic, ptr %3, i32 0, i32 3
%5 = load ptr, ptr %4, align 8
%6 = call i32 %5(ptr noundef %3, i32 noundef 42)
```

字段下标从 0 数：`0=isa, 1=flags, 2=reserved, 3=invoke, 4=descriptor`。所以取的正是 `invoke`，然后间接调用它，把 Block 自身作为第一个实参传进去，原始参数依次追加。

```c
((int (*)(void *, int))((struct Block_layout *)blk)->invoke)(blk, 42);
```

跟 Objective-C 方法把 `self` 当第一参数是同一个套路。invoke 函数体里访问捕获的变量，靠的就是从这个指针往后偏移——所以第三节那组 size 数据才有意义。

---

## 七、几个说法需要辨析

> "Block 是语法糖，不是真正的对象。"

它有 isa，`isKindOfClass:[NSObject class]` 返回 YES，继承链三级。

> "访问了 auto 变量的 Block 就是 `__NSStackBlock__`。"

只在 MRC 下成立。ARC 下赋值给 strong 变量就已经是 `__NSMallocBlock__`。

> "ARC 下看不到栈 Block。"

看得到，别用会触发 copy 的方式去观察。见第五节。

> "`copy` 一个 Block 就是深拷贝一份内存。"

不一定。`_Block_copy` 的逻辑是：已经在堆上（`BLOCK_NEEDS_FREE`）就只把引用计数加一直接返回；是全局 Block 就原样返回；只有确实是栈 Block 时才 `malloc` + `memmove`。多数情况下只是一次引用计数递增。

> "MRC 下 Block 不会持有捕获的对象。"

这条我第一版写反了。MRC 下 copy 一个捕获了对象的 Block，对象**会**被 retain：

```text
alloc 之后                retainCount=1
栈上的 block 字面量        retainCount=1   ← 栈 block 不持有任何东西
block copy 之后            retainCount=2   ← copy 那一刻才 retain
block release 之后         retainCount=1
```

真正不 retain 的是 `__block id`：

```text
__block id：alloc 之后     retainCount=1
__block id：copy 之后      retainCount=1
```

`_Block_object_assign` 里那条分支的源码注释写得很清楚：`// Note this is MRC unretained __block only.`

这个区别正是 MRC 时代 `__block id weakSelf` 能打破循环引用的原因。ARC 把 `__block` 也改成默认 strong 了，所以那条老经验作废——下一篇有实测数据。

> "`clang -rewrite-objc` 能看到编译器真实生成的代码。"

两个问题：它走的是一套独立的教学用改写逻辑，不是真实 codegen 的旁路输出；而且它在现代 Xcode 上已经无法执行。想验证真实行为，用 `-S -emit-llvm` 看 IR，或者直接跑起来打印。

---

## 总结

ARC 下你依然能观察到栈 Block，只要别用 `id` 参数或 strong 变量去接它——那个动作本身就会把它搬到堆上。这是本文最想说的一件事，因为它同时解释了为什么"ARC 下看不到栈 block"这个说法能流传开：所有人的观察姿势都一样。

`BLOCK_IS_NOESCAPE` 那一位也是类似的故事，只不过换了个方向：规范里定义得清清楚楚，Apple 的 clang 就是没实现。我一度以为是自己不会测。

**遇到结果和预期不符，先怀疑仪器，再怀疑结论。**

下一篇讲捕获的细节：[[iOS Block 的变量捕获与 __block]]。

## 参考资料

### 规范与源码

- [Clang — Block Implementation Specification](https://clang.llvm.org/docs/Block-ABI-Apple.html)：最权威，但 descriptor 部分是简化版，别照着它做指针运算。`BLOCK_IS_NOESCAPE` 的语义写在 enum 上方的注释里
- [Clang ARC Specification — Blocks](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#blocks)：什么时候等价于 `Block_copy`、`__strong` 形参那条例外
- [apple-oss-distributions/libclosure](https://github.com/apple-oss-distributions/libclosure)：`Block_private.h` 里是真实的三段式 descriptor、完整 flags 枚举、`_Block_descriptor_3` 和几个可用的 SPI；`runtime.cpp` 里是 `_Block_copy` / `_Block_release` / `_Block_object_assign`
- [llvm-project — clang/test/CodeGenObjC/noescape.m](https://github.com/llvm/llvm-project/blob/main/clang/test/CodeGenObjC/noescape.m)：上游对 noescape codegen 的期望，拿它跟 Apple clang 的输出对照

### 第三方实证

- [steipete/Aspects — Aspects.m](https://github.com/steipete/Aspects/blob/master/Aspects.m)：不敢用 SPI 的上架代码怎么手算 descriptor 偏移

### 拓展

- [Desgard — 浅谈 block](https://github.com/Desgard/iOS-Source-Probe/tree/master/Objective-C/Runtime)：中文资料里质量较高的一份，含真实的 `-rewrite-objc` 产物，而且作者自己指出了这个工具的局限。注意那些转写代码今天已经无法自己复现
- [halfrost — 深入研究 Block 捕获外部变量和 __block 实现原理](https://halfrost.com/ios_block/)

### 本地

- [[iOS Block 的变量捕获与 __block]]
- [[iOS weak 的实现：SideTable 与置 nil 的时机]]
- [[iOS 属性关键字：从所有权推导，而不是从类型名猜]]

---

实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -target arm64-apple-ios17.0-simulator`，ARC 与 MRC 两份分别构建。所有输出块都是真实运行结果。

复现方式：手写一份和 `Block_private.h` 布局一致的结构体，把 Block 用 `__bridge` 强转过去读字段。注意别把中转函数的参数类型写成 `id`，否则 ARC 会插入一次 copy，你测到的就不是原来那个 Block 了。

flags 的位定义和 Foundation 的内部类名属于实现细节，可能随版本变化。稳定的是"三种存储位置"和"捕获的变量顺序追加在结构体后面"这两层语义。
