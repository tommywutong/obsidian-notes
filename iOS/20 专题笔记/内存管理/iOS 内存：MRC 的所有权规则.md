---
title: 【iOS】MRC 的所有权规则：retain、release 与 autorelease
published: 2026-07-26
description: ARC 没有取消所有权规则，只是把配平代码自动化了。从 Apple 的四条规则出发，看 retain 的引用计数存在哪、autorelease 池怎么组织，并用实验测出 extra_rc 到底有几位。
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

## 为什么今天还要学 MRC

已经没人写 MRC 了，但所有权规则一天都没有消失。

ARC 做的事情是**替你写配平代码**，不是**取消配平的必要性**。编译器插入 `objc_retain` / `objc_release` 的依据，恰恰就是下面要讲的那四条规则——它是照着规则来插的。所以"ARC 到底做了什么"这个问题，如果不先知道规则本身，就只能答到"自动管理内存"这个层次。

另外几个现实理由：Core Foundation 对象至今仍需手动 `CFRetain` / `CFRelease`；`__bridge_retained` / `__bridge_transfer` 表达的就是所有权转移；老项目里 `-fno-objc-arc` 标记的文件还在编译；崩溃日志里的过度释放问题，不理解引用计数就无从下手。

这篇讲规则本身和它在运行时的实现。ARC 怎么把规则翻译成代码，放在下一篇 [[iOS 内存：ARC 的两半]]。

---

## 一、四条规则

Apple 的 Memory Management Policy 只有四句话，值得原文引用，因为中文转述经常把第一条讲歪。

> **You own any object you create.** You create an object using a method whose name begins with "alloc", "new", "copy", or "mutableCopy".

> **You can take ownership of an object using retain.**

> **When you no longer need it, you must relinquish ownership of an object you own.** You relinquish ownership of an object by sending it a `release` message or an `autorelease` message.

> **You must not relinquish ownership of an object you do not own.**

第一条的判定标准是**方法名前缀**，不是"这个方法有没有返回一个新对象"。这个区别很重要，因为它是一个纯粹的命名约定契约，编译器和运行时都不会去检查方法内部到底干了什么。

举个最容易搞错的例子：

```objc
NSString *a = [[NSString alloc] initWithFormat:@"%d", 1];  // alloc 开头，我拥有它，必须 release
NSString *b = [a stringByAppendingString:@"x"];            // 不是那四个前缀，我不拥有它
[b release];                                                // ← 错，这是过度释放
```

`stringByAppendingString:` 确实分配了一个新字符串，但方法名不以那四个词开头，所以按约定它返回的是一个**你不拥有**的对象。你要留住它，得自己 `retain`。

反过来，一个叫 `newItem` 的方法，哪怕内部只是返回一个缓存对象，按约定也表示"调用方获得了所有权"——如果实现没有相应地 retain 一次，就是这个方法自己写错了。

这套约定纯靠自觉。但它不是"仅供参考"：**ARC 的返回值优化正是靠静态匹配这些方法名前缀来决定要不要插 retain 的**。命名约定在 ARC 时代反而变成了编译器行为的一部分。

---

## 二、手写一遍，才知道 ARC 删掉了什么

把同一段逻辑用 MRC 写出来，是理解 ARC 最直接的方式。

```objc
@interface Box : NSObject {
    NSString *_label;
}
- (NSString *)label;
- (void)setLabel:(NSString *)label;
@end

@implementation Box

- (NSString *)label { return _label; }

- (void)setLabel:(NSString *)label {
    if (_label != label) {
        [_label release];
        _label = [label retain];
    }
}

- (void)dealloc {
    [_label release];
    [super dealloc];      // ARC 下写这一句是编译错误
}
@end
```

setter 里那两行的顺序不能随便写。Apple 官方文档给的标准写法其实是反过来的：

```objc
- (void)setCount:(NSNumber *)newCount {
    [newCount retain];
    [_count release];
    _count = newCount;
}
```

并且明确解释了为什么必须**先 retain 新值**：

> You must send this after `[newCount retain]` in case the two are the same object—you don't want to inadvertently cause it to be deallocated.

如果新旧是同一个对象，且它的引用计数正好是 1，先 `release` 会当场把它干掉，接下来的 `retain` 就作用在一块已经释放的内存上。上面那个 `if (_label != label)` 的写法是另一种规避方式，两种都对，但先 retain 的版本更稳妥——它不依赖那个判断写没写。

这个顺序问题在 ARC 里没有消失，只是搬进了运行时函数。`objc_storeStrong` 内部就是"先 retain 新值、再赋值、最后 release 旧值"。

