---
title: 【iOS】MRC 的所有权规则：retain、release 与 autorelease
published: 2026-07-26
description: 从 Apple 的四条所有权规则出发，说明 MRC 中 retain、release 与 autorelease 的使用方式，并结合当前 objc4 源码和实验观察引用计数及对象销毁过程。
tags:
  - iOS
  - Objective-C
  - Memory
  - MRC
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 4
draft: true
---
# MRC 的所有权规则：retain、release 与 autorelease

MRC 的全称是 Manual Reference Counting，即手动引用计数。在 MRC 下，开发者需要明确表示什么时候开始持有一个对象，以及什么时候不再需要它。

#### dealloc 方法

- 当一个对象的引用计数器值为 0 时，这个对象即将被销毁，其占用的内存被系统回收。
- 对象即将被销毁时系统会自动给对象发送一条 `dealloc` 消息（因此，从 `dealloc` 方法有没有被调用，就可以判断出对象是否被销毁）
- `dealloc` 方法的重写（**注意是在 MRC 中**）
    - 一般会重写 `dealloc` 方法，在这里释放相关资源，`dealloc` 就是对象的遗言
    - 一旦重写了 `dealloc` 方法，就必须调用 `[super dealloc]`，并且放在最后面调用。

> `dealloc` 使用注意：

- 不能直接调用 `dealloc` 方法。
- 一旦对象被回收了, 它占用的内存就不再可用，坚持使用会导致程序崩溃（野指针错误）。

#### 3.4 野指针和空指针

- 只要一个对象被释放了，我们就称这个对象为「僵尸对象（不能再使用的对象）」。
- 当一个指针指向一个僵尸对象（不能再使用的对象），我们就称这个指针为「野指针」。
- 只要给一个野指针发送消息就会报错（EXC_BAD_ACCESS 错误）。


```javascript
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *p = [[Person alloc] init]; // 执行完引用计数为 1。

        [p release]; // 执行完引用计数为 0，实例对象被释放。
        [p release]; // 此时，p 就变成了野指针，再给野指针 p 发送消息就会报错。
        [p release]; // 报错
    }
    return 0;
}
```

- 为了避免给野指针发送消息会报错，一般情况下，当一个对象被释放后我们会将这个对象的指针设置为空指针。
- 空指针：
    - 没有指向存储空间的指针（里面存的是 nil, 也就是 0）。
    - 给空指针发消息是没有任何反应的。

```javascript
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *p = [[Person alloc] init]; // 执行完引用计数为 1。

        [p release]; // 执行完引用计数为 0，实例对象被释放。
        p = nil; // 此时，p 变为了空指针。
        [p release]; // 再给空指针 p 发送消息就不会报错了。
        [p release];
    }
    return 0;
}
```

## 一、四条规则

Apple 的 [Memory Management Policy](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html) 给出了四条所有权规则。

### 规则一：自己创建的对象，自己拥有，自己负责释放

> **You own any object you create.** You create an object using a method whose name begins with "alloc", "new", "copy", or "mutableCopy" (for example, `alloc`, `newObject`, or `mutableCopy`).

通过 `alloc`、`new`、`copy`、`mutableCopy` 方法族取得对象时，调用方拥有该对象。

>（由各类实现的 `copyWithZone:` 方法和 `mutableCopyWithZone:` 方法将生成并持有对象的副本。）

另外，由上面四种方法名称开头的方法名，也将生成并持有对象：

- `allocMyObject`
- `newMyObject`
- `copyMyObject`
- `mutableCopyMyObject`

对应的内存管理思想是：**自己生成的对象，自己持有。**

这里的“拥有”表示调用方取得了一份需要自己负责的所有权。当这份所有权不再需要时，调用方必须执行一次对应的 `release` 或 `autorelease`。

判断依据是方法名所属的家族，而不是方法内部到底有没有重新分配一块内存。例如，不可变对象执行 `copy` 时可能直接返回自己，但调用方仍然拥有 `copy` 方法族返回的对象。

### 规则二：不是自己创建的对象，也可以取得所有权

> **You can take ownership of an object using retain.** You use `retain` in two situations: in the implementation of an accessor method or an `init` method, to take ownership of an object you want to store as a property value; and to prevent an object from being invalidated as a side-effect of some other operation.

有些方法会返回一个调用方不拥有的对象。调用方可以暂时使用它；如果需要跨越当前作用范围继续保存，就可以调用 `retain`，主动取得一份所有权。既然取得了这份所有权，以后也必须负责将它释放。

### 规则三：不再需要时，放弃自己的所有权

> **When you no longer need it, you must relinquish ownership of an object you own.** You relinquish ownership of an object by sending it a `release` message or an `autorelease` message.

Apple 的规则是：不再需要自己拥有的对象时，必须调用 `release` 或 `autorelease` 放弃所有权。

对应的内存管理思想是：**不再需要自己持有的对象时，释放它。**

一次 `release` 只表示放弃自己的一份所有权，不等于要求对象立即销毁。如果还有其他所有者，对象仍然会继续存在；只有最后一份所有权也被放弃，引用计数归零时，对象才会进入销毁流程。

