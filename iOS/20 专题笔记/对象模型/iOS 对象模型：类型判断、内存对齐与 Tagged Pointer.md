---
title: 【iOS】对象模型：类型判断、内存对齐与 Tagged Pointer
published: 2026-07-26
description: 三次启动的指针数据里，有一位从头到尾没变过——顺着它能读出 arm64 的 tagged pointer 布局。外加两个被中文社区讲错的结论：ivar 重排常常一个字节都省不了，对象的 16 字节下限也不是 malloc 定的。
tags:
  - iOS
  - Objective-C
  - Runtime
  - Memory
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 3
draft: true
---
# 对象模型：类型判断、内存对齐与 Tagged Pointer

系列第一篇 [[iOS 内存：从虚拟地址空间到堆与栈]] 结尾留了一句没展开的话：

> 并非所有看起来像对象的值都对应一次普通堆分配，例如字符串字面量和 Tagged Pointer。

这篇还这笔账。顺带处理一个几乎每次面试都会出现、又极容易被"背结果表"糊弄过去的问题——`isKindOfClass:` 和 `isMemberOfClass:` 在实例和类对象上为什么给出反直觉的答案。

两件事共用一个前提：一个 Objective-C 值到底是不是"有 isa、有实例变量、住在堆上的对象"。类型判断依赖 isa 链走不走得通，Tagged Pointer 则是一类根本没有 isa 可走的值。

对象、类与元类的结构本身，[[Runtime/Part 1 - 对象与类的本质]] 已经完整梳理过，这里只回顾推导需要的最小结论。全文会反复回到同一个区分：哪些是稳定语义、哪些只是当前实现——后者在过去几年里几乎全都变过。

---

## 一、类型判断：从源码推导，不背结果

### 先把两条链摆出来

推导只需要三条结论。实例的 isa 指向类，类的 isa 指向元类，元类的 isa 指向根元类。类沿 superclass 走到 `NSObject`，再走一步是 `nil`。元类沿 superclass 走到根元类，而根元类的 superclass 不是 `nil`，是绕回 `NSObject` 类对象本身。

第三条是全文最关键、也最容易被忽略的一条。它的存在是为了让类对象能响应 `NSObject` 的实例方法，代价是元类链在末端和类链接上了。

```text
实例 d ──isa──▶ Dog ──isa──▶ Dog 元类 ──isa──┐
                 │             │              │
            superclass    superclass          │
                 ▼             ▼              │
              Animal      Animal 元类         │
                 │             │              │
            superclass    superclass          │
                 ▼             ▼              │
             NSObject ◀── NSObject 元类 ◀─────┘
                 │        （根元类，isa 指向自己）
            superclass
                 ▼
                nil
```

注意 `NSObject 元类 ──superclass──▶ NSObject` 这条横向的边。后面所有反直觉的答案都从它推出来。

先确认这张图不是画着玩的：

```text
Dog                     = 0x1004381f0
Dog 的元类              = 0x1004381c8 (Dog)
NSObject                = 0x1efeba748
NSObject 元类           = 0x1efeba6f8
NSObject 元类的 superclass = 0x1efeba748
```

根元类的 superclass 和 `NSObject` 是同一个地址。另外注意 `class_getName` 对元类返回的名字仍然是 `"Dog"`——类和元类同名，调试时不能靠打印名字区分，只能靠地址或 `class_isMetaClass`。

### 四个方法，不是两个

`isKindOfClass:` 和 `isMemberOfClass:` 在 `NSObject` 上各有实例方法和类方法两个版本，共四个实现。这是所有困惑的源头——面试题里"receiver 是类对象"的那些例子，走的根本不是常见的那份代码。

```objc
+ (BOOL)isMemberOfClass:(Class)cls {
    return self->ISA() == cls;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}

+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = self->ISA(); tcls; tcls = tcls->getSuperclass()) {
        if (tcls == cls) return YES;
    }
    return NO;
}

- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->getSuperclass()) {
        if (tcls == cls) return YES;
    }
    return NO;
}
```

只需要记住起点：

