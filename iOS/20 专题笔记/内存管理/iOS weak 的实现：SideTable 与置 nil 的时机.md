---
title: 【iOS】weak 的实现：SideTable、weak_table_t 与置 nil 的时机
published: 2026-07-26
description: 「weak 要等 dealloc 跑完才置 nil，中间有一段野指针窗口」——这个说法很流行，一个实验就能推翻，而且 objc4 的源码注释里 Apple 亲口说了为什么这么设计。
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


## 一、四层结构，每层的 key 都不一样

`SideTable` 中存储了对象的引用计数以及所关联的弱引用指针，它是 `SideTables()` 这样一个全局哈希表的 `value`，其数据结构如下图所示：

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260808171652362.png)


```text
SideTables()                     全局入口
  └── StripedMap<SideTable>      按对象地址分片
        key   : 对象地址
        映射  : indexForPointer = ((addr>>4) ^ (addr>>9)) % StripeCount
        │
        └── SideTable
              └── weak_table_t
                    key   : 被弱引用的对象（referent）
                    value : weak_entry_t
                    │
                    └── weak_entry_t
                          ≤4 个引用者：inline_referrers[4]，就是个数组
                          >4 个引用者：referrers，独立的开放寻址哈希表
                                key : __weak 变量自己的地址
```

`weak_table_t` 的 key 是被弱引用的对象，不是 weak 指针。




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

一个 union：引用者不超过 4 个时直接放在内联数组里，省掉一次分配和一层间接。这个"小规模内联、大规模外置"的模式在 runtime 里反复出现——`class_rw_t` 的方法列表在单个 list 和 list-of-lists 之间切换是同一个思路，关联对象的存储也是。

升级过程其实是两步，不是一步。第 5 个引用者到来时，`append_referrer` 先 `calloc` 一张只有 4 槽的表把内联的搬过去，紧接着就撞上 `num_refs >= TABLE_SIZE * 3/4`（4 ≥ 3）触发扩容，扩到 8 并重哈希。源码里对那张中间表的注释是 "This constructed table is invalid, but grow_refs_and_insert will fix it"。所以第一张真正可用的外置表是 8 槽的。

各层的伸缩阈值都是写死的常量：

| 层 | 触发条件 | 动作 |
| --- | --- | --- |
| `weak_entry_t` 内联→外置 | 4 个内联槽用满，第 5 个到来 | calloc 4 槽，随即扩到 8 |
| `weak_entry_t` 外置扩容 | `num_refs >= TABLE_SIZE * 3/4` | 翻倍，全量重哈希 |
| `weak_table_t` 扩容 | `num_entries >= old_size * 3/4` | `old_size ? old_size*2 : 64` |
| `weak_table_t` 收缩 | `old_size >= 1024 && old_size/16 >= num_entries` | 缩到 `old_size/8` |

收缩条件苛刻得很显眼：表至少 1024 项、且实际占用不到十六分之一才缩，缩完也只到八分之一。这是明确的取舍——宁可浪费内存也不频繁 rehash，只有大表才值得回收。

至于 `DisguisedPtr<T>`，它做的是取负而不是位运算：

```c
static void *disguise(T* ptr) { return (void *)-(uintptr_t)ptr; }
```

目的源码里写得很直白，两个文件各有一句：`// These pointers are stored disguised so memory analysis tools don't see lots of interior pointers from the weak table into objects.`，以及 `// we don't want the table to act as a root for leaks.`

顺带纠正一个我自己差点写错的：外置哈希表**不做**插入去重。`append_referrer` 的注释是 "Does not perform duplicate checking (b/c weak pointers are never added to a set twice)"——它依赖 `objc_storeWeak` 的调用契约保证不重复，而不是存储时检查。

### SideTable 不等于 weak 表

`SideTable` 里装着三样东西：

```text
SideTable
  ├── spinlock_t slock            一把锁
  ├── RefcountMap refcnts         引用计数溢出表
  └── weak_table_t weak_table     弱引用表
```

`weak_table` 只是其中一个成员。把两者划等号会引出错误推论，比如"给对象加 weak 会导致引用计数走 SideTable"。这两张表互不相干，只是碰巧共用同一把锁、住在同一个分片里。上一篇讲的 `extra_rc` 溢出走的是 `refcnts`。

