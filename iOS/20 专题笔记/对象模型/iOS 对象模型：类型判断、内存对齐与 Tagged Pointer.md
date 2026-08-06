---
title: 【iOS】对象模型：类型判断、内存对齐与 Tagged Pointer
published: 2026-07-26
description: 从一行 Dog *d = [Dog new] 出发，先分清指针、对象、类和元类，再理解 isKindOfClass、对象大小以及 Tagged Pointer 为什么能省掉一次堆分配。
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

我们从一行每天都会写的代码开始：

```objc
Dog *d = [Dog new];
```

这行代码里其实有两个东西：

- `d` 是一个指针变量，它保存的是地址；
- `[Dog new]` 创建出来的才是对象本身，普通情况下它在堆里。

这篇我们要解决三个问题：

1. `isKindOfClass:` 和 `isMemberOfClass:` 到底差在哪？
2. 一个 Objective-C 对象到底占多大内存？
3. 为什么有些 `NSNumber`、`NSString` 看起来像对象，却没有普通的堆地址？

---

## 一、先分清对象、类和元类

### 1. `d` 和对象不是同一个东西

还是这行代码：

```objc
Dog *d = [Dog new];
```

可以先把它画成这样：

```text
指针变量 d                         Dog 对象
┌──────────────┐                  ┌──────────────────┐
│ 保存对象的地址 ├─────────────────▶│ isa │ 其他实例变量 │
└──────────────┘                  └──┬───────────────┘
                                    │
                                    ▼
                                  Dog 类对象
```

`d` 指向 Dog 对象。Dog 对象开头保存的 `isa`，又能让 Runtime 找到 Dog 类对象。

所以：

```text
d  ──指向──▶  Dog 对象  ──isa──▶  Dog 类对象
```

这里先记住一句话：

> 实例对象通过 isa 找到自己的类。

### 2. 类对象里存什么

Dog 类对象不是某一只具体的狗。它更像 Dog 这个类型的说明书，里面能找到实例方法、实例变量的描述、父类以及实例大小等信息。

如果有继承关系：

```objc
@interface Animal : NSObject
@end

@interface Dog : Animal
@end
```

那么类对象会沿 `superclass` 连成一条链：

```text
Dog ──superclass──▶ Animal ──superclass──▶ NSObject ──▶ nil
```

这就是实例执行 `isKindOfClass:` 时要走的主要路线。

### 3. 元类先知道它是干什么的就够了

类对象也能接收消息：

```objc
[Dog new];
[Dog alloc];
```

这些类方法存在哪里？答案是元类。

```text
Dog 对象 ──isa──▶ Dog 类对象 ──isa──▶ Dog 元类
```

- Dog 对象找 Dog 类对象，是为了找实例方法；
- Dog 类对象找 Dog 元类，是为了找类方法。

第一次学习到这里就够了。你不需要先背根元类的完整闭环，等我们分析“类对象也调用 `isKindOfClass:`”时再补那一小段。

---

## 二、类型判断：先学日常用法

### 1. `isMemberOfClass:` 只认“正好就是这个类”

```objc
Dog *d = [Dog new];

[d isMemberOfClass:[Dog class]];       // YES
[d isMemberOfClass:[Animal class]];    // NO
```

`d` 的真实类型是 Dog，所以第一句为真。虽然 Dog 继承自 Animal，但 `isMemberOfClass:` 不会继续找父类。它只问：

> 这个对象是不是正好由这个类创建出来的？

### 2. `isKindOfClass:` 会继续找父类

```objc
[d isKindOfClass:[Dog class]];         // YES
[d isKindOfClass:[Animal class]];      // YES
[d isKindOfClass:[NSObject class]];    // YES
```

它从 Dog 开始，沿着父类链往上找：

```text
Dog → Animal → NSObject → nil
```

只要途中遇到目标类，就返回 `YES`。