| 方法 | receiver | 起点 | 之后怎么走 |
| --- | --- | --- | --- |
| `-isMemberOfClass:` | 实例 | `[self class]` | 直接比较 |
| `-isKindOfClass:` | 实例 | `[self class]` | 沿 superclass 遍历类链 |
| `+isMemberOfClass:` | 类对象 | `self->ISA()`，即元类 | 直接比较 |
| `+isKindOfClass:` | 类对象 | `self->ISA()`，即元类 | 沿 superclass 遍历元类链 |

receiver 是实例就从类链走，是类对象就从元类链走。反直觉的答案全都来自第二种情况——参数写的是类对象，遍历的却是元类。

### 十组判断题

先别看答案，用上面的表和那张图自己推。

实例接收（`Dog *d`，继承链 `Dog : Animal : NSObject`）：

| 表达式 | 推导 | 结果 |
| --- | --- | --- |
| `[d isKindOfClass:[Dog class]]` | 起点 `Dog`，第一步命中 | `1` |
| `[d isKindOfClass:[Animal class]]` | `Dog → Animal` | `1` |
| `[d isKindOfClass:[NSObject class]]` | `Dog → Animal → NSObject` | `1` |
| `[d isMemberOfClass:[Dog class]]` | `Dog == Dog` | `1` |
| `[d isMemberOfClass:[Animal class]]` | `Dog != Animal`，不往上走 | `0` |

类对象接收：

| 表达式 | 推导 | 结果 |
| --- | --- | --- |
| `[[NSObject class] isKindOfClass:[NSObject class]]` | `NSObject 元类 → superclass 是 NSObject`，命中 | `1` |
| `[[NSObject class] isMemberOfClass:[NSObject class]]` | `NSObject 元类 != NSObject` | `0` |
| `[[Dog class] isKindOfClass:[Dog class]]` | `Dog 元类 → Animal 元类 → NSObject 元类 → NSObject → nil`，全程没有 `Dog` | `0` |
| `[[Dog class] isMemberOfClass:[Dog class]]` | `Dog 元类 != Dog` | `0` |
| `[[Dog class] isKindOfClass:[NSObject class]]` | 元类链末端绕回 `NSObject`，命中 | `1` |

实测输出：

```text
[d isKindOfClass:Dog]        = 1
[d isKindOfClass:Animal]     = 1
[d isKindOfClass:NSObject]   = 1
[d isMemberOfClass:Dog]      = 1
[d isMemberOfClass:Animal]   = 0
[NSObject isKindOfClass:NSObject]   = 1
[NSObject isMemberOfClass:NSObject] = 0
[Dog      isKindOfClass:Dog]        = 0
[Dog      isMemberOfClass:Dog]      = 0
[Dog      isKindOfClass:NSObject]   = 1
```

十组全部与推导一致。但结果对不代表机制理解对——再补一个能区分假设的反证：如果 `+isMemberOfClass:` 真的比较的是元类，那把参数换成元类就应该为真。

```objc
[[Dog class] isMemberOfClass:object_getClass([Dog class])]   // 输出 1
```

前十条只能证明结果对，这一条能证明机制对：类方法版比较的是 `self->ISA()`，不是 `self`。

### 但你贴的这份代码，多数时候根本没跑

从 Xcode 11 起，clang 对 `-isKindOfClass:` 发出的往往不是消息发送，而是 `objc_opt_isKindOfClass`：

```c
BOOL objc_opt_isKindOfClass(id obj, Class otherClass) {
    if (slowpath(!obj)) return NO;
    Class cls = obj->getIsa();
    if (fastpath(!cls->hasCustomCore())) {
        for (Class tcls = cls; tcls; tcls = tcls->getSuperclass())
            if (tcls == otherClass) return YES;
        return NO;
    }
    return ((BOOL(*)(id, SEL, Class))objc_msgSend)(obj, @selector(isKindOfClass:), otherClass);
}
```

快路径用 `getIsa()` 而不是 `[self class]`，直接绕过消息发送。结果不变——receiver 是类对象时 `getIsa()` 就是元类，和 `+isKindOfClass:` 同源。