分片数在源码里是这样的：

```cpp
#if TARGET_OS_IPHONE && !TARGET_OS_SIMULATOR
    enum { StripeCount = 8 };
#else
    enum { StripeCount = 64 };
#endif
    struct PaddedT { T value alignas(CacheLineSize); };
```

iOS / watchOS / tvOS 真机 8 片，Mac 和所有模拟器 64 片。源码没写为什么，但那行 `alignas(CacheLineSize)` 给了线索：每片按 64 字节缓存行对齐是为了避免伪共享，代价是 64 片光对齐填充就有 4KB 常驻脏内存，还要加上 64 把锁和 64 个 DenseMap 的初始开销。手机上进程多、内存紧、核心少；Mac 核心多、并发压力大、内存宽裕。这是我的推断，源码里没写，但这个取舍链条应该站得住。

---

## 二、写入、读取、清除

### 注册

`__weak id p = obj;` 会走 `objc_initWeak`，之后的赋值走 `objc_storeWeak`，核心流程一样：

```
storeWeak(location, newObj)
  1. 取旧值，算出新旧两个对象各自所在的 SideTable
  2. lockTwo(oldTable, newTable)          ← 同时锁两张表
  3. 若 *location 已被别的线程改过，解锁重试
  4. 若 newObj 的类还没跑 +initialize，先解锁触发，再重试
  5. weak_unregister_no_lock(旧表, oldObj, location)
  6. weak_register_no_lock(新表, newObj, location)
  7. newObj 打上"被弱引用过"的标记
  8. *location = newObj                    ← 整个流程里唯一一次赋值
  9. unlockTwo
 10. callSetWeaklyReferenced(newObj)       ← 在锁外
```

第 2 步同时锁两张表有个经典的死锁风险：线程 A 把 `x` 从对象 1 改指向对象 2，线程 B 把 `y` 从对象 2 改指向对象 1，如果都"先锁旧再锁新"，就会各持一把互相等。

objc4 的解法是按锁的内存地址排序，地址小的先锁：

```cpp
// Address-ordered lock discipline for a pair of locks.
void lockWith(T& other) {
    if (this < &other) {
        T::lock();
        other.lock();
    } else {
        other.lock();
        if (this != &other) T::lock();
    }
}
```

无论调用顺序如何，所有线程获取两把锁的顺序都一致，环形等待不可能形成。注意 `else` 分支里那个 `this != &other`——新旧对象落进同一个分片时只锁一次，避免自死锁。

（这段代码不在 `objc-os.h` 里，它随着一次重构搬到了 `runtime/Threading/mixins.h`。按老文章的线索去 grep `lockTwo` 是找不到的。）

第 4 步和第 10 步是同一个模式的两次出现：**只要可能回调到用户代码，就必须先放锁。** 触发 `+initialize` 会跑用户代码，所以解锁重试；`callSetWeaklyReferenced` 也会，源码注释解释得很清楚：

```c
// This must be called without the locks held, as it can invoke
// arbitrary code. In particular, even if _setWeaklyReferenced
// is not implemented, resolveInstanceMethod: may be, and may
// call back into the weak reference machinery.
```

第 8 步也不是随手写的。整个流程里对 `*location` 只写一次，而且是在两张表都锁住、登记都完成之后——源码里那行注释是 `// Do not set *location anywhere else. That would introduce a race.`

### 读取

读一个 `__weak` 变量不是简单取值。编译器插入 `objc_loadWeakRetained`，然后 autorelease：

```
objc_loadWeakRetained(location)
  1. 取出裸指针；是 tagged pointer 或 nil 就直接返回，不加锁不查表
  2. 锁住对象所在的 SideTable
  3. 若 *location 被并发改过，解锁重试
  4. 查 weak_entry_for_referent 确认表项存在
  5. rootTryRetain()  ← 检查对象是否正在析构
       正在析构 → 失败，结果为 nil
       否则     → 引用计数 +1，拿到一个保证存活的强引用
  6. 解锁，返回
```

为什么非要 retain 一下？Clang ARC 规范说得很明确：

> For `__weak` objects, the current pointee is retained and then released at the end of the current full-expression. In particular, messaging a `__weak` object keeps the object retained until the end of the full expression. This must execute atomically with respect to assignments and to the final release of the pointee.