还有一条容易被忽略的官方建议：

> The only places you shouldn't use accessor methods to set an instance variable are in initializer methods and `dealloc`.

`init` 和 `dealloc` 是**唯一**应该直接碰 ivar 的地方。原因是这两个时刻对象处于不完整状态，而 setter 可能被子类覆写、可能触发 KVO 通知、可能访问别的还没初始化好的属性。

### 编译出来看看

把 MRC 版本编译成 LLVM IR，能看到里面确实有 `objc_retain` / `objc_release` / `objc_autorelease` 的调用：

```text
   5 @objc_release
   3 @objc_retain
   3 @objc_alloc_init
   2 @objc_msgSendSuper
   2 @objc_autorelease
```

有人看到这个会以为"MRC 下编译器也插了代码"。不是的——这些调用**和源码里我手写的每一句 `[x retain]` / `[x release]` 一一对应**。clang 有个优化：认出 `retain` / `release` / `autorelease` 这几个众所周知的选择器时，不走 `objc_msgSend`，直接调用对应的运行时函数。省一次消息发送，语义完全一样。

MRC 和 ARC 的区别不在"有没有这些调用"，而在**谁写的**。

---

## 三、autorelease 解决什么问题

规则第一条说，方法名不以那四个前缀开头，返回的对象调用方就不拥有。那么一个工厂方法该怎么写？

```objc
Box *makeBox(void) {
    Box *b = [[Box alloc] init];   // +1，此刻我拥有它
    return b;                       // 但我不能 release，一 release 对象就没了
}                                   // 也不能不 release，那就泄漏了
```

进退两难。`autorelease` 就是为这个场景造的：

```objc
Box *makeBox(void) {
    Box *b = [[Box alloc] init];
    return [b autorelease];   // 我放弃所有权，但推迟到"稍后某个时刻"再真正 release
}
```

调用方拿到的是一个"暂时还活着、但我不拥有"的对象。想留住它就自己 `retain`：

```objc
void keepBox(void) {
    Box *b = makeBox();     // +0，不能 release
    [gKept release];
    gKept = [b retain];     // 想留住，自己拿所有权
}
```

"稍后某个时刻"具体是什么时候，取决于当前自动释放池什么时候被排空。主线程上通常是一次 RunLoop 迭代结束——这条线索留到第五周的 [[iOS RunLoop 与 AutoreleasePool]] 展开。

---

## 四、引用计数到底存在哪

这是本篇最值得亲手验的部分，因为中文资料在这里几乎全体过时了。

### nonpointer isa

早期的 isa 就是一个纯粹的类指针，引用计数全部存在一张全局的 side table 里，每次 retain/release 都要加锁查表。iOS 7 前后引入了 **nonpointer isa**：既然 64 位指针有大量位用不上，就把一些常用信息直接塞进 isa 里，其中就包括引用计数（`extra_rc` 字段）。

arm64 上的位域排布（objc4 `isa.h`）：

```c
uintptr_t nonpointer        : 1;    // 是否是 nonpointer isa
uintptr_t has_assoc         : 1;    // 有没有关联对象
uintptr_t weakly_referenced : 1;    // 有没有被弱引用
uintptr_t shiftcls_and_sig  : 52;   // 类指针 + 指针签名
uintptr_t has_sidetable_rc  : 1;    // 引用计数是否溢出到了 SideTable
uintptr_t extra_rc          : 8;    // 内联的引用计数
```

`retain` 的快速路径就是把 `extra_rc` 加一，用 `ldxr` / `stxr`（ARM64 的独占访问指令）做一个无锁 CAS 循环，根本不碰锁。只有 `extra_rc` 装不下了才会溢出到 SideTable。

### 实验：extra_rc 有几位

网上几乎所有中文文章都说 `extra_rc` 占 **19 位**。这个数字来自 arm64e 和指针认证普及之前的源码。今天在 arm64e 设备（A12 及以后，也就是 iPhone XS 之后）和模拟器上，52 位被 `shiftcls_and_sig` 占走了，`extra_rc` 只剩 **8 位**。

这个差别不是无关紧要——8 位意味着内联能装下的引用计数上限是 255，19 位是 524287，差了三个数量级，溢出到 SideTable 的门槛完全不同。

直接测。按上面的位域排布解析 isa，然后反复 retain：

```objc
static void dumpISA(id obj, const char *tag) {
    uintptr_t isa = *(uintptr_t *)(__bridge void *)obj;
    printf("%-22s isa=0x%016lx  nonpointer=%lu  has_sidetable_rc=%lu  extra_rc=%lu\n",
           tag, isa,
           (isa >> 0)  & 0x1,
           (isa >> 55) & 0x1,
           (isa >> 56) & 0xff);
}

NSObject *o = [NSObject new];
for (int i = 1; i <= 300; i++) {
    CFRetain((CFTypeRef)o);
    dumpISA(o, ...);
}
```