这里有个连锁反应值得注意。快路径的开关是 `hasCustomCore()`，而 objc4 的 `isCoreSelector` 判定表里就包含 `class`、`isKindOfClass:`、`respondsToSelector:`、`new`、`self` 这几个选择器。KVO 生成的中间类覆写了 `-class`，于是整条快路径被关掉、退回 `objc_msgSend`——下面那个 KVO 的例子之所以还成立，靠的正是这个。

### 三个边界

`[self class]` 和 `object_getClass(self)` 不等价。实例方法版用的是 `[self class]`，那是一次真实的消息发送，可以被覆写；`object_getClass(self)` 直接读 isa。对一个被 KVO 观察的对象：

```objc
[obj class]              // 返回原始类，看起来一切正常
object_getClass(obj)     // 返回 NSKVONotifying_XXX
```

判断"真实的类"必须用后者。这一点在第三周的 KVO 笔记里还会再用到。

`isKindOfClass:` 对元类链的遍历会绕回类链。所以"任何类对象都 `isKindOfClass:[NSObject class]`"恒为真，而"任何类对象都 `isKindOfClass:` 自己"恒为假——`NSObject` 除外，因为它恰好在元类链末端被绕回来命中了。

想问"这个类是不是那个类的子类"，同一个文件里就有现成的答案：

```objc
+ (BOOL)isSubclassOfClass:(Class)cls {
    for (Class tcls = self; tcls; tcls = tcls->getSuperclass()) {
        if (tcls == cls) return YES;
    }
    return NO;
}
```

起点是 `self` 而不是 `self->ISA()`，走的是类链。`[[Dog class] isSubclassOfClass:[Dog class]]` 是 `YES`。

### 一句私货

这道题被当成面试标配，我一直觉得性价比不高。真实代码里需要 `isKindOfClass:` 做主分支的场合极少：问"能不能响应某个方法"该用 `respondsToSelector:`，问"是不是某协议的实现者"该用 `conformsToProtocol:`，问"是不是某个类的子类"该用上面那个 `+isSubclassOfClass:`。把类型判断当成主要的分支手段，通常说明抽象层次出了问题。

它唯一的价值是逼你把两条链真的画对。画对了答案是推出来的；背结果表恰恰说明没画对。

---

## 二、对象有多大

要理解 Tagged Pointer 省下了什么，得先知道一个普通对象的最低成本。

### instanceSize 不是 malloc_size

`class_getInstanceSize(cls)` 是所有实例变量（含 isa）按 C 结构体规则排完之后、再按字长对齐的大小。`malloc_size(obj)` 是分配器实际交出的内存块大小。两者的差值就是内部碎片。

```text
类名           class_getInstanceSize   malloc_size
NSObject                8                  16
Empty                   8                  16
OneByte                16                  16
ThreeChars             16                  16
OneByteOnePtr          24                  32
Mixed                  32                  32
Nine                   80                  80
```

`NSObject` 的 instanceSize 是 8——一个没有任何实例变量的对象也有 isa，arm64 上占 8 字节。

但 `malloc_size` 是 16。这一档不是 malloc 决定的。objc4 的 `objc_class::instanceSize()` 里有一行：

```cpp
// CF requires all objects be at least 16 bytes.
if (size < 16) size = 16;
```

runtime 在调 `calloc` 之前就把 8 抬成了 16，理由写在注释里——CoreFoundation 要求。所以"一个 Objective-C 对象最少 16 字节"这句话是 runtime 硬编码的，不是分配器的粒度。

分配器的 16 字节粒度确实存在，但它解释的是下面那档：`OneByteOnePtr` 的 instanceSize 24 被提升到 malloc_size 32。

### ivar 重排：一个我照抄了流行说法然后被打脸的地方

`OneByteOnePtr` 的布局是 isa(8) + char(1) + 7 字节填充 + id(8) = 24。中文社区的标准说法是：把大成员放前面就能消掉这 7 字节填充。

我原来也是这么写的。实测：

```text
{char a; id b;}         = 24
{id b; char a;}         = 24   ← 重排后，一个字节都没省
{char a; id b; char c;} = 32
{id b; char a; char c;} = 24   ← 重排后，省了 8 字节
```