保活的粒度是整个 full-expression。从"读出指针"到"用完它"之间，对象随时可能在另一个线程被释放，所以必须在持锁的前提下把它 retain 住。

第 5 步的 `rootTryRetain()` 只是快路径。重写了 retain/release 的类走的是 `-retainWeakReference`，`NSObject` 的默认实现是 `return [self _tryRetain];`，殊途同归。这条岔路后面还会用到。

第 4 步那个表项检查值得单说，因为它给下一节的结论加了个限定：

```c
if (slowpath(entry == NULL)) {
    table->unlock();
    weakTableScan();
    _objc_fault_and_log("Weak reference loaded from %p contains %p which is not "
                        "in the weak references table", location, obj);
    return objc_retain(obj);
}
```

表项对不上时 runtime 报错，然后**不走 tryRetain**，直接 retain 返回。正常程序里表项一定在，但严格说"读 weak 必然经过 tryRetain 把关"这句话有这么一个例外。

### 清除

```
引用计数归零 → 对象进入析构状态     ← 早于 -dealloc 方法体执行
      ↓
objc_msgSend(obj, @selector(dealloc))
      ↓
… 一路 super 到 NSObject 根实现 …
      ↓
object_dispose → objc_destructInstance → clearDeallocating
      ↓
weak_clear_no_lock：遍历 entry 里的每个 referrer，
                    若 *referrer == referent 就写成 nil，
                    然后把整条 weak_entry_t 摘掉
```

"进入析构状态"这个说法比"置 DEALLOCATING 位"准确。nonpointer isa 上根本没有这个位：

```cpp
bool isDeallocating() const {
    return extra_rc == 0 && has_sidetable_rc == 0;
}
```

引用计数归零本身就是"正在析构"。上一篇讲过，`extra_rc` 的语义从 `retainCount - 1` 改成 `retainCount` 之后，那个独立的标志位就被省掉了。只有 raw-pointer isa 的对象才在 side table 的引用计数里用 `SIDE_TABLE_DEALLOCATING` 位表示。两条路径殊途同归，`tryRetain` 都会失败。

清除时那个 `if (*referrer == referent)` 是防御性检查。`*referrer` 为 `nil` 时什么都不做；非 nil 且不等于 referent 才报错，措辞是：

```text
__weak variable at %p holds %p instead of %p.
This is probably incorrect use of objc_storeWeak() and objc_loadWeak().
Break on objc_weak_error to debug.
```

`objc_weak_error` 是个 `BREAKPOINT_FUNCTION`，默认不 abort，可以直接下断点。

---

## 三、dealloc 里的 weak 到底是不是 nil

上面的流程里有个容易误读的地方。进入析构状态发生在 `dealloc` 之前，而真正把 `*referrer` 写成 `nil` 要等 `dealloc` 一路走完、进到 `object_dispose` 才发生。中间隔着你自己写的 `dealloc` 方法体。

于是流传出这样一个说法：`dealloc` 执行期间 weak 指针还没被清空，所以存在一段"野指针窗口"。

测一下。让 `dealloc` 同时打印三个东西——之前注册好的 weak、一个 `unsafe_unretained`、以及 `self`：

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

```text
  [存活时] weak = <Probe: 0x600000010020>
  [dealloc 中] 读之前注册的 weak     = (null)
  [dealloc 中] 读 unsafe_unretained = 0x600000010020
  [dealloc 中] self 本身             = 0x600000010020
  [dealloc 后] weak = (null)
```

`self` 是有效指针，`unsafe_unretained` 也还指着它，唯独 weak 已经读出 `nil`。

原因就在 `objc_loadWeakRetained` 的第 5 步：每次读取都要过 `tryRetain`，而对象一旦进入析构状态，`tryRetain` 必然失败。`dealloc` 期间内存里那个 `*referrer` 字段确实还是旧值。但你没有办法读到它——读 weak 变量只有 `objc_loadWeak` 那一条路，路上有 `tryRetain` 把关。

### 为什么不干脆提前清零

这才是最有意思的部分。Apple 曾经就是那么做的，后来故意改了回来，理由写在 `objc_loadWeakRetained` 正上方：