| 方法 | 它真正问的问题 | 会不会找父类 |
| --- | --- | --- |
| `isMemberOfClass:` | 是不是正好这个类 | 不会 |
| `isKindOfClass:` | 是不是这个类或它的子类 | 会 |

### 3. 最常见的五个结果

| 表达式 | 结果 | 为什么 |
| --- | --- | --- |
| `[d isKindOfClass:[Dog class]]` | `YES` | 起点就是 Dog |
| `[d isKindOfClass:[Animal class]]` | `YES` | Dog 的父类是 Animal |
| `[d isKindOfClass:[NSObject class]]` | `YES` | 继续向上能找到 NSObject |
| `[d isMemberOfClass:[Dog class]]` | `YES` | 真实类型正好是 Dog |
| `[d isMemberOfClass:[Animal class]]` | `NO` | 它不是正好由 Animal 创建的 |

这五个才是平时最可能用到的情况。

### 4. 为什么类对象调用时结果很怪

下面这种题不是日常写法，但面试很喜欢问：

```objc
[[Dog class] isKindOfClass:[Dog class]];           // NO
[[Dog class] isKindOfClass:[NSObject class]];      // YES
```

这次接收消息的不是 Dog 实例，而是 `[Dog class]` 这个类对象。类对象执行类型判断时，起点不是 Dog 类，而是 Dog 元类：

```text
Dog 元类
   ↓
Animal 元类
   ↓
NSObject 元类
   ↓
NSObject 类
   ↓
nil
```

这条链里没有 Dog 类，因此第一句为 `NO`；末尾能走到 NSObject 类，因此第二句为 `YES`。

不要背结果。看到 receiver 是类对象时，只做两步：

1. 从它的元类开始；
2. 沿元类的 `superclass` 往上找。

### 5. 实际开发该用哪个方法

- 想知道对象能不能调用某个方法：`respondsToSelector:`；
- 想知道对象是否遵守协议：`conformsToProtocol:`；
- 想知道某个类是不是另一个类的子类：`isSubclassOfClass:`；
- 真的需要区分具体类型时，才考虑 `isKindOfClass:` 或 `isMemberOfClass:`。

---

## 三、一个普通对象到底有多大

### 1. 先别把两种“大小”混在一起

```objc
class_getInstanceSize([Dog class]);
malloc_size((__bridge const void *)d);
```

它们问的不是同一个问题。

| 数值 | 含义 |
| --- | --- |
| `class_getInstanceSize` | 按照 isa、实例变量和对齐规则，这个对象至少需要多少字节 |
| `malloc_size` | 分配器实际给了这个对象多大的内存块 |

```text
对象布局需要的空间       分配器实际给出的空间
class_getInstanceSize    malloc_size
          24        →        32
```

多出来的是分配时的空余空间，不是新的实例变量。

### 2. 为什么空对象也要占空间

一个没有自定义实例变量的 `NSObject`，仍然需要保存 `isa`。在 64 位环境里，指针是 8 字节。

实验中：

```text
class_getInstanceSize([NSObject class]) = 8
malloc_size(object)                     = 16
```

当前 objc4 实现会把小于 16 字节的对象分配大小抬到 16，源码注释给出的原因是 CoreFoundation 的要求。

因此要把两句话分开：

- 从布局看，空对象至少要放下 8 字节的 isa；
- 从当前普通堆分配结果看，它通常会拿到至少 16 字节。

第二句属于实现细节，不要把 16 当成永远不变的语言规则。

### 3. 内存对齐到底在对齐什么

```objc
char a;      // 1 字节
int b;       // 4 字节
double c;    // 8 字节
id d;        // 8 字节
```

在 64 位环境里，一个可能的布局是：

```text
0  ~ 7    isa
8          char a
9  ~ 11   填充
12 ~ 15   int b
16 ~ 23   double c
24 ~ 31   id d
```

`int` 希望从 4 的倍数地址开始，`double` 和指针希望从 8 的倍数地址开始。为了满足这些要求，中间会出现填充字节。这和 C 结构体的对齐规则是同一件事。

