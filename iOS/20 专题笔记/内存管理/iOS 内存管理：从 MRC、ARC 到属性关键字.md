---
title: 【iOS】Objective-C 内存管理：从 MRC、ARC 到属性关键字
published: 2026-07-26
description: 从 MRC 的四条所有权规则出发，追到 ARC 的编译器插桩与 runtime 支持，最后落到 strong、copy、weak、atomic 等属性关键字的实际语义。
tags:
  - iOS
  - Objective-C
  - Memory
  - MRC
  - ARC
  - LLVM
  - 并发
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 4
draft: true
covers:
  - MRC 所有权规则
  - ARC 编译器插桩与 runtime 支持
  - Objective-C 属性关键字
---
# Objective-C 内存管理：从 MRC、ARC 到属性关键字

MRC、ARC 和属性关键字不是三套知识。MRC 给出所有权规则，ARC 把规则翻译成编译器插桩与 runtime 操作，属性关键字再把所有权、原子性和接口暴露方式写进类的公开契约。

本文按这条因果链整合原来的三篇文章。原有论证、代码、实验、纠错、参考资料和环境说明全部保留，只调整章节层级、过渡语句和文内导航。

---

## 第一部分：MRC 的所有权规则：retain、release 与 autorelease


### 一、四条规则

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

#### Core Foundation 说的是同一件事的另一种方言

CF 没有 `alloc`/`new`/`copy` 这四个词，它有自己的一套：

- **Create Rule**：函数名里含 `Create` 或 `Copy` 的，返回 +1，你负责 `CFRelease`。
- **Get Rule**：函数名里含 `Get` 的，返回 +0，不许 release。

和 ObjC 那四个前缀是同一个思想——靠函数名约定表达所有权，不检查实现。两边桥接时用三个关键字划边界：`__bridge` 只做类型转换不动所有权；`__bridge_retained`（等价于 `CFBridgingRetain`）把对象交给 CF 侧并 +1；`__bridge_transfer`（等价于 `CFBridgingRelease`）把 CF 侧的所有权交还给 ARC。

搞清楚这两套方言其实是同一件事，桥接就不用死记了：问一句"这次转换之后，谁负责最后那次 release"。

---

### 二、手写一遍，才知道 ARC 删掉了什么

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

#### 编译产物里的一个小陷阱

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

### 三、autorelease 解决什么问题

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

### 四、引用计数存在哪：一个能测出来的问题

中文资料在这一节几乎全体过时了，而且过时得很有系统性。

#### nonpointer isa

早期的 isa 就是一个纯类指针，引用计数全部放在一张全局表里，每次 retain/release 都要加锁查表。iOS 7 前后引入 nonpointer isa：64 位指针有大量位闲着，干脆把常用信息塞进去，其中就包括引用计数。

`retain` 的快速路径变成把 isa 里的计数字段加一，用 `ldxr` / `stxr`（ARM64 的独占访问指令）做无锁 CAS 循环，根本不碰锁。只有装不下了才溢出到 SideTable。

关键在于"装不下"的门槛是多少。

#### 两个分支，两套位宽

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

#### 实测

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

#### 顺带修正一个我自己差点写错的地方

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

#### 别拿 retainCount 调试

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

### 五、autorelease 池的实现

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

#### 三个老文章里没有的机制

中文圈关于 autorelease 最经典的两篇写于 2014 年前后，下面这些当时还不存在。今天用新 Xcode 抓到的行为和那些文章的截图对不上，不是你搞错了。

**空池占位符。** 大量自动释放池从创建到销毁一个对象都没装过——RunLoop 每轮迭代都会 push/pop 一个池，但多数迭代里什么都没发生。运行时为此做了优化：如果当前线程只 push 了一个池、且从没真正塞过对象，就把热页指针设成哨兵值 `1`，完全不分配内存。等真要塞第一个对象时才补边界并分配页。

**相邻指针合并。** `objc-config.h` 里 `SUPPORT_AUTORELEASEPOOL_DEDUP_PTRS` 的注释写的是"combine consecutive pointers to the same object"，实现上会回看栈顶最近 4 个条目，命中就在已有条目上递增计数并把它挪到栈顶（LRU），pop 时按计数循环 release。循环里反复 `[NSNumber numberWithInt:]` 是典型受益场景。

**返回值优化让很多 autorelease 根本没入池。** `objc_autoreleaseReturnValue` 会先判断调用方是不是马上就要 retain 这个返回值，是的话对象直接以 +1 状态交出去，通过线程本地存储接力，完全不经过池页。判断方式本身也换过代——`objc-config.h` 里 `HAS_RETURNADDR_AUTORELEASE_ELISION` 在 arm64 上是 1，注释说得很直白：

> autorelease elision based on comparing the return address of the call and claim. When 0, we only support the older scheme that inspects the caller's code for a claim call or sentinel NOP.

也就是说"读调用方指令流找哨兵 NOP"是旧方案，当前 arm64 走的是返回地址比对。对应还多了一个 `objc_claimAutoreleasedReturnValue` 入口，签名注释写着 "without a NOP in the caller on ARM64"。具体机制在下一篇展开。

所以"autorelease 就是把 release 延后"这句话今天已经不完整了。

#### 池什么时候排空

主线程上由 RunLoop 的 observer 负责：进入时 push，即将休眠时 pop 再 push，退出时 pop。所以主线程的 autoreleased 对象最迟活到当前这轮 RunLoop 结束。

子线程麻烦一些。`NSThread` 手动起的线程有隐式池，但排空时机不受你控制；GCD 队列会自动排空，时机同样不保证。所以在子线程跑一个产生大量临时对象的长任务时，得自己套 `@autoreleasepool`，否则内存会一路涨到任务结束。

如果一个对象在完全没有池的上下文里被 autorelease，运行时会打日志：

```text
Object 0x... of class ... autoreleased with no pool in place - just leaking
```

这个函数在源码里是 `BREAKPOINT_FUNCTION(void objc_autoreleaseNoPool(id obj))`，意味着你可以直接 `b objc_autoreleaseNoPool` 断在现场。

至于"循环里套 `@autoreleasepool` 能降低内存峰值"这条标准答案，今天需要打个折。套一层池本身有成本（push/pop，可能还要分配页），只有循环体确实产生大量 autoreleased 对象时才划算——`imageWithContentsOfFile:`、`stringWithFormat:`、JSON 解析这类。而且有了上面说的指针合并和返回值优化，2014 年那些 demo 里的效果今天已经明显缩水。先测再改，别照抄。

---

### 六、release 到 0 之后

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

#### 所以 dealloc 里该写什么

Apple 的态度很明确：

> You should typically not manage scarce resources such as file descriptors, network connections, and buffers or caches in a `dealloc` method. In particular, you should not design classes so that `dealloc` will be invoked when you think it will be invoked.

理由是 `dealloc` 的调用时机不可控——可能被延迟，可能因为应用退出而根本不执行。

对 iOS 最毒的是线程问题：一个对象的最后一次 release 发生在哪个线程，它的 `dealloc` 就在哪个线程执行。如果某个后台任务持有了最后一个引用，`dealloc` 里碰 UIKit 就直接炸，而且这种崩溃极难复现——它取决于哪条引用链最后一个断开。

`dealloc` 里合适做的事只有一类：解除本对象和外界的关联。移除通知观察者、置空 delegate、取消定时器。真正的资源释放应该有显式的生命周期方法。

---

### 七、NSZombie 与它的替代品

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

### 八、几个已经不准的说法

- **"extra_rc 占 19 位。"** 只在无指针认证的 arm64（A7~A11）上成立。arm64e 真机、所有 iOS 模拟器、以及所有 x86_64 都是 8 位。19 位在 Intel Mac 上从来就没对过。
- **"isa 里有一个 deallocating 位。"** 现在没有了。`extra_rc` 的语义从 `retainCount - 1` 改成 `retainCount` 之后，`extra_rc == 0 && has_sidetable_rc == 0` 就能表示正在析构，那一位被省掉了。
- **"AutoreleasePoolPage 固定 4096 字节。"** 数值对，但源码里是平台宏而非字面常量；控制它的 `PROTECT_AUTORELEASEPOOL` 在发布版 objc4 里没有定义。
- **"NSAutoreleasePool 和 @autoreleasepool 是新旧两种写法。"** `@autoreleasepool {}` 编译后直接调 `objc_autoreleasePoolPush` / `Pop`，走运行时内建的分页栈；`NSAutoreleasePool` 是一个真实的类，走另一套代码路径。效果等价，实现是两条路。老代码里的 `[pool drain]` 则是 GC 时代的遗留，GC 早已移除，今天它和 `release` 没有区别。
- **"autorelease 就是把 release 延后。"** 返回值优化会让很多 autorelease 调用根本没入池，见第五节。
- **"过度释放会立刻崩溃。"** 对象销毁后指针没被清空，继续用只是看起来正常，直到那块内存被复用。这就是野指针 bug 难查的根本原因。
- **"SideTable 是一张全局哈希表。"** 是 `StripedMap<SideTable>`，按对象地址哈希分片，各片独立加锁。分片数在真机上是 8，Mac 和模拟器上是 64。
- **"retain/release 都是 C 实现的。"** 部分架构上快速路径直接在汇编里，tagged pointer 和 nil 的判断在汇编层就短路了，只有慢速路径才落到 C++ 代码。