```c
/*
  Once upon a time we eagerly cleared *location if we saw the object 
  was deallocating. This confuses code like NSPointerFunctions which 
  tries to pre-flight the raw storage and assumes if the storage is 
  zero then the weak system is done interfering. That is false: the 
  weak system is still going to check and clear the storage later. 
  This can cause objc_weak_error complaints and crashes.
  So we now don't touch the storage until deallocation completes.
*/
```

有代码会去"预检"那块原始存储，看到零就以为 weak 系统已经收工了——而实际上 runtime 稍后还要再来检查并清理一次，于是撞出 `objc_weak_error` 和崩溃。

所以"指针清零晚于状态变化"不是一个实现上的懒惰，是一次明确的设计回退。上面那个实验测出来的现象，源码注释里有人签了字。

### dealloc 里的 weakSelf 一定是 nil

这个很多人撞过的困惑，到这里就有解释了。想在 `dealloc` 里用自己，直接用 `self`。

不过要补一句：这是 `NSObject` 默认实现的结果，不是语言保证。前面提过读取路径上有 `-retainWeakReference` 这条岔路——一个类只要重写它，让它在析构期间返回 YES，`dealloc` 里的 weak 就不是 `nil` 了。我想不出什么正当理由这么干，但机制上它是敞开的。

### 在 dealloc 里注册新的 weak 会直接崩

既然对象已经在析构，再登记一条弱引用就没意义了——没人会再来清它。

```objc
- (void)dealloc {
    NSLog(@"进入 dealloc，self = %p", self);
    __weak typeof(self) weakSelf = self;
    NSLog(@"%@", weakSelf);
}
```

```text
进入 dealloc，self = 0x600000008040
objc[73993]: Cannot form weak reference to instance (0x600000008040) of class Boom.
It is possible that this object was over-released, or is in the process of deallocation.
Child process terminated with signal 6: Abort trap
```

这条崩溃在线上很常见，提示里给了两种可能：过度释放，或者正在析构。排查时如果堆栈里有 `dealloc`，就是后一种。

崩不崩其实是可选的。`objc-weak.h` 里定义了三种策略：

```cpp
enum WeakRegisterDeallocatingOptions {
    ReturnNilIfDeallocating,
    CrashIfDeallocating,
    DontCheckDeallocating
};
```

`objc_storeWeak` / `objc_initWeak` 用 `CrashIfDeallocating`，也就是上面这条。还有一对 SPI `objc_storeWeakOrNil` / `objc_initWeakOrNil` 用 `ReturnNilIfDeallocating`，同样场景下静默存 `nil`。`DontCheckDeallocating` 全 runtime 只有 `objc_moveWeak` 在用——移动的前提是这条弱引用已经存在，存活性早就验证过了。

---

## 四、编译器实际会发哪个函数

前面只讲了 `storeWeak` 和 `loadWeakRetained`，实际上编译器会根据场景发五个不同的函数。用 `clang -S -emit-llvm` 编一份包含各种用法的代码，逐个函数看它调了什么：

```text
-[H setWn:]  / -[H setWa:]              storeWeak
-[H wn]      / -[H wa]                  loadWeakRetained
-[H .cxx_destruct]                      destroyWeak
make                                    initWeak storeWeak copyWeak destroyWeak
__Block_byref_object_copy_              moveWeak
__copy_helper_block_e8_32r40w48w        copyWeak
__destroy_helper_block_e8_32r40w48w     destroyWeak
```

对应关系是：

| 场景 | 调用 |
| --- | --- |
| `__weak id w = obj;` 局部变量初始化 | `objc_initWeak` |
| `w = obj2;` 已初始化后赋值 | `objc_storeWeak` |
| 局部 weak 变量出作用域 | `objc_destroyWeak` |
| weak **ivar** 随对象销毁 | `objc_destroyWeak`（在 `.cxx_destruct` 里，每个一次） |
| `__weak id b = a;` 用 weak 初始化 weak | `objc_copyWeak` |
| **block 被 copy 到堆，它捕获了 `__weak` 变量** | `objc_copyWeak` |
| `__block __weak` 的 byref 结构从栈搬到堆 | `objc_moveWeak` |

倒数第二行是每天都在写、却几乎没人讲成本的那个。`objc_copyWeak` 的实现只有三行：

```cpp
void objc_copyWeak(id *dst, id *src) {
    id obj = objc_loadWeakRetained(src);
    objc_initWeak(dst, obj);
    objc_release(obj);
}
```