### 4. 把大成员放前面，不一定马上省空间

```text
{ char a; id b; } = 24
{ id b; char a; } = 24
```

第二种虽然消掉了中间填充，但结构体末尾仍要补齐到 8 的倍数，所以总大小没变。

```text
{ char a; id b; char c; } = 32
{ id b; char a; char c; } = 24
```

第二组里两个小成员能挨在一起，共用末尾那次对齐，才真的省下 8 字节。

> 调整成员顺序可能减少填充，但是否真的缩小总大小，要看最终对齐结果。

---

## 四、Tagged Pointer：指针里直接放值

### 1. 它想解决什么问题

假设只想保存整数 `42`。普通对象会这样做：

```text
NSNumber 指针 ──▶ 堆里的 NSNumber 对象 ──▶ 对象内部保存 42
```

为了这么小的值，系统仍要分配堆内存、初始化对象、管理引用计数，最后再销毁对象。

对于足够小的值，Runtime 可以不再把 64 位“指针”当成普通地址，而是把类型信息和值直接编码进去：

```text
普通对象：指针里放地址 ──▶ 去堆上找值
Tagged Pointer：指针里直接放类型和值
```

这就是 Tagged Pointer。

### 2. 它看起来像对象，但不是普通堆对象

```objc
int runtimeValue = 42;
NSNumber *small = @(runtimeValue);
```

`small` 仍然可以接收 `integerValue`、`description` 等消息，但它背后不一定有普通堆内存。Runtime 会识别 Tagged Pointer，再从位模式中找到对应的类和值。

所以它不是“没有类型”，而是“类型和值没有放在普通对象内存里”。

### 3. 小值可能 tagged，大值可能退回堆对象

```text
小 NSNumber     tagged，malloc_size = 0
大 NSNumber     普通堆对象，malloc_size = 32
短 NSString     tagged，malloc_size = 0
长 NSString     普通堆对象，malloc_size = 64
普通 NSObject   普通堆对象，malloc_size = 16
```

“小”不是简单按十进制位数判断。payload 还要保存类型信息；浮点数、字符串编码和系统实现也会影响结果。

业务代码不应该依赖某个具体数值或字符串一定会 tagged。

### 4. Tagged Pointer 省下了什么

| 环节 | 普通小对象 | Tagged Pointer |
| --- | --- | --- |
| 创建 | 分配并初始化堆对象 | 通过位运算编码 |
| 堆内存 | 需要 | 不需要普通堆分配 |
| retain / release | 维护普通引用计数 | 不走普通引用计数路径 |
| 读取值 | 通过地址找到对象，再读数据 | 从位模式解码 |
| 销毁 | `dealloc` + 释放内存 | 没有普通堆对象要销毁 |

它省下的不只是几个字节，而是“分配 → 引用计数 → 销毁”这一整条普通对象路径。

### 5. 最容易写错的是 `==`

同一个进程里，相同类型、相同值的 Tagged Pointer 通常会得到相同的位模式，此时 `a == b` 可能为真。

换成装不下的值后，两次创建可能得到两个不同的堆对象，此时 `a == b` 又可能为假。

所以比较 `NSNumber`、`NSString` 的内容，应该使用值语义：

```objc
[a isEqual:b]
```

不要让“某些小值上 `==` 恰好能用”骗过你。

### 6. 两个调试陷阱

第一，`malloc_size == 0` 不能单独证明它是 Tagged Pointer。字符串字面量和某些数字字面量可能是编译期常量对象，它们同样不来自 malloc。

第二，不要只用被编译器折叠的字面量验证：

```objc
id fromLiteral = @(42);       // 可能被编译成常量对象

int value = 42;
id fromRuntime = @(value);    // 更适合观察运行期装箱结果
```

实验时同时看类名、Tagged Pointer 标记和内存区域，不要只看一个指标。

---