只有一个小成员时，重排完全无效——省下的中间填充会原样变成尾部填充，因为结构体总大小还要按 8 对齐。`{id b; char a;}` 的未对齐大小是 17，对齐后仍然是 24。

重排要见效，得有两个及以上的小成员可以并到一起。这时 `{char a; ... char c;}` 分散在两处各自吃填充，合并后共用一次。

所以那条流行建议不算错，但它省略了前提，而按最常见的两成员例子去试，会发现一个字节都省不下来。

### Mixed 的偏移

```text
a    offset=8   type=c   (char)
b    offset=12  type=i   (int)
c    offset=16  type=d   (double)
d    offset=24  type=@   (id)
```

`a` 在 8 占 1 字节；`b` 是 `int` 需要 4 字节对齐，跳过 9~11 落在 12；`c` 是 `double` 需要 8 字节对齐，落在 16；`d` 落在 24。总计 32。Objective-C 对象的实例变量区，本质就是一个 C 结构体。

### 从这里看 Tagged Pointer 的动机

一个 `NSNumber` 只想表示整数 42，代价是：一次 `malloc`（加锁、找空闲块、更新元数据）、至少 16 字节、之后每次 retain/release 都要动引用计数、销毁时走一遍 `dealloc` 和 `free`。

42 只要 6 个 bit，指针有 64 位。剩下 58 位与其空着，不如把值本身放进去。

---

## 三、Tagged Pointer

### arm64 的布局：标签在低位，标记位在最高位

先纠正一个我自己犯过的错。网上大量中文资料写"Tagged Pointer 的最低位为 1"，也有资料说 arm64 是 MSB 方案。两种说法都不完整。

`objc-internal.h` 的分支顺序是这样的：

```c
#if __arm64__
#   define OBJC_SPLIT_TAGGED_POINTERS 1    // arm64 走这条
...
#if OBJC_SPLIT_TAGGED_POINTERS             // 这个分支在前
#   define _OBJC_TAG_MASK (1UL<<63)
#   define _OBJC_TAG_INDEX_SHIFT 0         // 标签索引在低位
#elif OBJC_MSB_TAGGED_POINTERS             // arm64 根本到不了这里
```

注释说得很直白：*ARM64 uses a new tagged pointer scheme where normal tags are in the low bits, extended tags are in the high bits.*

所以 arm64 上的实际布局是：

```text
bit63     ── 标记位：1 表示这是 tagged pointer
bit3-62   ── payload（60 位）
bit0-2    ── 标签索引（3 位），标识这是哪个类
```

3 位只能表示 8 个类，不够用。索引 `0b111` 是逃逸出口——命中它就切到扩展表示：高位放 8 位扩展标签，payload 缩到 52 位。常见的标签取值有 `NSAtom=0`、`NSString=2`、`NSNumber=3`、`NSIndexPath=4`、`NSDate=6`，扩展段里有 `NSColor=16`、`NSIndexSet=19`、`Constant_CFString=136` 等。

至于"最低位为 1"那种 LSB 布局，条件是：

```c
#if (TARGET_OS_OSX || TARGET_OS_MACCATALYST) && __x86_64__
#   define OBJC_MSB_TAGGED_POINTERS 0    // LSB
#else
#   define OBJC_MSB_TAGGED_POINTERS 1
#endif
```

只出现在 Intel Mac 上的原生 macOS 或 Mac Catalyst 进程里。**iOS 模拟器不算**——它的 `TARGET_OS_OSX` 是 0，哪怕宿主是 Intel Mac，标记位也在 bit63。那批"最低位为 1"的中文资料，多半是在 Intel Mac 上写命令行 macOS demo 测出来的。

### 探针

打印类名、指针、bit63、bit62、低 3 位、`malloc_size`：

```text
smallNum  class=__NSCFNumber           ptr=0xb1a35bb175e0915a  bit63=1  malloc_size=0
bigNum    class=__NSCFNumber           ptr=0x00006000002078a0  bit63=0  malloc_size=32
shortStr  class=NSTaggedPointerString  ptr=0xb1a35bb175f83050  bit63=1  malloc_size=0
longStr   class=__NSCFString           ptr=0x0000600001704480  bit63=0  malloc_size=64
date      class=__NSTaggedDate         ptr=0xe74a649089e08446  bit63=1  malloc_size=0
plainObj  class=NSObject               ptr=0x0000600000008070  bit63=0  malloc_size=16
```