也就是说，**每次一个捕获了 `weakSelf` 的 block 被 copy 到堆上，都要做一次完整的 loadWeakRetained 加一次 initWeak**——两次加锁、两次查表。在循环里创建这种 block 是有实打实代价的。

还有一条顺便回答了高频疑问：weak 属性写不写 `atomic`，生成的代码完全一样。上面 `setWn:`（nonatomic）和 `setWa:`（atomic）都是 `storeWeak`，getter 也都是 `loadWeakRetained`。因为 `objc_storeWeak` 自带 SideTable 锁，原子性由 runtime 保证，跟 `atomic` 关键字没关系。

### 那到底慢多少

公开资料里没有 weak 读取的 benchmark——Mike Ash 两篇都没有量化数据，verdagon 那篇给的是他自己的方案对比引用计数的数字，跟这个无关，Apple 从未公布过。既然没有，自己测一个。

一千万次属性读取，`-O2`：

```text
第 1 轮  strong=16.8ns  unsafe=18.1ns  weak=46.1ns   weak/strong=2.7x
第 2 轮  strong=11.3ns  unsafe=11.5ns  weak=46.3ns   weak/strong=4.1x
```

三到四倍。这跟源码路径对得上：一次加锁 + 一次哈希查表 + 一次 CAS，对比一次普通 load。

绝对值不大，46 纳秒一次。所以我的判断是：除非是逐帧执行几千次的热路径，否则不值得为这点开销放弃自动置 nil。真要优化，先做的应该是把循环里反复读的 weak 变量用局部强引用接住——这个改动零风险，收益还比换成 `unsafe_unretained` 大。

（测试条件：iOS 模拟器 arm64，Apple Silicon Mac，通过合成 getter 访问以免被优化掉。真机数字会不同，尤其真机只有 8 个锁分片，争用更容易发生。）

---

## 五、Tagged Pointer 完全绕开这套机制

`weak_register_no_lock` 一开始就短路：

```c
if (_objc_isTaggedPointerOrNil(referent)) return referent_id;
```

`objc_loadWeakRetained` 同理，取出的值是 tagged pointer 就直接返回，不加锁不查表。

