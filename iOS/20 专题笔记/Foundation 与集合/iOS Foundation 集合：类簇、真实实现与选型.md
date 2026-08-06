---
title: 【iOS】Foundation 集合：类簇、真实实现与选型
published: 2026-07-27
description: 给一个六万四千元素的可变数组发 copy，耗时和一千个元素时一样，都是 50 纳秒。Foundation 的容器早就是写时复制了。外加一条我差点写进正文的错误结论：头插比尾插快，是测量假象。
tags:
  - iOS
  - Objective-C
  - Foundation
  - 数据结构
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 29
draft: true
---
# Foundation 集合：类簇、真实实现与选型

先看三行输出，都是这台 Mac 上刚跑出来的：

```text
@[@1, @2]                     class = NSConstantArray       malloc_size = 0
[[NSArray alloc] init]        class = __NSArray0            malloc_size = 0
[NSArray alloc]               class = __NSPlaceholderArray
```

没有一行符合流传的说法。数组字面量的真实类不是 `__NSArrayI`。它根本不在堆上。`[[NSArray alloc] init]` 返回的是一个进程级单例。`alloc` 单独拿出来看，得到的是一个叫"占位符"的东西。

这篇把 `NSArray` / `NSDictionary` / `NSSet` 挖到能拿数字说话的程度。最值钱的两条在第三节和第四节。一条是可变数组的存储是个真正的环，另一条是 `copy` 早就成了写时复制。第九节的选型建议，每条都挂着前面某个实验的数字。