`autorelease` 放弃的也是一份所有权，只是把对应的 `release` 推迟到自动释放池处理时执行。

### 规则四：不能放弃不属于自己的所有权

> **You must not relinquish ownership of an object you do not own.**

Apple 的规则是：不要释放自己并不拥有的对象。

对应的内存管理思想是：**非自己持有的对象，不能释放。**

这里的“不能释放”不等于“不能使用”。调用方可以在对象仍然有效的时间内使用一个自己不拥有的对象，只是不能直接对它执行 `release`。如果确实需要长期保存，应先通过 `retain` 取得自己的所有权，再在不需要时放弃这份所有权。



是否拥有返回对象，依据的是方法命名约定，而不是方法内部是否真的创建了新对象。`newObject` 也符合规则，因为它的名字以 `new` 开头。

下面的方法可能返回一个新字符串，但调用方没有取得所有权：

```objc
NSString *a = [[NSString alloc] initWithFormat:@"%d", 1];  // alloc 开头，我拥有，必须 release
NSString *b = [a stringByAppendingString:@"x"];            // 前缀不匹配，我不拥有
[b release];                                       // 错误：释放了不拥有的对象
```

`stringByAppendingString:` 不属于上述四个方法族，因此调用方不拥有它的返回值。调用方可以使用 `b`，但不能直接对它执行 `release`；如果需要长期保存，应先 `retain`。

反过来，`copy` 可能返回原对象，但调用方仍然取得了所有权：

```objc
NSString *s = @"abc";
NSString *c = [s copy];   // 前缀是 copy，我拥有它，必须 release
                          // 对不可变对象，c 可能与 s 是同一个对象
```

不可变对象执行 `copy` 时可以直接返回自身并增加引用计数。即使 `c == s`，调用方仍然拥有 `copy` 的返回值，之后必须执行一次 `release`。

这两个例子说明：所有权规则描述的是调用方与 API 之间的约定，与方法内部是否分配新对象没有直接关系。

MRC 下 `@property (copy)` 的 setter 也遵循同一规则。调用 `[newValue copy]` 后，setter 拥有返回的对象，因此属性所属对象销毁时，需要对保存的值执行 `release`。

ARC 仍然使用这些方法族判断返回值的所有权，只是对应的引用计数操作改由编译器生成。

### Core Foundation 的所有权规则

> Objective-C 是一门编程语言。
> 
>Core Foundation，简称 CF，是苹果提供的一套用 C 语言写的基础框架。
>
>关系大概是：
>iOS / macOS 应用  
↓  
Foundation：Objective-C 风格  
↓  
Core Foundation：C 语言风格  
↓  
更底层的系统能力

Core Foundation 使用类似的命名约定：

- **Create Rule**：函数名中包含 `Create` 或 `Copy` 时，调用方通常拥有返回值，并负责 `CFRelease`；
- **Get Rule**：函数名中包含 `Get` 时，调用方通常不拥有返回值，不能直接释放，除非另外取得了所有权。

Objective-C 对象与 Core Foundation 对象桥接时，三个关键字分别表示：

- `__bridge`：只转换类型，不转移所有权；
- `__bridge_retained`：转换后由 CF 一侧持有，之后需要 `CFRelease`；
- `__bridge_transfer`：把 CF 一侧已有的所有权交给 ARC 管理。

判断桥接代码时，核心仍然是确认转换完成后由哪一侧负责释放对象。

---

## 二、MRC 怎样保存一个对象

前面讨论的是“单个对象”的内存管理：谁创建对象，谁负责在不需要时释放；如果通过 `retain` 获得了对象的所有权，也需要在之后对应一次 `release`。

