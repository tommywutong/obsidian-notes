---
title: 【iOS】weak 的实现：SideTable、weak_table_t 与置 nil 的时机
published: 2026-07-26
description: 「weak 要等 dealloc 跑完才置 nil，中间有一段野指针窗口」——这个说法很流行，但一个实验就能推翻。顺便理清 SideTables 到 weak_entry_t 那四层里，每一层的 key 和 value 到底是什么。
tags:
  - iOS
  - Objective-C
  - Memory
  - Runtime
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 7
draft: true
---
# weak 的实现：SideTable、weak_table_t 与置 nil 的时机

`__weak` 的语义只有一句话：不持有对象，对象销毁后自动变成 `nil`。

前半句靠编译器——它不插 retain 就行了。后半句是个真问题：对象自己都要没了，凭什么还能回头把散落在各处的指针一个个改掉？它得先知道有哪些指针指着自己。

所以 weak 的实现本质上是一张反向索引表。这篇文章把这张表的结构拆开，然后回答一个被广泛讲错的问题——那些指针到底是在哪一刻变成 `nil` 的。

上一篇 [[iOS 内存：ARC 的两半]] 里说"ARC 不追踪引用关系"，唯一的例外就是这里。

---

## 一、四层结构，每层的 key 和 value 都不一样

中文资料在这一节最容易出错，因为四层套下来，很容易把某一层的 key 说成另一层的。先把全貌摆出来：

```text
SideTables()                     全局入口，一个函数
  └── StripedMap<SideTable>      按对象地址哈希分片
        key   : ((addr >> 4) ^ (addr >> 9)) % StripeCount
        value : SideTable
        分片数：真机 8，Mac 与模拟器 64
        │
        └── SideTable             每片一个，各自独立加锁
              ├── spinlock_t slock
              ├── RefcountMap refcnts      ← 溢出的引用计数住这儿
              │     key   : 被引用的对象
              │     value : 计数 + 标志位（DEALLOCATING / WEAKLY_REFERENCED 等）
              └── weak_table_t weak_table  ← 弱引用住这儿
                    key   : 被弱引用的对象（referent）
                    value : weak_entry_t
                    │
                    └── weak_entry_t
                          ├── ≤4 个引用者：inline_referrers[4]，就是个数组
                          └── >4 个引用者：referrers，独立的开放寻址哈希表
                                key   : __weak 变量自己的地址
                                value : 同上（去重存储，无 payload）
```

**`weak_table_t` 的 key 是被弱引用的对象，不是 weak 指针。** 这是最常被说反的一条。查表方向是"给我一个对象，告诉我有哪些 weak 变量指着它"——因为对象销毁时需要的正是这个方向。`__weak` 变量的地址是第二层（`weak_entry_t` 内部）的东西。

源码里 `weak_entry_for_referent(weak_table_t *, objc_object *referent)` 这个函数签名已经说明一切了。

结构体定义（`objc-weak.h`）：

```c
#define WEAK_INLINE_COUNT 4
#define REFERRERS_OUT_OF_LINE 2

struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };
};
```

一个 union：引用者不超过 4 个时直接放在内联数组里，省掉一次分配和一层间接；第 5 个进来才 `calloc` 一张真正的哈希表，把内联的四个搬过去。这个"小规模内联、大规模外置"的模式在 runtime 里到处都是。

另一个细节：`DisguisedPtr<T>` 会把指针做一次位运算伪装。目的不是安全，是让 `leaks` 这类内存分析工具扫描时不把 weak 表当成强引用的根——否则每个被弱引用的对象看起来都是可达的。

### SideTable 不等于 weak 表

`SideTable` 里装着三样东西：一把锁、一张引用计数溢出表、一张弱引用表。`weak_table` 只是其中一个成员。

把两者划等号是个很常见的误解，而且会引出错误的推论——比如"给对象加 weak 会导致引用计数走 SideTable"。实际上这两张表互不相干，只是碰巧共用同一把锁、住在同一个分片里。上一篇讲的 `extra_rc` 溢出走的是 `refcnts`，跟 weak 一点关系没有。

分片的意义也值得说一句。如果全局只有一把锁，任意两个毫不相干的对象做 retain/release 都会互相阻塞。按地址哈希分成 8 或 64 片之后，锁竞争被限制在"地址恰好落进同一片"的对象之间。真机分 8 片、Mac 和模拟器分 64 片，源码里没注释原因，我也不猜。

---

## 二、写入、读取、清除

### 注册

`__weak id p = obj;` 编译成 `objc_storeWeak`，核心流程：