## 五、现在回头看，三部分其实是一条线

```text
指针变量
   ↓ 保存地址
实例对象（isa + ivar）
   ↓ isa
类对象
   ↓ isa
元类
```

- 类型判断是在类链或元类链上寻找目标；
- 对象大小是在计算 `isa + ivar + 对齐`；
- Tagged Pointer 跳过普通实例对象，把类型和值直接编码进“指针”本身。

Tagged Pointer 不是突然冒出来的黑魔法。它只是对“小对象也要分配一块堆内存”这件事做了优化。

---

## 六、先用这几道题检查自己

### 题 1：`isKindOfClass:` 和 `isMemberOfClass:` 有什么区别？

`isKindOfClass:` 会沿父类链查找，判断“是不是这个类或它的子类”；`isMemberOfClass:` 只比较当前类，判断“是不是正好这个类”。

### 题 2：`class_getInstanceSize` 和 `malloc_size` 为什么可能不同？

前者是对象布局至少需要的大小，后者是分配器实际给出的内存块大小。分配粒度可能让后者更大。

### 题 3：为什么空的 `NSObject` 也不是 0 字节？

普通实例至少要保存 `isa`。在 64 位环境里，光这个指针就占 8 字节。

### 题 4：Tagged Pointer 的“指针”还是真正的地址吗？

不是普通对象地址。它的位模式直接编码了类型和值，因此没有对应的普通堆对象可供解引用。

### 题 5：为什么不能用 `==` 比较两个 `NSNumber` 的值？

`==` 比较的是位模式或地址。小值恰好使用 Tagged Pointer 时可能相等，大值退回不同堆对象后又可能不等。值比较应使用 `isEqual:`。

---

## 七、进阶：理解以后再看源码和实验

这一部分是为了回答“上面的结论从哪里来”，不是第一次阅读的必背内容。

### 1. 四个类型判断实现的起点

```objc
+ (BOOL)isMemberOfClass:(Class)cls {
    return self->ISA() == cls;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}

+ (BOOL)isKindOfClass:(Class)cls {
    for (Class current = self->ISA(); current; current = current->getSuperclass()) {
        if (current == cls) return YES;
    }
    return NO;
}

- (BOOL)isKindOfClass:(Class)cls {
    for (Class current = [self class]; current; current = current->getSuperclass()) {
        if (current == cls) return YES;
    }
    return NO;
}
```

不用背代码，只看起点：

| receiver | 判断起点 |
| --- | --- |
| 实例对象 | 它的类 |
| 类对象 | 它的元类 |

现代 Runtime 还可能通过 `objc_opt_isKindOfClass` 走快路径，直接读取 isa。若相关核心方法被自定义，例如 KVO 中间类覆写 `class`，Runtime 可能退回正常消息发送。这个优化不改变前面讲的判断结果。

### 2. 当前 arm64 Tagged Pointer 布局

```text
bit 63      Tagged Pointer 标记
bit 3~62    payload
bit 0~2     标签索引
```

普通标签不够用时还有扩展标签方案。Intel Mac 上也曾使用低位标记方案，所以“Tagged Pointer 永远看最低位”或“永远看最高位”都不可靠。

业务代码应该使用 Runtime 提供的对象语义，不要自己硬编码位掩码解码。

### 3. 为什么重新启动后指针会变

Runtime 会在进程启动时使用混淆值和标签置换表处理 Tagged Pointer。于是同一个值：

- 在同一次进程运行中，位模式保持一致；
- 重新启动进程后，位模式可能改变。

这再次说明不能把某次实验打印出的十六进制数写进业务判断。

### 4. 几个会暴露实现差异的边界

这些现象知道即可，不建议拿来设计业务逻辑：