---

### 总结

四条规则的判定标准是方法名前缀，不是方法实际做了什么。`[s copy]` 返回同一个对象你也得 release，`stringByAppendingString:` 返回新对象你也不能 release——这个契约不讲理，但正因为不讲理才可靠。CF 的 Create/Get Rule 是同一套思想的另一种方言。

引用计数优先存在 nonpointer isa 里。实测在模拟器和 arm64e 上是 8 位，第 254 次 retain 溢出，一半（`RC_HALF` = 128）挪到 SideTable。19 位那个数字只属于 A7~A11。而且 `extra_rc` 现在直接就是引用计数，不是 `retainCount - 1`——这个语义改动顺带干掉了旧版的 `deallocating` 位。

自动释放池是分页的指针栈，哨兵是 `nil`。空池占位符、相邻指针合并、返回值优化这三个机制在 2014 年那批经典文章里都还不存在，今天抓到的行为和那些截图对不上是正常的。

引用计数归零后 `dealloc` 是一次真实的消息发送。如果对象没有弱引用、关联对象、C++ 析构、SideTable 计数，销毁就是一句 `free`。但 `dealloc` 的执行线程取决于谁持有最后一个引用，所以里面不要碰 UIKit，也不要管理稀缺资源。

最后一条方法论：这一篇里所有"网上说的不对"的结论，都不是靠读更多文章得出来的，而是靠按源码的位域定义解析 isa、然后跑一遍看数字对不对。**遇到位宽、常量、时机这类问题，测一次比读十篇文章可靠。**

