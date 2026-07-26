---
title: 【iOS】对象模型：类型判断、内存对齐与 Tagged Pointer
published: 2026-07-26
description: 沿 isa 链与 superclass 链推导 isKindOfClass / isMemberOfClass 的全部结果，再从内存对齐走到 Tagged Pointer——为什么有些"对象"根本没有堆分配，以及哪些细节属于版本实现不能背死。
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
# iOS 对象模型：类型判断、内存对齐与 Tagged Pointer

## 前言

系列前两篇回答的是"地址"和"记账"两个问题：[[iOS 内存：从虚拟地址空间到堆与栈]] 说明一段虚拟地址属于哪里、由谁放进去，[[iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint]] 说明这些页面此刻是否真的占用物理内存。第一篇结尾留下了一句没有展开的话：

> 并非所有看起来像对象的值都对应一次普通堆分配，例如字符串字面量和 Tagged Pointer。

这篇文章就来还这笔账。同时还要处理一个几乎每次面试都会出现、但极容易被"背结果表"糊弄过去的问题——`isKindOfClass:` 与 `isMemberOfClass:` 在实例对象和类对象上为什么会给出反直觉的答案。

这两件事看起来不相关，其实共用同一个前提：**一个 Objective-C 值到底是不是一个"有 isa、有实例变量、住在堆上的对象"**。类型判断依赖 isa 链能不能走通，Tagged Pointer 则是一类根本没有 isa 字段可走的值。所以本文把它们放在一起。

对象、类与元类的结构本身，笔者在 [[Runtime/Part 1 - 对象与类的本质]] 里已经完整梳理过，本文只回顾推导所需的最小结论，不重复展开 `objc_object`、`class_data_bits_t` 与 nonpointer isa 的位布局。

## 本文主线

```text
回顾两条链：isa 链与 superclass 链
        ↓
从 objc4 的四个方法实现推导类型判断
        ↓
用 8 组判断题验证推导，而不是背答案
        ↓
转向"对象有多大"：ivar 布局与两种对齐
        ↓
从"最小 16 字节"引出 Tagged Pointer 的动机
        ↓
判定、收益、边界，以及哪些位不能背死
```

全文反复使用的一组区分：

| 层次 | 问题 | 是否可以依赖 |
| --- | --- | --- |
| 稳定语义 | 继承关系怎样决定类型判断；Tagged Pointer 不做普通堆分配 | 可以写进代码逻辑 |
| 当前实现 | 标记位在最高位还是最低位、掩码常量、混淆器算法、`@42` 生成什么类 | 只能用于理解，不能写进代码逻辑 |

---

## 一、先回到两条链