小值走 tagged，大值退回堆对象。tagged 的高位是 1，普通堆地址高位是 0，`malloc_size` 为 0——分配器不认识这个"地址"，因为它根本不来自分配器。

### 什么样的值才会被 tag

我原来写"`numberWithDouble:1e100` 装不下所以退回堆对象"，这个解释太粗糙。实测：

```text
numberWithDouble:1.0      bit63=1  tagged=1
numberWithDouble:1.5      bit63=0  tagged=0     ← 1.5 也"装不下"？
numberWithLongLong:2^55   bit63=0  tagged=0     ← 远小于 60 bit
numberWithBool:YES        __NSCFBoolean, tagged=0
```

真实规律是：tagged `NSNumber` 存的是整数形态的 payload。浮点数只有恰好是整数、能无损转换时才 tagged，`1.5` 有小数部分直接退堆。整数的上限也远低于 60 位——`2^55` 就已经不 tagged 了，因为 payload 里还要放类型信息。`BOOL` 根本不走这条路，`__NSCFBoolean` 是单例。

字符串有个能精确测出来的硬上限：

```text
7 个 a    NSTaggedPointerString   tagged=1
8 个 a    NSTaggedPointerString   tagged=1
9 个 a    NSTaggedPointerString   tagged=1
10 个 a   NSTaggedPointerString   tagged=1
11 个 a   NSTaggedPointerString   tagged=1
12 个 a   __NSCFString            tagged=0
```

11 个字符是分界线。原因是编码分三档：长度 0–7 每字符 8 位原样存；8–9 用 6 位编码，字母表是 `eilotrm.apdnsIc ufkMShjTRxgC4013bDNvwyUL2O856P-B79AFKEWV_zGJ/HYX`，按英文字频排序；10–11 只能用这张表的前 32 项做 5 位编码。所以 11 个 `a` 能装下，但 10 个生僻字符可能就装不下——这个上限跟字符本身有关，不只是长度。

### 混淆器：三次启动的数据里藏着源码的分支

同一个二进制连续跑三次，打印同样几个 tagged pointer 的 bit63、bit62 和低 3 位：

```text
--- 第 1 次 ---            bit63  bit62  low3
NSNumber(42)                 1      0      4
NSNumber(7)                  1      0      4
NSString(短)                 1      0      6
NSString(短2)                1      0      6
NSDate                       1      1      5
--- 第 2 次 ---
NSNumber(42)                 1      0      2
NSString(短)                 1      0      0
NSDate                       1      1      4
--- 第 3 次 ---
NSNumber(42)                 1      0      1
NSString(短)                 1      0      4
NSDate                       1      1      3
```

我第一版写的是"除标记位以外的所有位每次启动都不同"。这句话被我自己的数据否定了——`bit62` 三次纹丝不动，而且按类分成两组：`NSNumber` 和 `NSString` 恒为 0，`NSDate` 恒为 1。

低 3 位也不是噪声。同一次运行里，所有 `NSNumber` 共享一个值、所有 `NSTaggedPointerString` 共享另一个值；换一次启动，这组值被整体换掉（NSNumber 依次是 4→2→1，NSString 是 6→0→4）。这不是随机，是置换。

源码里写得很清楚（`objc-runtime-new.mm`）：

```c
arc4random_buf(&objc_debug_taggedpointer_obfuscator, sizeof(...));
objc_debug_taggedpointer_obfuscator &= ~_OBJC_TAG_MASK;
#if OBJC_SPLIT_TAGGED_POINTERS
    // The obfuscator doesn't apply to any of the extended tag mask or the no-obfuscation bit.
    objc_debug_taggedpointer_obfuscator &= ~(_OBJC_TAG_EXT_MASK | _OBJC_TAG_NO_OBFUSCATION_MASK);
    // Shuffle the first seven entries of the tag permutator.
```