关键的几行输出：

```text
刚创建           isa=0x02000001efeba749  nonpointer=1  has_sidetable_rc=0  extra_rc=2
retain 第 251 次  isa=0xfd000001efeba749  nonpointer=1  has_sidetable_rc=0  extra_rc=253
retain 第 252 次  isa=0xfe000001efeba749  nonpointer=1  has_sidetable_rc=0  extra_rc=254
retain 第 253 次  isa=0xff000001efeba749  nonpointer=1  has_sidetable_rc=0  extra_rc=255
retain 第 254 次  isa=0x80800001efeba749  nonpointer=1  has_sidetable_rc=1  extra_rc=128
retain 第 255 次  isa=0x81800001efeba749  nonpointer=1  has_sidetable_rc=1  extra_rc=129
最终 CFGetRetainCount = 301
```

这几行把整个机制摊开了：

- `extra_rc` 一路涨到 **255** 就到顶了。它是 8 位，不是 19 位。
- 第 254 次 retain 触发溢出：`has_sidetable_rc` 从 0 翻成 1，同时 `extra_rc` **回落到 128**。
- 128 正是 objc4 里的 `RC_HALF`。溢出的处理策略不是"全部搬到 SideTable"，而是**留一半在 isa 里、把另一半挪过去**。这样后续的 release 在 `extra_rc` 减到 0 之前都不用碰 SideTable，减少加锁次数。
- `CFGetRetainCount` 最终返回 301，说明 isa 内联的那部分和 SideTable 里的那部分被正确地加在了一起。

顺带解释一下开头那个 `extra_rc=2`。对象刚创建时引用计数是 1，`extra_rc` 应该是 0。读到 2 是因为这是 `-O0` 构建，ARC 在把对象传给 `dumpISA` 时插入了额外的 retain。这个"多出来的计数"本身就是下一篇要讲的内容——**ARC 在未优化构建下插的桩比你想象的多**。

> 这组数据来自 iOS 模拟器（arm64）。模拟器和 arm64e 真机走的是同一个位域分支，但真机上是否完全一致需要自己复现一遍。上面那段代码原样拿到真机跑即可。

---

## 五、autorelease 池的实现

### 结构

自动释放池不是一个对象列表，而是**一个按页组织的指针栈**。objc4 源码的注释说得很清楚：

> A thread's autorelease pool is a stack of pointers. Each pointer is either an object to release, or `POOL_BOUNDARY` which is an autorelease pool boundary... The stack is divided into a doubly-linked list of pages.

几个要点：

**哨兵就是 `nil`。** 源码里 `#define POOL_BOUNDARY nil`。每次 `objc_autoreleasePoolPush` 往栈里压一个 `nil` 作为边界，并把这个 `nil` 的地址返回；`objc_autoreleasePoolPop` 拿着这个地址，把栈弹到那个位置为止，沿途每个对象发一次 `release`。嵌套的池就是栈里多个哨兵，天然支持。

**页是双向链表。** 一页装满了就挂一个新页上去，`parent` / `child` 互相串起来。线程本地存储里记着当前的"热页"。

**页大小不是硬编码的 4096。** 源码里写的是 `PROTECT_AUTORELEASEPOOL ? PAGE_MAX_SIZE : PAGE_MIN_SIZE`。日常构建下确实等于 4096，但"固定 4096 字节"这个说法不严谨，调试构建下会变成 `PAGE_MAX_SIZE`。

**`autoreleaseFast` 的三个分支**：

```c
static inline id *autoreleaseFast(id obj) {
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);                 // 热页有空间，直接塞
    } else if (page) {
        return autoreleaseFullPage(obj, page);  // 热页满了，找或建下一页
    } else {
        return autoreleaseNoPage(obj);          // 一页都还没有
    }
}
```

**pop 时会涂抹释放过的槽位**，涂的值是 `SCRIBBLE = 0xA3`。所以在 lldb 里读一块刚被排空的池内存，会看到成片的 `0xA3A3A3A3`。这是个能直接观察到的特征。

### 几个新机制，老文章里没有

中文圈关于 autorelease 最经典的两篇文章写于 2014 年前后，成文时下面这些东西还不存在。今天用新 Xcode 抓到的行为和那些文章的截图对不上，不是你搞错了。