下一部分是 [[#第二部分：ARC 的两半：编译器插桩与 runtime 支持|ARC 的两半]]。

### 参考资料

#### 官方

- [Memory Management Policy](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html)：四条规则的权威措辞
- [Practical Memory Management](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html)：setter 顺序和 dealloc 禁忌的官方理由
- [CoreFoundation Ownership Policy](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Concepts/Ownership.html)：Create Rule / Get Rule
- [Clang ARC Specification](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)：ARC 下被禁用的方法清单
- [apple-oss-distributions/objc4](https://github.com/apple-oss-distributions/objc4)：`isa.h` 是所有位宽争议的最终裁判；`objc-config.h` 管着几个关键开关；`NSObject.mm` 里是池的完整实现

#### 经典

- [Always Processing — objc_retain](https://alwaysprocessing.blog/2023/07/22/objc-retain)：少见的跟着当前 objc4 走读的文章，比多数中文教程新
- [draven — 黑箱中的 retain 和 release](https://draven.co/rr/)：概念讲得清楚，位宽数字已过时
- [sunnyxx — 黑幕背后的 Autorelease](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/)：中文圈引用率最高的一篇，双向链表和哨兵的说法源头，成文早于第五节那三个新机制
- [draven — 自动释放池的前世今生](https://draven.co/autoreleasepool/)：三篇里和当前源码吻合度最高的

#### 本地

- [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]]
- [[#第二部分：ARC 的两半：编译器插桩与 runtime 支持|ARC 的两半]]

---

实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -target arm64-apple-ios17.0-simulator`，ARC 与 MRC 两份分别构建。

isa 位域实验依赖 objc4 当前的定义，属于实现细节。这个实验真正可复用的是方法本身：照着 `isa.h` 解析位，用实际行为验证源码，而不是相信任何一篇文章给出的数字。

> 待真机补测：同一组实验在 iPhone 15 / iOS 26.5 上复现，确认 arm64e 真机的位宽与溢出临界点是否与模拟器一致。

---

## 第二部分：ARC 的两半：编译器插桩与 runtime 支持

面试里问"ARC 是什么"，最常见的回答是"编译器在编译时自动插入 retain 和 release"。这句话不算错，但它把一半的事实说没了。

真正的分工是：编译器决定在哪插、插哪一个函数，运行时决定这个函数具体怎么干。前者是纯静态分析，运行时对此一无所知；后者涉及引用计数存在 isa 的哪几位、要不要加锁、能不能走快速路径，编译器同样一无所知。

有一个地方能把这种分工看得特别清楚——方法返回值的所有权交接。两边谁都看不见对方的代码，只能隔着一个返回地址对暗号。这篇文章最长的一节花在这上面。

本文第一部分 [[#第一部分：MRC 的所有权规则：retain、release 与 autorelease|MRC 的所有权规则]] 讲规则本身，这一部分讲规则怎么被翻译成代码。

---
![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260806211750893.png)


![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260806211757541.png)




### 先看编译器插了什么

把一段普通的 ARC 代码编成 LLVM IR。测试代码覆盖常见的所有权场景：

```objc
@interface Box : NSObject
@property (nonatomic, strong) NSString *label;
@end

void localStrong(void) {            // 局部强引用进出作用域
    Box *b = [[Box alloc] init];
    NSLog(@"%@", b);
}

void setLabel(Box *b, NSString *s) { // 属性赋值
    b.label = s;
}

Box *makeBox(void) {                 // 返回一个 +0 对象
    Box *b = [[Box alloc] init];
    return b;
}

void useBox(void) {                  // 接收返回值
    Box *b = makeBox();
    NSLog(@"%@", b);
}

void weakDance(Box *b) {             // weak 的读写
    __weak Box *w = b;
    Box *strong = w;
    NSLog(@"%@", strong);
}

void poolScope(void) {               // 自动释放池
    @autoreleasepool {
        Box *b = [[Box alloc] init];
        NSLog(@"%@", b);
    }
}
```

`clang -fobjc-arc -S -emit-llvm -O0` 编出来，把每个函数里的 `objc_*` 调用抽出来：

```text
--- localStrong ---   objc_alloc_init, storeStrong
--- setLabel ---      storeStrong × 4
--- makeBox ---       objc_alloc_init, retain, storeStrong, autoreleaseReturnValue
--- useBox ---        [asm marker], retainAutoreleasedReturnValue, storeStrong
--- weakDance ---     storeStrong, initWeak, loadWeakRetained, storeStrong,
                      destroyWeak, storeStrong, destroyWeak
--- poolScope ---     autoreleasePoolPush, objc_alloc_init, storeStrong, autoreleasePoolPop
```

`[[Box alloc] init]` 没有变成两次 `objc_msgSend`，而是一个 `objc_alloc_init`。这组调用优化由 `-fobjc-runtime` 的版本门控，把它切换一下就能看到分界线在哪：

```text
-fobjc-runtime=ios-12.0   : objc_alloc
-fobjc-runtime=ios-13.0   : objc_alloc_init  objc_opt_new
```

`objc_alloc` 早得多，`objc_alloc_init` 和 `objc_opt_new` 是 iOS 13 那一代运行时的产物。它们和所有权无关，纯粹是把高频消息发送换成直调运行时函数，省掉一次方法查找。

局部强引用出作用域时清理用的是 `objc_storeStrong(&slot, nil)`。这个函数把"retain 新值、赋值、release 旧值"打包成一次调用，传 `nil` 就退化成纯释放。ARC 大量复用它，所以在 IR 里出现得非常频繁。

`weakDance` 那一行把 weak 三件套完整暴露了：`initWeak` 建槽、`loadWeakRetained` 读、`destroyWeak` 销毁。读一个 weak 变量为什么要 retain，是因为从读出来到用它之间对象可能在别的线程被释放——必须在持锁的前提下把它 retain 住，要么拿到活着的对象，要么拿到 `nil`，不允许有中间态。这部分留给 [[iOS weak 的实现：SideTable 与置 nil 的时机]]。

---

### 那些桩在优化后大部分会消失

同一份代码换成 `-O1`：

```text
             -O0                                    -O1
localStrong  objc_alloc_init, storeStrong           objc_alloc_init, release
setLabel     storeStrong × 4                        （一个都没有）
makeBox      alloc_init, retain, storeStrong,       alloc_init, autoreleaseReturnValue
             autoreleaseReturnValue
useBox       marker, retainAutoreleasedReturnValue, alloc_init, release
             storeStrong
poolScope    poolPush, alloc_init, storeStrong,     poolPush, alloc_init, release, poolPop
             poolPop
```

`setLabel` 从四个 `storeStrong` 变成零。`useBox` 里整套返回值握手直接蒸发——`makeBox` 被内联了，编译器发现对象从创建到销毁全在一个函数里，中间的配平操作一个都不需要。

所以"ARC 插了这么多桩是不是很浪费"这个问题，答案是：**拿 `-O0` 的 IR 去论证 ARC 昂贵是无效论证。** LLVM 有一个专门的 ObjCARC optimizer pass 负责消掉冗余，发布构建里剩下的调用远比这里看到的少。

但它不是万能的。注意 `makeBox` 在 `-O1` 下还剩一个 `autoreleaseReturnValue`——为什么没被消掉？因为这个 pass 只被允许**成对**消除 retain/release，而这次 autorelease 的配对方在别的函数里。跨函数它不敢动。这也解释了为什么发布构建里 ARC 仍然有开销：所有跨越函数边界的所有权转移，都消不掉。

这还顺带解释了第一部分那个小疑点：为什么刚创建的对象读出来 `extra_rc` 是 2 而不是 1。`-O0` 下每个 `id` 参数都会被 `objc_storeStrong` 进一个本地槽位，那次多出来的 retain 是**被调方**在函数体开头做的，不在调用点。换 `-O1` 这个槽位直接消失。

---

### 返回值的所有权交接

`useBox` 在 `-O0` 下的完整 IR：

```llvm
define void @useBox() #1 {
  %1 = alloca ptr, align 8
  %2 = call ptr @makeBox()
  call void asm sideeffect "mov\09fp, fp\09\09// marker for objc_retainAutoreleaseReturnValue", ""()
  %3 = call ptr @llvm.objc.retainAutoreleasedReturnValue(ptr %2) #2
  store ptr %3, ptr %1, align 8
  ...
}
```

编成 arm64 汇编：

```asm
bl	_makeBox
; InlineAsm Start
mov	x29, x29	; marker for objc_retainAutoreleaseReturnValue
; InlineAsm End
bl	_objc_retainAutoreleasedReturnValue
```

`mov x29, x29` 把帧指针赋值给自己，是一条彻底的空操作。

（顺带解释一个字面上的怪事：那条注释写的是 `marker for objc_retainAutoreleaseReturnValue`，少了个 d，而实际配对的是带 d 的 `retainAutoreleasedReturnValue`。这是 clang 里从 2011 年留到现在的一处笔误，别被它带偏。）

#### 要解决的问题

Cocoa 的约定是：方法名不以 `alloc`/`new`/`copy`/`mutableCopy` 开头，就该返回一个调用方不拥有的对象。老实的做法是被调方 `retain` 一次再 `autorelease` 一次，调用方要留住就自己再 `retain`。

一次简单的 getter 调用，为此产生了一次入池、一次出池、两次引用计数操作。而绝大多数情况下调用方拿到值马上就要持有它——中间这趟池完全是白跑的。更糟的是对象存活时间变得不可控：它要等池排空才真正释放，在没有 RunLoop 及时排空的线程上会堆积成内存峰值。

顺带一提，那个"方法名前缀"的约定在编译器里不是约定，是强制的类型信息。我给一个属性起名叫 `copyLabel`，编译直接失败：

```text
error: property follows Cocoa naming convention for returning 'owned' objects
note: explicitly declare getter '-copyLabel' with
      __attribute__((objc_method_family(none))) to return an 'unowned' object
```

编译器把这些前缀识别成 method family，并隐式给它们加上 `ns_returns_retained`。所以上一篇说的"命名约定纯靠自觉"，在 ARC 时代只对方法**实现**成立，方法**声明**这一侧编译器是会管的。

#### 暗号怎么对

被调方的 `objc_autoreleaseReturnValue` 不会立刻决定要不要 autorelease，而是先把对象扣在线程本地存储里，连同"我是被谁调用的"这个返回地址一起：

```c
setReturnAutoreleaseInfo({obj, cameFromRootAutorelease, disposition, clientReturnAddress()});
```

对象没进池，只是暂存。真正的验证发生在调用方一侧的 `objc_retainAutoreleasedReturnValue` 里，它算一个差值：

```c
const uintptr_t expectedDeltaWithNOP = 8;
const uintptr_t expectedDeltaNoNOP   = 4;
...
uintptr_t delta = currentReturnAddress - previousReturnAddress;
if (delta == expectedDelta) return info.getReturnDisposition();
```

两次调用的返回地址如果只差一条指令（无 marker，4 字节）或两条指令（有 marker，8 字节），就说明这两次调用在源码上确实是紧挨着的同一件事。握手成立，对象以 +1 原样交出，两次引用计数操作和一次入池全部省掉。

那 marker 呢？在当前 arm64 的实现里它降级成了兜底——只有 delta 对不上、且调用方声明自己会发 marker 时，才回过头去读那条 `mov x29, x29` 确认一次：

```c
if (expectsNOP) {
    if (callerAcceptsOptimizedReturn(info.getReturnAddress()))
        return info.getReturnDisposition();
}
```

这个分支由 `objc-config.h` 里的开关控制：

```c
// Define HAS_RETURNADDR_AUTORELEASE_ELISION where we support autorelease
// elision based on comparing the return address of the call and claim. When 0,
// we only support the older scheme that inspects the caller's code for a claim
// call or sentinel NOP.
#if __arm64__
#   define HAS_RETURNADDR_AUTORELEASE_ELISION 1
#else
#   define HAS_RETURNADDR_AUTORELEASE_ELISION 0
#endif
```

注释里那句 "the older scheme that inspects the caller's code" 说的就是读指令那套。arm64 已经不走它了，但很多中文资料（包括我这篇的第一版）仍然把它当成现行机制在讲。

握手失败最坏就是白跑一趟：被调方老实入池，调用方老实 `objc_retain`，退回没有优化时的行为，不会出错。这个"失败也不会错"的性质，才是它敢在每一次方法调用上生效的原因。

#### x86_64 靠指令序列当签名

`HAS_RETURNADDR_AUTORELEASE_ELISION` 在非 arm64 上是 0，所以 x86_64 走的仍然是读指令那条路。同一份代码换 target，`-O0` 编译：

```asm
callq	_makeBox
movq	%rax, %rdi
callq	_objc_retainAutoreleasedReturnValue
```

没有 marker。因为 `movq %rax, %rdi` 紧跟一个 `callq` 到 `objc_retainAutoreleasedReturnValue`，这串指令本身就够独特了。

但判定过程比"匹配四个字节"复杂得多，要走三跳：

```c
if (*ra4 != 0xe8c78948) return false;                          // movq %rax,%rdi; callq
ra1 += (long)*(const unaligned_int32_t *)(ra1 + 4) + 8l;        // 按相对位移跳到 dyld stub
if (*(unaligned_uint16_t *)ra1 != 0x25ff) return false;         // stub 里必须是 jmpq *sym(%rip)
ra1 += 6l + (long)*(const unaligned_int32_t *)(ra1 + 2);        // 顺着 RIP 相对寻址解引用
if (*(const void **)ra1 != objc_retainAutoreleasedReturnValue &&
    *(const void **)ra1 != objc_unsafeClaimAutoreleasedReturnValue)
    return false;
```

先比字节，再跳到 stub 确认形状，最后拿到真正的函数指针比对。这个判定硬编码了 dyld stub 的布局，脆得很。

白名单里还有一处沉默的信息：`objc_claimAutoreleasedReturnValue` 不在里面。免 marker 那条路在 x86_64 上从一开始就走不通，因为这里判定的依据就是那串指令本身——把指令拿掉，什么都不剩了。

arm64 和 x86_64 的差别，本质是两种回答同一个问题的方式：

| | 怎么确认"调用方马上就要 retain" |
| --- | --- |
| arm64 | 比对两次调用的返回地址差值，marker 只是兜底 |
| x86_64 | 从返回地址读机器指令，跳三次确认符号 |

这个对照说明 marker 不是什么魔法，只是识别调用方意图的一种手段。手段可以换，也确实换过。

#### IR 里的表达变过，但 marker 没消失

`-O1` 及以上，marker 不再以 inline asm 形式出现在 IR 里，而是变成挂在调用指令上的 operand bundle。这段要用一个跨函数、无法内联的调用点才看得到（文章开头那六个函数在 `-O1` 下会被整段优化掉）：

```objc
void consume(NSString *s);
void useGetter(Box *b) { NSString *s = b.label; consume(s); }
```

```llvm
%2 = tail call ptr @"objc_msgSend$label"(ptr noundef %0, ptr noundef undef)
       [ "clang.arc.attachedcall"(ptr @llvm.objc.retainAutoreleasedReturnValue) ]
```

注意 marker 本身并没有消失。同一份代码的 `-O1` 汇编里 `mov x29, x29` 照样在（我数了一下，两处），只是不再带 `; InlineAsm` 的注释——它现在由后端根据 bundle 发出，而不是前端硬塞进 IR。变的是谁做决定，不是发不发。

这个改动是为了给 `objc_claimAutoreleasedReturnValue` 铺路：它完全不需要 marker，靠 bundle 就能完成绑定。

时间线上有个挺有意思的落差。运行时这一侧，`objc_claimAutoreleasedReturnValue` 从 iOS 16 起就在 libobjc 里导出了；编译器这一侧，上游 LLVM 直到 2025 年 5 月才合入生成它的支持。三年空窗期里符号一直在，只是没人调。

更有意思的是空窗期还没结束。我用 Xcode 26 的 clang，target 从 `ios17.0` 一路拉到 `ios26.0`，`-O1` 下 bundle 绑定的仍然是 `retainAutoreleasedReturnValue`，一次 claimARV 都没发出来。所以对写业务代码的人来说，"免 marker"到今天仍然不是一个你能感知、更谈不上能控制的特性——它是运行时和编译器之间的一次协议升级，两边什么时候真正接上，你说了不算。

#### 第四种收尾

返回值优化在调用方一侧其实有三个入口，前面只讲了一个。如果拿到的值根本不需要持有：

```objc
void unsafeSink(Box *b) { __unsafe_unretained NSString *s = [b direct]; }
```

生成的是 `objc_unsafeClaimAutoreleasedReturnValue`——握手成功就直接 release，失败就什么都不做。用途是避免"retain 一下马上又 release"这种无谓的往返。

---

### 编译器和运行时各自管什么

编译器管静态的那半边。所有权限定符（`__strong` / `__weak` / `__unsafe_unretained` / `__autoreleasing`）的推导是纯类型系统的事，不需要运行任何代码就能定下来；哪个语法位置该插哪个 `objc_*` 调用，同理。它还负责生成 `.cxx_destruct`、在 `-dealloc` 末尾补上对父类的转发、判断某个调用点能不能替换成 `objc_alloc` 这类快捷入口。

运行时管动态的那半边，全是编译期问不出答案的问题。引用计数存在 isa 的哪几位、CAS 循环怎么写、溢出了往 SideTable 挪多少；这个类有没有重写 `retain`/`release`，能不能走快速路径；弱引用表怎么管、对象销毁时怎么批量置 `nil`。以及上面那个靠返回地址对暗号的机制——编译器根本不知道它存在。

一句话切开：**"这里要不要调 `objc_retain`"是编译器的问题，"`objc_retain` 怎么让计数加一"是运行时的问题。**

ARC 用到的运行时函数不算多：

| 用途 | 函数 |
| --- | --- |
| 基本增减 | `objc_retain` / `objc_release` / `objc_autorelease` / `objc_retainAutorelease` |
| 强引用赋值 | `objc_storeStrong` |
| 弱引用 | `objc_initWeak` / `objc_storeWeak` / `objc_loadWeak` / `objc_loadWeakRetained` / `objc_destroyWeak` / `objc_copyWeak` / `objc_moveWeak` |
| 返回值优化（被调方） | `objc_autoreleaseReturnValue` / `objc_retainAutoreleaseReturnValue` |
| 返回值优化（调用方） | `objc_retainAutoreleasedReturnValue` / `objc_unsafeClaimAutoreleasedReturnValue` / `objc_claimAutoreleasedReturnValue` |
| 自动释放池 | `objc_autoreleasePoolPush` / `objc_autoreleasePoolPop` |
| Block | `objc_retainBlock` |

表里 `objc_claimAutoreleasedReturnValue` 是个例外：Clang ARC 规范的 Runtime support 一节里没有它，规范至今只写了带 `unsafe` 的那个。这个"运行时先有、规范后有（或者干脆没有）"的顺序本身就说明了两边的关系——规范划的是编译器必须遵守的语义，运行时可以在语义不变的前提下自己加快路。

`objc_retainAutoreleaseReturnValue` 也是同一种情况。规范给的等价实现是 `objc_autoreleaseReturnValue(objc_retain(value))`，而 objc4 在 arm64 上走的是另一条路，disposition 从 `ReturnAtPlus0` 变成了 `ReturnAtPlus1`。语义不变，实现换了。

#### 合成访问器有一条专门的捷径

合成的 `nonatomic strong` getter，在 `-O0` 下一个 `objc_*` 调用都不生成，就是裸的取地址、load、返回。

我第一版把这解释成"编译器判定 ivar 不可能被释放"。这个解释是错的——如果真是通用的数据流推理，那手写的同形状代码也该省掉。实测它没省：

```objc
- (NSString *)direct { return _label; }   // → retainAutoreleaseReturnValue
```

所以这是 CodeGen 对**合成访问器**开的专门捷径，而且只对 `nonatomic` 生效。同一个类里三个属性，三种结果：

```text
-[Box label]           （无 objc_* 调用）           nonatomic strong getter
-[Box atomicLabel]     objc_getProperty             atomic strong getter
-[Box setTextValue:]   objc_setProperty_nonatomic_copy   nonatomic copy setter
```

Clang ARC 规范并没有为这条捷径背书，它只在 Unretained return values 那一节留了口子："callers must not assume that the value is actually in the autorelease pool."

于是"getter 会做 retain + autorelease"这个说法要拆成两半看：被调方那一半在最常见的情况下确实不做了；调用方那一半照做不误——`b.label` 的调用点仍然会发 `objc_retainAutoreleasedReturnValue`。真正的风险在混编：`-fobjc-arc` 是 per-file 的，MRC 文件里调这个 getter，拿到的是一个没进池的 +0 对象，谁也不替你续命。这就是 Mike Ash 那篇《When an Autorelease Isn't》讲的坑，只不过今天触发它的路径更多了。

顺带说，`objc_retainAutoreleaseReturnValue`（无 d）在我那六个函数里一次都没出现，但它一点都不冷门。任何返回 +0 值的函数都会生成它——上面那个 `direct` 就是。它没出现只是因为我的测试函数要么返回新建对象，要么不返回对象。

---

### .cxx_destruct 是编译期生成的

ARC 下不用手写释放 ivar 的代码，那这活谁干的？

一个叫 `.cxx_destruct` 的方法。名字来自 C++——它本来是给带 C++ 成员对象的类做析构的，ARC 把它借来承载 ivar 的自动释放。编译产物里能直接看到：

```llvm
define internal void @"\01-[Box .cxx_destruct]"(ptr noundef %0, ptr noundef %1) #1 {
  ...
  call void @llvm.objc.storeStrong(ptr %6, ptr null) #2
  ...
}
```

就是对每个 ARC 托管的 ivar 调一次 `objc_storeStrong(&ivar, nil)`。

运行时销毁对象时并不发消息，而是沿继承链从子类往父类走，每一层直接取 IMP 就调，并且显式跳过消息转发：

```c
for ( ; cls; cls = cls->getSuperclass()) {
    if (!cls->hasCxxDtor()) return;
    dtor = lookupMethodInClassAndLoadCache(cls, SEL_cxx_destruct);
    if (dtor != _objc_msgForward_impcache) (*dtor)(obj);
}
```

每个类只清自己那几个 ivar，谁也管不着谁。编译期按类生成，运行期按链调用——这个小细节本身就是那套分工的缩影。

ARC 下手写 `[super dealloc]` 会直接编译失败（`ARC forbids explicit message send of 'dealloc'`），因为编译器自己会在 dealloc 末尾发 `objc_msgSendSuper2`。不过要注意，ivar 的销毁和这次转发不是同一个时机——规范说 ivar 是在"控制流进入**根类** dealloc 之后的某个时刻"被销毁的，而且销毁顺序是未指定的。

对象销毁的完整链条在第一部分讲过。这里补一句：如果对象没有弱引用、没有关联对象、没有 C++ 析构、引用计数没溢出过，`rootDealloc` 会走快速路径直接 `free`，`.cxx_destruct` 根本不会被调用。

---

### 几个说法需要纠正

**"ARC 是垃圾回收。"** 不是。GC 要扫描对象图判断可达性，可能有停顿；ARC 是编译期插桩的确定性引用计数，没有扫描、没有停顿。这是两种完全不同的范式，ARC 从来没做过任何图遍历。

**"ARC 全是编译器做的，运行时只是被动执行。"** 返回地址差值的判定、`hasCustomRR` 的分支、`extra_rc` 溢出的迁移策略，全都是运行时主动做的动态决策。编译器只负责把调用点摆对位置。

**"ARC 会追踪对象之间的引用关系。"** 强引用不追踪。ARC 只维护每个对象自身的一个计数值，谁持有它、持有了几次，全部不记。唯一的例外是 `__weak`——运行时确实在 SideTable 里记着哪些槽指向我，好在销毁时逐个置 `nil`，但这份账不参与计数，也不构成可达性图。所以循环引用无解：检测环需要遍历强引用图，而这张图从来没被建出来过。

**"用了 ARC 就不会内存泄漏。"** 强引用环照样泄漏。Block 捕获 `self` 和 delegate 忘了标 weak 是两个最常见的来源，这个话题在 [[iOS Block 循环引用与 weak-strong dance]] 展开。

**"`.cxx_destruct` 是运行时动态生成的。"** 是编译期生成的真实方法，能在编译产物的方法列表里找到。

**"返回值优化靠一条固定的 `mov fp, fp` 实现。"** 只有 x86_64 是真的在读指令，而它读的还不是这条；arm64 早就换成返回地址比对了，marker 在那边只是兜底。

---

### 总结

ARC 的分工可以压缩成一句话：编译器做静态的所有权分析并决定插桩位置，运行时做动态的计数与快速路径判定，两边靠一组 `objc_*` 函数对接。

这句话有一个容易被误读的推论——插桩多不等于开销大。`-O0` 下 `setLabel` 那四个 `storeStrong`，到 `-O1` 一个不剩。但 ARC optimizer 只能成对消除，跨函数的所有权转移它不敢动，这才是发布构建里 ARC 仍有开销的真正原因。

分工最锋利的地方在返回值交接上。两边谁都看不见对方的代码，只能靠一个返回地址对暗号：arm64 把对象扣在 TLS 里等调用方来认领，靠返回地址差值是 4 还是 8 来判断；x86_64 走的仍是老路，从返回地址读机器指令、跳三次确认符号。暗号对不上就退回传统的 autorelease + retain，只赔性能不赔正确性——这个"失败也不会错"的设计，才是它敢在每一次方法调用上生效的原因。

`.cxx_destruct` 是同一套分工的另一个样本，方向反过来：编译器按类生成，运行时按继承链调用，而且在对象满足快速销毁条件时压根不调。

最后一条方法论，和第一部分一样：这篇里几处"网上说的不对"，包括我自己第一版写错的两处，都不是靠读更多文章发现的，而是换个 `-fobjc-runtime` 版本、换个 target、把手写 getter 和合成 getter 摆在一起编一次。凡是来自记忆的断言都出过错，凡是来自编译产物的断言都对。

再往下走就是属性声明如何表达这些所有权语义，也就是本文第三部分；weak 和 Block 的完整机制仍由系列后续专题展开。

### 参考资料

#### 规范与源码

- [Clang ARC Specification](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)：运行时函数清单、method family、unretained return values 的语义定义
- [objc4 — objc-object.h / NSObject.mm](https://github.com/apple-oss-distributions/objc4)：`callerAcceptsOptimizedReturn`、`prepareOptimizedReturn`、四个返回值优化函数
- [objc4 — objc-config.h](https://github.com/apple-oss-distributions/objc4)：`HAS_RETURNADDR_AUTORELEASE_ELISION` 这个开关决定了 arm64 走哪条路
- [llvm-project PR #138696](https://github.com/llvm/llvm-project/pull/138696)：上游 LLVM 支持生成 `objc_claimAutoreleasedReturnValue`，2025 年 5 月

#### 文章

- [Mike Ash — Automatic Reference Counting](https://www.mikeash.com/pyblog/friday-qa-2011-09-30-automatic-reference-counting.html)：ARC 早期最清晰的整体讲解，但文中关于 getter 用哪个返回值优化函数的描述，和现在编译器的实际行为已经对不上
- [Mike Ash — When an Autorelease Isn't](https://www.mikeash.com/pyblog/friday-qa-2014-05-09-when-an-autorelease-isnt.html)：返回值优化误伤导致过早释放的真实调试案例
- [sunnyxx — ARC 下 dealloc 过程及 .cxx_destruct 的探究](https://blog.sunnyxx.com/2014/04/02/objc_dig_arc_dealloc/)：用 lldb watchpoint 实测 `.cxx_destruct` 的执行时机，但没提 `rootDealloc` 的快速路径会完全跳过它
- [Always Processing — objc_retain](https://alwaysprocessing.blog/2023/07/22/objc-retain)：跟着当前 objc4 走读引用计数实现

#### 本地

- [[#第一部分：MRC 的所有权规则：retain、release 与 autorelease|MRC 的所有权规则]]
- [[#第三部分：属性关键字：从所有权推导，而不是从类型名猜|属性关键字：从所有权推导，而不是从类型名猜]]
- [[iOS weak 的实现：SideTable 与置 nil 的时机]]

---

实验环境：Xcode 26.6（Apple clang 21），target 为 `arm64-apple-ios17.0-simulator` 与 `x86_64-apple-ios17.0-simulator`。

复现命令：

```shell
SDK=$(xcrun --sdk iphonesimulator --show-sdk-path)
clang -fobjc-arc -S -emit-llvm -O0 -isysroot "$SDK" \
      -target arm64-apple-ios17.0-simulator -o out_O0.ll own.m
clang -fobjc-arc -S -O0 -isysroot "$SDK" \
      -target x86_64-apple-ios17.0-simulator -o out_x86.s own.m
```

换 `-O1` 再编一遍做对照，差异最明显的是 `setLabel` 和 `useBox`。想看 `objc_alloc_init` 的版本门控，加 `-fobjc-runtime=ios-12.0` 和 `ios-13.0` 各编一次。

IR 里的函数名带 `llvm.objc.` 前缀，那是 LLVM 的 intrinsic 形式，最终会降级成对同名运行时函数的调用；汇编里看到的才是真实符号 `_objc_storeStrong`。

---

## 第三部分：属性关键字：从所有权推导，而不是从类型名猜

网上流传的属性关键字口诀大概是这样：NSString 用 copy，delegate 用 weak，Block 用 copy，基本类型用 assign。

记住这四条能答对大部分面试题，但它解释不了任何一个"为什么"，也处理不了没见过的情况——一个自定义的 value object 该用什么？一个需要跨线程读写的配置项该用什么？

问题在于这十来个关键字看起来像一个需要死记的排列组合，其实它们在回答三个互不相干的问题。

**所有权**：这个属性持有对象吗，持有到什么程度。`strong` / `copy` / `weak` / `assign` / `unsafe_unretained` 五选一。

**原子性**：`atomic` / `nonatomic` 二选一。

**暴露方式**：`readwrite` / `readonly`，加上 `getter=` / `setter=` 这些命名控制。

三栏正交，各选各的。把它们摊平成一张十几行的表然后去背——这就是为什么怎么背都记不住。

本文所有输出块都是真跑出来的，环境是 Xcode 26.6 加 iOS 模拟器（arm64），完整配置放在文末。

---

### 一、所有权那一栏

#### 编译器到底生成了什么

讲属性关键字最有效的方式是直接看编译器生成的访问器。这里有个分野：并不是所有属性的访问器都长得一样，它们分成两条完全不同的路径。

一条是编译器直接内联展开，setter 里就是几条指令；另一条是调用 objc4 提供的通用访问器函数，把工作丢给运行时。

分界线在这里：

| 属性声明 | setter | getter |
| --- | --- | --- |
| `nonatomic, strong` | `objc_storeStrong`（内联展开） | 裸 load |
| `nonatomic, copy` | `objc_setProperty_nonatomic_copy` | 裸 load |
| `atomic, strong` | `objc_setProperty_atomic` | `objc_getProperty(…, true)` |
| `atomic, copy` | `objc_setProperty_atomic_copy` | `objc_getProperty(…, true)` |
| `weak`（atomic 与否都一样） | `objc_storeWeak` | `objc_loadWeakRetained` + autorelease |
| `CGRect` 这类结构体 + `atomic` | `objc_copyStruct` | `objc_copyStruct` |
| 标量 + `nonatomic` | 裸 store | 裸 load |
| 标量 + `atomic` | 裸 store，无锁无函数调用 | 裸 load |

规律是：`copy` 的 setter 一定走运行时函数，`atomic` 的 getter 一定走运行时函数，`weak` 两头都走自己的专用入口。只有"nonatomic + 非 copy"的对象属性和标量，才真的是内联展开。

我第一版把 `nonatomic copy` 也写成了内联，是错的。clang 在 `PropertyImplStrategy` 里写死了，注释第一句就是答案：

```cpp
// If we have a copy property, we always have to use setProperty.
// If the property is atomic we need to use getProperty, but in
// the nonatomic case we can just use expression.
if (IsCopy) {
  Kind = IsAtomic ? GetSetProperty : SetPropertyAndExpressionGet;
  return;
}
```

编译一次也能看到：

```text
-[C setNCopy:]     objc_setProperty_nonatomic_copy
-[C setNStrong:]   llvm.objc.storeStrong
```

这个更正对后面是加分的——第三节那个经典崩溃的根源，正是 setter 里那次真实的 `copyWithZone:` 调用。既然连 `nonatomic copy` 都进了运行时函数，这个因果就更硬。

几个运行时访问器的签名：

```objc
id   objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic);
void objc_setProperty(id self, SEL _cmd, ptrdiff_t offset, id newValue,
                      BOOL atomic, signed char shouldCopy);
void objc_copyStruct(void *dest, const void *src, ptrdiff_t size,
                     BOOL atomic, BOOL hasStrong);
id   objc_storeWeak(id *location, id newValue);      // 返回 newValue
void objc_storeStrong(id *location, id obj);
```

`shouldCopy` 是 `signed char` 而不是 `BOOL`，因为它要容纳第三个值：

```c
#define MUTABLE_COPY 2
bool copy        = (shouldCopy && shouldCopy != MUTABLE_COPY);
bool mutableCopy = (shouldCopy == MUTABLE_COPY);
```

有意思的是这条分支 clang 永远走不到——Objective-C 里压根没有 `@property (mutableCopy)` 这个关键字。它是 GCC 运行时时代的遗留。这一点后面还会用到：想要一个既独立又可变的属性，语言层面根本没给你这个选项。

`objc_setProperty` 这类没有公开头文件声明，只能从源码看；但 `objc_storeWeak` / `objc_loadWeak` 是例外，它们在 `objc/runtime.h` 里是公开 API。

`objc_setProperty` 内部转到 `reallySetProperty`：

```cpp
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset,
                                     bool atomic, bool copy, bool mutableCopy) {
    id oldValue;
    id *slot = (id*)((char*)self + offset);

    if (copy)             newValue = [newValue copyWithZone:nil];
    else if (mutableCopy) newValue = [newValue mutableCopyWithZone:nil];
    else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }

    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;
        slotlock.unlock();
    }

    objc_release(oldValue);                 // 在锁外释放
}
```

这段代码里有三个地方是特意这么写的。

`copy` 关键字对应的调用是 `copyWithZone:`，不是 `mutableCopyWithZone:`。记住这一行，第三节整节都是它的后果。

锁的粒度。`PropertyLocks` 是一个 `StripedMap`，按 ivar 地址分片。而分片数——

```cpp
#if TARGET_OS_IPHONE && !TARGET_OS_SIMULATOR
    enum { StripeCount = 8 };
#else
    enum { StripeCount = 64 };
#endif
```

真机上整个进程只有 8 把锁。这意味着两件事，都比"同一对象的不同属性各用各的锁"更值得说：同一个对象的两个 atomic 属性有八分之一的概率撞进同一条 stripe，你无法依赖任何一边；反过来，全 App 所有 atomic 对象属性共享这 8 把自旋锁，两个毫不相干的对象可以互相阻塞。

`objc_release(oldValue)` 特意放在解锁之后。释放旧值可能触发 `dealloc`，而 `dealloc` 里可以做任意事情——在锁内执行很容易撞出死锁。同样的手法在 getter 里也有，锁内 retain、锁外 autorelease：

```cpp
id objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic) {
    id *slot = (id*)((char*)self + offset);
    if (!atomic) return *slot;

    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    id value = objc_retain(*slot);
    slotlock.unlock();

    return objc_autoreleaseReturnValue(value);
}
```

这个 getter 说明了 atomic 唯一真正提供的保证：返回值在调用方拿到它之前不会被别的线程释放掉。它在锁的保护下把对象 retain 住了。至于调用方拿到之后要干什么、期间属性有没有被改成别的对象，atomic 一概不管。

#### 五个所有权关键字

**`strong`** 是对象属性的默认值，语义是"我拥有它，它至少要活得和我一样久"。setter 展开成 `objc_storeStrong`，内部顺序是先 retain 新值、再赋值、最后 release 旧值。这个顺序不能颠倒——新旧是同一个对象且引用计数为 1 时，先 release 会直接把它干掉。

**`copy`** 的语义是"我要一份独立的、不会被外部修改的快照"。这是所有权隔离，不是内容复制。下一节整节都在讲这件事。

**`weak`** 表示"我引用它，但不拥有它，也不该延长它的生命周期"。对象销毁时自动置 `nil`。具体实现见 [[iOS weak 的实现：SideTable 与置 nil 的时机]]。

**`assign`** 是给标量准备的：`int`、`BOOL`、`CGFloat`、结构体。一次位拷贝，不碰引用计数。

**`unsafe_unretained`** 修饰对象时和 `assign` 行为完全一致——不 retain、不置 nil。区别在于表达意图：`assign` 用在对象上不会报警告，容易是手滑；`unsafe_unretained` 是 ARC 时代专门造的显式声明，写下它等于在说"我知道这会产生悬垂指针，我有把握"。

我在 code review 里看到 `assign` 修饰对象一律打回，不听解释。想要那个语义就写 `unsafe_unretained`，让下一个读代码的人知道你是故意的。

#### weak 与 unsafe_unretained 的实测差异

```objc
__weak id weakRef = nil;
__unsafe_unretained id unsafeRef = nil;
@autoreleasepool {
    NSObject *o = [NSObject new];
    weakRef = o;
    unsafeRef = o;
}   // o 在这里释放
```

```text
对象存活时  weak=0x600000010000  unsafe=0x600000010000
对象销毁后  weak=0x0             unsafe=0x600000010000
```

`weak` 被置成了 `nil`，`unsafe_unretained` 还指着那块已经归还给分配器的内存。此时访问它是未定义行为——可能什么事都没有（内存还没被复用），可能拿到一个完全不相干的新对象，也可能直接崩。这种"有时候好使"正是它最危险的地方，它不会在开发阶段稳定复现。

那什么时候会故意选 `unsafe_unretained`？读一次 weak 的代价是加锁、查表、`tryRetain`、入池，比读普通指针长得多。在这台机器上量了一下，一千万次属性读取：

```text
第 1 轮  strong=16.8ns  unsafe=18.1ns  weak=46.1ns   weak/strong=2.7x
第 2 轮  strong=11.3ns  unsafe=11.5ns  weak=46.3ns   weak/strong=4.1x
```

三到四倍，不是有些文章说的一个量级。绝对值也不大，46 纳秒一次。所以除非是逐帧执行几千次的热路径，而且你能自证生命周期安全，否则不值得为这点开销放弃自动置 nil。

关于 delegate 为什么用 weak，流行的解释是"防止循环引用"。这个说法不算错，但把因果讲反了。真正的理由是所有权方向：`UITableView` 不应该拥有它的 delegate，因为那通常是一个更上层、生命周期更长的对象。循环引用只是用错 strong 之后的一个后果。按所有权推导，答案是唯一的；按"防循环引用"推导，遇到没有环的场景就不知道该选什么了。

---

### 二、copy 不是深拷贝

#### 不可变对象的 copy 返回的是它自己

这条你自己跑一次比看十篇文章管用，代码就四行。

```objc
NSString *longImm = [NSString stringWithFormat:@"a string long enough to avoid tagging %d", argc];
NSString *c1 = [longImm copy];
printf("源      %p class=%s\n", longImm, object_getClassName(longImm));
printf("copy    %p class=%-16s 同一对象=%d\n", c1, object_getClassName(c1), c1 == longImm);
```

```text
源      0x60000170c180 class=__NSCFString
copy    0x60000170c180 class=__NSCFString     同一对象=1
```

地址一模一样。容器也是：

```text
NSArray      源 0x60000020cb00  copy 0x60000020cb00  同一对象=1
NSDictionary 源 0x60000020cba0  copy 0x60000020cba0  同一对象=1
```

对一个不可变对象来说，"复制一份"没有任何意义——内容永远不会变，多个持有者共享同一块内存完全安全。所以这些类的 `copyWithZone:` 实现直接就是 `return [self retain];`。

这条没写进任何规范，纯粹是 Foundation 自己愿意这么干。哪天它改了你也没处说理去——虽然二十年没改过，实践中可以放心依赖。

#### 四种组合

```text
不可变源  0x88b5a991d0b93da5  class=NSTaggedPointerString
  copy        -> 0x88b5a991d0b93da5  class=NSTaggedPointerString  同一对象=1
  mutableCopy -> 0x600000c00090      class=__NSCFString           同一对象=0