不参与异或的是 bit63、bit62 和低 3 位共五位；低 3 位另有一张 7 元置换表每次启动重新洗牌。所以"不能硬编码"这个结论仍然成立，但理由不是"只剩一位可信"，而是**可信的那几位不足以让你自己解码**。

混淆器存在的目的是防止伪造：如果编码规则固定，任意一个可控的 64 位整数都能被伪装成某个 Foundation 对象。

顺带一提，扩展标签里有一档是完全不混淆的——`_OBJC_TAG_NO_OBFUSCATION_MASK` 命中时编码函数直接原样返回，`Constant_CFString = 136` 就靠这个存在，它让编译期构造 tagged 字符串成为可能。

### 字面量测不出来，但这条"陷阱"本身是有版本的

用 `@42` 去验证会得到反常结果：

```text
@42       class=NSConstantIntegerNumber  bit63=0  malloc_size=0
run42     class=__NSCFNumber             bit63=1  malloc_size=0
@"hi"     class=__NSCFConstantString     bit63=0  malloc_size=0
```

`@42` 的地址落在主程序映像范围内，它是编译期就构造好、随 Mach-O 一起映射进来的常量对象。`@"hi"` 同理，住在 `__DATA_CONST` 里。

所以验证 Tagged Pointer 必须用运行期才能确定的值（我用了 `argc` 参与运算）。

但这条"陷阱"本身是工具链版本相关的。它来自 clang 的 `-fobjc-constant-literals`，Xcode 13 起默认开启；在此之前 `@42` 走的是运行期 `+numberWithInt:`，那时它确实是一个 tagged pointer。也就是说，2020 年前那批写"`@42` 是 tagged pointer"的博客，在当时并没有错。

这个自我打脸值得留着——一篇通篇强调"不能背死实现细节"的文章，自己差点把一条版本相关的行为写成了永恒真理。

另外纠正一个流传很广的判据：`malloc_size == 0` 不能单独用来判定 Tagged Pointer。上面三行里 `@42` 和 `@"hi"` 的 `malloc_size` 也是 0，因为它们同样不来自分配器。要区分是 tagged 还是常量对象，得看标记位或类名。

### 值即身份

```text
tagged: n1=0xb1a35bb175e0915a n2=0xb1a35bb175e0915a  相同=1
堆对象: o1=0x600000008070    o2=0x600000008080     相同=0
```

两次独立创建的"值 42"得到完全相同的指针。既然值编码在指针里，相同的值必然产生相同的位模式。

这条性质有个实际影响：对 tagged pointer 用 `==` 比较，行为看起来像值比较；对普通对象用 `==` 是地址比较。所以判断两个 `NSNumber` 是否相等永远要用 `isEqual:`，否则会写出"小整数上一直对、换成大整数突然错"的 bug——触发条件是数值大小，极难复现。

### 没有真正的引用计数

```text
tagged CFGetRetainCount = 9223372036854775807
堆对象 CFGetRetainCount = 1
```

`9223372036854775807` 是 `INT64_MAX`，是 runtime 对不朽对象统一返回的值。retain/release 的入口先判断是不是 tagged pointer，是就直接返回，不做任何计数操作。

由此可以回答 Tagged Pointer 到底省了什么：

| 环节 | 普通小对象 | Tagged Pointer |
| --- | --- | --- |
| 创建 | `malloc` + 初始化 isa 和 ivar | 位运算 |
| 内存 | 至少 16 字节堆内存 | 0 |
| retain / release | 修改引用计数，可能加锁 | 直接返回 |
| 访问值 | 解引用一次内存 | 从指针位取出 |
| 销毁 | `dealloc` + `free` | 无 |

省下的不只是那 16 字节，是整条"分配 → 引用计数 → 销毁"的路径。至于这个收益具体有多大，得看装箱密度——本文没有测，不给数字。

### 四个会漏出来的地方

**关联对象。** 这条常被写成"不能用"。实测是能用的：

```text
t1 == t2 ? 1
objc_getAssociatedObject(t1) = 来自 t1
objc_getAssociatedObject(t2) = 来自 t1  ← 从没给 t2 设过
objc_getAssociatedObject(h1) = 来自 h1
```

