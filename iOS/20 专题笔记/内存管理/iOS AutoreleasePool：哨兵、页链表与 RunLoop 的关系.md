---
title: 【iOS】AutoreleasePool：哨兵、页链表与 RunLoop 的关系
published: 2026-07-27
description: 一页装 505 个是对的，但你永远数不满，因为最新的那个被扣在 TLS 里。以及：异常抛出去池不会排空，主线程的池今天是每次 callout 一个而不是每轮 RunLoop 一个。
tags:
  - iOS
  - Objective-C
  - Memory
  - AutoreleasePool
  - RunLoop
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 18
draft: true
---
# AutoreleasePool：哨兵、页链表与 RunLoop 的关系

我写了个循环，一直往池里灌对象，直到页满为止。灌了 505 次。然后把页打印出来数条目，是 504 个。

差一个。

第一反应当然是我数错了，或者哪个对象被合并了。都不是。少掉的那个躺在线程本地存储里，它要等下一次 `autorelease` 把它挤进来。这个行为在中文资料里我一篇都没见过，而它恰好是理解今天这套机制的入口。

本文只管池本身：数据结构、哨兵、页链表、push/pop、线程绑定、嵌套，以及池和 RunLoop 的那一段接缝。RunLoop 的 mode/source/observer 是 [[iOS RunLoop：mode、source 与那张流程图今天还对不对]] 的活。池的基本形状在 [[iOS 内存：MRC 的所有权规则#五、autorelease 池的实现|MRC 的所有权规则]] 第五节已经描过一遍，这篇负责把每个数字都测出来，顺便修两处我自己在那篇里写得不够准的地方。

三条先摆在前面：

- 一页 505 个槽位这个数在所有 64 位 Apple 平台上都成立，但它是 `PAGE_MIN_SIZE` 而不是系统页大小算出来的。这台 Mac 的系统页是 16384。
- **异常从 `@autoreleasepool {}` 里抛出去，池不会排空。** Clang ARC 规范白纸黑字写着，编译产物里也确实没有对应的 cleanup。
- 主线程上的池不是"每轮 RunLoop 一个"。CoreFoundation 今天是**每次 callout 一个**，UIKit 那两个著名的 observer 在现代系统上根本不装。

---

## 一、先把池打印出来

libobjc 导出了一个私有函数，声明一下就能调：

```objc
extern void _objc_autoreleasePoolPrint(void);
```

它把当前线程的整条页链表按栈序打出来，包括每页地址、每个哨兵、每个入池对象的类名。这是全文最好用的仪器。嵌套三层的样子：

```text
AUTORELEASE POOLS for thread 0x1fa339d80
8 releases pending.
[0xb3500c000]  ................  PAGE  (hot) (cold)
[0xb3500c038]  ################  POOL 0xb3500c038
[0xb3500c040]       0x100a50ee0  NSObject
[0xb3500c048]  ################  POOL 0xb3500c048
[0xb3500c050]       0x100a50130  NSObject
[0xb3500c058]  ################  POOL 0xb3500c058
[0xb3500c060]       0x100a52770  __NSArrayM
[0xb3500c068]       0x100a527c0  __NSCFString
[0xb3500c070]  ################  POOL 0xb3500c070
```

三个 `POOL` 就是三层嵌套的哨兵，夹在中间的是各层入池的对象。嵌套没有任何特殊机制，就是同一个栈上多插几个哨兵。

哨兵的值是 `nil`。`objc_autoreleasePoolPush()` 的返回值就是哨兵那个槽的地址：

```text
push 返回值 token   = 0xc98c08038
哨兵槽里存的值      = 0x0   (POOL_BOUNDARY)
token 所在页首      = 0xc98c08000
token - 页首        = 56 字节
```

56 这个偏移待会儿要用。`pop(token)` 拿地址反推出页，然后把栈弹回这个位置，沿途每个非哨兵条目发一次 `release`，弹过的槽位涂成 `0xA3`：

```text
pop 之后已释放槽位的字节 = a3 a3 a3 a3 a3 a3 a3 a3
```

还有一个不占内存的池。在一个刚创建的线程上第一次 push，返回的是 `0x1`：

```text
线程 1: 全新线程上第一次 push 返回 0x1
AUTORELEASE POOLS for thread 0x16de17000
0 releases pending.
[0x1]  ................  PAGE (placeholder)
[0x1]  ################  POOL (placeholder)
```

这就是 `EMPTY_POOL_PLACEHOLDER`。源码注释说得很直白，连受益者的名字都点了：

> EMPTY_POOL_PLACEHOLDER is stored in TLS when exactly one pool is pushed and it has never contained any objects. This saves memory when the top level (i.e. libdispatch) pushes and pops pools but never uses them.

我把 `objc_autoreleasePoolPush` / `Pop` 用 `__DATA,__interpose` 拦下来打调用方，正好抓到了注释里点名的那位：

```text
{PUSH  token=0x1   调用方= _dispatch_last_resort_autorelease_pool_push  (libdispatch.dylib)}
<全局队列的 block 里>
{POP   token=0x1   调用方= _dispatch_last_resort_autorelease_pool_pop   (libdispatch.dylib)}
```

GCD 每跑一个 block 就 push/pop 一次。绝大多数 block 里一个对象都不入池，占位符让这些 push 一分钱内存都不花。

---

## 二、一页 505 个，这个数字在哪些平台上成立

先把变量列清楚，因为这一节最容易翻车。页容量由四件事决定：`AutoreleasePoolPage::SIZE`、页头结构体的大小、指针宽度，以及哨兵占不占槽。四个都得单独确认，只扫一个就下结论必错。

### SIZE

```c
static size_t const SIZE =
#if PROTECT_AUTORELEASEPOOL
    PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
    PAGE_MIN_SIZE;  // size and alignment, power of 2
#endif
```

我在 MRC 那篇里写过一句"`PROTECT_AUTORELEASEPOOL` 在发布的 objc4 里根本没有定义"。这话不准。它定义了，就在 `NSObject-internal.h`：

```c
// Set this to 1 to mprotect() autorelease pool contents
#define PROTECT_AUTORELEASEPOOL 0
```

结论没变（走 `PAGE_MIN_SIZE` 那支），但"没定义"和"定义成 0"是两回事，改一下。

### PAGE_MIN_SIZE 不等于系统页大小

这台 Apple Silicon Mac 上：

```text
getpagesize()      = 16384
vm_page_size       = 16384
PAGE_MAX_SIZE      = 16384
PAGE_MIN_SIZE      = 4096
```

系统页是 16 KB，池页是 4 KB。实测四页连续分配出来的地址差正好 4096：

```text
第一页 0xc98c08000  →  第二页 0xc98c09000  →  第三页 0xc98c0a000
两页地址差 = 4096 字节
```

也就是说四个池页挤在同一个物理页里。`operator new` 用的是 `posix_memalign(&result, SIZE, SIZE)`，对齐和大小都是 4096，`pageForPointer` 才能靠 `p % SIZE` 从任意条目地址反推页首。

那 `PAGE_MIN_SIZE` 会不会在别的平台上变？把六个 target 的平台头扫了一遍：

| target | PAGE_MIN_SHIFT | PAGE_MAX_SHIFT |
| --- | --- | --- |
| arm64-apple-macos26.0 | 12 | 14 |
| x86_64-apple-macos26.0 | 12 | 14 |
| arm64-apple-ios17.0（真机） | 12 | 14 |
| arm64-apple-ios17.0-simulator | 12 | 14 |
| x86_64-apple-ios17.0-simulator | 12 | 14 |
| arm64_32-apple-watchos10.0 | 12 | 14 |

清一色 4096。

### 页头 56 字节

不用自己数字段，libobjc 把偏移全导出来了：

```text
magic  offset = 0
next   offset = 16
thread offset = 24
parent offset = 32
child  offset = 40
depth  offset = 48
hiwat  offset = 52
begin  offset = 56  (= sizeof(AutoreleasePoolPageData))
```

对得上 `AutoreleasePoolPageData` 的声明。`magic_t` 是四个 `uint32_t`，16 字节。`next`、`thread`、`parent`、`child` 四个指针各 8 字节。`depth` 和 `hiwat` 两个 `uint32_t` 各 4 字节。合计 56。

`(4096 - 56) / 8 = 505`。跑出来的数一致：

```text
刚 push 完:  page=0xc98c08000 depth=0 used=1   cap=505
第一页满:    page=0xc98c08000 depth=0 used=505 cap=505
第二页:      page=0xc98c09000 depth=1 used=1   cap=505 parent=0xc98c08000
```

所以 505 是槽位数。开池的那一页里有一个槽给了哨兵，能装 504 个对象；后续页没有哨兵，能装满 505 个。中文资料统一写 505，指的是槽位，没错，只是很少有人说清楚那一个哨兵。

至于 32 位平台，指针 4 字节、页头缩到 40 字节，`(4096 - 40) / 4 = 1014`。arm64_32 的 watchOS 我编得出来但跑不了，这个数只是按同一套公式推的，没实测。

> 待真机补测：在 iPhone 上跑同一份 `e4.m`，确认 `objc_debug_autoreleasepoolpage_begin_offset` 仍是 56、相邻两页地址差仍是 4096。复现方法就是把本文第二节那段读页头的代码原样拿过去，真机上 `getpagesize()` 会返回 16384，别被它带偏。

### 顺带一个对不上的地方

`AutoreleasePoolEntry` 用高位存"同一个对象连续入池多少次"。公开的 objc4（最新 tag 是 951.7）里写的是：

```c
struct AutoreleasePoolEntry {
    static constexpr uintptr_t pointerBits = 48;
    static constexpr uintptr_t countBits = 16;
    static constexpr uintptr_t pointerMask = MASK(pointerBits);
    uintptr_t ptr: pointerBits;
    uintptr_t count: countBits;
};
```

照这个，`pointerMask` 应该是 `0xffffffffffff`，count 占 bit48~63。但 macOS 26.5 上跑着的 libobjc 自己导出的常量是：

```text
libobjc 报告的 ptr_mask = 0x0f00ffffffffffff
```

指针位是 bit0~47 加上 bit56~59，不连续。我把同一个对象连着 `autorelease` 70 次，看栈顶那个字的变化：

```text
第  3 次: raw^addr=0x1000000000000000      count = 1
第  9 次: raw^addr=0x7000000000000000      count = 7
第 17 次: raw^addr=0xf000000000000000      count = 15
第 25 次: raw^addr=0x7001000000000000      count = 23
第 33 次: raw^addr=0xf001000000000000      count = 31
```

低四位在 bit60~63，进位进到 bit48~55。count 是 12 位不是 16 位，中间让出 4 位给指针。公开源码停在 951.7，系统上这份显然更新，布局改了。我没有更新的源码可查，所以只报测量结果，不猜它为什么这么改。

---

## 三、少掉的那一个

回到开头。505 次 `autorelease` 之后页才满，但页里只有 504 个对象加一个哨兵。逐次打印就能看到它一直差一拍：

```text
push 后 used=1
  第 1 次 autorelease 后 used=1     ← 没进去
  第 2 次 autorelease 后 used=2
  第 3 次 autorelease 后 used=3
  ...
  第 505 次 autorelease 后 used=505  ← 满了
```

答案在 `rootAutorelease()` 里：

```c
ALWAYS_INLINE id objc_object::rootAutorelease()
{
    if (isTaggedPointer()) return (id)this;
    ...
    if (prepareOptimizedReturn((id)this, true, ReturnAtPlus1)) return (id)this;
    if (slowpath(isClass())) return (id)this;
    return rootAutorelease2();
}
```

第二个参数 `cameFromRootAutorelease` 传的是 `true`。而 `prepareOptimizedReturn` 里那道"检查调用方是不是真的打算认领"的关卡，只对 `false` 生效：

```c
if (!cameFromRootAutorelease && obj->ISA()->hasCustomRR())
    if (!callerAcceptsOptimizedReturn(clientReturnAddress()))
        return false;

setReturnAutoreleaseInfo({obj, cameFromRootAutorelease, disposition, clientReturnAddress()});
return true;
```

所以每一次普通的 `objc_autorelease` 都会先把对象扣在 TLS 里，连同返回地址一起，然后直接返回。它压根没进页。等下一次有对象要入池，或者有人 push/pop，`moveTLSAutoreleaseToPool` 才把上一个挪进去。

这不是返回值优化的副产品，是 `autorelease` 本身现在的第一步。[[iOS 内存：ARC 的两半#返回值的所有权交接|ARC 的两半]] 里讲的 `objc_autoreleaseReturnValue` 走的是同一个 `prepareOptimizedReturn`，只是 `cameFromRootAutorelease` 传 `false`，多一道关卡而已。

我的判断是：任何靠"数池里有几个对象"来推断行为的实验，都得先把这个差一算进去，否则每个数都对不上。我第一版就是被它坑了半小时。

---

## 四、`@autoreleasepool {}` 编译成什么

`-O0` 的 IR，干净得没什么可讲：

```llvm
define void @blockVersion() #1 {
  %1 = call ptr @llvm.objc.autoreleasePoolPush() #2
  ...
  call void @llvm.objc.autoreleasePoolPop(ptr %1)
  ret void
}
```

MRC 下的 `NSAutoreleasePool` 对象版编出来的是另一副样子：

```llvm
define void @objectForm() local_unnamed_addr #0 {
  %2 = tail call ptr @objc_alloc_init(ptr %1)   ; [[NSAutoreleasePool alloc] init]
  tail call void @body()
  tail call void @objc_release(ptr %2)          ; [p release]
  ret void
}
```

我在 MRC 那篇里说这两者"走另一套代码路径"。用 interpose 一测，这话得收回：

```text
{PUSH  token=0x8a7008038   调用方= _CFAutoreleasePoolPush  (CoreFoundation)}
NSAutoreleasePool 实例 = 0x100bb2890, class = NSAutoreleasePool, 父类 = NSObject
实例大小 = 40 字节
{POP   token=0x8a7008038   调用方= _CFAutoreleasePoolPop   (CoreFoundation)}
```

`-init` 里面调 `_CFAutoreleasePoolPush`，`-drain` 调 `_CFAutoreleasePoolPop`。两者最终落到同一对 `objc_autoreleasePoolPush/Pop`，同一条页链表。它是一个 40 字节的真实对象，包着 token，如此而已。

两种写法真正的差别只有三点：对象版多一次 alloc，多一次消息发送，以及 token 可以被你不小心传到别的作用域去。

### 抛异常的时候，池不排空

这条我原本以为是常识的反面。`@autoreleasepool` 处理 `return` 和 `break` 是没问题的，IR 里两条路汇到同一个 pop：

```text
%2 = call ptr @llvm.objc.autoreleasePoolPush()
br i1 %4, label %5, label %6      ← if (cond()) return;
...
call void @llvm.objc.autoreleasePoolPop(ptr %2)
ret void
```

但异常路径上没有 pop。我把 `-fobjc-arc-exceptions` 和 `-fexceptions` 都打开试过，也换成 ObjC++ 试过（那边异常默认开）。landingpad 里只有局部强引用的 `objc_release`。`autoreleasePoolPop` 一次都没出现在 cleanup 里。

Clang ARC 规范说得比我的实验还直接：

> Upon entry to this block, the current state of the autorelease pool is captured. When the block is exited normally, whether by fallthrough or directed control flow (such as `return` or `break`), the autorelease pool is restored to the saved state, releasing all the objects in it. When the block is exited with an exception, the pool is not drained.

运行时验证一遍。在内层池里塞两个能打 `dealloc` 日志的对象，然后抛异常，外面 `@catch`：

```text
外层 push 后 used=1
池内 used=3
catch 到 Boom
catch 之后 used=7   （若池被 pop 应回到 1）

AUTORELEASE POOLS for thread 0x1fa339d80
[0x990c04038]  ################  POOL 0x990c04038     ← 我的外层池
[0x990c04040]  ################  POOL 0x990c04040     ← 内层池的哨兵还在
[0x990c04048]       0x101475d80  Probe9
[0x990c04050]       0x101476530  Probe9
[0x990c04058]       0x101476760  __NSCFString
[0x990c04060]       0x1014766d0  NSException
[0x990c04068]       0x101477030  _NSCallStackArray
```

内层哨兵原地不动，两个 `Probe9` 一个都没 dealloc。异常对象和调用栈数组还顺手挤了进来。

这个设计其实说得通：栈已经在展开了，池里那些对象可能正被异常处理逻辑引用着，排空的风险大于收益。但它意味着一件事，**如果你的代码真的在用 ObjC 异常做控制流，`@autoreleasepool` 挡不住内存堆积**。要挡就自己写 `@try/@finally`，或者干脆别用异常做控制流。

---

## 五、池跟着线程走

页头里有个 `thread` 字段。两个线程各自开池，各自的页、各自的 `thread`：

```text
线程 1 (pthread=0x16de17000): 页=0xa20c0e000 页头 thread 字段=0x16de17000 depth=0
线程 2 (pthread=0x16dea3000): 页=0xa20c0f000 页头 thread 字段=0x16dea3000 depth=0
```

两页的地址只差 4096，是从同一个 16 KB 系统页里切出来的，但两条链表毫不相干。

页头里那个 `thread` 不只是给人看的。`check()` 会核对 `thread != objc_thread_self()`，对不上直接 `busted_die()` 走 `_objc_fatal`。不过发布配置下 `CHECK_AUTORELEASEPOOL` 是 0，热路径走的是只看 `magic.m[0]` 的 `fastcheck()`，所以这道检查不是每次入池都跑。跨线程把 token 传来传去具体会不会当场崩，我没测。

热页指针存在 TLS 里，用的是 `tls_direct`。所以"每个线程一个池栈"这句话在实现上就是"热页指针是个 thread-local"。

### 不建池会怎样

经典说法是会 leak，并且打一行 warning。实测两条都要打折。

```text
=== 完全不建池的 pthread ===
  三个对象 autorelease 完了，都还活着
AUTORELEASE POOLS for thread 0x16de17000
2 releases pending.
[0xa20c0f000]  ................  PAGE  (hot) (cold)
[0xa20c0f038]       0x102829300  T9
[0xa20c0f040]       0x10282dd80  T9
  线程函数即将 return
      >> T9(92) dealloc, thread=0x16de17000
      >> T9(91) dealloc, thread=0x16de17000
      >> T9(90) dealloc, thread=0x16de17000
```

默认一行 warning 都没有。对象照样进了页，只是页里没有哨兵（`pop` 那边对这种情况有专门的注释："an object is autoreleased with no pool"）。线程退出时 TLS 的析构器 `HotPageDealloc` 把整条链清掉，三个对象全部释放。没有泄漏。

那句 `autoreleased with no pool in place - just leaking` 要开环境变量才出来：

```shell
OBJC_DEBUG_MISSING_POOLS=YES ./prog
```

```text
objc[19266]: MISSING POOLS: (0x16b013000) Object 0x104ead300 of class T9 autoreleased
with no pool in place - just leaking - break on objc_autoreleaseNoPool() to debug
```

三个对象只报了两条，因为最新那个还在 TLS 里，没走到入池那一步。第三节那个差一，在这儿又冒出来一次。

所以子线程要建池的真正理由不是"不建就泄漏"，而是"不建就得等到线程结束才释放"。对一个跑几秒就退出的线程无所谓，对一个常驻的工作线程就是内存一路涨。Apple 的文档在这点上比中文转述准确：

> If your application or thread is long-lived and potentially generates a lot of autoreleased objects, you should use autorelease pool blocks (like AppKit and UIKit do on the main thread); otherwise, autoreleased objects accumulate and your memory footprint grows.

跟池有关的开关一共六个。`OBJC_HELP=1` 全列得出来：

```text
OBJC_PRINT_POOL_HIGHWATER: log high-water marks for autorelease pools
OBJC_DEBUG_MISSING_POOLS: warn about autorelease with no pool in place, which may be a leak
OBJC_DEBUG_POOL_ALLOCATION: halt when autorelease pools are popped out of order, ...
OBJC_DEBUG_POOL_DEPTH: log fault when at least a set number of autorelease pages has been allocated
OBJC_DISABLE_AUTORELEASE_COALESCING: disable coalescing of autorelease pool pointers
OBJC_DISABLE_AUTORELEASE_COALESCING_LRU: disable coalescing ... using look back N strategy
```

第一个查内存峰值特别顺手：

```text
objc[71564]: POOL HIGHWATER: new high water mark of 1215 pending releases for thread 0x1fa339d80:
objc[71564]: POOL HIGHWATER:     1   libobjc.A.dylib  ... popPageDebug + 256
objc[71564]: POOL HIGHWATER:     2   hw               ... main + 120
```

它连调用栈一起打，直接告诉你哪段代码在灌池。

---

## 六、循环里加池，今天还有用吗

这是全文我最想测清楚的一节。标准答案是"大循环里手动加池防止内存峰值"，Apple 文档也这么写：

> If you write a loop that creates many temporary objects. You may use an autorelease pool block inside the loop to dispose of those objects before the next iteration.

先量。每个配置单独起一个进程，循环 20 万次，取 `phys_footprint` 峰值减基线。

单独起进程这条是被逼出来的。我第一版把六组配置串在一个进程里跑，前一轮撑大的 footprint 会把后一轮的分配吃掉，`arrayWithObjects` 那行硬是测出个假的 +0.03 MB。

| 循环体 | ARC 不加池 | ARC 加池 | MRC 不加池 | MRC 加池 |
| --- | --- | --- | --- | --- |
| `stringWithFormat:`（长串） | +14.00 MB | +0.17 MB | +13.98 MB | +0.20 MB |
| `dataWithLength:4096` | +0.05 MB | +0.05 MB | +1014.89 MB | +0.05 MB |
| `arrayWithObjects:` | +10.78 MB | +0.03 MB | +10.78 MB | +0.03 MB |
| `numberWithDouble:` | +0.03 MB | +0.03 MB | +3.86 MB | +0.05 MB |

`dataWithLength:` 那一行是整张表的重点。同一份代码，MRC 涨 1 GB，ARC 涨 0.05 MB。加不加池对 ARC 完全没有区别。

`stringWithFormat:` 和 `arrayWithObjects:` 那两行则相反，ARC 和 MRC 涨得一模一样，加池省下 14 MB 和 10 MB。

### 到底谁进了池

内存是间接证据，直接数条目更硬。我写了个函数遍历整条页链，把每个条目的合并计数也算进去，然后每种工厂方法调 100 次：

| 调用 | ARC | MRC |
| --- | --- | --- |
| `[NSString stringWithFormat:]` 长串 | 100 | 99 |
| `[NSString stringWithFormat:]` 短串 | 0（tagged） | 0（tagged） |
| `[NSString stringWithUTF8String:]` | 0 | 100 |
| `[NSArray arrayWithObjects:]` | 100 | 100 |
| `[NSArray arrayWithObject:]` | 0 | 100 |
| `[NSMutableData dataWithLength:]` | 0 | 100 |
| `[NSNumber numberWithDouble:]` | 0 | 50 |
| `[NSMutableArray array]` | 0 | 100 |
| `[NSDate date]` | 0（tagged） | 0（tagged） |

MRC 那一列老老实实，除了 tagged pointer（tagged 指针在 `rootAutorelease` 第一行就被特判返回，从不入池）。`numberWithDouble:` 那个 50 是因为一半的值能塞进 tagged 表示。

ARC 那一列只剩两个非零：`stringWithFormat:` 和 `arrayWithObjects:`。

两个都是变参方法。我当时觉得抓到了：变参调用破坏了返回值握手。

这个结论写下来正准备收工，又觉得样本太整齐、整齐得可疑，于是补了个对照实验。自己写一对函数体逐字相同的类方法，一个定参一个变参，都用 MRC 编译，再从 ARC 调用方调：

```objc
+ (id)fixedOne:(int)a       { (void)a; return [[[Fac alloc] init] autorelease]; }
+ (id)variadicOne:(int)a, ... { (void)a; return [[[Fac alloc] init] autorelease]; }
```

```text
调用方 ARC；被调方一律 MRC，函数体逐字相同
  [Fac fixedOne:] 定参             100 次 → 入池    0 次
  [Fac variadicOne:,...] 变参      100 次 → 入池    0 次
```

变参一样能免掉。假设当场废掉。

真正的原因在被调方的收尾指令。`fixedOne:` 的汇编最后一句是：

```asm
	ldp	x29, x30, [sp], #16
	b	_objc_autorelease          ; 尾调用，不是 bl
```

关键在这个 `b`。因为是尾调用，`objc_autorelease` 里读到的 `clientReturnAddress()` 是主调方返回后的那个地址，不是 Foundation 内部的地址。而调用方一侧：

```c
uintptr_t delta = currentReturnAddress - previousReturnAddress;
uintptr_t expectedDelta = expectsNOP ? 8 : 4;   // 两条指令 / 一条指令
if (delta == expectedDelta) return info.getReturnDisposition();
```

差 8 字节（一条 `mov x29, x29` marker 加一条 `bl`）就认领成功，对象直接以 +1 交出，不入池。差不上就退回读 marker 兜底，再不行就老老实实入池。

所以 `stringWithFormat:` 和 `arrayWithObjects:` 免不掉，只能说明 Foundation 里这两个方法的实现没有以尾调用的形式把 autorelease 交出来。变参只是相关，不是原因。这条我确认不了更多，因为 Foundation 不开源，也没法反汇编到能下定论的程度。

### 所以什么时候该加

我自己现在的判断：

- 循环体里只调定参的 Foundation 工厂方法，ARC 下加池基本白加。测出来的差值在 0.05 MB 量级，就是噪声。
- 循环体里有变参工厂方法（`stringWithFormat:`、`arrayWithObjects:`、`dictionaryWithObjectsAndKeys:` 这一类），加池省的是真金白银，十几 MB 起。
- 循环体里调的是你自己或者第三方 MRC 代码写的工厂方法，一律加。上表 MRC 那一列就是它的样子。
- 循环体里有 `imageWithContentsOfFile:`、`stringWithContentsOfURL:` 这种一次分配几 MB 的，别赌，加。

比"加不加"更实用的一条是：真要查，别猜，跑一次 `OBJC_PRINT_POOL_HIGHWATER=YES`，它会把水位和调用栈一起打出来。

2014 年那批文章写"循环里必须加池"的时候，返回值优化才刚落地，Foundation 里参与握手的方法远没有今天多。那个结论在当时是对的。今天它变成了一个要分情况的判断，而分情况的依据只能靠测。

---

## 七、和 RunLoop 的接缝

流传最广的版本是：主线程 RunLoop 里挂着两个 observer，进入时 push，即将休眠时 pop 再 push，退出时 pop。所以主线程的 autoreleased 对象最迟活到本轮 RunLoop 结束。

我在 MRC 那篇里也是这么写的。它今天不成立。

先看 CFRunLoop 自己带不带这样的 observer。macOS 命令行工具里把主 RunLoop 的描述打出来：

```text
observers = (null),
timers = (null),
```

一个都没有。CFRunLoop 本身不管池。

那池是谁 push 的？把 `objc_autoreleasePoolPush/Pop` interpose 掉，加一个覆盖全部 activity 的 observer 和一个 timer，跑两轮：

```text
--- 进 CFRunLoopRun ---
    {PUSH  token=0xa99010038   调用方= _CFAutoreleasePoolPush  (CoreFoundation)}
[Entry]
    {POP   token=0xa99010038   调用方= _CFAutoreleasePoolPop   (CoreFoundation)}
    {PUSH  token=0xa99010038   调用方= _CFAutoreleasePoolPush  (CoreFoundation)}
[BeforeTimers]
    {POP   token=0xa99010038   调用方= _CFAutoreleasePoolPop   (CoreFoundation)}
    {PUSH  token=0xa99010038   调用方= _CFAutoreleasePoolPush  (CoreFoundation)}
[BeforeSources]
    ...
    {PUSH  token=0xa99010038   调用方= _CFAutoreleasePoolPush  (CoreFoundation)}
<timer 回调>
    {POP   token=0xa99010038   调用方= _CFAutoreleasePoolPop   (CoreFoundation)}
    ...
[Exit]
    {POP   token=0xa99010038   调用方= _CFAutoreleasePoolPop   (CoreFoundation)}
--- 出 CFRunLoopRun ---
```

每一次 callout 一对 push/pop。observer 回调是一次，timer 回调是一次，source 回调也是一次。不是每轮迭代一个池，是每次回调一个池。

token 地址每次都一样，因为上一次的池已经排空了，哨兵落在同一个槽。

CoreFoundation 为此有一对导出符号：

```text
__CFRunLoopPerCalloutAutoreleasepoolEnabled
__CFRunLoopSetPerCalloutAutoreleasepoolEnabled
```

在 macOS 26.5 上 `_CFRunLoopPerCalloutAutoreleasepoolEnabled()` 返回 1。

### 那两个著名的 observer 还在不在

在，但装不上去了。

先说名字。iOS 18.3 的 UIKitCore 里已经找不到 `_wrapRunLoopWithAutoreleasePoolHandler` 了。今天叫 `_UIApplicationInstallAutoreleasePoolsIfNecessaryForMode`。名字里的 "IfNecessary" 不是客套，反汇编它开头三条指令就明白了：

```asm
__UIApplicationInstallAutoreleasePoolsIfNecessaryForMode:
	mov	w0, #0x1
	bl	__CFRunLoopSetPerCalloutAutoreleasepoolEnabled
	cbnz	w0, 0x919060                 ; 返回非零就直接 ret，什么都不装
```

第一句就是问 CoreFoundation："你自己会做 per-callout 的池吗？"会，那 UIKit 掉头就走。

只有 CF 说不会的时候，才走到下面这段：

```asm
	mov	x0, #0x0                      ; allocator = NULL
	mov	w1, #0x1                      ; activities = kCFRunLoopEntry
	mov	w2, #0x1                      ; repeats = true
	mov	x3, #-0x7fffffff              ; order = -2147483647
	bl	_CFRunLoopObserverCreate
	...
	mov	w1, #0xa0                     ; activities = BeforeWaiting | Exit
	mov	w2, #0x1
	mov	w3, #0x7fffffff               ; order = 2147483647
	bl	_CFRunLoopObserverCreate
	...
	ldr	x22, [x8]                     ; _kCFRunLoopCommonModes
	bl	_CFRunLoopAddObserver
	bl	_CFRunLoopAddObserver
```

流传的那几个数字一个不差：activities `0x1` 配 order `-2147483647`，activities `0xa0`（`kCFRunLoopBeforeWaiting | kCFRunLoopExit`）配 order `2147483647`，都加到 `kCFRunLoopCommonModes`。order 取两端极值，是为了让第一个 observer 排在所有 observer 之前、第二个排在所有之后，把整轮迭代包住。

这套讲法当年完全正确。今天它是一条 fallback 分支，前面那句 `cbnz` 把它跳过去了。

> 待模拟器/真机补测：本节 UIKit 部分是对 iOS 18.3 模拟器运行时里的 UIKitCore 做静态反汇编，没有实际跑起来。要确认的是 `_CFRunLoopPerCalloutAutoreleasepoolEnabled()` 在 iOS 上也返回 1、以及 `CFRunLoopGetMain()` 的 observer 列表里确实找不到那两个 order 为 ±2147483647 的 observer。做法：在一个空白 iOS 工程的 `application:didFinishLaunchingWithOptions:` 里 `extern` 声明那个函数打印返回值，再 `NSLog(@"%@", (__bridge id)CFRunLoopGetMain())` 把 observer 列表整个打出来数。

### 结论对使用者变了吗

基本没变，但边界更紧了。以前的说法是"autoreleased 对象最迟活到本轮 RunLoop 结束"，现在是"最迟活到当前这次回调结束"。你在 `viewDidLoad` 里拿到的 autoreleased 对象，到 `viewWillAppear` 未必还活着，因为这两个回调之间可能已经隔了好几次 callout、好几次 pop。

网上那个经典的"用 `__weak` 引用一个 autoreleased 对象，在 `viewWillAppear` 里发现它还在"的小实验，结论建立在同一个池上。这个前提今天需要重新验证。

顺带验一条相关的老说法。sunnyxx 写过容器的 block 版枚举器内部会加池：

```text
--- for-in ---
--- enumerateObjectsUsingBlock: ---
    {PUSH  token=0xcbe800040   调用方= _CFAutoreleasePoolPush  (CoreFoundation)}
    {POP   token=0xcbe800040   调用方= _CFAutoreleasePoolPop   (CoreFoundation)}
--- 完 ---
```

确实加，但整个枚举只加一个，不是每个元素一个。所以"用 block 枚举器就不用在循环里加池了"这个推论不成立。元素多的时候，池还是要你自己在 block 里面加。

---

## 八、几个不准的说法

「AutoreleasePoolPage 一页 4096 字节，因为虚拟内存一页就是 4096。」 前半句对，后半句在 Apple Silicon 上早就不成立了，系统页是 16384。4096 来自 `PAGE_MIN_SIZE`，四个池页共用一个系统页。

「autorelease 就是把对象地址写进页里。」 最新的那个不在页里，在 TLS 里，要等下一次 autorelease 或者一次 push/pop 才挪进去。

「`@autoreleasepool` 是异常安全的。」 反了。规范原文是 "When the block is exited with an exception, the pool is not drained."，编译产物里 landingpad 也确实没有 pop。`return` 和 `break` 倒是处理得好好的。

「`NSAutoreleasePool` 和 `@autoreleasepool` 是两套实现。」 这是我自己在第 4 篇里写的，收回。`-init` 调 `_CFAutoreleasePoolPush`，`-drain` 调 `_CFAutoreleasePoolPop`，最终是同一对运行时函数、同一条页链表。它只是一个 40 字节的壳。

「子线程不建池，autoreleased 对象会泄漏，并且会打 warning。」 默认既不打 warning 也不泄漏，线程退出时 TLS 析构器会清干净。要 warning 得开 `OBJC_DEBUG_MISSING_POOLS`。真正的代价是"要等到线程结束"。

「主线程 RunLoop 靠两个 observer 管池，进入 push、休眠 pop 再 push、退出 pop。」 那两个 observer 的参数一字不差，但 UIKit 现在先问 CoreFoundation 支不支持 per-callout 池，支持就一个都不装。macOS 上实测是每次 callout 一对 push/pop。

「循环里加 `@autoreleasepool` 一定能降峰值。」 看循环体。ARC 下调定参 Foundation 工厂方法，实测差值 0.05 MB；换成 `dataWithLength:` 的 MRC 版本，差 1 GB。

「`PROTECT_AUTORELEASEPOOL` 在发布版 objc4 里没有定义。」 也是第 4 篇里我自己写的。它定义了，值是 0。

---

## 总结

池是一条 4096 字节页组成的双向链表，哨兵是 `nil`，push 返回哨兵槽的地址，pop 按地址把栈弹回去。这部分二十年没变，中文资料讲得也对。

变了的是外围。最新一次 autorelease 的对象扣在 TLS 里不进页，所以你永远数不满 505。ARC 下大量临时对象靠尾调用加返回地址比对完成交接，根本不进池，循环里加池的收益因此从"必然"变成"看循环体"。主线程的池今天由 CoreFoundation 按 callout 粒度管，UIKit 那两个 observer 只在 CF 不支持时才装。

有三条我本来当常识、根本没打算验的说法，验完发现全是错的：`@autoreleasepool` 遇异常不排空，子线程不建池不会泄漏，`NSAutoreleasePool` 走的是同一套实现。

方法论还是那条，这次多一个注脚。第六节那张表九行数据整整齐齐，我据此得出"变参破坏返回值优化"，补了个对照实验，当场推翻。**相关不是原因，样本再整齐也不是。** 那个对照实验只花了三分钟。

下一篇 [[iOS RunLoop：mode、source 与那张流程图今天还对不对]] 讲 RunLoop 本身。

## 参考资料

### 源码与规范

- [apple-oss-distributions/objc4](https://github.com/apple-oss-distributions/objc4)：`NSObject.mm` 里 `AutoreleasePoolPage` 全部实现，`NSObject-internal.h` 里 `AutoreleasePoolPageData` 和 `PROTECT_AUTORELEASEPOOL`，`objc-object.h` 里 `rootAutorelease` / `prepareOptimizedReturn` / `acceptOptimizedReturn`。当前最新 tag 是 objc4-951.7，比 macOS 26.5 上跑的那份旧，第二节末尾那个 entry 布局的出入就是这么来的
- [Clang ARC Specification](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)：`@autoreleasepool` 一节写明了异常退出不排空
- [Apple — Using Autorelease Pool Blocks](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmAutoreleasePools.html)：三种该自己建池的场合，以及每线程一个池栈的官方措辞
- [apple-oss-distributions/CF](https://github.com/apple-oss-distributions/CF)：公开的这份 `CFRunLoop.c` 停在 2015 年，里面一处 autorelease 都没有，和今天 CoreFoundation 的行为已经对不上

### 经典

- [sunnyxx — 黑幕背后的 Autorelease](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/)：双向链表、哨兵、TLS 优化的中文源头。页结构那部分至今准确，返回值优化那部分讲的是已经退役的读汇编方案
- [draven — 自动释放池的前世今生](https://draven.co/autoreleasepool/)
- [Mike Ash — Let's Build NSAutoreleasePool](https://www.mikeash.com/pyblog/friday-qa-2011-09-02-lets-build-nsautoreleasepool.html)：自己造一个池，用来理解 page/stack 的思路，别当成 Apple 当前实现

### 本地

- [[iOS 内存：MRC 的所有权规则#五、autorelease 池的实现|MRC 的所有权规则]]
- [[iOS 内存：ARC 的两半#返回值的所有权交接|ARC 的两半]]
- [[iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint]]
- [[iOS RunLoop：mode、source 与那张流程图今天还对不对]]

---

实验环境：macOS 26.5.2，Apple Silicon（arm64），Xcode 26 自带 clang。全部 macOS 原生编译运行，没开模拟器：

```shell
clang -fobjc-arc     -framework Foundation -o out prog.m && ./out
clang -fno-objc-arc  -framework Foundation -o out prog.m && ./out    # MRC 对照
clang -fobjc-arc -S -emit-llvm -O0 -o out.ll prog.m                  # 看 IR
```

读页头用的是照着 `AutoreleasePoolPageData` 手抄的一份等价 struct，配合 `objc_autoreleasePoolPush()` 返回的 token 反推页首（`token & ~4095`）。注意全新线程上第一次 push 返回的是 `0x1`，直接拿去做位运算会当场段错误，我就是这么发现占位符的。

interpose 那套用 `__DATA,__interpose` 段加 `DYLD_INSERT_LIBRARIES` 注入自己编的 dylib，在回调里 `dladdr(__builtin_return_address(0))` 打调用方符号。只对自己编的二进制有效，系统二进制被 SIP 拦着。

UIKit 那段是对 `/Library/Developer/CoreSimulator/.../iOS 18.3.simruntime/.../UIKitCore` 做 `nm -n` 加 `otool -arch arm64 -tV -p` 静态反汇编，只读文件，没有 boot 任何模拟器。