可变源    0x600000c000c0      class=__NSCFString
  copy        -> 0x88b5a991d0b93da5  class=NSTaggedPointerString  同一对象=0
  mutableCopy -> 0x600000c00090      class=__NSCFString           同一对象=0
```

```
[不可变对象 copy]        → 同一个对象（本质是 retain）
[不可变对象 mutableCopy] → 新对象，可变
[可变对象   copy]        → 新对象，不可变
[可变对象   mutableCopy] → 新对象，可变
```

`copy` 的结果永远不可变，`mutableCopy` 的结果永远可变，这两条不看源对象。只有 `[不可变 copy]` 这一格会退化成 retain。

#### 一个意外：短字符串的 copy 撞回了 Tagged Pointer

上面那组输出里，可变源 `copy` 之后的地址 `0x88b5a991d0b93da5`，和不可变源本身完全一样。这两个对象是分别构造的，凭什么撞在一起？

```text
不可变短串        0xa0b62b44cfead36e
可变短串 copy 后  0xa0b62b44cfead36e
两者相同=1
```

因为内容相同，而短字符串的 copy 结果是 `NSTaggedPointerString`——[[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer#值即身份|值就是身份]]，相同的值必然产生相同的位模式。不是"复制到了同一个对象"，是两个值相同的 Tagged Pointer 天然长得一样。

顺便提醒一个读地址的陷阱。上面第一组里 `[不可变源 mutableCopy]` 是 `0x600000c00090`，而后面另一组实验里一个完全无关的对象地址也是 `0x600000c00090`。前一个对象已经释放，分配器把这块内存回收后又发给了下一个请求。跨时间点比地址没有意义，能比的只有同一时刻存活的两个对象。

#### 容器只复制一层

```text
原容器 0x600000004020  元素 0x600000c00090
mutableCopy              容器 0x600000c00180 (变了=1)  元素 0x600000c00090 (变了=0)
initWithArray:copyItems: 容器 0x600000004060 (变了=1)  元素 0x90231e27fe1d3b05 (变了=1)
```

容器地址变了，元素没变——元素还是原来那几个对象，只是被装进了一个新数组。想让元素也被复制，需要 `initWithArray:copyItems:YES`，它会对每个元素发一次 `copyWithZone:`。

但这依然只有一层。元素本身如果还是容器，孙子那一层照样共享。真正的完全深拷贝在 Foundation 里没有通用方案，实务上只能靠归档解档走一圈。

"`mutableCopy` 就是深拷贝"是中文技术文章里被简化得最厉害的一句，实际上它连元素都没碰。

#### copy 修饰可变类型：一个必然发生的崩溃

回到前面埋的那一行：`copy` 生成的调用是 `copyWithZone:`。

所以无论你把属性类型声明成什么，`copy` 修饰的 setter 存进去的一定是不可变对象。类型声明只骗过了编译器。

```objc
@property (nonatomic, copy)   NSMutableArray *badArr;   // 故意写错
@property (nonatomic, strong) NSMutableArray *goodArr;
```

```text
声明类型都是 NSMutableArray *
  copy   修饰 -> 实际类 = __NSSingleObjectArrayI
  strong 修饰 -> 实际类 = __NSArrayM
  对 copy 修饰的调用 addObject: -> 抛异常:
      -[__NSSingleObjectArrayI addObject:]: unrecognized selector sent to instance 0x600000004080