设得进、读得回、不崩溃。问题有两个。一是相同值共享同一个实例，两个"不同"的小整数会读到同一份关联——上面 `t2` 从没设过却读出了 `t1` 的值。二是关联表只在 `objc_destructInstance` 里通过 `_object_remove_associations` 清理，而 tagged pointer 永远不 dealloc，挂上去的东西会一直留到进程结束。第二条比第一条危险，因为它不表现为逻辑错误，只表现为内存慢慢涨。

这里有个坑我自己踩过，写下来提醒后来人：**这组实验不能用 `@(42)` 这种字面量来造 tagged pointer。**前面第四节刚讲过，`@42` 和 `@(42)` 在 Xcode 13 之后都被折叠成 `NSConstantIntegerNumber`，bit63 是 0，压根不是 tagged。用它跑出来的"同值共享"看着结论一样，机制却是另一回事——常量对象共享是因为编译器只发了一份，tagged pointer 共享是因为值直接编在指针位里。要真的拿到 tagged pointer，装箱的值必须来自运行期：

```objc
int rt = 42;
id fromLiteral = @(42);     // NSConstantIntegerNumber，bit63 = 0
id fromRuntime = @(rt);     // __NSCFNumber，bit63 = 1，这才是 tagged
```

上面那组关联对象的输出用的是运行期值。同一篇文章里两处结论建立在同一个字面量上却指向不同机制，是很容易出的错。

**弱引用。** 可以声明 `__weak` 指向 tagged pointer，运行时在 `weak_register_no_lock` 开头就短路了（`_objc_isTaggedPointerOrNil` 判断），根本不写进弱引用表。因为它永远不会被销毁，弱引用也就永远不会被置 `nil`。实测 `weak 指向 tagged: 42`，对象"释放"后依然是 42。这不崩溃，但如果逻辑依赖"对象销毁后 weak 变 nil"，在小整数上就会失效。

**并发写入。** 假设有 `@property (nonatomic, copy) NSString *name`，多线程并发赋值。赋短字符串时是 tagged pointer，赋值就是一次指针写入，不涉及 release 旧值，看起来"很安全"；一旦某次赋了长字符串，并发的 release 就可能过度释放而崩溃。表现是"平时好好的，某个用户名字长一点就闪退"。

这里必须写 `nonatomic`——不写默认是 atomic，而 atomic 的 setter 在锁内交换槽位，每个线程拿到互不相同的旧值，恰恰不会过度释放。中文社区转述这个案例时经常漏掉这个前提，导致整段推理不成立。正确做法是从一开始就保证赋值的线程安全，而不是依赖"短字符串碰巧不崩"。

**调试观察。** 在 LLDB 里对 tagged pointer 执行 `memory region` 或 `x/` 会失败，因为它不是有效地址。第一篇的真机实测表里已经出现过：`@(runtimeValue)` → `0x9af4dd61519470e0` → 无普通 VM Region。

### Swift 那边是另一条路

Swift 原生 `String` 不用 tagged pointer，而是 `_StringObject` 的 small-string 形式——16 字节的 struct 里直接内联最多 15 个 UTF-8 字节，比 Objective-C 的 11 字符宽松，因为它不需要保留指针语义。只有 bridge 到 `NSString` 时才可能变成 tagged pointer。Swift 的 class 实例同样是 16 字节起步（isa + 引用计数）。

同一个思想的两种实现：值小到一定程度就别走堆。

---

## 四、稳定语义 vs 当前实现