- **弱引用**：Tagged Pointer 不会像普通对象那样销毁，weak 不会因为它“释放”而自动变成 `nil`；
- **关联对象**：相同 tagged 值可能共享同一位模式，而且没有普通 `dealloc` 时机；
- **并发属性写入**：短字符串碰巧走 tagged 路径，不代表 `nonatomic` 属性就线程安全；
- **LLDB 内存命令**：对 Tagged Pointer 使用 `memory region` 或按普通地址读取会失败。

### 5. Swift 的小字符串不是同一套实现

Swift 原生 `String` 有自己的 small-string 优化。只有桥接到 `NSString` 时，才可能进入 Objective-C 的 Tagged Pointer 路径。

思想相似：值足够小时，尽量不做额外堆分配；具体布局不是同一套。

---

## 八、稳定语义和实现细节要分开

| 写代码时可以依赖 | 只用于理解当前实现 |
| --- | --- |
| 实例能找到自己的类，类有父类关系 | nonpointer isa 的具体位布局 |
| `isKindOfClass:` 会考虑继承关系 | 当前是否走快路径 |
| `isMemberOfClass:` 只判断当前类 | 当前内部类名和打印地址 |
| 实例变量遵循对齐规则 | 当前普通对象的具体分配粒度 |
| Tagged Pointer 没有普通堆对象 | 标记位位置、标签编号和混淆算法 |
| 对象值应使用值语义比较 | 某个具体值是否会 tagged |

实验可以帮助理解实现，但业务代码不能把实验中的位布局、类名和阈值当成永久规则。

---

## 一句话面试答案

**`isKindOfClass:` 和 `isMemberOfClass:` 的区别？**

> `isKindOfClass:` 会沿 superclass 查找，能匹配当前类和父类；`isMemberOfClass:` 只比较当前类。receiver 是类对象时，判断从元类开始，所以会出现一些反直觉结果。

**一个 Objective-C 对象有多大？**

> 要区分对象布局大小和分配器实际块大小。`class_getInstanceSize` 计算 isa、ivar 和对齐后的需求，`malloc_size` 反映实际分配结果，后者可能因为分配粒度更大。

**什么是 Tagged Pointer？**

> 对足够小的值，Runtime 把类型和值直接编码进指针位，不创建普通堆对象，从而省掉堆分配、普通引用计数和销毁路径。具体位布局和可编码范围属于实现细节。

---

## 参考资料

### 官方与源码

- [Apple — NSObject class and type information](https://developer.apple.com/documentation/objectivec/nsobject)
- [WWDC20 — Advancements in the Objective-C runtime](https://developer.apple.com/videos/play/wwdc2020/10163/)
- [objc4 — objc-internal.h](https://github.com/apple-oss-distributions/objc4)：Tagged Pointer 标签和平台分支
- [objc4 — objc-runtime-new.mm](https://github.com/apple-oss-distributions/objc4)：类链、快路径和 Tagged Pointer 混淆逻辑
- [objc4 — objc-runtime-new.h](https://github.com/apple-oss-distributions/objc4)：对象分配大小

### 延伸阅读

- [Always Processing — Objective-C Tagged Pointers](https://alwaysprocessing.blog/2023/03/19/objc-tagged-ptr)
- [Mike Ash — Tagged Pointer Strings](https://mikeash.com/pyblog/friday-qa-2015-07-31-tagged-pointer-strings.html)
- [Andrew Madsen — Constant Literals in Objective-C](https://blog.andrewmadsen.com/2021/06/07/constant-literals-in.html)

### 本地笔记

- [[20 专题笔记/Runtime/Part 1 - 对象与类的本质]]
- [[iOS 内存：从虚拟地址空间到堆与栈]]
- [[iOS 内存管理：从 MRC、ARC 到属性关键字#第一部分：MRC 的所有权规则：retain、release 与 autorelease|MRC 的所有权规则]]

---

原实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -fobjc-arc -target arm64-apple-ios17.0-simulator`。

Tagged Pointer 的位模式、混淆方式、内部类名和可编码范围都可能随平台与系统版本变化。本文保留实验数据用于解释机制，不把它们当成跨版本承诺。