**空池占位符（`EMPTY_POOL_PLACEHOLDER`）。** 大量自动释放池从创建到销毁一个对象都没装过——比如 RunLoop 每轮迭代都会 push/pop 一个池，但大部分迭代里什么都没发生。为此运行时做了个优化：如果当前线程只 push 了一个池、且从没真正塞过对象，就把热页指针设成一个哨兵值 `1`（不是真实页地址），**完全不分配内存**。等真要塞第一个对象时才补上边界并分配页。

**相邻指针去重。** 现在的 `add` 会检查栈顶最近几个槽位，如果同一个指针被连续 autorelease（循环里反复 `[NSNumber numberWithInt:]` 就是典型），不再压新槽位，而是在已有条目上做计数递增，pop 时按计数循环 release。这是把小整数计数压进指针高位的位运算技巧，目的是压缩栈深度。

**返回值优化让很多 autorelease 根本没入池。** 这条最颠覆——现代实现里，`objc_autoreleaseReturnValue` 会先检查调用方是不是"马上就要 retain 这个返回值"，如果是，对象直接以 +1 的状态交出去，**完全不进池**。所以"autorelease 就是把 release 延后"这个说法今天已经不完整了。具体机制在下一篇讲。

---

## 六、release 到 0 之后发生了什么

```
最后一次 release，计数归零
      │  isa.deallocating 置位
      ▼
objc_msgSend(this, @selector(dealloc))     ← 是一次真正的消息发送，可被覆写、可被 NSZombie 替换
      │
-[NSObject dealloc] → _objc_rootDealloc(self)
      │
rootDealloc()
      │
      ├─ 快速路径：nonpointer && !weakly_referenced && !has_assoc
      │            && !has_cxx_dtor && !has_sidetable_rc
      │            → free(this)，直接结束
      │
      └─ 慢速路径：object_dispose → objc_destructInstance
                    ① object_cxxDestruct   （执行 .cxx_destruct，释放 ARC 托管的 ivar）
                    ② 移除关联对象
                    ③ clearDeallocating    （清空所有弱引用、清理 SideTable 残留）
                    → free
```

快速路径那个判断值得留意：**只要这个对象没被弱引用过、没挂关联对象、没有 ARC 托管的 ivar、引用计数没溢出过，销毁就是一句 `free`**，整套析构流程全部跳过。nonpointer isa 把这四个条件直接编码成了位，一次判断就知道能不能抄近道。

慢速路径里那三步的顺序在源码里有一句注释说 "This order is important"。原因是 `.cxx_destruct` 要读取并释放各个 ivar，此刻它们必须还是有效的；如果先清了弱引用表，析构过程中万一有逻辑依赖弱引用语义就会拿到 `nil`。

---

## 七、NSZombie 是怎么做到的

过度释放的表现通常是"随机崩溃"——对象已经销毁，但指针还在，那块内存要等到被别人复用才会出问题。所以崩溃点往往离真正的 bug 很远。

`NSZombieEnabled` 的做法是把 `dealloc` 的行为整个换掉：

1. 根据对象原来的类名拼出一个 `_NSZombie_原类名` 的僵尸类，这个类没有父类、没有任何方法实现。
2. 调用 `objc_destructInstance` 做析构（执行 `.cxx_destruct`、清关联对象、清弱引用），但**不 free 内存**。
3. 把对象的 isa 改指向僵尸类。
4. 内存留着不还。

之后任何人再给这个指针发消息，因为僵尸类什么方法都没有，会走消息转发失败路径，被拦截下来打印：

```text
-[Person printName]: message sent to deallocated instance 0x108a08180
```

这行错误是僵尸类主动打印的，不是系统默认的野指针崩溃信息。代价是**内存永久泄漏**——僵尸对象永远不会被真正释放，所以只能在调试时开。

---

## 八、几个流传很广但已经不准的说法

**"extra_rc 占 19 位"** — 只在没有指针认证的 arm64 上成立。arm64e 设备和模拟器上是 8 位，实测已经在第四节验证过。这条不能怪原作者，2016 年前后 arm64e 还没出现。

**"AutoreleasePoolPage 固定 4096 字节"** — 源码里是平台宏 `PAGE_MIN_SIZE` / `PAGE_MAX_SIZE`，不是字面常量。日常构建下数值确实是 4096。

**"NSAutoreleasePool 和 @autoreleasepool 是新旧两种写法"** — 不准确。`@autoreleasepool {}` 编译后直接调 `objc_autoreleasePoolPush` / `Pop`，走的是运行时内建的分页栈；`NSAutoreleasePool` 是一个真实的类，走的是另一套代码路径。两者效果等价，但不是同一个东西。

