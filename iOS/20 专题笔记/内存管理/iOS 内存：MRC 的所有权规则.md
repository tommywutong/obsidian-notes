---
title: 【iOS】MRC 的所有权规则：retain、release 与 autorelease
published: 2026-07-26
description: 网上说 extra_rc 占 19 位的文章，在今天的模拟器、A12 之后的 iPhone 和所有 Intel Mac 上都是错的。从 Apple 的四条所有权规则讲起，用实验把引用计数从 isa 一路追到 SideTable。
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

> [!note] 已整合
> 本文已与 ARC、属性关键字两篇合并为 [[iOS 内存管理：从 MRC、ARC 到属性关键字#第一部分：MRC 的所有权规则：retain、release 与 autorelease|Objective-C 内存管理：从 MRC、ARC 到属性关键字]]。本文件保留为分篇原稿，以兼容既有链接。

已经没人写 MRC 了，但所有权规则一天都没有消失。ARC 替你写配平代码，依据恰恰就是下面这四条规则——它是照着规则插的。不知道规则本身，"ARC 到底做了什么"就只能答到"自动管理内存"这个层次。

还有两个更现实的理由。Core Foundation 对象至今要手动 `CFRetain` / `CFRelease`，`__bridge_transfer` 这些关键字表达的就是所有权转移；线上崩溃日志里的过度释放问题，不理解引用计数根本无从下手。

顺带先把本文最硬的一个结论摆在前面：**网上讲 `extra_rc` 占 19 位的文章，在你现在用的模拟器、iPhone XS 起的所有真机、以及所有 Intel Mac 上都是错的，实测是 8 位。** 第四节有完整实验和源码依据。

ARC 怎么把规则翻译成代码，放在下一篇 [[iOS 内存：ARC 的两半]]。

---

## 一、四条规则

Apple 的 Memory Management Policy 只有四句话，值得看原文，因为中文转述经常把第一条讲歪。

> **You own any object you create.** You create an object using a method whose name begins with "alloc", "new", "copy", or "mutableCopy" (for example, `alloc`, `newObject`, or `mutableCopy`).

> **You can take ownership of an object using retain.** You use `retain` in two situations: in the implementation of an accessor method or an `init` method, to take ownership of an object you want to store as a property value; and to prevent an object from being invalidated as a side-effect of some other operation.

> **When you no longer need it, you must relinquish ownership of an object you own.** You relinquish ownership of an object by sending it a `release` message or an `autorelease` message.

> **You must not relinquish ownership of an object you do not own.**

第一条的判定标准是方法名前缀，不是"这个方法有没有返回一个新对象"。括号里那个 `newObject` 是官方特意给的例子，说明匹配的是前缀而非全名。

这个契约纯靠自觉，编译器和运行时都不会去检查方法内部做了什么。两个方向的例子放在一起才能看清它有多"不讲理"。

一个是分配了新对象但你不拥有：

```objc
NSString *a = [[NSString alloc] initWithFormat:@"%d", 1];  // alloc 开头，我拥有，必须 release
NSString *b = [a stringByAppendingString:@"x"];            // 前缀不匹配，我不拥有
[b release];                                                // 过度释放
```

`stringByAppendingString:` 确实分配了一个新字符串，但名字不以那四个词开头，按约定它返回的就是一个你不拥有的对象。

另一个方向更尖锐——没分配新对象，但你拥有：

```objc
NSString *s = @"abc";
NSString *c = [s copy];   // 前缀是 copy，我拥有它，必须 release
                          // 可是 c == s，copy 内部就是 retain，一个字节都没复制
```

不可变字符串的 `copyWithZone:` 实现是 `return [self retain]`，压根没有新对象。但方法名以 `copy` 开头，所以你确实持有了一份所有权，必须配平。这两个例子摆在一起，"命名约定是契约、和实现无关"这句话才真正立住。

MRC 下 `@property (copy)` 的 setter 也是同一套逻辑：`_s = [newValue copy]` 拿到 +1，所以 `dealloc` 里必须 `release`。

顺带说，这套约定在 ARC 时代不但没作废，反而变成了编译器行为的一部分——返回值优化正是靠静态匹配这些前缀来决定要不要插 retain。

### Core Foundation 说的是同一件事的另一种方言

CF 没有 `alloc`/`new`/`copy` 这四个词，它有自己的一套：

- **Create Rule**：函数名里含 `Create` 或 `Copy` 的，返回 +1，你负责 `CFRelease`。
- **Get Rule**：函数名里含 `Get` 的，返回 +0，不许 release。

和 ObjC 那四个前缀是同一个思想——靠函数名约定表达所有权，不检查实现。两边桥接时用三个关键字划边界：`__bridge` 只做类型转换不动所有权；`__bridge_retained`（等价于 `CFBridgingRetain`）把对象交给 CF 侧并 +1；`__bridge_transfer`（等价于 `CFBridgingRelease`）把 CF 侧的所有权交还给 ARC。

搞清楚这两套方言其实是同一件事，桥接就不用死记了：问一句"这次转换之后，谁负责最后那次 release"。

---

## 二、手写一遍，才知道 ARC 删掉了什么

```objc
@interface Box : NSObject {
    NSString *_label;
}
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

setter 那两行的顺序不能随便写。Apple 给的标准写法其实是反过来的：

```objc
- (void)setCount:(NSNumber *)newCount {
    [newCount retain];
    [_count release];
    _count = newCount;
}
```

并且明确解释了为什么必须先 retain 新值：

> You must send this after `[newCount retain]` in case the two are the same object—you don't want to inadvertently cause it to be deallocated.

新旧是同一个对象、引用计数正好为 1 时，先 `release` 会当场把它干掉，接下来的 `retain` 就作用在已释放的内存上。上面用 `if (_label != label)` 是另一种规避方式，两种都对，但先 retain 的版本更稳妥——它不依赖那个判断写没写。

这个顺序在 ARC 里没有消失，只是搬进了 `objc_storeStrong`。

还有一条容易被忽略的官方建议：`init` 和 `dealloc` 是唯一应该直接碰 ivar 的地方。这两个时刻对象处于不完整状态，而 setter 可能被子类覆写、可能触发 KVO 通知、可能访问别的还没初始化好的属性。

### 编译产物里的一个小陷阱

把 MRC 版本编译成 LLVM IR，里面确实有 `objc_retain` / `objc_release` / `objc_autorelease`：

```text
   5 @objc_release
   3 @objc_retain
   3 @objc_alloc_init
   2 @objc_msgSendSuper
   2 @objc_autorelease
```

看到这个别急着说"MRC 下编译器也插了代码"。这些调用和我手写的每一句 `[x retain]` / `[x release]` 一一对应——clang 认出 `retain` / `release` / `autorelease` 这几个众所周知的选择器时，不走 `objc_msgSend`，直接调对应的运行时函数，省一次消息发送，语义完全一样。

MRC 和 ARC 的区别不在有没有这些调用，而在谁写的。

---

## 三、autorelease 解决什么问题

规则第一条说，方法名不匹配那四个前缀，返回的对象调用方就不拥有。那工厂方法该怎么写？

```objc
Box *makeBox(void) {
    Box *b = [[Box alloc] init];   // +1，此刻我拥有它
    return b;                       // 不能 release，一 release 对象就没了
}                                   // 也不能不 release，那就泄漏了
```

进退两难。`autorelease` 就是为这个场景造的：

```objc
Box *makeBox(void) {
    Box *b = [[Box alloc] init];
    return [b autorelease];   // 放弃所有权，但推迟到"稍后某个时刻"再真正 release
}
```

调用方拿到一个暂时还活着、但不归自己的对象。想留住就自己 `retain`。

"稍后某个时刻"是什么时候，取决于当前自动释放池什么时候排空。第五节末尾会讲。

---

## 四、引用计数存在哪：一个能测出来的问题

中文资料在这一节几乎全体过时了，而且过时得很有系统性。

### nonpointer isa

早期的 isa 就是一个纯类指针，引用计数全部放在一张全局表里，每次 retain/release 都要加锁查表。iOS 7 前后引入 nonpointer isa：64 位指针有大量位闲着，干脆把常用信息塞进去，其中就包括引用计数。

`retain` 的快速路径变成把 isa 里的计数字段加一，用 `ldxr` / `stxr`（ARM64 的独占访问指令）做无锁 CAS 循环，根本不碰锁。只有装不下了才溢出到 SideTable。

关键在于"装不下"的门槛是多少。

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

19 位在第二个分支里。指针认证要占掉 52 位来放类指针加签名，`extra_rc` 就只剩 8 位了。

模拟器为什么也走第一个分支，`isa.h` 里有现成的注释：

> ARM64 simulators have a larger address space, so use the ARM64e scheme even when simulators build for ARM64-not-e.

还有一件事几乎没人提：`__x86_64__` 分支的 `extra_rc` **也是 8 位**，`RC_HALF` 同样是 `1ULL<<7`。所以 19 位这个数字在 Intel Mac 上从来就没成立过。它只存在于无指针认证的 arm64 这一个分支里——对应 A7 到 A11，iPhone 5s 到 iPhone X。

顺带纠正一个容易搞混的地方：决定走哪个分支的是 **libobjc 自己怎么编的**（`__has_feature(ptrauth_calls)`），不是你 App 的架构 slice。普通 App 编出来是 arm64 而不是 arm64e，但 A12 起设备的 dyld 共享缓存是 arm64e，libobjc 走的就是 8 位那个分支。

### 实测

按第一个分支的位域解析 isa，同时读 `CFGetRetainCount` 做对照：

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

`extra_rc` 涨到 255 就到顶了，8 位。第 254 次触发溢出，`has_sidetable_rc` 翻成 1，`extra_rc` 回落到 128。

把溢出那一行的 isa 拆开看更清楚：

```text
0x80 80 0001efeba749
  │   └── bit55 = 1      → has_sidetable_rc
  └────── bit56-63 = 0x80 = 128 = RC_HALF
```

128 正是源码里的 `RC_HALF`。溢出的处理不是把计数全搬去 SideTable，而是留一半在 isa 里、把另一半挪过去。这样后续 release 在 `extra_rc` 减到 0 之前都不用碰锁。

### 顺带修正一个我自己差点写错的地方

最后一列"差值"恒为 0，这说明 `extra_rc` 直接就是引用计数，不是老文章说的 `retainCount - 1`。源码可以印证——`initIsa` 里是：

```c
#if ISA_HAS_INLINE_RC
        newisa.extra_rc = 1;
#endif
```

`rootRetainCount()` 也不再 +1：

```c
uintptr_t rc = bits.extra_rc;
if (bits.has_sidetable_rc) { rc += sidetable_getExtraRC_nolock(); }
return rc;
```

这个语义变化还有一个连带后果，很少有人提：旧版 isa 里有一个独立的 `deallocating` 位，现在没有了。因为 `extra_rc` 既然直接是引用计数，`extra_rc == 0` 天然就表示没人持有：

```cpp
bool isDeallocating() const {
    return extra_rc == 0 && has_sidetable_rc == 0;
}
```

所以那一位被省掉了。你回头看上面贴的两个位域分支，确实一个 `deallocating` 都找不到。

至于"刚创建"那行为什么是 2 而不是 1——这是 `-O0` 构建，ARC 在把对象传进 `probe` 时多插了一次 retain。剥掉这一次，`1 + 300 = 301`，和跑满 300 次后 `CFGetRetainCount` 的返回值对得上。

### 别拿 retainCount 调试

上面刚用完这个数字，得马上说清楚它为什么不能信。

首先是 tagged pointer。`rootRetainCount()` 第一行就是特判：

```c
if (isTaggedPointer()) return (uintptr_t)this;
```

但实际测出来的结果和这行代码不一样：

```text
tagged 指针值             = 0xb68cd92197b5cfee
tagged  [obj retainCount] = 9223372036854775807
tagged  CFGetRetainCount  = 9223372036854775807
        retainCount 是否 == 指针值? 0
堆对象  [obj retainCount] = 1
```

返回的是 `INT64_MAX` 而不是指针值，因为 `NSNumber` 是 CF 桥接类、有自定义的 retain/release 实现，压根不走 `objc_object::rootRetainCount()`。这个例子挺能说明问题：只读一个函数的源码就下结论，很容易翻车。不管走哪条路，结论都一样——tagged pointer 的引用计数是个没有意义的数。

其次，返回值不包含自动释放池里挂着的待释放次数，所以"看着是 1"和"马上要归零"根本分不出来。常量字符串、单例这类对象返回的也是无意义的极大值。

ARC 下 `retain` / `release` / `autorelease` / `retainCount` 以及显式调用 `dealloc` 一律是编译错误，Clang ARC 规范里有明确清单。这不是"防止你手滑"，是这些操作在 ARC 的语义模型下没有位置。

> 上面这组数据来自 iOS 模拟器（arm64）。模拟器和 arm64e 真机走同一个位域分支，但真机上是否完全一致需要自己复现。代码原样拿到真机跑即可。

---

## 五、autorelease 池的实现

自动释放池是一个按页组织的指针栈。objc4 源码的注释说得很清楚：

> A thread's autorelease pool is a stack of pointers. Each pointer is either an object to release, or `POOL_BOUNDARY` which is an autorelease pool boundary... The stack is divided into a doubly-linked list of pages.

哨兵就是 `nil`（`#define POOL_BOUNDARY nil`）。每次 `objc_autoreleasePoolPush` 往栈里压一个 `nil` 当边界，并把这个位置的地址返回；`objc_autoreleasePoolPop` 拿着地址把栈弹回去，沿途每个对象发一次 `release`。嵌套的池就是栈里多个哨兵，天然支持。

页之间用 `parent` / `child` 串成双向链表，线程本地存储里记着当前的热页。`autoreleaseFast` 按三种情况分流：

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

pop 时会把释放过的槽位涂成 `SCRIBBLE = 0xA3`。在 lldb 里读一块刚排空的池内存，能看到成片的 `0xA3A3A3A3`。

关于页大小有个流传很广的说法需要修正。源码里写的确实不是字面常量 4096，而是 `PROTECT_AUTORELEASEPOOL ? PAGE_MAX_SIZE : PAGE_MIN_SIZE`。但 `PROTECT_AUTORELEASEPOOL` 在发布的 objc4 里**根本没有定义**，它是给运行时开发者自己重编 libobjc 时用的开关，不会随 Xcode 的 Debug 构建自动打开。所以实践中 4096 这个数值是对的，错的只是把它当成源码里的硬编码常量。

### 三个老文章里没有的机制

中文圈关于 autorelease 最经典的两篇写于 2014 年前后，下面这些当时还不存在。今天用新 Xcode 抓到的行为和那些文章的截图对不上，不是你搞错了。

**空池占位符。** 大量自动释放池从创建到销毁一个对象都没装过——RunLoop 每轮迭代都会 push/pop 一个池，但多数迭代里什么都没发生。运行时为此做了优化：如果当前线程只 push 了一个池、且从没真正塞过对象，就把热页指针设成哨兵值 `1`，完全不分配内存。等真要塞第一个对象时才补边界并分配页。

**相邻指针合并。** `objc-config.h` 里 `SUPPORT_AUTORELEASEPOOL_DEDUP_PTRS` 的注释写的是"combine consecutive pointers to the same object"，实现上会回看栈顶最近 4 个条目，命中就在已有条目上递增计数并把它挪到栈顶（LRU），pop 时按计数循环 release。循环里反复 `[NSNumber numberWithInt:]` 是典型受益场景。

**返回值优化让很多 autorelease 根本没入池。** `objc_autoreleaseReturnValue` 会先判断调用方是不是马上就要 retain 这个返回值，是的话对象直接以 +1 状态交出去，通过线程本地存储接力，完全不经过池页。判断方式本身也换过代——`objc-config.h` 里 `HAS_RETURNADDR_AUTORELEASE_ELISION` 在 arm64 上是 1，注释说得很直白：

> autorelease elision based on comparing the return address of the call and claim. When 0, we only support the older scheme that inspects the caller's code for a claim call or sentinel NOP.

也就是说"读调用方指令流找哨兵 NOP"是旧方案，当前 arm64 走的是返回地址比对。对应还多了一个 `objc_claimAutoreleasedReturnValue` 入口，签名注释写着 "without a NOP in the caller on ARM64"。具体机制在下一篇展开。

所以"autorelease 就是把 release 延后"这句话今天已经不完整了。

### 池什么时候排空

主线程上由 RunLoop 的 observer 负责：进入时 push，即将休眠时 pop 再 push，退出时 pop。所以主线程的 autoreleased 对象最迟活到当前这轮 RunLoop 结束。

子线程麻烦一些。`NSThread` 手动起的线程有隐式池，但排空时机不受你控制；GCD 队列会自动排空，时机同样不保证。所以在子线程跑一个产生大量临时对象的长任务时，得自己套 `@autoreleasepool`，否则内存会一路涨到任务结束。

如果一个对象在完全没有池的上下文里被 autorelease，运行时会打日志：

```text
Object 0x... of class ... autoreleased with no pool in place - just leaking
```

这个函数在源码里是 `BREAKPOINT_FUNCTION(void objc_autoreleaseNoPool(id obj))`，意味着你可以直接 `b objc_autoreleaseNoPool` 断在现场。

至于"循环里套 `@autoreleasepool` 能降低内存峰值"这条标准答案，今天需要打个折。套一层池本身有成本（push/pop，可能还要分配页），只有循环体确实产生大量 autoreleased 对象时才划算——`imageWithContentsOfFile:`、`stringWithFormat:`、JSON 解析这类。而且有了上面说的指针合并和返回值优化，2014 年那些 demo 里的效果今天已经明显缩水。先测再改，别照抄。

---

## 六、release 到 0 之后

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

快速路径那个判断值得留意：只要对象没被弱引用过、没挂关联对象、没有 C++ 析构、引用计数没溢出过，销毁就是一句 `free`，整套析构流程全部跳过。

但那四个条件并不都存在 isa 里。前面贴过，ptrauth / 模拟器分支的 `ISA_HAS_CXX_DTOR_BIT` 是 0，位域里根本没有 `has_cxx_dtor`，所以源码在这里分了岔：

```cpp
#if ISA_HAS_CXX_DTOR_BIT
                 !isa().has_cxx_dtor                  &&
#else
                 !isa().getClass(false)->hasCxxDtor() &&
#endif
```

在 x86_64 和无指针认证的 arm64 上它是 isa 位，在 arm64e 和模拟器上要回退去查类对象的标志。又一个位域分支差异的例证。

慢速路径那三步的顺序，源码里有一句注释：`// This order is important.` 但源码没有给理由。合理的推断是：`.cxx_destruct` 要读取并释放各个 ivar，此刻它们必须还有效；如果先清了弱引用表，析构过程中万一有逻辑依赖弱引用语义就会拿到 `nil`。

补一句和上一节呼应的：ARC 下不用写 `[super dealloc]`，也不用手动释放 ivar，是因为这两件事都由编译器合成——ivar 的释放代码进了 `.cxx_destruct`，对父类的转发由编译器在 `-dealloc` 末尾插入。所以第二节里"ARC 下写 `[super dealloc]` 是编译错误"和这里说的是同一件事的两面。

另外，对一个正在 dealloc 的对象注册新的弱引用会直接崩溃（`Cannot form weak reference to instance ... being deallocated`），这条留到 weak 那一篇讲。

### 所以 dealloc 里该写什么

Apple 的态度很明确：

> You should typically not manage scarce resources such as file descriptors, network connections, and buffers or caches in a `dealloc` method. In particular, you should not design classes so that `dealloc` will be invoked when you think it will be invoked.

理由是 `dealloc` 的调用时机不可控——可能被延迟，可能因为应用退出而根本不执行。

对 iOS 最毒的是线程问题：一个对象的最后一次 release 发生在哪个线程，它的 `dealloc` 就在哪个线程执行。如果某个后台任务持有了最后一个引用，`dealloc` 里碰 UIKit 就直接炸，而且这种崩溃极难复现——它取决于哪条引用链最后一个断开。

`dealloc` 里合适做的事只有一类：解除本对象和外界的关联。移除通知观察者、置空 delegate、取消定时器。真正的资源释放应该有显式的生命周期方法。

---

## 七、NSZombie 与它的替代品

过度释放的表现通常是"随机崩溃"——对象已经销毁，指针还在，那块内存要等被别人复用才出问题。所以崩溃点往往离真正的 bug 很远。

`NSZombieEnabled` 的做法是把 `dealloc` 的行为整个换掉。Foundation 没有开源，下面这几步来自社区对反汇编的整理，和 mikeash 那篇重实现一致，不是一手源码：

1. 按原类名找到或复制出一个 `_NSZombie_原类名` 的僵尸类，这个类没有父类、没有任何方法实现，并按类名缓存。
2. 调 `objc_destructInstance` 做析构（执行 `.cxx_destruct`、清关联对象、清弱引用），但不 free 内存。
3. 把对象的 isa 改指向僵尸类。
4. 内存留着不还。

之后任何人再给这个指针发消息，因为僵尸类什么方法都没有，会走消息转发失败路径被拦截，打印出来：

```text
-[Person printName]: message sent to deallocated instance 0x108a08180
```

随后进程立即终止。这个"立即"才是 zombie 的全部价值——它把一个随机时刻的诡异崩溃换成了一个确定的、可以下断点的崩溃。代价是内存永久泄漏，所以只能调试时开。

今天 zombie 已经不是首选了。**Address Sanitizer** 能抓 use-after-free，而且同时给出分配栈和释放栈，信息量比 zombie 大得多，也不会永久占着内存，Xcode 的 Scheme 里勾一下就行。`MallocStackLogging` 配 `malloc_history <pid> <addr>` 可以拿到那块内存完整的 alloc/free 调用栈，和 zombie 配合用效果最好——zombie 告诉你地址，malloc_history 告诉你是谁释放的。

还有个用工具的常见误区值得单说：Instruments 的 Leaks 只能找到**不可达**的内存，找不到"还被强引用着但一直在涨"的那类问题。后者要用 Allocations 做 Generation 对比。很多人查内存增长时选错了工具，然后得出"没有泄漏"的结论。

顺带一提，`MallocScribble` 会把 free 掉的内存填成 `0x55`，和第五节讲的 `SCRIBBLE = 0xA3` 是同一类技巧。

---

## 八、几个已经不准的说法

- **"extra_rc 占 19 位。"** 只在无指针认证的 arm64（A7~A11）上成立。arm64e 真机、所有 iOS 模拟器、以及所有 x86_64 都是 8 位。19 位在 Intel Mac 上从来就没对过。
- **"isa 里有一个 deallocating 位。"** 现在没有了。`extra_rc` 的语义从 `retainCount - 1` 改成 `retainCount` 之后，`extra_rc == 0 && has_sidetable_rc == 0` 就能表示正在析构，那一位被省掉了。
- **"AutoreleasePoolPage 固定 4096 字节。"** 数值对，但源码里是平台宏而非字面常量；控制它的 `PROTECT_AUTORELEASEPOOL` 在发布版 objc4 里没有定义。
- **"NSAutoreleasePool 和 @autoreleasepool 是新旧两种写法。"** `@autoreleasepool {}` 编译后直接调 `objc_autoreleasePoolPush` / `Pop`，走运行时内建的分页栈；`NSAutoreleasePool` 是一个真实的类，走另一套代码路径。效果等价，实现是两条路。老代码里的 `[pool drain]` 则是 GC 时代的遗留，GC 早已移除，今天它和 `release` 没有区别。
- **"autorelease 就是把 release 延后。"** 返回值优化会让很多 autorelease 调用根本没入池，见第五节。
- **"过度释放会立刻崩溃。"** 对象销毁后指针没被清空，继续用只是看起来正常，直到那块内存被复用。这就是野指针 bug 难查的根本原因。
- **"SideTable 是一张全局哈希表。"** 是 `StripedMap<SideTable>`，按对象地址哈希分片，各片独立加锁。分片数在真机上是 8，Mac 和模拟器上是 64。
- **"retain/release 都是 C 实现的。"** 部分架构上快速路径直接在汇编里，tagged pointer 和 nil 的判断在汇编层就短路了，只有慢速路径才落到 C++ 代码。

---

## 总结

四条规则的判定标准是方法名前缀，不是方法实际做了什么。`[s copy]` 返回同一个对象你也得 release，`stringByAppendingString:` 返回新对象你也不能 release——这个契约不讲理，但正因为不讲理才可靠。CF 的 Create/Get Rule 是同一套思想的另一种方言。

引用计数优先存在 nonpointer isa 里。实测在模拟器和 arm64e 上是 8 位，第 254 次 retain 溢出，一半（`RC_HALF` = 128）挪到 SideTable。19 位那个数字只属于 A7~A11。而且 `extra_rc` 现在直接就是引用计数，不是 `retainCount - 1`——这个语义改动顺带干掉了旧版的 `deallocating` 位。

自动释放池是分页的指针栈，哨兵是 `nil`。空池占位符、相邻指针合并、返回值优化这三个机制在 2014 年那批经典文章里都还不存在，今天抓到的行为和那些截图对不上是正常的。

引用计数归零后 `dealloc` 是一次真实的消息发送。如果对象没有弱引用、关联对象、C++ 析构、SideTable 计数，销毁就是一句 `free`。但 `dealloc` 的执行线程取决于谁持有最后一个引用，所以里面不要碰 UIKit，也不要管理稀缺资源。

最后一条方法论：这一篇里所有"网上说的不对"的结论，都不是靠读更多文章得出来的，而是靠按源码的位域定义解析 isa、然后跑一遍看数字对不对。**遇到位宽、常量、时机这类问题，测一次比读十篇文章可靠。**

下一篇 [[iOS 内存：ARC 的两半]]。

## 参考资料

### 官方

- [Memory Management Policy](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html)：四条规则的权威措辞
- [Practical Memory Management](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html)：setter 顺序和 dealloc 禁忌的官方理由
- [CoreFoundation Ownership Policy](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Concepts/Ownership.html)：Create Rule / Get Rule
- [Clang ARC Specification](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)：ARC 下被禁用的方法清单
- [apple-oss-distributions/objc4](https://github.com/apple-oss-distributions/objc4)：`isa.h` 是所有位宽争议的最终裁判；`objc-config.h` 管着几个关键开关；`NSObject.mm` 里是池的完整实现

### 经典

- [Always Processing — objc_retain](https://alwaysprocessing.blog/2023/07/22/objc-retain)：少见的跟着当前 objc4 走读的文章，比多数中文教程新
- [draven — 黑箱中的 retain 和 release](https://draven.co/rr/)：概念讲得清楚，位宽数字已过时
- [sunnyxx — 黑幕背后的 Autorelease](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/)：中文圈引用率最高的一篇，双向链表和哨兵的说法源头，成文早于第五节那三个新机制
- [draven — 自动释放池的前世今生](https://draven.co/autoreleasepool/)：三篇里和当前源码吻合度最高的

### 本地

- [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]]
- [[iOS 内存：ARC 的两半]]

---

实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -target arm64-apple-ios17.0-simulator`，ARC 与 MRC 两份分别构建。

isa 位域实验依赖 objc4 当前的定义，属于实现细节。这个实验真正可复用的是方法本身：照着 `isa.h` 解析位，用实际行为验证源码，而不是相信任何一篇文章给出的数字。

> 待真机补测：同一组实验在 iPhone 15 / iOS 26.5 上复现，确认 arm64e 真机的位宽与溢出临界点是否与模拟器一致。