推导类型判断只需要三条结论，都来自 [[Runtime/Part 1 - 对象与类的本质#isa 走位与继承链|isa 走位与继承链]]：

1. **实例的 isa 指向类对象，类对象的 isa 指向元类，元类的 isa 指向根元类，根元类的 isa 指向自己。**
2. **类对象沿 superclass 走到父类，最终到 `NSObject`，再走一步是 `nil`。**
3. **元类沿 superclass 走到父元类，最终到根元类；根元类的 superclass 不是 `nil`，而是绕回 `NSObject` 类对象本身。**

第三条是全文最关键、也最容易被忽略的一条。它的存在是为了让类对象能够响应 `NSObject` 的实例方法（例如 `description`、`isKindOfClass:` 本身），代价是让元类链在末端和类链接上了。

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

注意图中 `NSObject 元类 ──superclass──▶ NSObject` 这条横向的边。后面所有"反直觉"的答案都从它推出来。

先用实验确认这张图不是画着玩的：

```objc
printf("Dog                     = %p\n", (__bridge void *)[Dog class]);
printf("Dog 的元类              = %p (%s)\n",
       (__bridge void *)object_getClass([Dog class]),
       class_getName(object_getClass([Dog class])));
printf("NSObject                = %p\n", (__bridge void *)[NSObject class]);
printf("NSObject 元类           = %p\n", (__bridge void *)object_getClass([NSObject class]));
printf("NSObject 元类的 superclass = %p\n",
       (__bridge void *)class_getSuperclass(object_getClass([NSObject class])));
```

本次运行输出：

```text
Dog                     = 0x1004381f0
Dog 的元类              = 0x1004381c8 (Dog)
NSObject                = 0x1efeba748
NSObject 元类           = 0x1efeba6f8
NSObject 元类的 superclass = 0x1efeba748
```

两个观察：

- `NSObject 元类的 superclass` 与 `NSObject` 是同一个地址（`0x1efeba748`），根元类确实绕回了类对象。
- `class_getName` 对元类返回的名字仍然是 `"Dog"`。**类和元类同名**，所以调试时不能靠打印名字区分二者，只能靠地址或 `class_isMetaClass`。

---

## 二、类型判断：从源码推导，不背结果

### 2.1 四个方法，不是两个

`isKindOfClass:` 和 `isMemberOfClass:` 在 `NSObject` 上**各有实例方法和类方法两个版本**，共四个实现。这是所有困惑的源头——面试题里"receiver 是类对象"的那些例子，走的根本不是常见的那份代码。

objc4 中 `NSObject.mm` 的实现（省略了无关代码）：

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

把四份实现整理成一张表，只需要记住**起点**：

| 方法 | receiver | 起点 | 之后怎么走 |
| --- | --- | --- | --- |
| `-isMemberOfClass:` | 实例 | `[self class]`，即实例的类 | 不走，直接比较 |
| `-isKindOfClass:` | 实例 | `[self class]` | 沿 superclass 遍历类链 |
| `+isMemberOfClass:` | 类对象 | `self->ISA()`，即**元类** | 不走，直接比较 |
| `+isKindOfClass:` | 类对象 | `self->ISA()`，即**元类** | 沿 superclass 遍历**元类链** |

**一句话概括**：receiver 是实例时从类链出发，receiver 是类对象时从元类链出发。反直觉的答案全都是因为第二种情况下，参数写的是类对象，而遍历的却是元类。

### 2.2 推导 8 组判断题

先不看答案，用上面的表和第一节的图自己推。

**实例接收（`Dog *d`，`Dog : Animal : NSObject`）**

| 表达式 | 推导过程 | 结果 |
| --- | --- | --- |
| `[d isKindOfClass:[Dog class]]` | 起点 `Dog`，第一步命中 | `1` |
| `[d isKindOfClass:[Animal class]]` | `Dog → Animal` 命中 | `1` |
| `[d isKindOfClass:[NSObject class]]` | `Dog → Animal → NSObject` 命中 | `1` |
| `[d isMemberOfClass:[Dog class]]` | `Dog == Dog` | `1` |
| `[d isMemberOfClass:[Animal class]]` | `Dog != Animal`，不再往上走 | `0` |

实例这一侧完全符合直觉：`isKindOfClass:` 认祖宗，`isMemberOfClass:` 只认本人。

**类对象接收**

| 表达式 | 推导过程 | 结果 |
| --- | --- | --- |
| `[[NSObject class] isKindOfClass:[NSObject class]]` | 起点是 NSObject 元类；`NSObject 元类 → superclass 是 NSObject` 命中 | `1` |
| `[[NSObject class] isMemberOfClass:[NSObject class]]` | `NSObject 元类 != NSObject` | `0` |
| `[[Dog class] isKindOfClass:[Dog class]]` | `Dog 元类 → Animal 元类 → NSObject 元类 → NSObject → nil`，全程没有 `Dog` | `0` |
| `[[Dog class] isMemberOfClass:[Dog class]]` | `Dog 元类 != Dog` | `0` |
| `[[Dog class] isKindOfClass:[NSObject class]]` | 元类链末端绕回 `NSObject`，命中 | `1` |

### 2.3 实验验证

```objc
printf("[d isKindOfClass:Dog]        = %d\n", [d isKindOfClass:[Dog class]]);
printf("[d isKindOfClass:Animal]     = %d\n", [d isKindOfClass:[Animal class]]);
printf("[d isKindOfClass:NSObject]   = %d\n", [d isKindOfClass:[NSObject class]]);
printf("[d isMemberOfClass:Dog]      = %d\n", [d isMemberOfClass:[Dog class]]);
printf("[d isMemberOfClass:Animal]   = %d\n", [d isMemberOfClass:[Animal class]]);

printf("[NSObject isKindOfClass:NSObject]   = %d\n", [[NSObject class] isKindOfClass:[NSObject class]]);
printf("[NSObject isMemberOfClass:NSObject] = %d\n", [[NSObject class] isMemberOfClass:[NSObject class]]);
printf("[Dog      isKindOfClass:Dog]        = %d\n", [[Dog class] isKindOfClass:[Dog class]]);
printf("[Dog      isMemberOfClass:Dog]      = %d\n", [[Dog class] isMemberOfClass:[Dog class]]);
printf("[Dog      isKindOfClass:NSObject]   = %d\n", [[Dog class] isKindOfClass:[NSObject class]]);
```

本次运行输出：

```text
### A. isKindOfClass / isMemberOfClass ###
[d isKindOfClass:Dog]        = 1
[d isKindOfClass:Animal]     = 1
[d isKindOfClass:NSObject]   = 1
[d isMemberOfClass:Dog]      = 1
[d isMemberOfClass:Animal]   = 0
--- receiver 换成类对象（经典面试题）---
[NSObject isKindOfClass:NSObject]   = 1
[NSObject isMemberOfClass:NSObject] = 0
[Dog      isKindOfClass:Dog]        = 0
[Dog      isMemberOfClass:Dog]      = 0
[Dog      isKindOfClass:NSObject]   = 1
```

十组全部与推导一致。再补一个决定性的反证——如果 `+isMemberOfClass:` 真的比较的是元类，那么把参数换成元类就应该为真：

```objc
printf("[Dog isMemberOfClass:Dog的元类] = %d\n",
       [[Dog class] isMemberOfClass:object_getClass([Dog class])]);
```

```text
[Dog isMemberOfClass:Dog的元类] = 1
```

这条输出比前面十条加起来更有说服力：它直接证明了类方法版本比较的对象是 `self->ISA()`，而不是 `self`。

### 2.4 三个必须说清楚的边界

**第一，`[self class]` 与 `object_getClass(self)` 不等价。** 实例方法版用的是 `[self class]`，它是一次真实的消息发送，可以被子类覆写、也会被 KVO 的 isa-swizzling"骗过"——KVO 生成的中间类会覆写 `-class` 返回原始类。而 `object_getClass(self)` 直接读 isa，返回的是真实的（可能是 KVO 中间类的）类。所以：

```objc
// 对一个被 KVO 观察的对象
[obj class]              // 返回原始类，看起来一切正常
object_getClass(obj)     // 返回 NSKVONotifying_XXX
```

这一点在第三周的 KVO 笔记里还会再用到，这里先埋一个明确的记号。

**第二，`isKindOfClass:` 对元类链的遍历会绕回类链。** 这不是 bug，是根元类 superclass 设计的必然结果。所以"任何类对象都 `isKindOfClass:[NSObject class]`"恒为真，而"任何类对象都 `isKindOfClass:` 自己"恒为假（`NSObject` 除外，因为它恰好在元类链末端被绕回来命中了）。

**第三，实际工程中很少需要这类判断。** 需要"这个对象能不能响应某个方法"时，`respondsToSelector:` 比 `isKindOfClass:` 更贴近意图；需要"这是不是某个协议的实现者"时应该用 `conformsToProtocol:`。把类型判断当作主要的分支手段，通常说明抽象层次出了问题。这道题的价值在于**它逼你把 isa 链和 superclass 链真正画对**，而不在于结果本身。

---

## 三、对象有多大：两种对齐

要理解 Tagged Pointer 省下了什么，得先知道一个普通对象的最低成本是多少。

### 3.1 三个容易混淆的"大小"

| 函数 / 概念 | 含义 | 对齐规则 |
| --- | --- | --- |
| `class_getInstanceSize(cls)` | 所有实例变量（含 isa）按布局排完之后的大小 | 按字长对齐，arm64 上是 8 字节 |
| `malloc_size(obj)` | 分配器实际交出的内存块大小 | 由分配器决定，arm64 上以 16 字节为粒度 |
| `sizeof(SomeStruct)` | 编译期结构体大小 | 编译器按成员对齐规则计算 |

前两个的差值就是"内部碎片"：分配器为了管理效率，不会按字节精确切割。

### 3.2 实验

```objc
@interface Empty : NSObject @end
@interface OneByte : NSObject { char a; } @end
@interface ThreeChars : NSObject { char a; char b; char c; } @end
@interface OneByteOnePtr : NSObject { char a; id b; } @end
@interface Mixed : NSObject { char a; int b; double c; id d; } @end
@interface Nine : NSObject { id a; id b; /* … 共 9 个指针 … */ } @end
```

本次运行输出：

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

逐行解释：

- **`NSObject` / `Empty`：instanceSize = 8**。一个没有任何实例变量的对象也有 isa，isa 是一个指针，arm64 上占 8 字节。
- **`malloc_size` 却是 16**。分配器的最小粒度是 16 字节，所以哪怕只需要 8 字节也会拿到 16。**"一个 Objective-C 对象最少占 16 字节"这句话说的是分配粒度，不是 instanceSize。**面试里把这两个数字混着说是常见失分点。
- **`OneByte` / `ThreeChars`：instanceSize = 16**。isa 占 8 字节，后面的 1 或 3 个 `char` 被补齐到 8 的倍数，得到 16。
- **`OneByteOnePtr`：instanceSize = 24，malloc_size = 32**。isa(8) + char(1) + 为了让 `id b` 落在 8 字节边界而插入的 7 字节填充 + id(8) = 24；再被分配器提升到 32。**这 8 个字节纯粹是成员声明顺序造成的浪费**——把 `id b` 写在 `char a` 前面，就能压回 16 字节。
- **`Nine`：instanceSize = 80 = malloc_size**。8 + 9×8 = 80，正好是 16 的倍数，没有浪费。

再看 `Mixed` 的每个 ivar 偏移：

```text
a    offset=8   type=c   (char)
b    offset=12  type=i   (int)
c    offset=16  type=d   (double)
d    offset=24  type=@   (id)
```

`a` 在 8，占 1 字节；`b` 是 `int`，需要 4 字节对齐，所以跳过 9~11 落在 12；`c` 是 `double`，需要 8 字节对齐，落在 16；`d` 落在 24。总计 32。这正是 C 结构体的对齐规则——**Objective-C 对象的实例变量区，本质就是一个 C 结构体**。

### 3.3 从这里看 Tagged Pointer 的动机

现在可以量化问题了。一个 `NSNumber` 只想表示整数 `42`：

- 需要一次 `malloc`（有加锁、查找空闲块、更新元数据的成本）；
- 至少拿走 16 字节；
- 之后每次 retain/release 都要访问引用计数（可能落到 SideTable，涉及加锁）；
- 销毁时还要走一遍 `dealloc` 和 `free`。

而 `42` 本身只需要 6 个 bit。指针有 64 位，装一个小整数绰绰有余。**既然指针的位宽远大于它要表示的值，为什么不把值直接放进指针里？**这就是 Tagged Pointer。

---

## 四、Tagged Pointer

### 4.1 是什么

Tagged Pointer 是一种**把值本身编码进指针**的优化。这个"指针"不指向任何内存，它的某些位标记"我不是地址，是一个带类型标签的立即数"，其余位存放类型标签和数据。

因此一个 Tagged Pointer：

- 没有堆分配，创建和销毁的成本接近于一次赋值；
- 没有 isa 字段可读，runtime 必须通过标签位查表得到它的类；
- 没有真正的引用计数，retain/release 是空操作。

Foundation 中常见的 Tagged Pointer 类型有小整数 `NSNumber`、短 `NSString`、`NSDate`、`NSIndexPath` 等，具体范围随系统版本变化。

### 4.2 实验：怎么识别一个 Tagged Pointer

探针同时打印四个信息：类名、原始指针值、最高位、`malloc_size`。

```objc
static void probe(const char *name, id obj) {
    uintptr_t p = (uintptr_t)(__bridge void *)obj;
    printf("%-9s class=%-22s ptr=0x%016lx  bit63=%lu  low4=0x%lx  malloc_size=%zu\n",
           name, object_getClassName(obj), p,
           (p >> 63) & 1, p & 0xf, malloc_size((__bridge void *)obj));
}

NSInteger base = argc;   // 运行期才确定的值，用来绕开编译期常量折叠
probe("smallNum", [NSNumber numberWithInteger:(base + 41)]);
probe("bigNum",   [NSNumber numberWithDouble:(base * 1e100)]);
probe("shortStr", [NSString stringWithFormat:@"h%ld", (long)base]);
probe("longStr",  [NSString stringWithFormat:@"a fairly long string %ld not tagged", (long)base]);
probe("date",     [NSDate dateWithTimeIntervalSince1970:(double)base]);
probe("plainObj", [NSObject new]);
```

本次运行输出：

```text
smallNum  class=__NSCFNumber           ptr=0xb1a35bb175e0915a  bit63=1  low4=0xa  malloc_size=0
bigNum    class=__NSCFNumber           ptr=0x00006000002078a0  bit63=0  low4=0x0  malloc_size=32
shortStr  class=NSTaggedPointerString  ptr=0xb1a35bb175f83050  bit63=1  low4=0x0  malloc_size=0
longStr   class=__NSCFString           ptr=0x0000600001704480  bit63=0  low4=0x0  malloc_size=64
date      class=__NSTaggedDate         ptr=0xe74a649089e08446  bit63=1  low4=0x6  malloc_size=0
plainObj  class=NSObject               ptr=0x0000600000008070  bit63=0  low4=0x0  malloc_size=16
```

三组对照关系：

1. **小值 vs 大值**：`numberWithInteger:42` 是 Tagged Pointer，`numberWithDouble:1e100` 装不下，退回普通堆对象（`malloc_size=32`）。字符串同理，短的走 `NSTaggedPointerString`，长的走 `__NSCFString`。
2. **地址形态完全不同**：普通堆对象的地址是 `0x0000600000008070` 这种规矩的样子，高位为 0；Tagged Pointer 的高位是 1，整个值看起来像随机数。
3. **`malloc_size` 为 0**：分配器不认识这个"地址"，因为它根本不来自分配器。

### 4.3 标记位在最高位，不在最低位

网上大量中文资料写"Tagged Pointer 的最低位为 1"。这在 **x86_64** 上成立，在 **arm64** 上不成立。上面的输出里 `low4` 分别是 `0xa`、`0x0`、`0x6`，毫无规律；真正稳定的是 `bit63 = 1`。

objc4 用 `OBJC_MSB_TAGGED_POINTERS` 区分这两种布局：arm64 把标记位放在最高位，x86_64 放在最低位。**在 Apple Silicon 上，无论真机还是模拟器，都是最高位。**

这也解释了一件事：如果你在 Intel Mac 的模拟器上按"最低位"验证过一次，换到 Apple Silicon 就会得到完全相反的结论。这不是环境出错，是平台差异。

### 4.4 为什么位模式每次都变：混淆器

连续三次启动同一个二进制，打印同样三个 Tagged Pointer：

```text
--- 第 1 次启动 ---
smallNum  ptr=0xb0b0f70f6586af9e  bit63=1
shortStr  ptr=0xb0b0f70f659e0e95  bit63=1
date      ptr=0xe659c82e9986ba84  bit63=1
--- 第 2 次启动 ---
smallNum  ptr=0xb3f9fcff4e1d4682  bit63=1
shortStr  ptr=0xb3f9fcff4e05e788  bit63=1
date      ptr=0xe510c3deb21d539b  bit63=1
--- 第 3 次启动 ---
smallNum  ptr=0xaac4553e466190cb  bit63=1
shortStr  ptr=0xaac4553e467931c2  bit63=1
date      ptr=0xfc2d6a1fba6185d1  bit63=1
```

同一个值 `42`，三次启动得到三个完全不同的位模式，只有 `bit63` 恒为 1。

原因是 runtime 会在启动时生成一个随机的 **混淆器（obfuscator）**，对编码后的 Tagged Pointer 做一次异或，读取时再异或回来。这么做是为了防止攻击者构造出一个伪造的 Tagged Pointer——如果编码规则固定，任意一个可控的 64 位整数都能被伪装成某个 Foundation 对象。objc4 中相关符号是 `objc_debug_taggedpointer_obfuscator` 和 `initializeTaggedPointerObfuscator`。

从实验能直接得出的结论只有两条，其余都属于实现细节：

- 标记位（`bit63`）没有被混淆，否则 runtime 自己也无法判断；
- **除标记位以外的所有位，每次启动都不同，因此绝对不能对它们做任何硬编码假设。**

这就是为什么本文标题里那句"哪些细节不能背死"必须单独强调：掩码常量、类型标签编号、字符串的 6 bit 编码表，这些在不同 iOS 版本上都改过。真正稳定的是"值被编码在指针里、不做堆分配"这一层语义。

### 4.5 一个容易踩的实验陷阱：字面量会被常量折叠

如果直接写 `@42` 去验证，会得到与预期不符的结果：

```text
@42       class=NSConstantIntegerNumber ptr=0x00000001004340f8  bit63=0  low4=0x8  malloc_size=0
run42     class=__NSCFNumber            ptr=0xb1a35bb175e0915a  bit63=1  low4=0xa  malloc_size=0
@"hi"     class=__NSCFConstantString    ptr=0x00000001004340c0  bit63=0  low4=0x0  malloc_size=0
```

`@42` 的 `bit63 = 0`，类是 `NSConstantIntegerNumber`，地址 `0x1004340f8` 落在主程序映像范围内——它是**编译期就构造好、随 Mach-O 一起映射进来的常量对象**，压根没走 runtime 的 Tagged Pointer 路径。`@"hi"` 同理，是 `__NSCFConstantString`，住在 `__DATA_CONST` 里（回到第一篇 [[iOS 内存：从虚拟地址空间到堆与栈#Segment 与 Section|Segment 与 Section]] 那张表）。

所以验证 Tagged Pointer 必须用**运行期才能确定的值**构造，例如上面用 `argc` 参与运算。用字面量测出来的是常量对象，不是 Tagged Pointer。

顺带纠正一个流传很广的判据：**`malloc_size == 0` 不能单独用来判定 Tagged Pointer**。上面三行里，`@42` 和 `@"hi"` 的 `malloc_size` 也是 0，因为它们同样不来自分配器。`malloc_size == 0` 只说明"这不是一次普通堆分配"，要区分是 Tagged Pointer 还是常量对象，还得看标记位或类名。

### 4.6 值即身份

```objc
NSNumber *n1 = [NSNumber numberWithInteger:(base + 41)];
NSNumber *n2 = [NSNumber numberWithInteger:(base + 41)];
NSObject *o1 = [NSObject new];
NSObject *o2 = [NSObject new];
```

```text
tagged: n1=0xb1a35bb175e0915a n2=0xb1a35bb175e0915a  相同=1
堆对象: o1=0x600000008070    o2=0x600000008080     相同=0
```

两次独立创建的"值 42"，得到的是**完全相同的指针**。这是 Tagged Pointer 的直接推论：既然值编码在指针里，相同的值必然产生相同的位模式。

这条性质有一个实际影响：对 Tagged Pointer 用 `==` 比较，行为看起来像值比较；对普通对象用 `==` 是地址比较。所以**代码里判断两个 `NSNumber` 是否相等，永远应该用 `isEqual:`**，否则会写出"在小整数上一直对、换成大整数突然错"的 bug——这类问题极难复现，因为触发条件是数值大小。

### 4.7 没有真正的引用计数

```objc
printf("tagged CFGetRetainCount = %ld\n", CFGetRetainCount((CFTypeRef)n1));
printf("堆对象 CFGetRetainCount = %ld\n", CFGetRetainCount((CFTypeRef)o1));
```

```text
tagged CFGetRetainCount = 9223372036854775807
堆对象 CFGetRetainCount = 1
```

`9223372036854775807` 就是 `INT64_MAX`。这是一个约定俗成的"不朽对象"标记：runtime 在 retain/release 的入口先判断是不是 Tagged Pointer，是就直接返回，不做任何计数操作。查询引用计数时返回一个最大值，表达"这个对象永远不会被释放"。

由此可以回答一个常见问题——**Tagged Pointer 到底省了什么**：

| 环节 | 普通小对象 | Tagged Pointer |
| --- | --- | --- |
| 创建 | `malloc` + 初始化 isa 和 ivar | 位运算 |
| 内存 | 至少 16 字节堆内存 | 0 字节堆内存 |
| retain / release | 修改引用计数，可能加锁访问 SideTable | 直接返回 |
| 访问值 | 解引用一次内存 | 从指针位取出 |
| 销毁 | `dealloc` + `free` | 无 |

省下的不只是那 16 字节，更重要的是**去掉了整条内存分配与引用计数的路径**。在大量小 `NSNumber`、短字符串装箱进集合的场景里，这个收益非常可观。

### 4.8 边界：什么时候会出问题

Tagged Pointer 不是完全透明的，有几个地方会漏出来：

**关联对象**。`objc_setAssociatedObject` 需要用对象地址作为 key 去关联表里登记。Tagged Pointer 没有唯一地址（相同值共享同一个位模式），因此给 Tagged Pointer 设置关联对象在语义上是有问题的——不同的"对象"会互相覆盖。

**弱引用**。可以对 Tagged Pointer 声明 `__weak`，但因为它永远不会被销毁，弱引用也就永远不会被置 `nil`。这不会崩溃，但如果你的逻辑依赖"对象销毁后 weak 变 nil"，在小整数上就会失效。

**并发写入**。这是最经典的一个坑：假设有一个 `@property (copy) NSString *name`，多线程并发赋值。如果赋的都是短字符串（Tagged Pointer），赋值就是一次指针写入，不涉及 release 旧值，看起来"很安全"；一旦某次赋了长字符串（普通堆对象），并发的 release 就可能导致过度释放而崩溃。**表现是"平时好好的，某个用户名字长一点就闪退"**，排查方向很容易跑偏。SDWebImage 历史上就出现过同类问题。正确的做法是从一开始就保证赋值的线程安全，而不是依赖"短字符串碰巧不崩"。

**调试观察**。在 LLDB 里对一个 Tagged Pointer 执行 `memory region` 或 `x/` 会失败，因为它不是有效地址。这一点在第一篇的真机实测表里已经出现过：

> `@(runtimeValue)` → `0x9af4dd61519470e0` → 无普通 VM Region

---

## 五、稳定语义 vs 当前实现

按计划 Day 6 的要求，把本文涉及的知识分成两栏：

| 稳定语义（可以依赖） | 当前实现（只用于理解） |
| --- | --- |
| 实例的 isa 指向类，类的 isa 指向元类 | nonpointer isa 的具体位布局 |
| 根元类的 superclass 绕回 `NSObject` | 各类的具体地址、类与元类同名 |
| `+isKindOfClass:` 从元类链出发 | objc4 具体版本的函数名（`getSuperclass()` vs `superclass`） |
| 对象的 ivar 区按 C 结构体规则对齐 | `malloc` 的 16 字节粒度、nano 分配器的分档 |
| 一个对象至少有 isa，因此 instanceSize ≥ 8 | "对象最少 16 字节"这个数字来自分配器，不是语言规定 |
| Tagged Pointer 不做普通堆分配 | 标记位在最高位还是最低位（arm64 / x86_64 不同） |
| Tagged Pointer 的 retain/release 是空操作 | 混淆器算法、`INT64_MAX` 这个具体返回值 |
| 相同值的 Tagged Pointer 位模式相同 | 哪些类、哪个数值范围会被 tag（随版本变化） |
| `@42`、`@"hi"` 是编译期常量对象 | `NSConstantIntegerNumber` 这个类名 |

右栏的每一条，在过去几年里都实际发生过变化。写代码时依赖右栏，等于给自己埋一个只在特定系统版本上触发的 bug。

---

## 总结

1. `isKindOfClass:` / `isMemberOfClass:` 各有实例方法和类方法两个实现，**唯一的区别是起点**：实例走类链，类对象走元类链。所有反直觉的答案都由此推出，不需要背结果表。
2. `[[Dog class] isKindOfClass:[Dog class]]` 为假、`[[NSObject class] isKindOfClass:[NSObject class]]` 为真，共同的原因是根元类的 superclass 绕回了 `NSObject` 类对象。
3. `[self class]` 会被覆写（KVO 就依赖这一点），`object_getClass(self)` 直接读 isa。判断"真实的类"必须用后者。
4. 对象的实例变量区本质是 C 结构体，按成员对齐规则布局；`class_getInstanceSize` 给出布局大小，`malloc_size` 给出分配器实际交付的大小，两者不是一回事。"对象最少 16 字节"说的是后者。
5. 成员声明顺序会影响对象大小。把大成员放前面、小成员合并在一起，可以消除填充字节。
6. Tagged Pointer 把值编码进指针，省掉的是整条"分配 → 引用计数 → 销毁"路径，而不只是那几十字节。
7. 验证 Tagged Pointer 必须用运行期值，字面量会被编译成常量对象；`malloc_size == 0` 不足以判定，常量对象也是 0。
8. arm64 的标记位在最高位，位模式每次启动都被混淆器打乱。**除"不做堆分配"这层语义外，其余全部属于实现细节。**
9. Tagged Pointer 在关联对象、弱引用置 nil、并发赋值三个场景下会露出与普通对象不同的行为，其中并发赋值最容易产生"偶现崩溃"。

下一篇转向所有权：[[iOS 内存：MRC 的所有权规则]] 会先用手写 `retain/release/autorelease` 建立规则，再看 ARC 到底替我们删掉了什么。

## 参考资料

### 官方资料

- [Apple — NSObject class and type information](https://developer.apple.com/documentation/objectivec/nsobject)
- [WWDC20 — Advancements in the Objective-C runtime](https://developer.apple.com/videos/play/wwdc2020/10163/)
- [Clang Type Attributes — aligned](https://clang.llvm.org/docs/AttributeReference.html#aligned)
- [Apple objc4 源码](https://github.com/apple-oss-distributions/objc4)（`NSObject.mm`、`objc-internal.h` 中的 `_OBJC_TAG_MASK`）

### 经典文章

- [Always Processing — Objective-C Class isa](https://alwaysprocessing.blog/2023/01/19/objc-class-isa)
- [Always Processing — Objective-C Tagged Pointers](https://alwaysprocessing.blog/2023/03/19/objc-tagged-ptr)
- [Mike Ash — Friday Q&A 2012-07-27: Let's Build Tagged Pointers](https://www.mikeash.com/pyblog/friday-qa-2012-07-27-lets-build-tagged-pointers.html)
- [Mike Ash — Friday Q&A 2015-07-31: Tagged Pointer Strings](https://mikeash.com/pyblog/friday-qa-2015-07-31-tagged-pointer-strings.html)

### 拓展

- [深入理解 Tagged Pointer](https://blog.devtang.com/2014/05/30/understand-tagged-pointer/)
- [Testing if an arbitrary pointer is a valid Objective-C object](https://blog.timac.org/2016/1124-testing-if-an-arbitrary-pointer-is-a-valid-objective-c-object/)
- [isMemberOfClass、isKindOfClass 原理分析](https://www.cnblogs.com/xgao/p/11277935.html)

### 本地

- [[Runtime/Part 1 - 对象与类的本质]]
- [[iOS 内存：从虚拟地址空间到堆与栈]]
- [[iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint]]

---

## 附：实验环境与复现

| 项目 | 环境 |
| --- | --- |
| Xcode | 26.6 |
| 运行环境 | iOS Simulator（arm64，Apple Silicon Mac） |
| 构建方式 | `clang -fobjc-arc -target arm64-apple-ios17.0-simulator` |
| 运行方式 | `xcrun simctl spawn booted ./binary` |

数据来源说明：本文所有 `###` 输出块都是上述环境下真实运行的 stdout，不是手工编写的示意。

**未在真机验证的部分**：Tagged Pointer 的位模式、混淆器行为、`NSConstantIntegerNumber` 这一类名在不同 iOS 版本上可能不同。真机复现只需把 target 换成 `arm64-apple-ios17.0` 并在设备上运行同一份代码，结论（bit63 为标记位、位模式每次不同、`malloc_size` 为 0）预期一致，具体数值预期不同。

> 待真机补测：iPhone 15 / iOS 26.5 上重跑 B、C、D、E 四组，确认标记位与混淆行为一致。
