---
title: 【iOS】属性关键字：从所有权推导，而不是从类型名猜
published: 2026-07-26
description: 一个 atomic 的 NSInteger 属性，和 nonatomic 的那个，编译出来是逐字节相同的机器码。从编译器真正生成的访问器出发，看清 copy 为什么不是深拷贝、atomic 到底保护了什么。
tags:
  - iOS
  - Objective-C
  - Memory
  - 并发
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 6
draft: true
---
# 属性关键字：从所有权推导，而不是从类型名猜

> [!note] 已整合
> 本文已与 MRC、ARC 两篇合并为 [[iOS 内存管理：从 MRC、ARC 到属性关键字#第三部分：属性关键字：从所有权推导，而不是从类型名猜|Objective-C 内存管理：从 MRC、ARC 到属性关键字]]。本文件保留为分篇原稿，以兼容既有链接。

网上流传的属性关键字口诀大概是这样：NSString 用 copy，delegate 用 weak，Block 用 copy，基本类型用 assign。

记住这四条能答对大部分面试题，但它解释不了任何一个"为什么"，也处理不了没见过的情况——一个自定义的 value object 该用什么？一个需要跨线程读写的配置项该用什么？

问题在于这十来个关键字看起来像一个需要死记的排列组合，其实它们在回答三个互不相干的问题。

**所有权**：这个属性持有对象吗，持有到什么程度。`strong` / `copy` / `weak` / `assign` / `unsafe_unretained` 五选一。

**原子性**：`atomic` / `nonatomic` 二选一。

**暴露方式**：`readwrite` / `readonly`，加上 `getter=` / `setter=` 这些命名控制。

三栏正交，各选各的。把它们摊平成一张十几行的表然后去背——这就是为什么怎么背都记不住。

本文所有输出块都是真跑出来的，环境是 Xcode 26.6 加 iOS 模拟器（arm64），完整配置放在文末。

---

## 一、所有权那一栏

### 编译器到底生成了什么

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

### 五个所有权关键字

**`strong`** 是对象属性的默认值，语义是"我拥有它，它至少要活得和我一样久"。setter 展开成 `objc_storeStrong`，内部顺序是先 retain 新值、再赋值、最后 release 旧值。这个顺序不能颠倒——新旧是同一个对象且引用计数为 1 时，先 release 会直接把它干掉。

**`copy`** 的语义是"我要一份独立的、不会被外部修改的快照"。这是所有权隔离，不是内容复制。下一节整节都在讲这件事。

**`weak`** 表示"我引用它，但不拥有它，也不该延长它的生命周期"。对象销毁时自动置 `nil`。具体实现见 [[iOS weak 的实现：SideTable 与置 nil 的时机]]。

**`assign`** 是给标量准备的：`int`、`BOOL`、`CGFloat`、结构体。一次位拷贝，不碰引用计数。

**`unsafe_unretained`** 修饰对象时和 `assign` 行为完全一致——不 retain、不置 nil。区别在于表达意图：`assign` 用在对象上不会报警告，容易是手滑；`unsafe_unretained` 是 ARC 时代专门造的显式声明，写下它等于在说"我知道这会产生悬垂指针，我有把握"。

我在 code review 里看到 `assign` 修饰对象一律打回，不听解释。想要那个语义就写 `unsafe_unretained`，让下一个读代码的人知道你是故意的。

### weak 与 unsafe_unretained 的实测差异

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

## 二、copy 不是深拷贝

### 不可变对象的 copy 返回的是它自己

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

### 四种组合

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

### 一个意外：短字符串的 copy 撞回了 Tagged Pointer

上面那组输出里，可变源 `copy` 之后的地址 `0x88b5a991d0b93da5`，和不可变源本身完全一样。这两个对象是分别构造的，凭什么撞在一起？

```text
不可变短串        0xa0b62b44cfead36e
可变短串 copy 后  0xa0b62b44cfead36e
两者相同=1
```

因为内容相同，而短字符串的 copy 结果是 `NSTaggedPointerString`——[[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer#值即身份|值就是身份]]，相同的值必然产生相同的位模式。不是"复制到了同一个对象"，是两个值相同的 Tagged Pointer 天然长得一样。

顺便提醒一个读地址的陷阱。上面第一组里 `[不可变源 mutableCopy]` 是 `0x600000c00090`，而后面另一组实验里一个完全无关的对象地址也是 `0x600000c00090`。前一个对象已经释放，分配器把这块内存回收后又发给了下一个请求。跨时间点比地址没有意义，能比的只有同一时刻存活的两个对象。

### 容器只复制一层

```text
原容器 0x600000004020  元素 0x600000c00090
mutableCopy              容器 0x600000c00180 (变了=1)  元素 0x600000c00090 (变了=0)
initWithArray:copyItems: 容器 0x600000004060 (变了=1)  元素 0x90231e27fe1d3b05 (变了=1)
```

容器地址变了，元素没变——元素还是原来那几个对象，只是被装进了一个新数组。想让元素也被复制，需要 `initWithArray:copyItems:YES`，它会对每个元素发一次 `copyWithZone:`。

但这依然只有一层。元素本身如果还是容器，孙子那一层照样共享。真正的完全深拷贝在 Foundation 里没有通用方案，实务上只能靠归档解档走一圈。

"`mutableCopy` 就是深拷贝"是中文技术文章里被简化得最厉害的一句，实际上它连元素都没碰。

### copy 修饰可变类型：一个必然发生的崩溃

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

## 三、atomic：一个能证伪自己的实验

### 先看数据

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

### 更要命的是，这两行代码根本没有区别

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

### 那 atomic 到底保证什么

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

## 四、Block 属性：copy 还是 strong

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

## 五、剩下那些不常被问到的

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

## 六、怎么选

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

## 总结

三栏正交——所有权、原子性、暴露方式，各选各的。看到一个新属性，按这三个问题过一遍，不要回忆表格。

`copy` 属性的 setter 真的会调 `copyWithZone:`，连 `nonatomic copy` 也走运行时函数。所以它只能修饰不可变类型，第三节那个 `unrecognized selector` 就是这一行的直接后果。

`copy` 不是深拷贝，`mutableCopy` 也不是。它们都只处理最外面那一层，不可变对象的 `copy` 甚至连新对象都不造。

`atomic` 保护的是单次访问不撕裂，仅此而已。对标量属性它连一条额外指令都不生成，`atomic` 和 `nonatomic` 编译出来是同一份机器码。需要线程安全就往上一层加锁，别在这两个关键字之间挑。

## 参考资料

### 官方与源码

- [Apple — Encapsulating Data](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/EncapsulatingData/EncapsulatingData.html)：atomicity 与 thread safety 的官方措辞出自 "Properties Are Atomic by Default" 一节的注释框
- [Apple — NSCopying](https://developer.apple.com/documentation/foundation/nscopying)
- [objc4 — objc-accessors.mm](https://github.com/apple-oss-distributions/objc4)：`objc_getProperty` / `reallySetProperty` 的完整实现
- [objc4 — objc-private.h](https://github.com/apple-oss-distributions/objc4)：`StripedMap` 的分片数
- [clang — CGObjC.cpp](https://github.com/llvm/llvm-project)：`PropertyImplStrategy` 决定了本文第一节那张分路表
- [clang — DeclObjC.h](https://github.com/llvm/llvm-project)：`getSetterKind`，block 属性上 `strong` 等价于 `copy` 的依据

### 经典

- [objc.io — Value Objects](https://www.objc.io/issues/7-foundation/value-objects/)：解释了为什么不可变类的 `copyWithZone:` 应该 retain 而不是真拷贝

### 本地

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