```
objc_storeWeak(location, newObj)
  └── storeWeak<...>(location, newObj)
        1. 取旧值，算出新旧两个对象各自所在的 SideTable
        2. lockTwo(oldTable, newTable)          ← 同时锁两张表
        3. 若 *location 已被别的线程改过，解锁重试
        4. 若 newObj 的类还没跑 +initialize，先解锁触发，再重试
        5. weak_unregister_no_lock(旧表, oldObj, location)   摘掉旧登记
        6. weak_register_no_lock(新表, newObj, location)     登记新的
        7. newObj->setWeaklyReferenced_nolock()               在 isa 上打标记
        8. *location = newObj                                 唯一一次赋值
        9. unlockTwo
```

第 2 步同时锁两张表这件事有个经典的死锁风险：线程 A 把 `x` 从对象 1 改指向对象 2，线程 B 把 `y` 从对象 2 改指向对象 1，如果都"先锁旧再锁新"，就会各持一把互相等。

objc4 的解法是**按锁的内存地址排序**，地址小的先锁。这样无论调用顺序如何，所有线程获取两把锁的顺序都一致，环形等待不可能形成。这是解决双资源死锁的教科书手法，不是 weak 专属技巧，但在这儿用得很典型。

第 8 步也值得留意：整个流程里对 `*location` 只写一次，而且是在两张表都锁住、登记都完成之后。避免了"表里登记了但变量还没更新"这类中间状态。

### 读取

读一个 `__weak` 变量不是简单的取值。编译器插入的是 `objc_loadWeakRetained`，然后 autorelease：

```
objc_loadWeakRetained(location)
  1. 取出裸指针；是 tagged pointer 或 nil 就直接返回，不加锁不查表
  2. 锁住对象所在的 SideTable
  3. 若 *location 被并发改过，解锁重试
  4. rootTryRetain()  ← 关键：内部检查 DEALLOCATING 标志
       已置位 → 失败，结果为 nil
       未置位 → 引用计数 +1，拿到一个保证存活的强引用
  5. 解锁，返回
```

为什么非要 retain 一下？因为从"读出指针"到"使用它"之间，对象随时可能在另一个线程被释放。Clang ARC 规范对此有明确要求：

> For `__weak` objects, the current pointee is retained and then released at the end of the current full-expression... This must execute atomically with respect to assignments and to the final release of the pointee.

所以每次读 weak 变量，实际发生的是加锁、查表校验、尝试 retain、解锁、入池。这条路径明显比读一个普通指针长。

这带来一个实际建议：循环里反复读同一个 weak 变量，应该先用一个局部强引用接住。这不是玄学，是能从源码路径上说清楚的。至于慢多少倍——我没有找到可靠的公开 benchmark，也不打算编一个数字，只说开销来源。

### 清除

对象销毁时：

```
引用计数归零 → DEALLOCATING 置位     ← 注意：这一步早于 -dealloc 方法体执行
      ↓
objc_msgSend(obj, @selector(dealloc))    ← 你写的 dealloc 在这里跑
      ↓
… 一路 super 到 NSObject 根实现 …
      ↓
object_dispose → objc_destructInstance → clearDeallocating
      ↓
weak_clear_no_lock(&table.weak_table, this)
      遍历 entry 的 inline_referrers[4] 或外置哈希表：
        for each referrer:
            if (*referrer == referent) *referrer = nil;
      然后把整条 weak_entry_t 从表里摘掉
```

遍历时那个 `if (*referrer == referent)` 是防御性检查。如果对不上说明内存被破坏或者有人误用了，会报 `objc_weak_error`。

---

## 三、置 nil 的时机：一个实验推翻的流行说法

上面的流程图里有个容易被误读的地方。DEALLOCATING 置位发生在**进入 `dealloc` 之前**，而真正把 `*referrer` 写成 `nil` 要等 `dealloc` 一路走完、进到 `object_dispose` 才发生。中间隔着你自己写的 `dealloc` 方法体。

于是流传出这样一个说法：在 `dealloc` 执行期间，weak 指针还没被清空，所以存在一段"野指针窗口"。

这个说法不对，而且一个实验就能测。让 `dealloc` 里同时打印三个东西——之前注册好的 weak、一个 `unsafe_unretained`、以及 `self` 本身：

```objc
static __weak id gWeak = nil;
static __unsafe_unretained id gUnsafe = nil;

@implementation Probe
- (void)dealloc {
    NSLog(@"  [dealloc 中] 读之前注册的 weak     = %@", gWeak);
    NSLog(@"  [dealloc 中] 读 unsafe_unretained = %p", gUnsafe);
    NSLog(@"  [dealloc 中] self 本身             = %p", self);
}
@end
```

输出：