但实际开发中更常见的是“一个对象保存另一个对象”。这部分可以结合 [iOS 开发：彻底理解 iOS 内存管理（MRC 篇）](https://itcharge.cn/blogs/tech/ios/memory-management-01/#_3-5-2-%E5%A4%9A%E4%B8%AA%E5%AF%B9%E8%B1%A1%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%80%9D%E6%83%B3) 的 3.5.2～3.5.6 阅读。

例如，`Box` 内部需要保存一个字符串：

```objc
@interface Box : NSObject {
    NSString *_name;
}

- (void)setName:(NSString *)name;
@end
```

这里实际上存在两个对象：

```text
Box 对象
   ↓ 持有
NSString 对象
```

这就从“单个对象内存管理”进入了“多个对象之间的内存管理”。

### 1. 为什么不能直接赋值

最简单的 setter 可能会写成：

```objc
- (void)setName:(NSString *)name {
    _name = name;
}
```

但这样只是让 `_name` 指向 `name`，并没有取得这个 `NSString` 对象的所有权。

```objc
NSString *str = [[NSString alloc] initWithString:@"Tom"];

[box setName:str];

[str release];
```

`str` 是外部通过 `alloc` 创建的，所以外部拥有这个对象。如果 `Box` 的 setter 只是简单执行 `_name = name`，那么外部执行 `[str release]` 后，字符串就可能被销毁，而 `_name` 仍然保存原来的地址。此时 `_name` 就成了野指针。

因此，如果 `Box` 想长期保存这个对象，就必须取得一份属于自己的所有权：

```objc
- (void)setName:(NSString *)name {
    [name retain];
    _name = name;
}
```

此时可以把持有关系理解为：

```text
外部 ───────→ NSString
                ↑
Box  ───────────┘
```

外部和 `Box` 各自拥有一份所有权。所以，即使外部执行 `[str release]`，也只是放弃了外部自己的所有权；`Box` 仍然持有字符串，对象不会被销毁。

### 2. 换成新对象时，释放旧对象

但是，上面的 setter 还有一个问题。假设 `Box` 原来保存的是：

```text
Box → NSString A
```

现在执行：

```objc
box.name = stringB;
```

`Box` 不再需要 `NSString A`，准备改为保存 `NSString B`。这时不能只执行：

```objc
[name retain];
_name = name;
```

因为 `Box` 对旧对象 A 的那次 `retain` 还没有对应的 `release`。指针一旦改为指向 B，`Box` 就失去了释放 A 的机会，A 会发生内存泄漏。

所以 setter 还要负责放弃旧对象：

```objc
- (void)setName:(NSString *)name {
    [_name release];    // 不再持有旧对象

    [name retain];      // 持有新对象
    _name = name;
}
```

整个过程是：

```text
原来：
Box ──retain──> NSString A

重新赋值：
box.name = stringB;

第一步：[_name release]
Box 放弃 NSString A

第二步：[name retain]
Box 取得 NSString B 的所有权

第三步：_name = name
Box ──retain──> NSString B
```


### 3. 还要考虑“自己赋值给自己”

上面的写法还存在最后一个问题。

假设第一次执行：

```objc
box.name = str;
```
之后又执行：
```objc
box.name = str;
```

此时 `_name == name`，也就是说“旧对象”和“新对象”其实是同一个对象。如果仍然按照下面的顺序执行，就存在风险：

```objc
[_name release];
[name retain];
```

如果这个字符串当前只剩 `Box` 的这一份所有权，`[_name release]` 就可能让引用计数变成 0，对象随即被销毁。接下来的 `[name retain]` 就是在向已经销毁的对象发送消息，此时 `name` 已经是野指针。

因此，setter 在修改之前，要先判断新旧对象是不是同一个对象：

```objc
- (void)setName:(NSString *)name {
    if (_name != name) {
        [_name release];
        [name retain];
        _name = name;
    }
}
```

只有新旧对象不同的时候，才需要执行一次“旧对象 `release`、新对象 `retain`”。

### 4. MRC setter 最终形式

`retain` 在增加引用计数的同时，还会返回对象本身，因此下面两行：

```objc
[name retain];
_name = name;
```

可以合并为：

```objc
_name = [name retain];
```

最终的 setter 可以写成：

```objc
- (void)setName:(NSString *)name {
    if (_name != name) {
        [_name release];
        _name = [name retain];
    }
}
```

它的执行顺序是：

```text
1. 判断新旧对象是不是同一个对象

2. release 旧对象
   → 放弃旧对象的所有权

3. retain 新对象
   → 取得新对象的所有权

4. _name 指向新对象
```

这里之所以可以先 `release`、再 `retain`，是因为前面的 `_name != name` 已经保证旧对象和新对象不是同一个对象。因此，不会出现“`release` 销毁了新对象，随后又对野指针执行 `retain`”的问题。

### 5. Box 自己销毁时也要 release

setter 中的 `_name = [name retain]` 意味着 `Box` 取得了这个对象的一份所有权。根据 MRC，`retain` 取得的所有权必须用一次对应的 `release` 放弃。

所以 `Box` 自己销毁时，也必须放弃对 `_name` 的所有权：

```objc
- (void)dealloc {
    [_name release];
    [super dealloc];
}
```

完整的 MRC 对象持有关系可以总结为：

```text
Box 开始保存对象
        ↓
retain 新对象

Box 更换对象
        ↓
release 旧对象
retain 新对象

Box 自己销毁
        ↓
release 当前持有的对象
```

最终的核心思想只有一句：

> MRC 中，一个对象如果想长期保存另一个对象，就必须取得它的所有权；不再需要这个对象时，就必须放弃对应的所有权。

也就是：

```text
要持有 → retain
不持有 → release

retain 一次
就必须有一次对应的 release
```

这就是多个对象之间进行 MRC 内存管理的核心。这里不是无条件记忆“先 `release`”或“先 `retain`”，而是因为这一版的最终方案先用 `_name != name` 排除了自赋值，所以后面可以采用“先释放旧对象，再持有新对象”的顺序。

> 补充：ARC 下不再手写这些 `retain` 和 `release`，但“保存新对象、放弃旧对象”的所有权变化仍然存在，通常由编译器和 Runtime 完成。

### 补充：MRC 代码里为什么也有 `objc_retain`

把 MRC 版本编译成 LLVM IR，里面确实有 `objc_retain` / `objc_release` / `objc_autorelease`：

```text
   5 @objc_release
   3 @objc_retain
   3 @objc_alloc_init
   2 @objc_msgSendSuper
   2 @objc_autorelease
```

这些调用与源码中手写的 `[x retain]`、`[x release]` 和 `[x autorelease]` 一一对应。Clang 识别这些特殊选择器后，可以直接生成相应的 Runtime 函数调用，而不必统一通过 `objc_msgSend`。这只是调用形式的优化，不代表编译器替开发者决定了对象应该在哪里释放。

因此，MRC 和 ARC 的区别不是编译产物中有没有 `objc_retain` 或 `objc_release`，而是谁负责决定这些调用出现的位置：MRC 由开发者明确写出，ARC 由编译器根据所有权语义生成。

---

## 三、autorelease 解决什么问题

不属于 `alloc`、`new`、`copy`、`mutableCopy` 方法族的方法，通常需要返回一个调用方不拥有的对象。下面的工厂函数先创建了一个自己拥有的对象：

```objc
Box *makeBox(void) {
    Box *b = [[Box alloc] init];   // 创建方拥有 b
    return b;
}
```

如果直接返回 `b`，创建方没有放弃所有权，会造成泄漏；如果在返回前立即 `release`，调用方拿到对象之前它就可能已经销毁。`autorelease` 用于推迟这次 `release`：

```objc
Box *makeBox(void) {
    Box *b = [[Box alloc] init];
    return [b autorelease];   // 调用方不拥有返回对象，release 将在之后执行
}
```

调用方可以在当前有效期内使用返回对象；如果需要让它继续存活，应当调用 `retain` 取得自己的所有权。

延迟的 `release` 通常在对应自动释放池排空时执行，第五节会继续说明池的结构和排空时机。

---

## 四、引用计数存在哪里

前面说明了引用计数怎样变化。本节继续看当前 objc4 怎样保存这项数据。这里涉及的是私有实现，不同架构和 Runtime 版本可能采用不同布局。

### nonpointer isa

早期的 `isa` 主要保存类指针，引用计数需要借助对象外部的数据结构维护。引入 non-pointer isa 后，Runtime 开始利用一个机器字中的部分位保存类信息、引用计数和对象状态。

在支持内联引用计数的平台上，`retain` 的快速路径会直接更新 isa 中的计数字段。ARM64 实现可以通过 `ldxr` / `stxr` 等独占访问指令完成原子更新。只有内联字段无法继续保存时，额外计数才会转移到 SideTable。

内联字段的宽度取决于当前架构分支。

### 两个分支，两套位宽

objc4 的 `isa.h` 里，同一个 `#if __arm64__` 下面有两个分支。

带指针认证的（也包括所有 iOS 模拟器）：

```c
#define ISA_HAS_CXX_DTOR_BIT 0
#define ISA_BITFIELD                        \
    uintptr_t nonpointer        : 1;        \
    uintptr_t has_assoc         : 1;        \
    uintptr_t weakly_referenced : 1;        \
    uintptr_t shiftcls_and_sig  : 52;       \
    uintptr_t has_sidetable_rc  : 1;        \
    uintptr_t extra_rc          : 8
#define RC_HALF              (1ULL<<7)      // = 128
```

不带指针认证的：

```c
#define ISA_HAS_CXX_DTOR_BIT 1
#define ISA_BITFIELD                        \
    uintptr_t nonpointer        : 1;        \
    uintptr_t has_assoc         : 1;        \
    uintptr_t has_cxx_dtor      : 1;        \
    uintptr_t shiftcls          : 33;       \
    uintptr_t magic             : 6;        \
    uintptr_t weakly_referenced : 1;        \
    uintptr_t unused            : 1;        \
    uintptr_t has_sidetable_rc  : 1;        \
    uintptr_t extra_rc          : 19
#define RC_HALF              (1ULL<<18)     // = 262144
```

无指针认证的 arm64 分支为 `extra_rc` 保留了 19 位；带指针认证的分支需要保存类指针及其签名，因此 `extra_rc` 只保留 8 位。

ARM64 模拟器也使用第一种布局，`isa.h` 中的说明是：

> ARM64 simulators have a larger address space, so use the ARM64e scheme even when simulators build for ARM64-not-e.

`__x86_64__` 分支中的 `extra_rc` 也是 8 位，`RC_HALF` 同样是 `1ULL << 7`。因此，“`extra_rc` 固定占 19 位”并不是通用结论；19 位对应的是无指针认证的 arm64 分支。

选择哪种布局取决于 **libobjc 自身的构建配置**，其中会判断 `__has_feature(ptrauth_calls)`，而不是只看 App 的架构 slice。普通 App 的 slice 可以是 arm64，但设备中实际使用的 libobjc 仍可能采用 arm64e 对应的布局。

### 实测

下面的实验按照第一种位域布局解析 isa，并使用 `CFGetRetainCount` 作为辅助对照：

```objc
static void probe(id obj, const char *tag) {
    uintptr_t isa = *(uintptr_t *)(__bridge void *)obj;
    unsigned long extra_rc  = (isa >> 56) & 0xff;
    unsigned long sidetable = (isa >> 55) & 0x1;
    long cf = CFGetRetainCount((CFTypeRef)obj);
    printf("%-16s extra_rc=%-4lu  has_sidetable_rc=%lu  CFGetRetainCount=%-4ld  差值=%ld\n",
           tag, extra_rc, sidetable, cf, cf - (long)extra_rc);
}
```

```text
刚创建                extra_rc=2     has_sidetable_rc=0  CFGetRetainCount=2     差值=0
retain 1 次           extra_rc=3     has_sidetable_rc=0  CFGetRetainCount=3     差值=0
retain 252 次         extra_rc=254   has_sidetable_rc=0  CFGetRetainCount=254   差值=0
retain 253 次         extra_rc=255   has_sidetable_rc=0  CFGetRetainCount=255   差值=0
retain 254 次（溢出）  extra_rc=128   has_sidetable_rc=1  CFGetRetainCount=256   差值=128
```

实验中，`extra_rc` 增长到 255 后无法继续增加。下一次 `retain` 触发溢出，`has_sidetable_rc` 变为 1，`extra_rc` 调整为 128。

对应的 isa 位值如下：

```text
0x80 80 0001efeba749
  │   └── bit55 = 1      → has_sidetable_rc
  └────── bit56-63 = 0x80 = 128 = RC_HALF
```

128 对应源码中的 `RC_HALF`。发生溢出时，Runtime 不会把全部计数移出 isa，而是保留一部分内联计数，并把另一部分转移到 SideTable。之后执行 `release` 时，只要 `extra_rc` 仍有可减的值，就不需要访问 SideTable。

### 当前版本中 `extra_rc` 的含义

实验表格中，溢出前的 `extra_rc` 与 `CFGetRetainCount` 一致。当前源码的 `initIsa` 也会把初始值设为 1：

```c
#if ISA_HAS_INLINE_RC
        newisa.extra_rc = 1;
#endif
```

`rootRetainCount()` 读取时也不再额外加 1：

```c
uintptr_t rc = bits.extra_rc;
if (bits.has_sidetable_rc) { rc += sidetable_getExtraRC_nolock(); }
return rc;
```

因此，在本文使用的 objc4 版本中，`extra_rc` 直接参与表示当前引用计数，而不是旧资料中常见的 `retainCount - 1`。当前位域里也不再单独保存 `deallocating`；Runtime 可以通过下面的条件判断对象是否正在析构：

```cpp
bool isDeallocating() const {
    return extra_rc == 0 && has_sidetable_rc == 0;
}
```

上面列出的两个位域分支中都没有独立的 `deallocating` 字段。

表格中“刚创建”的结果为 2，是因为实验使用 `-O0` 构建。ARC 在把对象传入 `probe` 时额外保留了一次；开启优化后，这类临时引用可能被消除。因此，这个数字用于说明本次实验的调用路径，不能当作所有新对象的固定初始观测值。

### 为什么不应使用 `retainCount` 调试生命周期

`CFGetRetainCount` 在上面的受控实验中只用于对照位域变化，不适合判断业务对象何时销毁。

先看 Tagged Pointer。`rootRetainCount()` 会先判断对象是不是 Tagged Pointer：

```c
if (isTaggedPointer()) return (uintptr_t)this;
```

对 Tagged Pointer 进行测试时，得到的结果如下：

```text
tagged 指针值             = 0xb68cd92197b5cfee
tagged  [obj retainCount] = 9223372036854775807
tagged  CFGetRetainCount  = 9223372036854775807
        retainCount 是否 == 指针值? 0
堆对象  [obj retainCount] = 1
```

这里返回 `INT64_MAX`，而不是 Tagged Pointer 的位值。原因是 `NSNumber` 属于与 Core Foundation 桥接的类簇，可以使用自定义的引用计数实现，并不一定经过 `objc_object::rootRetainCount()`。无论进入哪条实现路径，Tagged Pointer 都没有需要通过引用计数管理的独立对象本体，因此该返回值不能解释为真实持有次数。

此外，`retainCount` 无法反映自动释放池中尚未执行的 `release`，常量对象和单例也可能返回特殊的大数。即使当前结果是 1，也不能据此判断对象会在什么时刻销毁。

ARC 禁止显式调用 `retain`、`release`、`autorelease`、`retainCount` 和 `dealloc`，这些限制在 Clang ARC 规范中有明确说明。ARC 代码应通过所有权关系和工具观察生命周期，而不是依赖某个瞬时引用计数值。

> 上面这组数据来自 iOS 模拟器（arm64）。从 objc4-951.1 的条件编译可以确认，模拟器与启用指针认证的 arm64 运行时选择同一组位域定义；真机上的具体结果仍需单独实验。

---

## 五、autorelease 池的实现

在当前 objc4 实现中，自动释放池可以理解为一个按页组织的指针栈。栈中的条目要么是等待接收 `release` 的对象，要么是表示池边界的 `POOL_BOUNDARY`；多个页面通过双向链表连接。

`POOL_BOUNDARY` 的值是 `nil`。`objc_autoreleasePoolPush` 会把这个边界压入栈中，并返回边界所在位置；`objc_autoreleasePoolPop` 根据该位置向回清理，对沿途记录的对象执行 `release`。嵌套自动释放池会形成多个边界，每次 `pop` 只处理对应边界以内的记录。

页面通过 `parent` 和 `child` 组成双向链表，线程本地存储保存当前正在使用的页面。`autoreleaseFast` 根据页面状态分成三种情况：

```c
static inline id *autoreleaseFast(id obj) {
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);                 // 热页有空间
    } else if (page) {
        return autoreleaseFullPage(obj, page);  // 热页满了，找或建下一页
    } else {
        return autoreleaseNoPage(obj);          // 一页都还没有
    }
}
```

执行 `pop` 后，Runtime 会用 `SCRIBBLE = 0xA3` 覆盖已经处理的槽位。在 LLDB 中查看刚清理过的页面，可以观察到相应的填充值。

页面大小在源码中不是直接写成 4096，而是由 `PROTECT_AUTORELEASEPOOL ? PAGE_MAX_SIZE : PAGE_MIN_SIZE` 决定。公开的 objc4 构建没有启用 `PROTECT_AUTORELEASEPOOL`，所以通常使用 `PAGE_MIN_SIZE`，其结果是 4096 字节。应把 4096 理解为当前配置计算出的结果，而不是结构中不变的字面常量。

### 当前实现中的三项优化

早期文章主要介绍页链表和边界。当前 Runtime 还包含下面三项优化，因此新系统上的观察结果可能与旧文章不同。

**空池占位符。** 有些自动释放池从创建到销毁都没有记录对象。此时 Runtime 可以把热页指针设置为占位值 `1`，暂不分配页面；直到第一个对象需要加入池时，再创建页面并补上边界。

**相邻指针合并。** `SUPPORT_AUTORELEASEPOOL_DEDUP_PTRS` 启用后，Runtime 会检查栈顶附近是否已经记录了同一个对象。如果命中，就在已有条目中累计次数，而不是重复保存相同指针；`pop` 时再按累计次数执行 `release`。

**返回值优化。** `objc_autoreleaseReturnValue` 会尝试判断调用方是否马上需要持有返回对象。如果调用方能够直接接收所有权，对象就不必先加入自动释放池，再被调用方重新 `retain`。较早的方案会检查调用方的指令序列或哨兵 NOP；objc4-951.1 在支持 `HAS_RETURNADDR_AUTORELEASE_ELISION` 时，可以通过返回地址完成这次交接。`objc_claimAutoreleasedReturnValue` 还提供了不依赖调用方 NOP 的入口。具体过程放在 [[iOS 内存：ARC 的两半#返回值的所有权交接|ARC 的两半]] 中讨论。

从语义上看，`autorelease` 表示延后放弃所有权；从当前实现看，返回值优化可能使部分调用不必真的进入池页。两者描述的是语义和实现两个层次。

### 池什么时候排空

主线程的事件循环会通过 RunLoop observer 管理自动释放池：进入循环时建立池，在即将休眠时清理并重新建立，退出时再次清理。加入相应池中的对象会在边界清理时收到 `release`；如果还有其他所有者，对象仍会继续存活。

辅助线程也需要自动释放池。系统框架或 GCD 可能建立和清理内部池，但具体时机不属于业务代码可以依赖的契约。长时间运行并产生大量临时对象的任务，应在合适的循环或任务边界中显式使用 `@autoreleasepool`，以控制内存峰值。

如果一个对象在完全没有池的上下文里被 autorelease，运行时会打日志：

```text
Object 0x... of class ... autoreleased with no pool in place - just leaking
```

对应的源码入口是 `BREAKPOINT_FUNCTION(void objc_autoreleaseNoPool(id obj))`。调试时可以在 `objc_autoreleaseNoPool` 上设置断点，定位没有自动释放池的调用位置。

在循环中增加 `@autoreleasepool` 可能降低临时对象造成的内存峰值，但它本身也有 push、pop 和页面管理成本。是否需要增加，应根据循环是否持续产生大量 autoreleased 对象以及实际测量结果决定。图片加载、字符串格式化和 JSON 解析是常见的观察场景。

---

## 六、最后一次 `release` 之后

```
最后一次 release，引用计数归零
      │  isa 进入 deallocating 状态（extra_rc == 0 && has_sidetable_rc == 0）
      ▼
objc_msgSend(this, @selector(dealloc))     ← 真正的消息发送，可被覆写、可被 NSZombie 替换
      │
-[NSObject dealloc] → _objc_rootDealloc(self)
      │
rootDealloc()
      │
      ├─ 快速路径：nonpointer && !weakly_referenced && !has_assoc
      │            && 没有 C++ 析构 && !has_sidetable_rc
      │            → free(this)，直接结束
      │
      └─ 慢速路径：_object_dispose_nonnull_realized
                    → objc_destructInstance_nonnull_realized
                        ① object_cxxDestruct   （执行 .cxx_destruct，释放 ARC 托管的 ivar）
                        ② 移除关联对象
                        ③ clearDeallocating    （清空所有弱引用、清理 SideTable 残留）
                    → free
```

如果对象没有被弱引用、没有关联对象、没有 C++ 析构逻辑，而且引用计数没有进入 SideTable，Runtime 可以直接释放对象内存。否则需要进入慢速路径，逐项清理对象相关状态。

这些判断并不一定全部来自 isa。前面列出的 ptrauth 和模拟器分支中，`ISA_HAS_CXX_DTOR_BIT` 为 0，位域不包含 `has_cxx_dtor`，因此源码需要根据平台选择不同的查询方式：

```cpp
#if ISA_HAS_CXX_DTOR_BIT
                 !isa().has_cxx_dtor                  &&
#else
                 !isa().getClass(false)->hasCxxDtor() &&
#endif
```

在 x86_64 和无指针认证的 arm64 布局中，可以读取 isa 位域；在 arm64e 和对应的模拟器布局中，需要查询类对象中的标志。

慢速路径依次执行 `.cxx_destruct`、移除关联对象和 `clearDeallocating`。源码明确注明顺序不能改变，但没有在此处解释完整原因。可以确认的是，`.cxx_destruct` 执行时对象的 ivar 仍需保持有效；对更具体原因的解释应视为实现分析，而不是公开 API 保证。

ARC 下不需要手动释放 ivar，也不能调用 `[super dealloc]`。编译器会生成 `.cxx_destruct` 来清理 ARC 管理的 ivar，并在 `-dealloc` 的结束位置处理父类销毁流程。

如果尝试为正在销毁的对象建立新的弱引用，Runtime 会报告 `Cannot form weak reference to instance ... being deallocated`。具体判断和 weak table 的处理放在 [[iOS weak 的实现：SideTable 与置 nil 的时机]] 中说明。

### `dealloc` 中适合处理什么

Apple 在 [Practical Memory Management](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html) 中提醒开发者：文件描述符、网络连接、缓冲区等稀缺资源不应只依赖 `dealloc` 释放，也不应假设 `dealloc` 一定会在预想的时间执行。对象可能比预期更晚销毁，应用退出时也不能依赖每个对象都完成这条路径。

`dealloc` 的执行线程也取决于最后一个强引用在哪个线程被释放。如果最后一次 `release` 发生在后台线程，`dealloc` 也会在该线程执行。因此，不应在 `dealloc` 中执行要求主线程的 UIKit 操作。

`dealloc` 适合完成对象自身的必要收尾，例如解除仍需手动处理的注册关系。文件描述符、网络连接等需要确定关闭时机的资源，应提供显式的生命周期方法，不应只依赖对象最终销毁。

---

## 七、NSZombie 与它的替代品

对象被过度释放后，原来的指针不会自动清空。如果对应内存尚未被复用，错误可能暂时没有表现；一旦内存内容变化，后续消息发送才会崩溃。因此，实际崩溃位置可能晚于错误的释放位置。

`NSZombieEnabled` 会改变对象销毁后的处理方式。Foundation 没有公开这部分实现，下面的过程来自社区反汇编分析和重实现，不属于 Apple 公布的源码：

1. 按原类名找到或复制出一个 `_NSZombie_原类名` 的僵尸类，这个类没有父类、没有任何方法实现，并按类名缓存。
2. 调 `objc_destructInstance` 做析构（执行 `.cxx_destruct`、清关联对象、清弱引用），但不 free 内存。
3. 把对象的 isa 改指向僵尸类。
4. 内存留着不还。

之后再次向该地址发送消息时，Runtime 可以识别这是已经销毁的对象，并报告类似下面的错误：

```text
-[Person printName]: message sent to deallocated instance 0x108a08180
```

Zombie 的作用是让程序在第一次访问已销毁对象时就明确报错，便于定位使用已释放对象的问题。由于对象内存不会真正归还，开启 Zombie 会持续增加内存占用，只适合调试使用。

排查 use-after-free 时，还可以使用 **Address Sanitizer**。它能够报告非法访问，并提供与分配和释放相关的调用信息。`MallocStackLogging` 配合 `malloc_history <pid> <addr>` 可以查询指定地址的分配和释放记录。Zombie、Address Sanitizer 和 malloc history 的侧重点不同，应根据问题选择。

Instruments 的 Leaks 主要检测已经不可达、但仍未释放的内存。如果对象仍被强引用，Leaks 可能不会把它判断为泄漏；这类持续增长问题更适合使用 Allocations，通过 Generation 对比观察对象数量和存活情况。

`MallocScribble` 会用 `0x55` 填充已经释放的内存，便于在调试时识别释放后的内容。这与自动释放池清理槽位时使用 `0xA3` 的目的相似，但属于不同的调试机制。

---

## 八、需要结合版本理解的结论

- **“`extra_rc` 占 19 位。”** 这只适用于无指针认证的 arm64 分支。本文检查的 arm64e、iOS 模拟器和 x86_64 分支使用 8 位。
- **“isa 里有一个 `deallocating` 位。”** 本文检查的当前布局中没有这个独立字段。Runtime 使用 `extra_rc == 0 && has_sidetable_rc == 0` 判断对象是否正在析构。
- **“AutoreleasePoolPage 固定为 4096 字节。”** 4096 是本文所用构建配置的计算结果，源码通过平台宏选择页面大小，并没有把它写成不变的结构常量。
- **“`NSAutoreleasePool` 和 `@autoreleasepool` 只是语法不同。”** 两者都用于划定自动释放池的生命周期，但入口并不相同。`@autoreleasepool {}` 会被编译为 `objc_autoreleasePoolPush` / `objc_autoreleasePoolPop`；`NSAutoreleasePool` 则通过对象接口使用自动释放池。在引用计数环境中，向 `NSAutoreleasePool` 发送 `drain` 与发送 `release` 的效果相同。
- **“`autorelease` 一定会把对象加入池。”** 语义上它表示延后放弃所有权，但返回值优化可能避免对象真正进入池页，见第五节。
- **“过度释放一定会立刻崩溃。”** 对象销毁后，悬空指针仍然保存旧地址；错误可能在对应内存被复用后才表现出来。
- **“SideTable 是一张单独的全局哈希表。”** 当前实现使用 `StripedMap<SideTable>` 按对象地址分片，各分片独立加锁。具体分片数属于实现细节。
- **“`retain` 和 `release` 全部由 C/C++ 实现。”** 部分架构的快速路径直接写在汇编中，nil 和 Tagged Pointer 等情况可以提前返回，慢速路径才进入 C++ 实现。

---

## 总结

MRC 的核心是让每一份所有权都能够配平。通过 `alloc`、`new`、`copy`、`mutableCopy` 方法族取得对象时，调用方拥有返回对象；`retain` 会再取得一份所有权；不再需要时，要用 `release` 或 `autorelease` 放弃相应的所有权。判断依据是 API 的命名约定，而不是方法内部是否真的创建了新对象。Core Foundation 的 Create/Get Rule 遵循相同的思路。

在支持内联引用计数的当前实现中，常用计数优先保存在 non-pointer isa 的 `extra_rc` 中，空间不足时再把部分计数保存到 SideTable。本文在 arm64 模拟器上使用的布局中，`extra_rc` 占 8 位；实验观察到第 254 次额外 `retain` 触发溢出，并有 128 份计数转入 SideTable。无指针认证的 arm64 分支则使用 19 位。位宽和溢出条件都属于版本、架构相关的实现细节。

`autorelease` 的语义是延后放弃所有权。当前 Runtime 使用按页组织的指针栈记录待处理对象，并用值为 `nil` 的边界区分嵌套的池。空池占位、相邻指针合并和返回值所有权交接等优化，可能使实际运行结果与早期资料中的观察不同。

引用计数归零后，Runtime 会向对象发送 `dealloc`，再根据对象是否存在弱引用、关联对象、C++ 析构或 SideTable 状态选择清理路径。`dealloc` 在最后一份强引用被释放的线程上执行，因此不应把要求固定线程或确定时机的业务流程放在其中。

文中涉及的 isa 位宽、页面大小和优化条件都不是公开 API 保证。学习这类实现细节时，需要先确认 objc4 版本和目标架构，再用小实验核对实际结果。

下一篇 [[iOS 内存：ARC 的两半]]。

## 参考资料

### 官方

- [Memory Management Policy](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html)：四条规则的权威措辞
- [Practical Memory Management](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html)：setter 顺序和 dealloc 禁忌的官方理由
- [CoreFoundation Ownership Policy](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Concepts/Ownership.html)：Create Rule / Get Rule
- [Clang ARC Specification](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)：ARC 下被禁用的方法清单
- [apple-oss-distributions/objc4](https://github.com/apple-oss-distributions/objc4)：本文涉及的 isa 位域、编译开关和自动释放池实现

### 经典

- [Always Processing — objc_retain](https://alwaysprocessing.blog/2023/07/22/objc-retain)：结合较新 objc4 源码说明 `retain` 的实现路径
- [draven — 黑箱中的 retain 和 release](https://draven.co/rr/)：介绍 non-pointer isa、SideTable 与引用计数；其中具体位宽需要结合当前源码理解
- [sunnyxx — 黑幕背后的 Autorelease](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/)：介绍 AutoreleasePoolPage、页链表与边界标记；成文较早，未覆盖本文列出的后续优化
- [draven — 自动释放池的前世今生](https://draven.co/autoreleasepool/)：从使用方式和 Runtime 实现说明自动释放池

### 本地

- [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]]
- [[iOS 内存：ARC 的两半]]

---

实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -target arm64-apple-ios17.0-simulator`，ARC 与 MRC 两份分别构建。

isa 位域实验依赖本文所用 objc4 版本的定义，属于实现细节。复现实验时，应先按照对应版本的 `isa.h` 确认位域，再将解析结果与实际引用计数操作对照。

> 待真机补测：同一组实验在 iPhone 15 / iOS 26.5 上复现，确认 arm64e 真机的位宽与溢出临界点是否与模拟器一致。