```

`__NSArrayM` 是可变数组的实现类，`__NSSingleObjectArrayI` 是不可变的（末尾 `I` = Immutable，`M` = Mutable）。

这个 bug 恶劣在它不在赋值那一行崩，而在后面某个使用点崩，堆栈里看到的是一个类型对不上的对象，很容易怀疑到别的地方去。`copy` 只能用于不可变类型的属性声明。想要一个属性既独立又可变，得自己在 setter 里 `mutableCopy`——前面说过，语言层面没给你这个关键字。

---

### 三、atomic：一个能证伪自己的实验

#### 先看数据

一个 `atomic` 属性，十万次并发自增：

```objc
@property (atomic) NSInteger atomicCount;
...
dispatch_apply(100000, dispatch_get_global_queue(0, 0), ^(size_t i) {
    c.atomicCount = c.atomicCount + 1;
});
```

跑六次：

```text
run 1: atomic=53660  nonatomic=45865
run 2: atomic=36198  nonatomic=63288
run 3: atomic=67907  nonatomic=56442
run 4: atomic=40941  nonatomic=49217
run 5: atomic=65754  nonatomic=78538
run 6: atomic=40634  nonatomic=51938
期望都是 100000。把读和写整体加锁的对照组：六次全部精确 100000。
```

`atomic` 丢了一半左右的计数。

我第一版只跑了两次，两次都是 atomic 丢得更多，于是写下了"atomic 甚至比 nonatomic 丢得更多"，还为它编了个机制——说 atomic 每次访问要抢自旋锁、访问变慢、线程重叠窗口变大。

两句话都是错的。跑满六次之后，两边互有胜负，纯噪声。

#### 更要命的是，这两行代码根本没有区别

`@property (atomic) NSInteger` 是标量属性，走的是标量路径，压根不碰 `PropertyLocks`。两个 setter 的 arm64 汇编：

```asm
"-[C setNInt:]":          ; nonatomic NSInteger
	str	x2, [x0, #8]
	ret
"-[C setAInt:]":          ; atomic NSInteger
	str	x2, [x0, #16]
	ret
```

逐字节相同，偏移不同只因为是两个不同的 ivar。atomic 在这里连一条额外指令都没生成，更别说锁——`AtomicOrdering::Unordered` 在 arm64 上就退化成普通的 `str`。

所以这个实验真正的结论是：**对标量属性，`atomic` 和 `nonatomic` 在 arm64 上是同一份代码。** "atomic 会加锁"只对**对象属性**成立，那把锁在 `objc_setProperty` 里，不在标量路径上。

一篇通篇标榜"从机制推导、不要背结论"的文章，在自己最想强调的地方犯了"先有结论再找机制"的毛病。留着这段自我打脸，比删掉有用。

#### 那 atomic 到底保证什么

`self.atomicCount = self.atomicCount + 1` 看起来是一条语句，实际是三步：调 getter 读出旧值、加一、调 setter 写回。atomic 保证的是 getter 内部和 setter 内部各自不被打断，但这两次调用之间完全敞开。两个线程同时读到 41，各自算出 42，先后写回，就丢了一次。

Apple 官方文档在 "Properties Are Atomic by Default" 那一节的注释框里说得很克制：

> Property atomicity is not synonymous with an object's thread safety.

接下来那段举的例子是一个人的 first name 和 last name 分别是两个 atomic 属性——一个线程改名字的同时另一个线程读，完全可能读到"张 + Smith"这种拼接。这正好说明 atomic 的锁按 ivar 地址走，跟对象没关系。

还有个更常被忽略的失效场景：atomic 只保护指针，不保护指针指向的东西。

```objc
@property (atomic) NSMutableArray *arr;
// 线程 A: [self.arr addObject:@1];
// 线程 B: [self.arr addObject:@2];
```

`self.arr` 这次读取确实是原子的，两个线程都安全地拿到了同一个数组。然后它们各自对这个数组调 `addObject:`，而 `NSMutableArray` 自己没有任何同步。数组内部结构被并发写坏，轻则数据错乱，重则 `EXC_BAD_ACCESS`。atomic 在这里连边都没沾上。

我写了这些年 iOS，没有一次是靠 atomic 解决掉线程问题的。属性一律 `nonatomic`，线程安全往上一层放——加锁、串行队列、或者干脆用不可变数据绕开。顺带还省下了对象属性上那次抢锁的开销。

`nonatomic` 是不是就完全没风险？严格说也不是。指针本身的对齐写入在 arm64 上是原子的，但 ARC 插入的 retain/release 和指针写入不是一个整体，并发场景下理论上存在过度释放的窗口。这个窗口很窄，实践中很少直接触发。真要跨线程读写同一个属性，答案不是在这两个关键字之间挑，而是加同步。

---

### 四、Block 属性：copy 还是 strong

这场争论其实早有答案，只是它写在 clang 的 AST 头文件里而不是文档里：

```cpp
SetterKind getSetterKind() const {
    if (PropertyAttributes & ObjCPropertyAttribute::kind_strong)
      return getType()->isBlockPointerType() ? Copy : Retain;
    ...
}
```

`strong` 碰上 block 指针类型，编译器直接当 `copy` 处理。两个属性发射同一个 `objc_setProperty_nonatomic_copy`，不是"实测没差异"，是语言规则规定它们相同。

但别把结论推广到 `retain`。它走另一个分支，真的只 retain 不 copy，clang 会专门警告：

```text
warning: retain'ed block property does not copy the block
         - use copy attribute instead [-Wobjc-noncopy-retain-block-property]
```

所以 block 属性上 `copy` 和 `strong` 随便挑，`retain` 是错的。这才是这场"两派之争"里唯一真正有对错的地方。

我自己还是写 `copy`。理由不是怕出 bug（现在证明了不会），而是可读性——`copy` 明确告诉读代码的人这里发生了一次从栈到堆的搬迁。Block 在栈和堆之间搬迁的机制本身，放在 [[iOS Block 的结构：ABI、descriptor 与三种类型]] 里讲。

---

### 五、剩下那些不常被问到的

**`readonly`** 拦得住编译器，拦不住 KVC。`setValue:forKey:` 照样能改到底层 ivar。它表达的是接口契约。常见搭配是头文件里声明 `readonly`，在 `.m` 的 class extension 里重新声明成 `readwrite`——class extension 里重声明的属性会正常合成，做到对外只读、对内可写。

**`getter=`** 最常见的用途是让 `BOOL` 属性符合 `isXxx` 的命名习惯：`@property (getter=isFinished) BOOL finished;`。

**`class`** 声明的是类属性，不生成实例变量，也不参与自动合成，实现里通常自己用 `static` 变量存。对它写 `@synthesize` 会直接编译报错。

**`null_resettable`** 是一个组合语义：setter 接受 `nil`，getter 保证不返回 `nil`。系统里最典型的是 `UIView.tintColor`：

```objc
@property(null_resettable, nonatomic, strong) UIColor *tintColor;
```

不过它的行为比"设 nil 就回到系统默认色"更复杂一点。同一个头文件里的注释写着：`-tintColor` 返回的是**接收者 superview 链上第一个非默认值**（从自己开始往上找）。设成 `nil` 是把自己从这条链上摘出去，继承上层的值，找不到才落到系统默认。

**protocol 和 category 里的属性** 是个容易误解的地方。在 protocol 里写 `@property` 只声明了一对方法签名，不生成任何存储，遵循协议的类得自己实现，clang 会给一条 `-Wobjc-protocol-property-synthesis` 警告。

category 里写 `@property` 同样不生成 ivar，常规做法是用关联对象模拟存储或者标 `@dynamic`。但这里有个坑：clang 只在存在对应的 `@implementation Cat (A)` 时才警告；如果 category 只有接口声明没有实现块，编译器一声不吭，`-Wall` 也不响，运行时直接 `unrecognized selector`。所以你会看到一个编译得干干净净、跑起来必崩的 category。

---

### 六、怎么选

把决策顺序写成一串问题，比记表管用：

1. 是标量还是对象？标量、结构体一律 `assign`，到此结束。
2. 对象的话，我拥有它吗？不拥有（delegate、parent、observer 这类反向引用）→ `weak`。拥有 → 进第 3 步。
3. 调用方之后修改这个对象，会不会影响到我？会（典型是把一个 `NSMutableString` 或 `NSMutableArray` 存进 model）→ 需要独立副本 → `copy`，并且属性类型必须声明成不可变类型。不会 → `strong`。
4. 目标类不支持 weak，或者这是逐帧访问的热路径且我能自证生命周期？→ `unsafe_unretained`，并且写注释说明为什么。
5. 原子性怎么选？`nonatomic`。真正需要线程安全时加锁或用串行队列。
6. 对外要不要可写？不要 → 头文件 `readonly` + extension 里 `readwrite`。

整个过程里没有一步是从类型名推的。`NSString` 用 `copy` 不是因为它叫 NSString，而是因为传进来的可能是个 `NSMutableString`。如果能确定不会——比如这个值来自一个你完全掌控的、只产生不可变字符串的内部模块——`strong` 也是对的，还省一次可能的拷贝。

而且有比 `NSString` 更阴的。`[str dataUsingEncoding:NSUTF8StringEncoding]` 的返回类型写着 `NSData *`，但它在运行时是 `NSConcreteMutableData`——`isKindOfClass:[NSMutableData class]` 返回 YES，`appendBytes:length:` 照调不误。所以：

```objc
@property (nonatomic, strong) NSData *payload;   // 看着无懈可击
```

这行代码一样能被调用方在你背后改掉。类型名是 `NSData`，可变性是 `NSMutableData`。从类型名推，你永远推不出这里该用 `copy`。

这也是我对"背口诀"这件事最大的意见。口诀能让你答对面试题，但它给的是一张查找表；真实代码里遇到的类型不在表上，你就没有办法了。三栏推导给的是一个可以重复使用的方法。

---

### 总结

三栏正交——所有权、原子性、暴露方式，各选各的。看到一个新属性，按这三个问题过一遍，不要回忆表格。

`copy` 属性的 setter 真的会调 `copyWithZone:`，连 `nonatomic copy` 也走运行时函数。所以它只能修饰不可变类型，第三节那个 `unrecognized selector` 就是这一行的直接后果。

`copy` 不是深拷贝，`mutableCopy` 也不是。它们都只处理最外面那一层，不可变对象的 `copy` 甚至连新对象都不造。

`atomic` 保护的是单次访问不撕裂，仅此而已。对标量属性它连一条额外指令都不生成，`atomic` 和 `nonatomic` 编译出来是同一份机器码。需要线程安全就往上一层加锁，别在这两个关键字之间挑。

### 参考资料

#### 官方与源码

- [Apple — Encapsulating Data](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/EncapsulatingData/EncapsulatingData.html)：atomicity 与 thread safety 的官方措辞出自 "Properties Are Atomic by Default" 一节的注释框
- [Apple — NSCopying](https://developer.apple.com/documentation/foundation/nscopying)
- [objc4 — objc-accessors.mm](https://github.com/apple-oss-distributions/objc4)：`objc_getProperty` / `reallySetProperty` 的完整实现
- [objc4 — objc-private.h](https://github.com/apple-oss-distributions/objc4)：`StripedMap` 的分片数
- [clang — CGObjC.cpp](https://github.com/llvm/llvm-project)：`PropertyImplStrategy` 决定了本文第一节那张分路表
- [clang — DeclObjC.h](https://github.com/llvm/llvm-project)：`getSetterKind`，block 属性上 `strong` 等价于 `copy` 的依据

#### 经典

- [objc.io — Value Objects](https://www.objc.io/issues/7-foundation/value-objects/)：解释了为什么不可变类的 `copyWithZone:` 应该 retain 而不是真拷贝

#### 本地

- [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]]
- [[iOS weak 的实现：SideTable 与置 nil 的时机]]
- [[iOS Block 的结构：ABI、descriptor 与三种类型]]

---

实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -fobjc-arc -target arm64-apple-ios17.0-simulator`，`xcrun simctl spawn booted` 运行。

第一节那张分路表不是从文档抄的，是编译出来的：

```shell
clang -fobjc-arc -target arm64-apple-ios17.0-simulator -S -emit-llvm T.m -o T.ll
```

然后在 `T.ll` 里 grep 每个 setter 调了什么。想看汇编把 `-emit-llvm` 换成 `-O2`。这比下符号断点快，也不会漏掉被内联的情况。

两点偏差需要注意。并发实验的具体数字每次都不同，能引用的只有"大量丢失"和"加锁后精确"这两个结论。Foundation 的内部类名（`__NSSingleObjectArrayI`、`__NSCFString` 等）属于实现细节，不同系统版本可能变化。

还有一处环境差异会影响结论外推：`PropertyLocks` 的分片数在模拟器上是 64，真机上只有 8。本文关于锁争用的讨论在真机上会更严重，模拟器的数据不能直接搬过去。