```text
  [存活时] weak = <Probe: 0x600000010020>
  [dealloc 中] 读之前注册的 weak     = (null)
  [dealloc 中] 读 unsafe_unretained = 0x600000010020
  [dealloc 中] self 本身             = 0x600000010020
  [dealloc 后] weak = (null)
```

`self` 是有效指针，`unsafe_unretained` 也还指着它，唯独 weak 已经读出 `nil` 了。

原因就在 `objc_loadWeakRetained` 的第 4 步：它每次都要过一遍 `rootTryRetain()`，而 DEALLOCATING 一旦置位，`tryRetain` 必然失败，返回 `nil`。也就是说——

**指针清零的时机确实晚于标志置位，但读取的安全性从标志置位那一刻就已经生效了。** 内存里那个 `*referrer` 字段在 `dealloc` 期间确实还是旧值，但没有任何受支持的代码路径能看到它：读 weak 变量必然走 `objc_loadWeak` 系列，而那条路径上有 `tryRetain` 把关。

顺带解释了一个很多人撞过的困惑：`dealloc` 里的 `weakSelf` 一定是 `nil`。不是 bug，是这个机制的必然结果。想在 `dealloc` 里用自己，直接用 `self`。

### 在 dealloc 里注册新的 weak 会直接崩

既然对象已经在析构，再登记一条弱引用就没有意义了——没人会再来清它。runtime 的处理是直接终止进程：

```objc
- (void)dealloc {
    NSLog(@"进入 dealloc，self = %p", self);
    __weak typeof(self) weakSelf = self;      // ← 这里
    NSLog(@"%@", weakSelf);
}
```

```text
进入 dealloc，self = 0x600000008040
objc[73993]: Cannot form weak reference to instance (0x600000008040) of class Boom.
It is possible that this object was over-released, or is in the process of deallocation.
Child process terminated with signal 6: Abort trap
```

这条崩溃信息在线上很常见，而且提示里给了两种可能：对象被过度释放，或者正在析构。后半句才是这里的情况。实际排查时，如果堆栈里有 `dealloc`，多半是某个清理逻辑里不小心用了 `weakSelf` 或者把 `self` 传给了一个会内部持弱引用的 API。

---

## 四、Tagged Pointer 完全绕开这套机制

`weak_register_no_lock` 一开始就短路：

```c
if (_objc_isTaggedPointerOrNil(referent)) return referent_id;
```

`objc_loadWeakRetained` 同理，取出的值是 tagged pointer 就直接返回，不加锁不查表。