**"autorelease 就是把 release 延后"** — 不完整。返回值优化会让很多 autorelease 调用根本没有入池，见第五节末尾。

**"过度释放会立刻崩溃"** — 不一定。对象销毁后指针没被清空，继续用只是"看起来正常"，直到那块内存被复用。这就是野指针 bug 难查的根本原因，也是 NSZombie 存在的意义。

**"SideTable 是一张全局哈希表"** — 更准确的说法是 `StripedMap<SideTable>`，按对象地址哈希分成多个独立加锁的分片，不是单张表。分片数在真机上是 8，在 Mac 和模拟器上是 64。

**"retain/release 都是 C 实现的"** — 部分架构上快速路径直接在汇编里（`objc-msg-arm64.s`），tagged pointer 和 nil 的判断在汇编层就短路了，只有慢速路径才落到 C++ 代码。

---

## 总结

1. ARC 没有取消所有权规则，它是照着规则替你插代码。不知道规则就说不清 ARC 做了什么。
2. 四条规则里第一条最容易讲错：判定标准是方法名前缀（`alloc` / `new` / `copy` / `mutableCopy`），不是"有没有返回新对象"。这是纯粹的命名约定，但 ARC 的返回值优化真的在静态匹配这些前缀。
3. setter 必须先 retain 新值再 release 旧值，防止新旧同一对象时把它释放掉。这个顺序在 ARC 里搬进了 `objc_storeStrong`。
4. `init` 和 `dealloc` 是唯一应该直接操作 ivar 的地方。
5. `autorelease` 是为"返回一个调用方不拥有的对象"这个场景造的，不是一个通用的延迟释放工具。
6. 引用计数优先存在 nonpointer isa 的 `extra_rc` 里，用 CAS 无锁增减。**实测这个字段在模拟器和 arm64e 上是 8 位**，第 254 次 retain 溢出，一半（`RC_HALF` = 128）挪到 SideTable。"19 位"是过时说法。
7. 自动释放池是分页的指针栈，哨兵就是 `nil`，pop 时用 `0xA3` 涂抹释放过的槽位。空池占位符、相邻指针去重、返回值优化是老文章里没有的新机制。
8. 引用计数归零后 `dealloc` 是一次真实的消息发送。如果对象没有弱引用、关联对象、ARC ivar、SideTable 计数，销毁就是一句 `free`，整套析构流程跳过。
9. NSZombie 的原理是把 isa 换成一个空的僵尸类并保留内存，代价是永久泄漏，只能调试用。

下一篇 [[iOS 内存：ARC 的两半]] 讲编译器怎么把这些规则翻译成代码，以及那个让 autorelease "凭空消失"的返回值优化。

## 参考资料

### 官方

- [Memory Management Policy](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html)：四条规则的权威措辞，必须原文核对
- [Practical Memory Management](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html)：setter 顺序的官方理由出自这里
- [apple-oss-distributions/objc4](https://github.com/apple-oss-distributions/objc4)：`isa.h` 是所有位宽争议的最终裁判，`NSObject.mm` 里是 AutoreleasePoolPage 的完整实现

### 经典

- [Always Processing — objc_retain](https://alwaysprocessing.blog/2023/07/22/objc-retain)：少见的跟着新版 objc4 走读的文章，比多数中文教程新
- [draven — 黑箱中的 retain 和 release](https://draven.co/rr/)：概念讲得清楚，但位宽数字已过时
- [sunnyxx — 黑幕背后的 Autorelease](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/)：中文圈引用率最高的一篇，双向链表和哨兵的说法源头，成文早于本文第五节末尾提到的几个新机制
- [draven — 自动释放池的前世今生](https://draven.co/autoreleasepool/)：三篇里和当前源码吻合度最高的

### 本地

- [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]]
- [[iOS 内存：ARC 的两半]]

---

## 附：实验环境

| 项目 | 环境 |
| --- | --- |
| Xcode | 26.6 |
| 运行环境 | iOS Simulator（arm64，Apple Silicon Mac） |
| 构建 | `clang -fobjc-arc` / `-fno-objc-arc`，`-target arm64-apple-ios17.0-simulator` |

isa 位域实验依赖 objc4 当前的位域排布，属于实现细节，不同版本可能变化。实验本身的价值在于**方法**：按 `isa.h` 的定义解析位，用实际行为验证源码结论，而不是相信任何一篇文章给出的数字。

> 待真机补测：同一份 isa 实验在 iPhone 15 / iOS 26.5 上复现，确认 arm64e 真机的 `extra_rc` 位宽与溢出临界点是否与模拟器一致。