所以对 tagged pointer 声明 `__weak` 不会崩溃，也不会进表，更不会被置 `nil`——它压根没有"销毁"这个事件。[[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer#四个会漏出来的地方|上一篇实测过]]，weak 指向一个 tagged 的 `@42`，"释放"之后读出来还是 42。

准确的说法是它的 weak 退化成了一次普通赋值。不计数、不清零，因为它本来就不会消失。

这个坑值不值得担心？我的判断是不值得，但要知道它存在。它只在一种情况下中招：你用 weak 去观察一个可能是 `NSNumber` 或短 `NSString` 的对象的生命周期。而这种设计本身就有问题——值类型对象的生命周期不该被观察。真撞上了，说明该改的是上层设计，不是这行 weak。

---

## 六、几个说法需要辨析

#### "weak 表的 key 是 weak 指针"

反了。`weak_table_t` 的 key 是被弱引用的对象，value 是指向它的所有 weak 变量地址的集合。weak 变量的地址是 `weak_entry_t` 内部那一层的 key。

#### "SideTable 就是 weak 表"

`SideTable` 是复合结构，包含一把锁、引用计数溢出表、弱引用表三部分。

#### "weak 在 dealloc 结束后才置 nil，中间是野指针"

清零确实晚，但读取安全性从对象进入析构状态就生效了。而且这是 Apple 有意为之的回退，理由写在源码注释里，见第三节。

#### "weak 和 unsafe_unretained 只差会不会自动置 nil"

差别更靠前。`__weak` 会调 `objc_storeWeak` 把变量地址登记进表，`__unsafe_unretained` 只是一次裸赋值，运行时对它一无所知。所以后者不是"没被清零"，是根本没人知道要清它。

#### "weak 只有 ARC 才有"

zeroing weak 这个语义最早随 GC 引入（`objc-weak.h` 的注释里还留着 GC 时代的措辞），但实现是 ARC 期另起炉灶写的，不是复用 libauto 那套读写屏障。

至于"MRC 下没有 `__weak` 只是缺语法糖"——这句我第一版写错了。MRC 下直接写 `__weak` 是硬编译错误：

```text
error: cannot create __weak reference in file using manual reference counting
```

但 clang 有个 `-fobjc-weak`，打开后 MRC 文件里 `__weak` 完全可用，IR 里照样发 `objc_initWeak` / `objc_loadWeak` / `objc_destroyWeak`。zeroing weak 和 ARC 是两个可以拆开的特性，只是默认绑在一起了。

#### "有些类不能用 weak"

这条还成立，但那份广为流传的名单已经过期十几年了。中文圈几乎所有文章都在原样复制 Apple 2012 年《Transitioning to ARC Release Notes》里的清单：`NSFont`、`NSColorSpace`、`NSViewController`、`NSWindow`、`NSWindowController`……

在当前 SDK 上扫一遍就知道了：

```shell
grep -rl NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE \
  $(xcrun --sdk iphonesimulator --show-sdk-path)/System/Library/Frameworks/
```

iOS 26.5 SDK 上只命中两个文件：宏定义所在的 `NSObjCRuntime.h`，和 `Foundation/NSPort.h`。也就是说整个 iOS SDK 里现在只剩 `NSMachPort` 和 `NSMessagePort` 不支持 weak，UIKit 一个都没有。当年名单上那些类现在全都可以 weak。

还要区分两种拒绝机制：`NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE` 是编译期拒绝（clang 直接报错），`-allowsWeakReference` 返回 NO 是运行期拒绝（走 crash 或 OrNil）。

---

## 七、Swift 走的是另一条路

Objective-C 用全局共享的哈希表记录"谁指向我"，Swift 把这份账放进了对象自己的 side table。

当前实现（`RefCount.h` 的注释）是三个计数，不是两个：

> An object conceptually has three refcounts. ... When the strong RC reaches zero the object is deinited, unowned reference reads become errors, and weak reference reads become nil. ... When the unowned RC reaches zero the object's allocation is freed. ... When the weak RC reaches zero the object's side table entry is freed.
>
> Objects initially start with no side table. They can gain a side table when: a weak reference is formed ...
>
> Strong and unowned variables point at the object. **Weak variables point at the object's side table.**

几个关键点跟常见说法不一样。strong 和 unowned 计数内联在对象头里，weak 计数只存在于 side table；而 side table 是**第一次形成弱引用时才惰性分配**的，一旦分配不再回收。对象内存在 unowned 计数归零时就 free 了——"整个对象变成僵尸占位"是 unowned 造成的，跟 weak 无关，weak 计数归零释放的只是那个小小的 side table entry。而且 weak 变量指向的是 side table，不是对象。

（我第一版把这段写成了"双计数器 + 对象变僵尸"，那是 Swift 3 及以前的行为，Swift 4 就改了。更尴尬的是我在参考资料里挂的两篇，标题里就写着 Swift 4。）

两种方案其实是同一个惰性思路：ObjC 是"没有 weak 就完全不碰 weak 表"，Swift 是"没有 weak 就不分配 side table"，都赌绝大多数对象一辈子不会被弱引用。这个赌注赢面都很大。

差别在于代价的形态。ObjC 一旦用上 weak，每次读写都要吃全局分片表的锁——争用取决于地址哈希，你无法预测也无法控制。Swift 把成本前移到"第一次形成 weak 时分配一个堆对象"，之后的读写只是本对象上的原子操作，没有哈希查找、没有跨对象争用。

我更喜欢 Swift 这个设计，理由是它的开销是**可预测**的：你知道每个被弱引用的对象多花了多少内存，而 ObjC 那边的锁争用是个跟地址分布有关的运气问题，真机上只有 8 片就更是如此。当然代价是 side table 分配了不再回收——一个短暂被 weak 引用过的长寿对象，会一直背着它。

顺便，两边在一件事上行为完全一致：Swift 的状态机里 `DEINITING with side table` 状态下 "Weak variable load returns nil"。跟第三节测出来的 ObjC 行为一模一样，实现路径完全不同，结论殊途同归。

还有一座桥。Swift 的 `weak var` 指向 Objective-C 对象时，走的就是本文这套机制——`WeakReference.h` 里非 native 对象的分支直接调 `objc_initWeak` / `objc_storeWeak` / `objc_loadWeakRetained` / `objc_copyWeak` / `objc_moveWeak`。混编项目里写 `weak var delegate: SomeOCProtocol?`，用的就是 SideTable。注意它用的是 `objc_initWeak` 那个会崩的版本，不是 OrNil。

---

## 总结

这篇的结论只有一条：weak 是一张以"被引用对象"为 key 的反向索引，读取的安全性由 `tryRetain` 保证，而它比指针清零早——早在整个 `dealloc` 之前。所谓"野指针窗口"不存在，不是因为清零够快，是因为根本没有绕开 `tryRetain` 的读取路径。而且这是 Apple 有意设计的：他们曾经真的提前清零，后来因为撞出崩溃改了回来，理由就写在源码注释里。

下一篇转向 Block：[[iOS Block 的结构：ABI、descriptor 与三种类型]]。

## 参考资料

### 源码（这篇几乎全部结论都来自这里）

- [objc4 — objc-weak.h](https://github.com/apple-oss-distributions/objc4/blob/main/runtime/objc-weak.h) / [objc-weak.mm](https://github.com/apple-oss-distributions/objc4/blob/main/runtime/objc-weak.mm)：`weak_table_t`、`weak_entry_t`、注册与清除，以及各层的扩容阈值
- [objc4 — NSObject.mm](https://github.com/apple-oss-distributions/objc4/blob/main/runtime/NSObject.mm)：`objc_storeWeak`、`objc_loadWeakRetained`，以及第三节那段 "Once upon a time" 注释
- [objc4 — NSObject-private.h](https://github.com/apple-oss-distributions/objc4/blob/main/runtime/NSObject-private.h)：`SideTable` 的结构体本体。很多文章漏掉这个文件，于是把 SideTable 和 weak_table 混为一谈
- [objc4 — objc-private.h](https://github.com/apple-oss-distributions/objc4/blob/main/runtime/objc-private.h)：`StripedMap` 的分片数、哈希与缓存行对齐
- [objc4 — Threading/mixins.h](https://github.com/apple-oss-distributions/objc4/blob/main/runtime/Threading/mixins.h)：`lockWith` 的地址排序加锁，这段代码从 `objc-os.h` 搬过来了
- [Clang ARC Specification — Semantics](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#semantics)：`__weak` 读写的原子性要求
- [swift — RefCount.h](https://github.com/swiftlang/swift/blob/main/stdlib/public/SwiftShims/swift/shims/RefCount.h)：三计数与对象生命周期状态机
- [swift — WeakReference.h](https://github.com/swiftlang/swift/blob/main/stdlib/public/runtime/WeakReference.h)：Swift 的 weak 指向 ObjC 对象时怎么走

### 文章

- [Mike Ash — Zeroing Weak References in Objective-C](https://www.mikeash.com/pyblog/friday-qa-2010-07-16-zeroing-weak-references-in-objective-c.html)：写于官方机制落地之前，作者自己造了个轮子。这是历史背景，数据结构和后来的 objc4 不是一回事
- [Mike Ash — Swift 4 Weak References](https://mikeash.com/pyblog/friday-qa-2017-09-22-swift-4-weak-references.html)：第七节 side table 方案的来源
- [Surprising Weak-Ref Implementations](https://verdagon.dev/blog/surprising-weak-refs)：横向对比多种语言的弱引用实现。注意正文里的 Swift 部分是 Swift 4 之前的版本，页面上挂了勘误；对 ObjC 的描述也漏了分片这一层

### 本地

- [[iOS 内存：ARC 的两半|ARC 的两半]]
- [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]]
- [[iOS 属性关键字：从所有权推导，而不是从类型名猜]]

---

实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -fobjc-arc -target arm64-apple-ios17.0-simulator`。第三节两组输出、第四节的 IR 统计和 benchmark 都是真实运行结果。

分片数、`WEAK_INLINE_COUNT`、各级扩容阈值都是实现细节，会随版本变化。但"读取安全性早于指针清零"这条是由 `tryRetain` 检查析构状态这个设计保证的，不依赖任何具体常量——连 `deallocating` 标志位本身都被重构掉了，这个结论还立着。

真机与模拟器有一处差异会影响外推：`StripedMap` 的分片数在模拟器上是 64，真机只有 8，锁争用在真机上更容易发生。第四节那组 benchmark 数字不能直接搬到真机。