所以对 tagged pointer 声明 `__weak` 不会崩溃，也不会进弱引用表，更不会被置 `nil`——因为它压根没有"销毁"这个事件。[[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer#四个会漏出来的地方|上一篇实测过]]：weak 指向一个 tagged 的 `@42`，"释放"之后读出来还是 42。

准确的表述不是"tagged pointer 不能用 weak"，而是**它的 weak 退化成了一次普通赋值**。不计数、不清零，因为它本来就不会消失。这在小整数上通常无害，但如果你的逻辑依赖"对象没了 weak 就变 nil"，在这里会静默失效。

---

## 五、几个说法需要辨析

**"weak 表的 key 是 weak 指针。"** 反了。`weak_table_t` 的 key 是被弱引用的对象，value 是指向它的所有 weak 变量地址的集合。weak 变量的地址是 `weak_entry_t` 内部那一层的 key。

**"SideTable 就是 weak 表。"** `SideTable` 是个复合结构，包含一把锁、引用计数溢出表 `refcnts`、弱引用表 `weak_table` 三部分。

**"weak 在 dealloc 结束后才置 nil，中间是野指针。"** 见第三节。清零确实晚，但读取安全性从 DEALLOCATING 置位就生效了，不存在能观察到的野指针窗口。

**"weak 和 unsafe_unretained 只差会不会自动置 nil。"** 差别更靠前：`__weak` 会调 `objc_storeWeak` 把变量地址登记进表，`__unsafe_unretained` 只是一次裸赋值，运行时对它一无所知。所以后者不是"没被清零"，是**运行时根本不知道它存在**。

**"weak 只有 ARC 才有。"** zeroing weak reference 这套机制最早是随 GC 引入的（`objc-weak.h` 的注释开头还留着痕迹），ARC 复用了它。MRC 下也能直接调 `objc_storeWeak` / `objc_loadWeak`，只是没有 `__weak` 这个语法糖。

**"weak 属性和普通属性一样快。"** 不成立，见第二节的读取路径。但具体慢多少，我没有可引用的数据，只能定性说。

---

## 六、Swift 走的是另一条路

Objective-C 用外部哈希表记录"谁指向我"，Swift 用的是双计数器：对象头里同时有强引用计数和弱引用计数。强计数归零时执行 `deinit` 并释放存储，但对象本身不立即回收，变成一个"僵尸"占位，直到弱计数也归零才真正释放。

两种方案的取舍很清楚。Objective-C 方案的常态开销是零——没有弱引用的对象完全不碰 weak 表；代价是有弱引用时要查哈希表、加锁。Swift 方案每个对象都多一个计数器字段，但弱引用的读写只是原子操作，没有哈希查找；代价是被弱引用过的对象会多占一段时间的内存。

顺带一提，C++ 的 `weak_ptr` 是第三种：控制块里放两个计数，`shared_ptr` 和 `weak_ptr` 共享。跟 Swift 思路接近。

---

## 总结

weak 的实现是一张反向索引：`SideTables` 按地址分片，每片的 `weak_table_t` 以被引用对象为 key，value 是指向它的所有 weak 变量地址。四层里每层 key 的含义都不同，"weak 表的 key 是 weak 指针"这个说法把最外面那层说反了。

`SideTable` 不只管 weak，它同时装着引用计数溢出表，两张表共用一把锁但互不相干。

置 nil 的时机分两步：DEALLOCATING 标志在进入 `dealloc` 之前就置位了，而真正写 `nil` 要等 `object_dispose`。这中间隔着你写的 `dealloc` 方法体，但读取安全性从置位那一刻就生效——实测在 `dealloc` 里 `self` 有效、`unsafe_unretained` 有效，唯独 weak 已经是 `nil`。所以"野指针窗口"是不存在的，而 `dealloc` 里的 `weakSelf` 必然为 `nil` 也就有了解释。

在正在析构的对象上注册新的弱引用会直接 abort，错误信息是 `Cannot form weak reference to instance`。线上遇到这条，堆栈里有 `dealloc` 的话基本可以锁定原因。

读 weak 变量的代价是加锁、查表、`tryRetain`、入池，比读普通指针长得多。热路径里应该先用局部强引用接住——这条建议有源码依据，不是经验之谈。

下一篇转向 Block：[[iOS Block 的结构：ABI、descriptor 与三种类型]]。

## 参考资料

### 源码（这一篇几乎全部结论都来自这里）

- [objc4 — objc-weak.h / objc-weak.mm](https://github.com/apple-oss-distributions/objc4)：`weak_table_t`、`weak_entry_t` 的定义，以及 `weak_register_no_lock` / `weak_clear_no_lock`
- [objc4 — NSObject.mm](https://github.com/apple-oss-distributions/objc4)：`objc_storeWeak`、`objc_loadWeakRetained`、`SideTable`
- [objc4 — NSObject-private.h](https://github.com/apple-oss-distributions/objc4)：`SideTable` 的结构体本体。很多文章漏掉这个文件，于是把 SideTable 和 weak_table 混为一谈
- [objc4 — objc-private.h](https://github.com/apple-oss-distributions/objc4)：`StripedMap` 的分片数与哈希算法
- [Clang ARC Specification](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)：`__weak` 读写的原子性要求

### 文章

- [Mike Ash — Zeroing Weak References in Objective-C](https://www.mikeash.com/pyblog/friday-qa-2010-07-16-zeroing-weak-references-in-objective-c.html)：写于官方机制落地之前，作者自己造了一个轮子。注意这是历史背景，数据结构和后来的 objc4 不是一回事
- [Mike Ash — Swift 4 Weak References](https://mikeash.com/pyblog/friday-qa-2017-09-22-swift-4-weak-references.html)：第六节那套双计数器方案的来源
- [Surprising Weak-Ref Implementations](https://verdagon.dev/blog/surprising-weak-refs)：横向对比 ObjC、Swift、C++、Rust 几种弱引用实现

### 本地

- [[iOS 内存：ARC 的两半]]
- [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]]
- [[iOS 属性关键字：从所有权推导，而不是从类型名猜]]

---

实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -fobjc-arc -target arm64-apple-ios17.0-simulator`。第三节两组输出都是真实运行结果。

分片数、`WEAK_INLINE_COUNT` 这些常量属于实现细节，可能随版本变化；但"读取安全性早于指针清零"这条是由 `tryRetain` 检查 DEALLOCATING 这个设计保证的，不依赖具体常量。

> 待补：`weak_entry_t` 从内联数组升级到外置哈希表这一步没有公开 API 可以直接观察，只能靠给同一个对象建 5 个以上 weak 引用、再用 Instruments 的 Allocations 看 `calloc` 是否被触发来间接佐证。这个实验我没做，留着。