| 可以依赖 | 只用于理解 |
| --- | --- |
| 实例的 isa 指向类，类的 isa 指向元类 | nonpointer isa 的具体位布局 |
| 根元类的 superclass 绕回 `NSObject` | 类与元类同名、具体地址 |
| `+isKindOfClass:` 从元类链出发 | 快路径 `objc_opt_isKindOfClass` 的存在与开关条件 |
| 对象的 ivar 区按 C 结构体规则对齐 | `malloc` 的 16 字节粒度 |
| 一个对象至少有 isa，instanceSize ≥ 8 | 16 字节下限来自 objc4 的硬编码（CF 要求） |
| Tagged Pointer 不做普通堆分配 | 标记位在最高位还是最低位（平台相关） |
| Tagged Pointer 的 retain/release 是空操作 | 混淆器算法、`INT64_MAX` 这个具体返回值 |
| 相同值的 Tagged Pointer 位模式相同 | 哪些类、哪个数值范围会被 tag |
| 字面量是编译期常量对象 | 这条本身依赖 `-fobjc-constant-literals`，Xcode 13 起才成立 |

右栏的每一条在过去几年里都实际变过。写代码时依赖右栏，等于埋一个只在特定系统版本上触发的 bug。

---

## 总结

四个类型判断方法唯一的区别是起点：实例走类链，类对象走元类链。所有反直觉的答案都是这一条加上"根元类 superclass 绕回 NSObject"推出来的，不需要背结果表。

两个 16 是不同来源的：instanceSize 从 8 抬到 malloc_size 16，是 objc4 里 `if (size < 16) size = 16` 干的，注释写着 CoreFoundation 的要求；24 抬到 32 才是分配器的粒度。ivar 重排能省内存的前提是有两个以上小成员可以合并，只有一个小成员时重排一个字节都省不下来。

Tagged Pointer 省掉的是整条"分配 → 引用计数 → 销毁"路径。可以依赖的只有"不做堆分配"和"相同值位模式相同"这两条；标记位位置、标签编号、混淆规则、哪些类会被 tag，全都变过。

最后是方法论。这一篇里几处"网上说的不对"，都不是靠读更多文章发现的：ivar 重排是编译一次就露馅，`bit62` 的稳定性就摆在我自己打印的十二个数字里。**取到数据只是第一步，让数据和源码互相质证才是第二步**——我第一版就是数据取到了却没盯着看，写下了被自己的输出否定的结论。

下一篇转向所有权：[[iOS 内存：MRC 的所有权规则]]。

## 参考资料

### 官方与源码

- [Apple — NSObject class and type information](https://developer.apple.com/documentation/objectivec/nsobject)
- [WWDC20 — Advancements in the Objective-C runtime](https://developer.apple.com/videos/play/wwdc2020/10163/)：`objc_opt_isKindOfClass` 这类快路径的出处
- [objc4 — objc-internal.h](https://github.com/apple-oss-distributions/objc4)：tag 枚举与三种布局的条件编译
- [objc4 — objc-runtime-new.mm](https://github.com/apple-oss-distributions/objc4)：tagged pointer 布局注释、混淆器初始化、`isCoreSelector`
- [objc4 — objc-runtime-new.h](https://github.com/apple-oss-distributions/objc4)：`instanceSize` 的 16 字节下限

### 经典

- [Always Processing — Objective-C Tagged Pointers](https://alwaysprocessing.blog/2023/03/19/objc-tagged-ptr)：60 位 payload、索引 7 逃逸到扩展表示
- [Mike Ash — Tagged Pointer Strings](https://mikeash.com/pyblog/friday-qa-2015-07-31-tagged-pointer-strings.html)：8/6/5 位三档编码和那张按字频排序的字母表
- [Andrew Madsen — Constant Literals in Objective-C](https://blog.andrewmadsen.com/2021/06/07/constant-literals-in.html)：`-fobjc-constant-literals` 与 Xcode 13

### 本地

- [[Runtime/Part 1 - 对象与类的本质]]
- [[iOS 内存：从虚拟地址空间到堆与栈]]
- [[iOS 内存：MRC 的所有权规则]]

---

实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -fobjc-arc -target arm64-apple-ios17.0-simulator`，`xcrun simctl spawn booted` 运行。所有输出块都是真实 stdout。

Tagged Pointer 的位模式、混淆行为、`NSConstantIntegerNumber` 这类类名在不同 iOS 版本上可能不同。真机复现只需把 target 换成 `arm64-apple-ios17.0` 跑同一份代码——结论（bit63 是标记位、bit62 与低 3 位不参与异或、`malloc_size` 为 0）预期一致，具体数值预期不同。