类簇和 `isKindOfClass:` 的关系、tagged pointer 的位布局，在 [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]] 里讲过。`copy` 修饰容器属性引发的那个必然崩溃，在 [[iOS 内存管理：从 MRC、ARC 到属性关键字#第三部分：属性关键字：从所有权推导，而不是从类型名猜|属性关键字：从所有权推导，而不是从类型名猜]] 里。两处只引用，不重讲。

---

## 一、真实类名摊开是什么样

`object_getClassName` 把类簇的伪装全部剥掉。把各种构造方式跑一遍：

| 构造方式 | 实际类 | 备注 |
|---|---|---|
| `[NSArray array]` / `@[]` / `[[NSArray alloc] init]` / `[NSArray new]` | `__NSArray0` | 全进程一个单例 |
| `@[@1, @2]`，元素全是编译期常量 | `NSConstantArray` | 在 `__DATA` 段里，不在堆上 |
| `@[@(x), @2]`，含运行期变量 | `__NSArrayI` | 退回普通不可变数组 |
| `arrayWithObject:` / `initWithObjects:count:1` | `__NSSingleObjectArrayI` | 单元素特化 |
| `arrayWithObjects:` 两个以上 | `__NSArrayI` | 元素内联在对象尾部 |
| `subarrayWithRange:` | `__NSArrayI_Transfer` | |
| `[NSMutableArray array]`，capacity 多少都一样 | `__NSArrayM` | |
| `[可变数组 copy]`，元素 ≤ 5 | `__NSArray0` / `__NSSingleObjectArrayI` / `__NSArrayI` | 真的复制了 |
| `[可变数组 copy]`，元素 ≥ 6 | `__NSFrozenArrayM` | 和原数组共用缓冲区 |

字典是同一套路数：`__NSDictionary0`（单例）、`__NSSingleEntryDictionaryI`、`__NSDictionaryI`、`__NSDictionaryM`、`__NSFrozenDictionaryM`。集合少一个空单例类。`[NSSet set]` 给的是 `__NSSetI`，但同样是单例。`NSOrderedSet` 另有一套。

### 空容器是真单例

同时持有六个来源不同的空数组，不给地址复用留机会：

```text
NSArray  空: 0x1fbab6b80 ×6  全等=1
NSDict   空: 0x1fbab6bb0 ×4  全等=1
NSSet    空: 0x1016437e0 ×3  全等=1
```

前两个的地址在 `0x1fb...`。`objc_getClass("__NSArray0")` 返回的类对象是 `0x1fa35a798`，同一个区间。而这个进程自己的堆对象在 `0x101...`。`malloc_size` 是 0。所以这两个单例住在 dyld 共享缓存里，跟着 Foundation 一起映射进来，不属于任何一次分配。

`NSSet` 那个不一样。地址落在普通堆区，运行期第一次需要时才建出来。

我第一版把"三次调用地址相同"当成了证据。这是错的：前一个对象释放之后地址被复用，也会给出同样的结果。改成同时持有再比对，结论才站得住。

### alloc 拿到的是占位符

```text
[NSArray alloc]           class = __NSPlaceholderArray       0x1fb9c9600
[NSArray alloc] 再来一次   class = __NSPlaceholderArray       0x1fb9c9600
[NSMutableArray alloc]    class = __NSPlaceholderArray       0x1fb9c9610
[NSDictionary alloc]      class = __NSPlaceholderDictionary   0x1fb9d1348
[NSString alloc]          class = NSPlaceholderString         0x1fbbc44e0
[NSNumber alloc]          class = NSPlaceholderNumber         0x1fbbc4840
```

两次 `[NSArray alloc]` 拿到同一个地址。这个地址也在共享缓存区间。可变版本的占位符是相邻的另一个对象，两者差 16 字节。

类簇的构造分两步。`alloc` 只回一个不占内存的哨兵，真正决定用哪个类是 `init` 系列方法的事。所以 `[[NSArray alloc] init]` 直接命中单例，而 `initWithObjects:count:3` 才去堆上开一个 `__NSArrayI`。

自定义类不走这套。`[NSObject alloc]` 和 `[Probe alloc]` 拿到的就是各自的实例。

### 这张表能拿来干什么

不能拿来写代码。私有类名会变。`__NSFrozenArrayM` 和 `NSConstantArray` 都是近几年才出现的，五年前的文章里一个都没有。

它有用的地方是解释现象。`copy` 一个可变数组拿到的对象，类型声明还写着 `NSMutableArray *`，一发 `addObject:` 就崩。看到 `__NSSingleObjectArrayI` 这个类名，因果链就闭合了。

> 类名和空单例这两件事只在 macOS 上验过，未在 iOS 上复核。Foundation 在两个平台上是同一份代码库的不同构建，但类簇的具体分派属于纯粹的实现细节，跨平台一致没有任何保证。

---

## 二、`@[@1, @2]` 为什么不在堆上

`NSConstantArray` 在中文资料里几乎搜不到。来源在编译产物里最清楚：

```text
$ nm -m f2.o | grep -i constant
    (undefined) external _OBJC_CLASS_$_NSConstantArray
    (undefined) external _OBJC_CLASS_$_NSConstantDictionary
    (undefined) external _OBJC_CLASS_$_NSConstantIntegerNumber
    (undefined) external ___CFConstantStringClassReference

$ otool -l f2.o | grep sectname
    sectname __objc_intobj      segname __DATA
    sectname __objc_arrayobj    segname __DATA
    sectname __cfstring         segname __DATA
    sectname __objc_dictobj     segname __DATA
```

`__objc_arrayobj` 和 `__cfstring` 是并列关系。常量字符串靠 `__cfstring` 段加一个外部类符号，这套实现有几十年了。现在数组、字典和整数 `NSNumber` 用的是同一招。

段里那 24 个字节直接读得出来：

```text
Contents of (__DATA,__objc_arrayobj) section
000000a0  00 00 00 00 00 00 00 00  02 00 00 00 00 00 00 00  |................|
000000b0  00 00 00 00 00 00 00 00                           |........|
```

isa 槽留空等链接器填 `NSConstantArray`，第二个字是元素个数 2，第三个字是指向元素数组的指针。运行期读出来完全对得上：

```text
isa = NSConstantArray   word1 = 3 (count)   word2 = 0x104878170
  elem[0] = 0x104878110  == @1 的地址?  1
  elem[1] = 0x104878128  == @2 的地址?  1
  elem[2] = 0x104878140  == @3 的地址?  1
malloc_size = 0
```

元素本身也是 `__objc_intobj` 段里的 `NSConstantIntegerNumber`。整条链上没有一次堆分配，也没有一次 `objc_msgSend`。

这件事需要和前面某一篇对账。[[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]] 里测出 `NSNumber(42)` 是 tagged pointer，用的是 `[NSNumber numberWithInt:42]`。写成字面量就不一样了：

```text
@42                          class = NSConstantIntegerNumber   bit63 = 0
[NSNumber numberWithInt:42]  class = __NSCFNumber              bit63 = 1
```

两条结论都对，差别在构造方式。前一篇没有写错，但"`@42` 是 tagged pointer"这句话今天需要加限定。

我换了五档 `-mmacosx-version-min`，从 12.0 到最新；又换成 iOS 15、17、18 的模拟器 target。三个 `NSConstant*` 符号都照样生成。所以这既不是 macOS 专属，也不是新 SDK 才打开的开关。它就是当前 clang 的默认行为。真正的门槛在运行时那一侧：旧系统的 Foundation 里没有这几个类，链接会失败。具体从哪个系统版本开始可用，我没有查到权威说明。

---

## 三、可变数组底层是一个环

"NSMutableArray 底层是数组还是链表"这个问题问错了。两个都不是。

### 先看形状

在一个 20 万元素的数组的第 p 位插入一个元素，测单次耗时：

```text
p =   0% (index 0)          8.8 ns
p =   5% (index 10000)    1263.1 ns
p =  10% (index 20000)    3466.7 ns
p =  25% (index 50000)    8507.0 ns
p =  40% (index 80000)   13947.5 ns
p =  50% (index 100000)  17352.1 ns
p =  60% (index 120000)  14202.4 ns
p =  75% (index 150000)   8853.9 ns
p =  90% (index 180000)   3459.3 ns
p =  95% (index 190000)   1283.1 ns
p = 100% (index 200000)    138.4 ns
```

一个对称的帐篷。5% 和 95% 是 1263 对 1283，25% 和 75% 是 8507 对 8854。中点最贵。两端比中点便宜三个数量级。

普通动态数组的曲线应该单调递增。越靠前插，要往后挪的元素越多。可这条曲线在中点折了回来。实现每次只挪较短的那一侧。

### 把内存读出来

`__NSArrayM` 一共 48 字节。把前六个字打出来，对着各种操作看哪个字段在动，布局就出来了：

```c
typedef struct {
    uintptr_t isa;
    uintptr_t cow;          // 平时是 0，第四节会讲它什么时候变非空
    uintptr_t buffer;       // 指向一块单独 malloc 的缓冲区
    uint32_t  offset;       // 第 0 个元素在缓冲区里的下标
    uint32_t  capacity;     // 缓冲区能放几个
    uint32_t  mutations;    // 每次结构性修改 +1
    uint32_t  count;
} __NSArrayM_layout;   // 照实测结果手写的，不是官方定义
```

对着这个布局跑几步操作：

```text
6 个（尾插）              offset=0 cap=6 count=6 mut=7
  removeObjectAtIndex:0   offset=1 cap=6 count=5 mut=8
  removeLastObject        offset=1 cap=6 count=4 mut=9
  insert@0                offset=0 cap=6 count=5 mut=10
```

删掉头元素只是把 `offset` 加一。一个字节都没搬。再往头部插入，`offset` 又减回去。

### 它真的会绕

`offset` 减到 0 之后再往头部插，会发生什么？把缓冲区的每个槽位直接打出来：

```text
尾插 1,2,3,4     offset=0 cap=6 used=4  槽位:  1 2 3 4 . .
头插 9           offset=5 cap=6 used=5  槽位:  1 2 3 4 . 9
头插 8           offset=4 cap=6 used=6  槽位:  1 2 3 4 8 9
逻辑顺序: 8,9,1,2,3,4
```

`offset` 从 0 绕到了 5。元素在缓冲区里断成两段，逻辑顺序靠取模算出来。**`__NSArrayM` 是环形缓冲区，头尾两端的增删都是 O(1)，中间是 O(n)。**

### 读 `CFArray.c` 是在读另一个类

开源的 CoreFoundation 里，可变数组确实是双端队列。但它不是环形的：

```c
struct __CFArrayDeque {
    uintptr_t _leftIdx;
    uintptr_t _capacity;
    /* struct __CFArrayBucket buckets follow here */
};
```

元素永远连续地放在 `[_leftIdx, _leftIdx + count)` 里。左边顶到头就得重新分配并居中，源码注释写的是 re-center everything。`__NSArrayM` 不需要居中，绕一圈就行。设计思想一致，数据结构不同。

我一开始想当然地把 CF 的 deque 当成 `__NSArrayM` 的实现，是看了自己打出来的槽位才发现不对。

这个坑 Ciechanowski 2013 年就踩过并且写下来了，他的措辞比我狠得多（基线是 iOS 7.0 SDK）：

> I was shocked to realize neither NSArray nor NSMutableArray have anything in common with CFArray.

> Basically, CFArray moves the memory around to accommodate the changes in the most efficient fashion, similarly to how `__NSArrayM` does its job. However, the CFArray does not use a circular buffer! Instead it has a larger buffer padded with zeros from both ends… Adding elements at either end simply eats up the remaining padding.

> What those two have in common? They're the concrete implementation of abstract data type known as deque.

抽象数据类型相同，内存布局不同。所以拿 `CFArray.c` 讲 `NSMutableArray`，讲复杂度能讲对，讲布局全错。十三年过去，这条边界没有变。

更彻底的一个证据：在 macOS 上你根本拿不到那份代码。

```text
CFArrayCreateMutable(NULL, 0, &kCFTypeArrayCallBacks)  -> __NSArrayM
CFArrayCreateMutable(NULL, 0, NULL)                    -> __NSCFArray
CFDictionaryCreateMutable(..., 标准回调)                -> __NSDictionaryM
CFDictionaryCreateMutable(NULL, 0, NULL, NULL)         -> __NSCFDictionary
```

用标准回调调 CF 的构造函数，回来的是 Foundation 的类。只有把 callbacks 传成 NULL、明确表示"我要装的不是对象"，才会落到真正的 CF 实现上。拿 `CFArray.c` 解释 `[NSMutableArray array]` 的行为，读的是另一个类的代码。

不过"挪较短那一侧"这条 CF 里写得非常直白，值得抄原文。`A` 是插入点左边的元素数，`C` 是右边的：

```c
if ((numNewElems < 0 && C < A) || (numNewElems <= R && C < A)) {	// move C
    // deleting: C is smaller
    // inserting: C is smaller and R has room
    ...
} else if ((numNewElems < 0) || (numNewElems <= L && A <= C)) {	// move A
    // deleting: A is smaller or equal (covers remaining delete cases)
    // inserting: A is smaller and L has room
```

上面那个对称的帐篷，就是这两个分支画出来的。

### 扩容倍率是黄金分割

跑到一百五十万个元素，把每次容量跃迁和相邻比值都记下来：

```text
cap 2 -> 4         2.000
cap 4 -> 6         1.500
cap 6 -> 10        1.667
cap 10 -> 16       1.600
cap 16 -> 28       1.750
cap 28 -> 48       1.714
cap 48 -> 80       1.667
cap 80 -> 160      2.000
cap 160 -> 320     2.000
   ...（一路 2.000 到 2560）
cap 2560 -> 6144   2.400
cap 6144 -> 10240  1.667
cap 10240 -> 18432 1.800
cap 18432 -> 30720 1.667
cap 30720 -> 51200 1.667
cap 51200 -> 83968 1.640
cap 83968 -> 137216   1.634
cap 137216 -> 223232  1.627
cap 223232 -> 362496  1.624
cap 362496 -> 587776  1.621
cap 587776 -> 952320  1.620
cap 952320 -> 1542144 1.619
```

1.640、1.634、1.627、1.624、1.621、1.620、1.619。单调地收敛到 φ = 1.61803。

只跑到几千个元素就停手，会看到 80 到 2560 那一段稳定的 2.000，然后写下"小容量 1.6 倍、大容量翻倍"。那是假的。容量乘 8 就是缓冲区字节数，缓冲区是 `malloc` 出来的。中间那段整齐的 2.000，是分配器的尺寸类把真实倍率量化掉了。

这个猜想可以直接验：

```
malloc_good_size((size_t)(旧容量 × 1.618) × 8) == 新容量 × 8
```

从 2 一路跑到 250 万，25 次跃迁，25 次全中：

```text
2        -> 4          malloc_good_size(3*8)       = 32         新容量*8 = 32         中
10       -> 16         malloc_good_size(16*8)      = 128        新容量*8 = 128        中
48       -> 80         malloc_good_size(77*8)      = 640        新容量*8 = 640        中
2560     -> 6144       malloc_good_size(4142*8)    = 49152      新容量*8 = 49152      中
137216   -> 223232     malloc_good_size(222015*8)  = 1785856    新容量*8 = 1785856    中
1542144  -> 2496512    malloc_good_size(2495188*8) = 19972096   新容量*8 = 19972096   中
                                                              （共 25/25）
```

所以真实策略只有一句：乘 1.618，然后交给 `malloc_good_size` 取整。那些看起来不是 1.618 的比值，全部是分配器尺寸类的量化结果。

### 换个架构再跑一遍

上面那句结论有个没扫过的维度：架构。整组实验都在 arm64 上跑，而假说本身就说"看到的比值取决于分配器"。不同架构的 `malloc` 尺寸类不一样，那序列就该跟着变。

换 `-target x86_64-apple-macos14` 走 Rosetta 重跑：

```text
arm64    2 4 6 10 16 28 48 80 160 320 640 1280 2560 6144 ...
x86_64   2 4 6 10 16 26 42 68 110 192 320 576  960 1600 2624 8192 ...
```

前五档一模一样，第六档就分了。更关键的是 arm64 上 80 到 2560 那一段整齐的 2.000，在 x86_64 上根本不存在，那边的比值是 1.625、1.615、1.619、1.618，一路贴着 φ。

同一份 Foundation 代码，在两台机器上能量出两个不同的"扩容倍率"。上面那句"整齐的 2.000 是尺寸类造成的假象"，到这里才算真的被证明，而不只是一个说得通的解释。

`malloc_good_size` 那个断言在 x86_64 上同样全中。两个架构合起来 41 步跃迁，一步不差。顺手把别的候选倍率也扫了：1.5 在 `16->28` 就崩，1.6 在 `80->160` 崩，1.65 在 `51200->83968` 崩，1.75 和 2.0 在第二步就崩。1.618 和 1.625 全程都对得上，`malloc` 的取整把这两个值之间的差别吃掉了，这组数据分辨不出来。

这一点顺带裁决了一桩旧公案。Ciechanowski 2013 年那篇（iOS 7 基线）说 1.625，老青菜 2020 年那篇说 2 倍。两边都真跑过，都没错，只是各自停在了自己那台机器的量化结果上。老青菜的样本只取到 `size 2->4`，那一档本来就是 2.000。

φ 是扩容倍率的经典最优解。小于 φ 时，历次释放的旧缓冲区之和迟早能装下新的一块，分配器可以复用。大于 φ 就永远追不上。C++ 社区为 1.5 还是 2 吵了很多年。Foundation 直接取了那条分界线。

头部插入产生的容量序列和尾部插入完全一样。扩容策略与从哪端插无关。缩容也是有的。塞满 4000 个（capacity 6144）再删到只剩 10 个，capacity 变成 12。`removeAllObjects` 之后 capacity 归零，缓冲区指针变成 NULL。

### 一次差点写进正文的错误结论

我第一版的测法是 `addObject:` 循环 N 次对比 `insertObject:atIndex:0` 循环 N 次。跑了六轮、五个规模，结果稳得不能再稳：

```text
N=80000   tail(addObject) = 0.931 ms   head(insert@0) = 0.612 ms   ×0.66
```

头插比尾插快三分之一。数据稳，样本够。我当时已经准备为它编一套解释了。换成同一个选择器再测：

```text
M=80000   insert@0 = 0.656 ms   insert@count = 0.612 ms   addObject = 0.890 ms
```

位置根本没有区别。`addObject:` 比 `insertObject:atIndex:` 慢 45%，那才是差值的来源。两个方法都是 O(1)。差的是方法本身的常数开销。换执行顺序重跑八轮，结论没变。

这里还有一层更值得记的教训。发现数据可疑之后，我第一批想到的补救全是在改测量的调度：固定顺序改成轮换顺序、再改成完全随机顺序、把中位数换成最小值，每种 31 轮。四种设计跑下来，那个 25% 的差距一次不落地全复现了。

因为混淆项根本不在调度里，它在循环体内部。尾插那行写的是 `[a insertObject:o atIndex:a.count]`，每轮比头插多发一次 `count` 消息。把它换成本地变量计数，差距当场归零：

```text
insert@0                    13.46 ns
insert@本地计数（尾插）       12.89 ns      头/尾 = 1.04x
insert@a.count（问 count）   19.39 ns
insert@0 + 白问一次 count    19.48 ns      ← 和上一行一致
```

最后那行是关键：给头插也补上一次白问的 `count`，它立刻和尾插一样慢。那 6.5 纳秒是消息发送的价钱，跟插到哪一端没有关系。

换顺序、加轮数、换统计量，这些都是在防噪声。噪声和系统性偏差是两回事，后者你跑一万轮也照样得到同一个错的数。

规范里那句"遇到结果和预期不符，先怀疑仪器"，这次要反过来用。结果和预期相符（网上都说头插慢），反而更该怀疑仪器。符合直觉的错误结论比违背直觉的更难被自己抓住。

---

## 四、copy 是写时复制

这一节是全篇最有实用价值的部分。

给一个可变数组发 `copy`，耗时随元素数怎么变：

```text
N          [m copy]    [im copy]   [m mutableCopy]   [NSArray arrayWithArray:m]
1000          84 ns       15 ns          53 ns              1941 ns
4000          56 ns       14 ns          57 ns              7813 ns
16000         52 ns       14 ns          42 ns             44725 ns
64000         49 ns       13 ns          43 ns            145962 ns
```

**`copy` 和 `mutableCopy` 的耗时与元素数无关，六万四千个元素也是 50 纳秒；`arrayWithArray:` 才是老老实实的 O(n)。**

内存布局对得上：

```text
n=5   -> __NSArrayI          与原缓冲区共享? 0
n=6   -> __NSFrozenArrayM    与原缓冲区共享? 1
n=50  -> __NSFrozenArrayM    与原缓冲区共享? 1
```

六个元素以上，`copy` 出来的 `__NSFrozenArrayM` 和原数组指向同一块缓冲区。五个及以下走真复制，大概是因为对小数组来说，共享的簿记成本比直接拷几个指针还贵。

那份复制什么时候发生？下一次写的时候。

```text
N=10000    copy 过再改，第一次 addObject =  15692 ns    没 copy 过 = 150 ns
N=40000    copy 过再改，第一次 addObject =  41952 ns    没 copy 过 =  77 ns
N=160000   copy 过再改，第一次 addObject = 184758 ns    没 copy 过 = 235 ns
```

第一次写要付一次完整的缓冲区复制。往后恢复正常。原数组换到新缓冲区，冻结的那份继续持有旧的：

```text
copy 前        m 缓冲区=0x104eb3a90   froz 缓冲区=0x104eb3a90   共享=1
m addObject 后  m 缓冲区=0x104eb3d10   froz 缓冲区=0x104eb3a90   共享=0
froz.count=50  m.count=51
```

第三节那个平时恒为 0 的 `cow` 字段，就是在这里派上用场的：

```text
copy 前     m.cow = 0x0            m.buf = 0x1035ee4e0
copy 后     m.cow = 0x1035edcc0    m.buf = 0x1035ee4e0
            froz.cow = 0x1035edcc0 froz.buf = 0x1035ee4e0
写之后      m.cow = 0x0            m.buf = 0x1035ee760
            froz.cow = 0x1035edcc0 froz.buf = 0x1035ee4e0
```

`copy` 的瞬间，两个对象的 `cow` 指向同一个 16 字节的小对象，缓冲区也是同一块。原数组一写，它的 `cow` 归零、换到新缓冲区，冻结的那份原地不动。这个字段就是"这块缓冲区还有别人在用"的标记。`mutableCopy` 也会给双方各插一份。

`mutableCopy` 同样是写时复制，两个方向都触发。改副本，副本换缓冲区；改原件，原件换缓冲区。字典和集合走同一套。`[dict copy]` 在一万六千项时还是 60 纳秒，共用内部指针，类名是 `__NSFrozenDictionaryM` 和 `__NSFrozenSetM`。

`arrayWithArray:` 和 `initWithArray:` 不参与这个机制，它们造出来的是 `__NSArrayI`，缓冲区独立。

所以"`copy` 是浅拷贝，会新建一个容器并把元素指针都拷一遍"今天只对小容器成立。对大容器它是常数时间，代价推迟到下一次写入。这条对 API 设计有直接影响。对外返回集合快照，用 `copy`，别用 `arrayWithArray:`。调用方只读的话，那次复制永远不会发生。

---

## 五、字典的 key 为什么必须 copy

拿一个 `NSMutableString` 当 key，塞完再改它：

```text
放进去的 key      0x9a4808180          class = __NSCFString
字典里存的 key    0xbab2e830f2b3b2f9   class = NSTaggedPointerString
同一个对象? 0

改完 mk 之后:
  mk 现在是 "key-changed"
  字典里的 key 还是 "key"
  d3[mk]      = (null)
  d3[@"key"]  = value
```

字典存的是一份 copy。短字符串的 copy 命中了 tagged pointer，连类名都变了。原对象改成什么样，那把钥匙都不受影响。

这个行为是 Foundation 加的，CF 不这么干。`CFDictionary.c` 里默认的 key 回调是 retain：

```c
const CFDictionaryKeyCallBacks kCFTypeDictionaryKeyCallBacks =
    {0, __CFTypeCollectionRetain, __CFTypeCollectionRelease, CFCopyDescription, CFEqual, CFHash};
const CFDictionaryKeyCallBacks kCFCopyStringDictionaryKeyCallBacks =
    {0, __CFStringCollectionCopy, __CFTypeCollectionRelease, CFCopyDescription, CFEqual, CFHash};
```

要 copy 得显式挑第二套。所以"字典会 copy key"准确的说法是：`NSDictionary` 这么做，`CFDictionary` 默认不做，`NSMapTable` 也不做。实测 `NSMapTable` 存进去的 key 就是传进去的那个对象。

这条差别有实际后果。直接用 `CFDictionarySetValue` 往一个 `CFMutableDictionaryRef` 里塞 key，默认只 retain，你得自己保证那个 key 不会被改。同一个字典对象桥接成 `NSMutableDictionary`、改走 `setObject:forKey:`，语义又变回 copy。两套 API 两种契约。

至于这次 copy 贵不贵，官方在 `NSCopying` 协议页给了明确指引：

> Implement `NSCopying` by retaining the original instead of creating a new copy when the class and its contents are immutable.

绝大多数 key 是字符串常量或者不可变字符串，这次 copy 退化成一次 retain，一个字节都不复制。所以"字典要拷贝 key"在性能上基本不用操心。真正要操心的是下一节那件事：你拿可变对象当 key。

### copy 只保护到这一层

自定义类当 key，copy 挡不住 hash 漂移：

```text
key 被 copy 了吗: 存的 0x9a4817a40  vs  传的 0x9a4817a30   同一对象=0
改掉原对象 k 之后 d4[k] = (null)
用一个 v=7 的新对象取 = seven
```

字典确实存了副本，但你手里那个 `k` 已经不等于副本了，拿它取当然取不到。

真要命的是 `NSSet`，它不 copy 元素：

```text
加入前 contains = 1
改掉参与 hash 的字段之后:
  count = 1（元素还在）
  containsObject: 自己 = 0
  removeObject: 之后 count = 1
```

集合里有这个对象，却找不到它。也删不掉它。这个元素会一直挂在那儿，直到集合本身销毁。做过 `NSSet` 元素或 `NSDictionary` key 的对象，参与 `hash` 的字段就不能再改了。

### hash 和 isEqual: 的契约

只重写 `isEqual:`：

```text
b1 isEqual b2 = 1
b1.hash = 4353072496
b2.hash = 4353073008   相等 = 0
两个「相等」的对象放进 NSSet，count = 2
[s containsObject:一个新造的同名对象] = 0
```

`NSObject` 的默认 `hash` 返回的就是对象地址，实测 `o.hash == (uintptr_t)o`。于是两个内容相等的对象哈希不等，落进不同的桶。去重和查找全失效。

`NSArray` 不受影响。`containsObject:` 和 `indexOfObject:` 只用 `isEqual:` 线性扫，都能找到。这是数组和集合表现分裂的根源。

还有个别处没见人提的细节。没实现 `NSCopying` 的类直接当字典 key，抛的是：

```text
-[Bad copyWithZone:]: unrecognized selector sent to instance 0x103769d70
```

没有任何"必须实现 NSCopying 协议"的检查，就是字典对 key 发了 `copyWithZone:` 而没人接。

`hash` 写成常数呢？契约没破，去重完全正确，代价在这儿：

```text
2000 个元素，2000 次 containsObject:
  hash 恒为 42:  329.46 ms
  正常 hash:       0.344 ms      慢 958 倍
```

哈希表退化成了链表。这就是"`hash` 要尽量分散"的量化版本。

### 集合自己的 hash 就是元素个数

```text
@[@1,@2].hash      = 2
@{@1:@2}.hash      = 1
NSSet(@1,@2).hash  = 2
```

三个都等于 `count`。这是合法实现。相等的集合，元素数必然相等。但它意味着把数组当字典 key 用时，所有等长的数组会撞进同一个桶，直接落进上面那个 958 倍的局面。

`NSString` 的 hash 是真算过内容的：200 个 `x` 改掉中间一个字符，hash 从 15776882586490664392 变成 9801666139086562761。

---

## 六、枚举：一次抓多少个，以及那个异常怎么抛

`for-in` 背后是 `countByEnumeratingWithState:objects:count:`。手动调它，看每次返回几个：

```text
N=5000  __NSArrayM             每次返回: 5000            共 1 批    itemsPtr==自带缓冲? 0
N=5000  不可变数组              每次返回: 5000            共 1 批    itemsPtr==自带缓冲? 0
N=5000  NSMutableSet           每次返回: 16 16 16 ...    共 313 批  itemsPtr==自带缓冲? 1
N=5000  NSMutableDictionary    每次返回: 16 16 16 ...    共 313 批  itemsPtr==自带缓冲? 1
```

"`for-in` 一次抓 16 个"只对集合和字典成立。数组一次把全部元素交出来，而且 `itemsPtr` 指向自己的缓冲区，调用方栈上那个 16 元素数组根本没用上。原因就是第三节那块连续缓冲区，指针一给就完事。

那绕了圈的数组呢？

```text
offset=5 cap=8 used=7  逻辑内容=102,101,100,0,1,2,3
countByEnumerating 每批: 3 4    共 2 批
```

分成两批。正好是环上断开的两段。同样 7 个元素、没绕圈的数组只要一批。这大概是环形缓冲区能被反推出来的最漂亮的一个侧面证据。

### mutationsPtr 指向哪

```text
对象地址            0x100d0acd0
mutations 字段地址   0x100d0acf0
state.mutationsPtr  0x100d0acf0
mutationsPtr 就是对象里的 mutations 字段? 1
```

可变数组把 `mutationsPtr` 指向自己内部那个计数器。`unsigned long` 是 8 字节。`mutations` 和 `count` 又是相邻的两个 32 位字段。所以这一次读同时盖住了修改次数和元素个数。

不可变数组的做法正好相反：

```text
class = NSConstantArray
mutationsPtr = 0x16f705e70
mutationsPtr 是不是指向 state 自己的 extra[0]? 1
```

指向调用方栈上的 `NSFastEnumerationState.extra[0]`。那块内存除了枚举器自己没人会碰。检测永远不会触发。不可变集合就是这样退出这套机制的。

不过"退出"的具体手法不止一种，换几个不可变类看就知道：

```text
__NSArrayI          mutationsPtr = 0x18e51beb8   *ptr = 1
__NSFrozenArrayM    mutationsPtr = 0x18e3daac0   *ptr = 1
__NSSetI            mutationsPtr = 0x18e3ded38   *ptr = 1
```

三个不同的地址，都落在 `0x18e...` 这段 dyld 共享缓存里，值都是 1。这些类各自指向一个只读常量，而不是调用方的栈。`NSConstantArray` 用栈上的 `extra[0]`，它们用共享缓存里的常量，效果一样：那个值不会变，比对永远通过。

开源 CF 里能看到这个手法的原型：

```c
    case __kCFArrayImmutable:
        if (state->state == ATSTART) {
            static const unsigned long const_mu = 1;
            state->mutationsPtr = (unsigned long *)&const_mu;
```

一个函数级的 `static const`。所以"不可变容器在枚举期间不会抛 mutation 异常"不是因为它没有 mutation 方法可调，而是它压根没接进这套检测。

SDK 里的结构体定义一共五个字段：

```c
typedef struct {
    unsigned long state;
    id __unsafe_unretained _Nullable * _Nullable itemsPtr;
    unsigned long * _Nullable mutationsPtr;
    unsigned long extra[5];
} NSFastEnumerationState;
```

### 检测发生在每次迭代，不是每批

编译出来的 IR 把这件事讲得很清楚。第一批取回来之后把 `*mutationsPtr` 存进 `%14` 当基准，进入循环头 `15:` 每一轮都重读比对：

```llvm
11:
  %13 = load ptr, ptr %12          ; mutationsPtr
  %14 = load i64, ptr %13          ; 基准值
  br label %15

15:                                ; 每次迭代都会回到这里
  %18 = load ptr, ptr %12
  %19 = load i64, ptr %18
  %20 = icmp eq i64 %19, %14
  br i1 %20, label %22, label %21

21:
  call void @objc_enumerationMutation(ptr noundef %8)
```

所以在 40 个元素的数组里第 1 次迭代改一下，第 1 次迭代就抛：

```text
NSGenericException
*** Collection <__NSArrayM: 0x100d0c110> was mutated while being enumerated.
```

但只有一个元素时不抛：

```text
1 个元素时改：没抛异常！
循环结束后 count = 2
```

没有下一次迭代，检查点根本没到。这个边界值得记住：同一段代码，容器里有一个元素时静悄悄跑过去，有两个元素时当场崩。靠测试撞出这类 bug 是撞不出来的。

`replaceObjectAtIndex:` 只改值不改结构，照样计数加一，照样抛。`sortUsingSelector:` 一次加了 6。

### 四种遍历方式的实测

100 万元素，热身后取七轮最小值：

```text
可变数组 下标            8.43 ms     8.4 ns/元素
可变数组 for-in          2.15 ms     2.2 ns/元素
可变数组 block           15.45 ms   15.4 ns/元素
可变数组 NSEnumerator    6.76 ms     6.8 ns/元素
不可变数组 下标           7.29 ms     7.3 ns/元素
不可变数组 for-in         2.14 ms     2.1 ns/元素
```

`for-in` 比下标快接近四倍。一次消息发送换来全部元素的指针，之后就是纯内存访问。block 版本最慢，每个元素一次调用。

需要下标就用 block 版本，不需要就用 `for-in`。这个建议现在有数字了。

---

## 七、弱引用容器的三个坑

`NSPointerArray` / `NSHashTable` / `NSMapTable` 都能装弱引用。五个对象加进去，全部释放，再看：

```text
都活着:    HashTable.count=5   MapTable.count=5   PointerArray.count=5
全部释放后:
  HashTable.count    = 5    allObjects    = 0 个
  MapTable.count     = 5    keyEnumerator = 0 个
  PointerArray.count = 5    allObjects    = 0 个
```

**三个容器的 `count` 在元素被释放之后全都不更新。** 想知道还剩几个活的，只能数 `allObjects` 或者枚举一遍。

`NSPointerArray` 还多一层：`pointerAtIndex:` 会返回 NULL，那些洞留在原地不动。`allObjects` 跳过它们，所以 `count` 和 `allObjects.count` 能差出很远。

官方给的清洞方法是 `compact`。它不工作。

```text
释放后 count = 3
直接 compact 之后 count = 3     ← 期望 0
先 addPointer:NULL 再 compact，count = 0
```

直接调 `compact` 什么也没发生。先 `addPointer:NULL` 再调就正常了。这个绕法社区里传了十年，我在实测里复现了。

那个"短路"猜测有人做过逆向，把标志的名字也挖出来了：

> `-compact` first checks whether an internal flag 'needsCompaction' is set… The only time the flag is set is if a nil pointer is inserted directly into the array through the public interface. It does not get set if a weakly referenced object is deallocated

按这个说法，`replacePointerAtIndex:withPointer:NULL` 也能置位，和 `addPointer:NULL` 等效。还有一份 open radar 记着这件事，编号 15396578。

这段的定性得说清楚：它是逆向推断的二手结论加一个 radar，不是 Apple 承认过的行为，我自己能担保的只有上面那三行输出。Apple 文档对 `compact()` 的全部描述是一句 "Removes `NULL` values from the receiver."，对这个坑一个字都没有。文档在这里的沉默，本身就是不要依赖它的理由。

我第一次测 `NSHashTable` 时踩了坑。对象建在循环里，出了循环体强引用就没了，三个对象在加进去的当场就 dealloc。于是"测出"加了三个之后 count 是 1。弱容器的实验必须先用一个强引用数组把对象钉住，再统一放手。

---

## 八、toll-free bridging 与三种 retainCount 哨兵

`__bridge` 转过去，地址一个字节都不变。

```text
NSArray -> CFArrayRef: 地址相同 = 1   CFArrayGetCount = 3
CFGetTypeID(nsArr) = 19   CFArrayGetTypeID() = 19
```

有意思的是 `NSConstantArray` 和 `__NSArray0` 这两个 Foundation 私有类，`CFGetTypeID` 一样返回 19。桥接不是几个特定类的特权，整个类簇都在里面。

反过来把 `CFMutableArrayRef` 当 `NSMutableArray` 用，`addObject:` 之后 `CFArrayGetCount` 也跟着变。同一个对象。两套 API。

需要留神的是 CF 容器不要求元素是对象。callbacks 传 NULL 就能往里塞裸整数：

```text
存裸整数 1,2,3 -> CFArrayGetCount = 3   取回 [1] = 2
桥回 NSArray: class = __NSCFArray  count = 3
```

`count` 读得出来。可里面的"对象"是 1、2、3 这三个地址，ObjC 侧任何一次 retain 都会立刻崩。桥接的边界在这儿。类型能过去，语义不一定。

### CFGetRetainCount 的三个哨兵值

```text
__NSArrayI                CFGetRetainCount = 2
__NSArrayM                CFGetRetainCount = 2
__NSDictionaryI / M       CFGetRetainCount = 2
__NSArray0（空单例）       CFGetRetainCount = -1
NSConstantArray           CFGetRetainCount = -1
NSConstantIntegerNumber   CFGetRetainCount = -1
__NSCFConstantString      CFGetRetainCount = 1152921504606846975   = 0x0FFFFFFFFFFFFFFF
NSTaggedPointerString     CFGetRetainCount = 9223372036854775807   = INT64_MAX
tagged NSNumber           CFGetRetainCount = 9223372036854775807   = INT64_MAX
```

不可变容器没有拿到哨兵值。`__NSArrayI` 返回的是真实引用计数 2。一个来自变量本身，一个来自 ARC 在 `-O0` 下传参时插的 retain。"不可变对象引用计数是极大值"这个印象，只对常量字符串和 tagged pointer 成立。

而"极大值"本身有三种。[[iOS 内存管理：从 MRC、ARC 到属性关键字#第一部分：MRC 的所有权规则：retain、release 与 autorelease|MRC 的所有权规则]] 里测过 tagged `NSNumber` 是 `INT64_MAX`，这次对上了。另外两种是新的：`-1` 归编译期常量对象和共享缓存里的单例，`0x0FFFFFFFFFFFFFFF` 归 `__NSCFConstantString`。三条路，三个魔数，再一次说明这个数不能拿来做任何判断。

---

## 九、选型

前面的数字够了。这节只做算术。

### 查找：数组是线性的，斜率能测出来

```text
N        NSArray     NSSet    NSDictionary   array/set
100        783 ns    43 ns       48 ns          18
400       7166 ns    70 ns       42 ns         103
1600     10181 ns    65 ns       68 ns         157
6400     40175 ns    78 ns       81 ns         515
12800    81238 ns    54 ns       60 ns        1493
```

`NSSet` 和 `NSDictionary` 这两列从 100 到 12800 就没动过，稳在 40 到 80 纳秒。`NSArray` 那列每翻一倍就翻一倍。

单元素的扫描成本是 6.35 纳秒，也就是 `81238 / 12800`。前提是探测对象和数组里的元素是同一个指针，`isEqual:` 走最快路径。换成每次新造的 `NSNumber`，这个数涨到 25 纳秒。

### 建表：数组便宜 7 倍

```text
N=80000   array = 1.33 ms   set = 9.98 ms   dict = 10.61 ms
```

折算成每个元素：数组 16.6 ns，集合 124.8 ns。集合每多一个元素多花 110 纳秒。

### 于是回本点是一个和 N 无关的数

建 `NSSet` 每个元素多付 110 ns。`NSArray` 每查一次全表，每个元素要付 6.35 到 25 ns。两边同时含一个 N，约掉：

**查大约 5 到 17 次就回本，和集合有多大没有关系。**

上界 17 对应最理想的指针命中。下界 5 对应真实的 `isEqual:` 比较。未命中的查询要扫完全表，命中的平均只扫一半，所以未命中占多数时回本更快。

所以实践中的判断标准很简单。这个集合建好之后，你要按内容查它超过几次？要，就用 `NSSet`。不要，用 `NSArray`。

有序而且只查不改的场景还有第三个选项。10 万个有序元素里找最后一个：

```text
indexOfObject:                      4741437 ns
indexOfObject:inSortedRange:...        686 ns      快 6909 倍
```

`NSArray` 的二分查找版本很少有人用。性能上它和哈希表是一个量级。

### 内存

```text
n=100000   dict 主缓冲 3440640 字节 (34.4 B/项)
           set        1720320 字节 (17.2 B/项)
           array      1097728 字节 (11.0 B/项)
```

数组每个元素 8 字节指针，加上不到 40% 的富余。集合大约 16 字节一项。字典大约 32 字节一项，正好是键值指针本身的两倍。哈希表要留一半空桶。

字典的扩容触发点测出来是 count 到 1、4、7、12、20、33、53、86、119、156、238、391、673、1066、1733、2796、4544、7392、12020、19303。相邻两档比值同样稳定在 1.6 附近。和第三节数组的收敛值是同一个数。

这个序列和开源 CF 里那张质数表对不上。`CFBasicHash.c` 的容量表是：

```c
static const uintptr_t __CFBasicHashTableSizes[64] = {
    0, 3, 7, 13, 23, 41, 71, 127, 191, 251, 383, 631, 1087, 1723,
    2803, 4523, 7351, 11959, 19447, 31231, 50683, 81919, 132607, ...
```

前两档 3 和 7 与实测吻合，往后就分家了。`__NSDictionaryM` 和 `CFBasicHash` 是两套实现。这和第三节里 `__NSArrayM` 与 `CFArray` 的关系一样。开源那份 CF 的最新 tag 是 CF-1153.18，`CFArray.c` 的版权行停在 2014 年。拿它解释今天 Foundation 的行为，只能对到设计思想那一层。

不过这条只对可变字典成立，不可变的那一半是反过来的。

Ciechanowski 用 Hopper 从 Foundation 二进制里 dump 出了 `__NSDictionaryI` 用的两张静态表（基线 iOS 7.1）：

```text
___NSDictionarySizes        0, 3, 7, 13, 23, 41, 71, 127, ...
___NSDictionaryCapacities   0, 3, 6, 11, 19, 32, 52,  85, ...
```

和 `__CFBasicHashTableSizes` / `__CFBasicHashTableCapacities` 逐位相同。

两套二进制、两种取证手段（读开源代码 vs 反汇编闭源框架）撞出同一串数，这比任何一边单独说话都硬。所以准确的说法是：不可变字典和 `CFBasicHash` 共用一套尺寸策略，可变字典自己另走一条按 1.6 倍增长的路。上面那个"两套实现"的结论要收窄到 `__NSDictionaryM`。

把两张表相除就是装载因子：

```text
buckets=41    capacity=32     0.780
buckets=127   capacity=85     0.669
buckets=191   capacity=118    0.618
buckets=1087  capacity=672    0.618
buckets=4523  capacity=2795   0.618
```

收敛到 0.618。又是 φ 的倒数。哈希表留出来的那 38% 空桶，就是"平均常数时间"这句话的全部前提，也是上面那个 34.4 字节每项的来源。

那张质数表的原注释还藏着一条线索：

> Prime numbers. Values above 100 have been adjusted up so that the malloced block size will be just below a multiple of 512; values above 1200 have been adjusted up to just below a multiple of 4096.

质数挑完之后又被往上推了一点，为的是让 `malloc` 出来的块正好卡在 512 或 4096 的倍数下面。这和第三节数组那边的 `malloc_good_size` 是同一个动作：先算一个理论值，再向分配器的尺寸类让步。两个毫不相干的数据结构，在同一件事上做了同样的妥协。

### 一句话版本

- 只遍历、只按下标取，用 `NSArray`。头尾频繁增删也归它，第三节测过是 O(1)。
- 要按内容判断在不在，用 `NSSet`，查 5 次以上就赚。
- 要按 key 取值，用 `NSDictionary`。key 必须实现 `NSCopying`，参与 `hash` 的字段进去之后不能再改。
- 容器要装弱引用，用 `NSHashTable` / `NSMapTable`，但别信它们的 `count`。
- 要在中间频繁插删，Foundation 没有合适的容器。`NSMutableArray` 中间插入在 8 万元素时是 268 毫秒插 8 万次。

---

## 十、几个已经不准的说法

- "`copy` 会把所有元素指针复制一遍，是 O(n)。" 六个元素以上是写时复制。`copy` 和 `mutableCopy` 都是常数时间，复制推迟到下一次写。见第四节。
- "`@[@1, @2]` 的真实类是 `__NSArrayI`。" 元素全是编译期常量时，clang 生成的是 `__DATA` 段里的 `NSConstantArray`。它不在堆上。
- "`NSMutableArray` 底层就是 `CFArray` 的双端队列。" CF 的 deque 不是环形的，`__NSArrayM` 是。挪较短一侧这个思想相同，数据结构不同。
- "扩容按 1.5 倍或 2 倍。" 真实策略是乘 1.618 再过一遍 `malloc_good_size`，25 次跃迁全中。只跑到几千个元素会看到一段整齐的 2.000，那是尺寸类造成的假象。
- "在 macOS 上调 `CFArrayCreateMutable` 就能拿到 `CFArray.c` 里那个实现。" 传标准回调回来的是 `__NSArrayM`，只有 NULL 回调才给 `__NSCFArray`。
- "`for-in` 一次抓 16 个对象。" 只对集合和字典成立。数组一次把全部交出来。绕了圈的数组分两批。
- "枚举期间修改必崩。" 检查在每次迭代的开头做。容器里只剩一个元素时改不会抛。
- "`NSDictionary` 会 copy key 是 CF 的行为。" `kCFTypeDictionaryKeyCallBacks` 是 retain。copy 由 `NSDictionary` 自己做，`NSMapTable` 不做。
- "不可变对象的 `CFGetRetainCount` 是 `INT64_MAX`。" `__NSArrayI` 返回真实计数。三种哨兵值分属编译期常量、常量字符串和 tagged pointer。
- "`NSPointerArray` 的 `compact` 能清掉 NULL 洞。" 直接调没有效果，得先 `addPointer:NULL`。
- "头部插入慢，尾部插入快。" 同一个选择器下两者相同。`addObject:` 确实比 `insertObject:atIndex:` 慢 45%，那是方法的常数开销。

---

## 总结

类簇的私有类名今天有十几个。`NSConstantArray` 和 `__NSFrozenArrayM` 是这几年新加的。它们能解释现象，不能写进代码。

`__NSArrayM` 是环形缓冲区，头尾 O(1)，中间 O(n)。那条对称的帐篷曲线是最直接的证据。扩容是乘 1.618 再交给 `malloc_good_size` 取整，25 次跃迁全中。开源 CF 里的 deque 与之设计同源、结构不同，而且在 macOS 上传标准回调根本走不到那份代码。

`copy` 和 `mutableCopy` 在六个元素以上是写时复制，六万四千个元素也是 50 纳秒。这是全篇最该带走的一条。它同时意味着"对外返回一份只读快照"在 Foundation 里几乎免费。

选型不需要背复杂度表。建 `NSSet` 每个元素多花 110 纳秒，`NSArray` 扫一遍每个元素花 6 到 25 纳秒。除一下就是回本次数。答案是 5 到 17 次，与集合大小无关。

最后一条方法论和上一篇同源，但这次多了个转折。本篇里推翻通说的结论没有一条来自读文章，全部来自打印内存和掐表。而我唯一一次差点写错，恰恰是因为测出来的结果和网上的说法一致，就没再多问一句。**符合直觉的错误结论，比违背直觉的更难被自己抓住。**

下一篇 [[iOS 持久化选型：沙盒、plist、NSUserDefaults 与 Keychain]]。

## 参考资料

### 一手源码与头文件

- [apple-oss-distributions/CF](https://github.com/apple-oss-distributions/CF)：`CFArray.c` 的 `__CFArrayDeque` 与 `__CFArrayRepositionDequeRegions`、`CFBasicHash.c` 的 `__CFBasicHashTableSizes`、`CFDictionary.c` 的 `kCFTypeDictionaryKeyCallBacks`。最新 tag CF-1153.18，`CFArray.c` 版权行到 2014 年，只能对到设计层面
- 本机 macOS SDK 的 `Foundation/NSEnumerator.h`：`NSFastEnumerationState` 的五个字段
- `clang -S -emit-llvm` 生成的 IR：`for-in` 的 mutation 检查在循环头，每次迭代都做

### 官方文档

- [NSDictionary](https://developer.apple.com/documentation/foundation/nsdictionary) / [NSMutableArray](https://developer.apple.com/documentation/foundation/nsmutablearray)：公开语义与线程安全边界
- [NSPointerArray](https://developer.apple.com/documentation/foundation/nspointerarray)：`compact` 的文档描述与实测行为不符

### 经典

下面三篇是这个主题公认的参考读物，但本文没有从中取用任何结论——正文里的每个数字都来自自己跑的实验，每段源码都抄自上面那份 CF。列在这里是给想读二手梳理的人。

- [Bartłomiej Ciechanowski — Exposing NSMutableArray](https://ciechanow.ski/exposing-nsmutablearray/)
- [Bartłomiej Ciechanowski — Exposing NSDictionary](https://ciechanow.ski/exposing-nsdictionary/)
- [CFArray 的历史渊源及实现原理](https://zhuanlan.zhihu.com/p/25063245)

### 本地

- [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]]：类簇与 `isKindOfClass:`、tagged pointer 的位布局
- [[iOS 内存管理：从 MRC、ARC 到属性关键字#第三部分：属性关键字：从所有权推导，而不是从类型名猜|属性关键字：从所有权推导，而不是从类型名猜]]：`copy` 修饰容器属性的崩溃
- [[iOS 内存管理：从 MRC、ARC 到属性关键字#第一部分：MRC 的所有权规则：retain、release 与 autorelease|MRC 的所有权规则]]：`retainCount` 为什么不能信
- [[iOS Mach-O：结构、符号绑定与 chained fixups]]：`__DATA` 段里的常量对象与外部类符号

---

实验环境：macOS 26.5.2（arm64，Apple Silicon），Apple clang 21.0.0，全部用 `clang -fobjc-arc -O0 -framework Foundation` 编成原生二进制直接跑，没有开模拟器。计时用 `mach_absolute_time`，每组至少六轮，取平均或最小值，并保留原始数据核对噪声。

私有类名、空容器单例、`__NSArrayM` 的字段偏移都属于实现细节，未在 iOS 上复核。第二节是例外：换成 iOS 15 / 17 / 18 的 target 编译，三个 `NSConstant*` 符号照样生成，所以至少编译器这一侧两个平台一致。

> 待真机补测：第四节的写时复制在 iPhone 15 / iOS 26 上复现一次。这条结论如果在真机上不成立，第九节的选型建议要跟着改。
