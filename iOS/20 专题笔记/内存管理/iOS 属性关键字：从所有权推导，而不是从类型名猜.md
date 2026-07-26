---
title: 【iOS】属性关键字：从所有权推导，而不是从类型名猜
published: 2026-07-26
description: strong / copy / weak / assign / atomic 不是一张需要背的记忆表，而是三个正交问题的答案。从编译器真正生成的 setter 出发，用实测数据说清 copy 不是深拷贝、atomic 不是线程安全，以及 copy 修饰可变类型为什么必然崩溃。
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

## 为什么这个话题值得单独写一篇

属性关键字是 iOS 面试里出现频率最高的题目之一，也是最容易被"背表"糊弄过去的一个。网上流传的版本大多长这样：NSString 用 copy，delegate 用 weak，Block 用 copy，基本类型用 assign。记住这四条确实能答对大部分面试题，但它解释不了任何一个"为什么"，更处理不了没见过的情况——比如一个自定义的 value object 该用什么，一个需要跨线程读写的配置项该用什么。

真正的问题是：这十来个关键字看起来像一个需要死记的排列组合，其实它们只在回答三个互不相干的问题。

一是**所有权**：这个属性持有对象吗，持有到什么程度。`strong` / `copy` / `weak` / `assign` / `unsafe_unretained` 在这一栏里五选一。

二是**访问器要不要同步**：`atomic` / `nonatomic` 二选一。

三是**对外暴露什么**：`readwrite` / `readonly`，加上 `getter=` / `setter=` 这些命名控制。

这三栏彼此正交，各选各的。把它们摊平成一张十几行的记忆表，是学不动的根源。

本文按这三栏展开，重点放在前两栏——因为它们背后是真实的运行时机制，而不是语法约定。第一周关于所有权规则的内容（[[iOS 内存：MRC 的所有权规则]]、[[iOS 内存：ARC 的两半]]）是本文的前置，那边讲的是"谁拥有对象"，这边讲的是"怎么把这个所有权关系写进属性声明"。

---

## 一、所有权那一栏

### 编译器到底生成了什么

讲属性关键字最有效的方式，是直接看编译器为每种组合生成了什么样的 setter。这里有一个很多人不知道的分野：**并不是所有属性的访问器都长得一样，它们分成两条完全不同的路径。**

一条是编译器直接内联展开——setter 里就是几条指令，不调用任何额外的运行时函数。另一条是调用 objc4 提供的通用访问器函数，把工作丢给运行时。

分界线大致是这样的：

| 属性声明 | setter 的实现方式 |
| --- | --- |
| `nonatomic, strong` | 编译器内联，展开成 `objc_storeStrong` |
| `nonatomic, copy` | 编译器内联，`copyWithZone:` + 赋值 + 释放旧值 |
| `atomic, strong` | 调用运行时 `objc_setProperty_atomic` |
| `atomic, copy` | 调用运行时 `objc_setProperty_atomic_copy` |
| `weak`（无论 atomic 与否） | 固定调用 `objc_storeWeak` |
| 结构体（如 `CGRect`）+ atomic | 调用 `objc_copyStruct` |
| 标量 + nonatomic | 直接赋值，没有任何函数调用 |

规律很清楚：**只要涉及 atomic，几乎一定走运行时函数；nonatomic 的对象属性倾向于内联；weak 是特例，永远走 `objc_storeWeak`。** 原因也不难理解——atomic 需要一把锁，而锁存在哪、怎么按 ivar 地址分片，这些是运行时的事，编译器没法内联；weak 需要访问全局的弱引用表，同理。

objc4 里那几个运行时访问器的签名是这样的（这些函数没有公开头文件声明，只能从源码里看到）：

```objc
id   objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic);
void objc_setProperty(id self, SEL _cmd, ptrdiff_t offset, id newValue,
                      BOOL atomic, BOOL shouldCopy);
void objc_copyStruct(void *dest, const void *src, ptrdiff_t size,
                     BOOL atomic, BOOL hasStrong);
void objc_storeWeak(id *location, id newValue);
id   objc_storeStrong(id *location, id obj);
```

`objc_setProperty` 内部会转到 `reallySetProperty`，核心逻辑是：

```cpp
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset,
                                     bool atomic, bool copy, bool mutableCopy) {
    id oldValue;
    id *slot = (id*)((char*)self + offset);

    if (copy)             newValue = [newValue copyWithZone:nil];
    else if (mutableCopy) newValue = [newValue mutableCopyWithZone:nil];
    else {
        if (*slot == newValue) return;      // 同一个对象，直接返回
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

    objc_release(oldValue);                 // 注意：在锁外释放
}
```

三个细节值得留意。

第一，`copy` 关键字对应的调用是 `copyWithZone:`，**不是** `mutableCopyWithZone:`。这一行是后面那个经典崩溃的全部原因，先记住。

第二，锁的粒度。`PropertyLocks[slot]` 是一个按 ivar 地址索引的分片锁表（`StripedMap`），不是每个对象一把锁，也不是全局一把锁。所以同一个对象的两个 atomic 属性之间**没有任何互斥关系**。

第三，`objc_release(oldValue)` 特意放在解锁之后。释放旧值可能触发 `dealloc`，`dealloc` 里可能做任意事情——如果在锁内执行，很容易撞出死锁。同样的手法在 getter 里也有：`objc_getProperty` 在锁内 retain，在锁外 autorelease。

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

这个 getter 解释了 atomic 唯一真正提供的保证：**返回值在调用方拿到它之前不会被别的线程释放掉。** 它在锁的保护下把对象 retain 住了。至于调用方拿到之后要干什么、期间属性有没有被改成别的对象，atomic 一概不管。

### 五个所有权关键字

有了上面的框架，逐个说就快了。

**`strong`** 是对象属性的默认值。语义是"我拥有它，它至少要活得和我一样久"。setter 展开成 `objc_storeStrong`，内部顺序是先 retain 新值、再赋值、最后 release 旧值——**这个顺序不能颠倒**，否则当新值和旧值是同一个对象、且引用计数为 1 时，先 release 会直接把对象干掉。

**`copy`** 的语义不是"深拷贝"，而是"我要一份独立的、不会被外部修改的快照"。这是所有权隔离，不是内容复制。下一节整节都在讲这件事。

**`weak`** 表示"我引用它，但不拥有它，也不该延长它的生命周期"。对象销毁时弱引用会自动置 `nil`。代价是每次读写都要过一遍弱引用表（哈希查找加锁），比直接指针访问慢一个量级。具体实现见 [[iOS weak 的实现：SideTable 与置 nil 的时机]]。

**`assign`** 是给标量准备的：`int`、`BOOL`、`CGFloat`、结构体。它就是一次位拷贝，不碰引用计数。

**`unsafe_unretained`** 和 `assign` 修饰对象类型时行为完全一致——不 retain、不置 nil。区别在于表达意图：`assign` 用在对象上不会报警告，容易是手滑；`unsafe_unretained` 是 ARC 时代专门造出来的显式声明，写下它等于在说"我知道这会产生悬垂指针，我有把握"。实践中"`assign` 修饰对象"应该一律视为坏味道改掉。

那什么时候会故意选 `unsafe_unretained`？答案是高频访问路径上，你能自证对象生命周期一定长于访问者的时候。weak 的查表开销在逐帧访问的场景里是能量到的。但这是个有意的权衡，不是默认选项。

---

## 二、copy 不是深拷贝

### 实测：不可变对象的 copy 返回的是它自己

这是最值得亲手验一遍的一条。

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

地址一模一样。容器也是同样的结果：

```text
NSArray      源 0x60000020cb00  copy 0x60000020cb00  同一对象=1
NSDictionary 源 0x60000020cba0  copy 0x60000020cba0  同一对象=1
```

原因不复杂：对一个不可变对象来说，"复制一份"没有任何意义——反正内容永远不会变，多个持有者共享同一块内存是完全安全的。所以这些类的 `copyWithZone:` 实现直接就是 `return [self retain];`。

这是一个性能优化，不是语言规定。但 Foundation 里所有常见的不可变类都这么实现，可以当成事实依赖。objc.io 那篇 Value Objects 把这个原则说得很直白：当类和它的内容都是不可变的，`NSCopying` 的正确实现是 retain 原对象，而不是造一个新副本。

### 四种组合的实测矩阵

把可变/不可变 × copy/mutableCopy 全跑一遍：

```text
不可变源  0x88b5a991d0b93da5  class=NSTaggedPointerString
  copy        -> 0x88b5a991d0b93da5  class=NSTaggedPointerString  同一对象=1
  mutableCopy -> 0x600000c00090      class=__NSCFString           同一对象=0
可变源    0x600000c000c0      class=__NSCFString
  copy        -> 0x88b5a991d0b93da5  class=NSTaggedPointerString  同一对象=0
  mutableCopy -> 0x600000c00090      class=__NSCFString           同一对象=0
```

整理成规则：

```
[不可变对象 copy]        → 同一个对象（本质是 retain）
[不可变对象 mutableCopy] → 新对象，可变
[可变对象   copy]        → 新对象，不可变
[可变对象   mutableCopy] → 新对象，可变
```

两条不依赖源对象可变性的铁律：`copy` 的结果**永远不可变**，`mutableCopy` 的结果**永远可变**。至于是"新对象"还是"原对象"，只有 `[不可变 copy]` 这一格会退化。

### 一个意外：短字符串的 copy 撞回了 Tagged Pointer

上面那组输出里有个反常的地方：可变源 `copy` 之后得到的地址 `0x88b5a991d0b93da5`，和不可变源本身的地址**一模一样**。这两个对象是分别构造的，凭什么撞在一起？

单独验一次就清楚了：

```text
不可变短串        0xa0b62b44cfead36e
可变短串 copy 后  0xa0b62b44cfead36e
两者相同=1
```

因为它们的内容相同，而短字符串的 copy 结果是一个 `NSTaggedPointerString`——[[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer#4.6 值即身份|Tagged Pointer 的值就是它的身份]]，相同的值必然产生相同的位模式。这不是"复制到了同一个对象"，而是"两个值相同的 Tagged Pointer 天然长得一样"。

顺带说一个读地址时的陷阱。上面第一组输出里 `[不可变源 mutableCopy]` 的地址是 `0x600000c00090`，而后面另一组实验里一个完全无关的对象地址也是 `0x600000c00090`。这不代表它们有任何关系——前一个对象已经被释放，分配器把这块内存回收后又发给了下一个请求。**跨时间点比较地址是没有意义的**，能比较的只有同一时刻存活的两个对象。

### 容器只复制一层

`mutableCopy` 一个数组，复制的是容器本身，不是里面的元素：

```text
原容器 0x600000004020  元素 0x600000c00090
mutableCopy              容器 0x600000c00180 (变了=1)  元素 0x600000c00090 (变了=0)
initWithArray:copyItems: 容器 0x600000004060 (变了=1)  元素 0x90231e27fe1d3b05 (变了=1)
```

容器地址变了，元素地址没变——元素还是原来那几个对象，只是被装进了一个新数组。想让元素也被复制，需要 `initWithArray:copyItems:YES`，它会对每个元素发一次 `copyWithZone:`。

但这依然只是**一层**。如果元素本身还是个容器，孙子那一层照样是共享的。真正的完全深拷贝在 Foundation 里没有通用方案，实务上只能靠归档解档（`NSKeyedArchiver` 走一圈）。

"`mutableCopy` 就是深拷贝"是中文技术文章里被简化得最厉害的一句话，实际上它连元素都没碰。

---

## 三、copy 修饰可变类型：一个必然发生的崩溃

现在回到前面埋的那一行：`copy` 关键字生成的调用是 `copyWithZone:`。

这意味着，**无论你把属性类型声明成什么，copy 修饰的 setter 存进去的一定是不可变对象。** 类型声明只骗过了编译器，骗不过运行时。

```objc
@interface Foo : NSObject
@property (nonatomic, copy)   NSMutableArray *badArr;   // 故意写错
@property (nonatomic, strong) NSMutableArray *goodArr;
@end

f.badArr  = [NSMutableArray arrayWithObject:@1];
f.goodArr = [NSMutableArray arrayWithObject:@1];
```

```text
声明类型都是 NSMutableArray *
  copy   修饰 -> 实际类 = __NSSingleObjectArrayI
  strong 修饰 -> 实际类 = __NSArrayM
  对 copy 修饰的调用 addObject: -> 抛异常:
      -[__NSSingleObjectArrayI addObject:]: unrecognized selector sent to instance 0x600000004080
```

`__NSArrayM` 是可变数组的实现类，`__NSSingleObjectArrayI` 是不可变数组（末尾的 `I` = Immutable，`M` = Mutable）。`copy` 那个属性拿到的是一个不可变数组，调 `addObject:` 直接 unrecognized selector。

这个 bug 的恶劣之处在于**它不在赋值那一行崩，而在后面某个使用点崩**，堆栈里看到的是一个类型对不上的对象，很容易怀疑到别的地方去。规避方式只有一条：**`copy` 只用于不可变类型的属性声明**。想要一个属性既独立又可变，得自己在 setter 里 `mutableCopy`。

---

## 四、atomic：它保证什么，以及为什么这个保证基本没用

### 先看实测

一个 `atomic` 属性，十万次并发自增：

```objc
@property (atomic) NSInteger atomicCount;
...
dispatch_apply(100000, dispatch_get_global_queue(0, 0), ^(size_t i) {
    c.atomicCount = c.atomicCount + 1;
});
```

两次运行的结果：

```text
--- 第 1 次 ---
期望 100000，实际 atomic    = 49948  （丢了 50052 次）
期望 100000，实际 nonatomic = 59432  （丢了 40568 次）
期望 100000，实际 nonatomic+锁 = 100000

--- 第 2 次 ---
期望 100000，实际 atomic    = 50941  （丢了 49059 次）
期望 100000，实际 nonatomic = 58153  （丢了 41847 次）
期望 100000，实际 nonatomic+锁 = 100000
```

`atomic` 丢了一半的计数。**而且它丢得比 `nonatomic` 还多。**

后半句值得停一下。这不是说 atomic 更不安全——两者在这个场景下都是完全错误的，谁丢得多只取决于线程交错的具体节奏。atomic 每次访问要抢一把自旋锁，访问变慢了，线程之间的重叠窗口反而变大，于是丢得更多。这个数字每次跑都不一样，但结论稳定：**在"读-改-写"这类复合操作面前，atomic 提供的保护等于零。**

对照组把整个"读+写"用 `os_unfair_lock` 包起来，结果精确到 100000。

### 为什么

`self.atomicCount = self.atomicCount + 1` 这一行看起来是一条语句，实际是三步：调 getter 读出旧值、加一、调 setter 写回。atomic 保证的是**getter 内部**和**setter 内部**各自不被打断，但这两次调用之间是完全敞开的。两个线程同时读到 41，各自算出 42，先后写回，结果就丢了一次。

Apple 官方文档对这一点的措辞很克制但很明确：

> Property atomicity is not synonymous with an object's thread safety.

### 还有两个更隐蔽的失效场景

**atomic 只保护指针，不保护指针指向的东西。**

```objc
@property (atomic) NSMutableArray *arr;
// 线程 A: [self.arr addObject:@1];
// 线程 B: [self.arr addObject:@2];
```

`self.arr` 这次读取确实是原子的，两个线程都安全地拿到了同一个 `NSMutableArray *`。然后它们各自对这个数组调 `addObject:`——而 `NSMutableArray` 自己压根没有任何同步措施。数组内部结构被并发写坏，轻则数据错乱，重则 `EXC_BAD_ACCESS`。atomic 在这里连边都没沾上。

**atomic 的锁是按属性分的，不是按对象分的。** 如果业务逻辑要求"`width` 和 `height` 必须同时更新才算一个合法状态"，把两个属性都标成 atomic 一点用都没有——它们各自用各自的锁，中间照样能被别的线程读到一个宽已改、高未改的中间态。

### 那 atomic 到底有什么用

它保证单次访问器调用不会撕裂：你不会读到一个"改了一半"的指针，也不会在读的瞬间对象被另一个线程释放掉（getter 里那次锁内 retain 就是干这个的）。

这个保证是真实的，但它解决的问题层次太低了。真实业务里需要的一致性几乎都是跨多次访问的，而那正是 atomic 覆盖不到的地方。所以 iOS 工程的通行做法是：**属性一律 `nonatomic`，线程安全放到更上层用锁、串行队列或者不可变数据来解决。** 顺带还省下了每次访问抢锁的开销。

`nonatomic` 是不是就完全没风险？严格说也不是。指针本身的对齐写入在 arm64 上是原子的，但 ARC 插入的 retain/release 和指针写入不是一个整体，并发场景下理论上存在过度释放的窗口。这个窗口很窄，实践中很少直接触发，但不能说"nonatomic 绝对安全"。真要跨线程读写同一个属性，答案不是在 atomic 和 nonatomic 之间挑，而是加同步。

---

## 五、weak 与 unsafe_unretained 的实测差异

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

`weak` 被置成了 `nil`，`unsafe_unretained` 还指着那块已经归还给分配器的内存。此时访问 `unsafeRef` 是未定义行为——可能什么事都没有（内存还没被复用），可能拿到一个完全不相干的新对象，也可能直接崩。**这种"有时候好使"正是它最危险的地方**，它不会在开发阶段稳定复现。

关于 delegate 为什么用 weak，流行的解释是"防止循环引用"。这个说法不算错，但把因果关系讲反了。更准确的说法是所有权方向：`UITableView` 不应该拥有它的 delegate（通常是一个 ViewController），因为 ViewController 是更上层、生命周期更长的对象。循环引用只是"如果用错 strong 会导致的后果之一"，不是选 weak 的理由本身。按所有权推导，答案是唯一的；按"防循环引用"推导，遇到没有环的场景就不知道该选什么了。

---

## 六、Block 属性：copy 还是 strong

这是本文唯一一个我不打算给出单一答案的地方。

历史上必须用 `copy`：Block 字面量默认在栈上创建，出了作用域就失效，属性想长期持有它就必须先 copy 到堆上。这个理由在 MRC 时代是硬性的。

ARC 引入后情况变了。编译器在"把 Block 赋值给 strong 变量"这类上下文里会自动插入 copy，所以 `strong` 修饰的 Block 属性在绝大多数情况下也能正常工作。于是产生了两派说法：一派认为 `copy` 已经是历史包袱，`strong` 完全等价；另一派认为显式写 `copy` 表达的是"这个属性需要一份堆上的副本"这个语义诉求，不该依赖编译器的自动行为。

我倾向后者，但理由不是"怕出 bug"，而是可读性：`copy` 明确告诉读代码的人这里发生了一次从栈到堆的搬迁。不过这确实是习惯问题，不是对错问题，两种写法在当前 ARC 下的实际行为没有观察到差异。Block 在栈和堆之间的搬迁机制本身，放在 [[iOS Block 的结构：ABI、descriptor 与三种类型]] 里详细讲。

---

## 七、剩下那些不常被问到的

**`readonly`** 只是不生成公开 setter，不是运行时的写保护。KVC 的 `setValue:forKey:` 照样能改到底层 ivar。它表达的是接口契约。常见搭配是头文件里声明 `readonly`，在 `.m` 的 class extension 里重新声明成 `readwrite`，做到对外只读、对内可写。

**`getter=`** 最常见的用途是让 `BOOL` 属性符合 `isXxx` 的命名习惯：`@property (getter=isFinished) BOOL finished;`。`setter=` 用得少得多。

**`class`** 声明的是类属性，不生成实例变量，也不参与自动合成，实现里通常自己用 `static` 变量存。它主要是为了和 Swift 的 type property 对齐。

**`null_resettable`** 是一个组合语义：setter 接受 `nil`（用来重置成默认值），但 getter 保证不返回 `nil`。要做到后者必须自己写 getter 兜底。系统里最典型的例子是 `UIView.tintColor`——设成 `nil` 会回到系统默认色，读出来永远有值。

**protocol 和 category 里的属性** 是个容易误解的地方。在 protocol 里写 `@property` 只是声明了一对方法签名，不会生成任何存储，遵循协议的类得自己实现。category 里写 `@property` 同样不生成 ivar（category 无法给已有类追加实例变量），常规做法是用关联对象模拟存储，或者标 `@dynamic` 表示"存取方法我自己实现"。

---

## 八、怎么选

把决策顺序写成一串问题，比记表管用：

1. **是标量还是对象？** 标量、结构体一律 `assign`，到此结束。
2. **对象的话，我拥有它吗？** 不拥有（delegate、parent、observer 这类反向引用）→ `weak`。拥有 → 进第 3 步。
3. **调用方之后修改这个对象，会不会影响到我？** 会（典型是把一个 `NSMutableString` 或 `NSMutableArray` 存进 model）→ 需要独立副本 → `copy`，并且**属性类型必须声明成不可变类型**。不会（比如一个自定义的不可变模型对象、一个子视图）→ `strong`。
4. **目标类不支持 weak，或者这是逐帧访问的热路径且我能自证生命周期？** → `unsafe_unretained`，并且写注释说明为什么。
5. **原子性怎么选？** 默认 `nonatomic`。除非你能说清楚"单次访问不撕裂"恰好就是你要的保证——这种情况很少。真正需要线程安全时，加锁或者用串行队列。
6. **对外要不要可写？** 不要 → 头文件 `readonly` + extension 里 `readwrite`。

整个过程里没有一步是从类型名推的。`NSString` 用 `copy` 不是因为它叫 NSString，而是因为传进来的可能是个 `NSMutableString`，别人改了会影响我。如果能确定不会（比如这个值来自一个你完全掌控的、只产生不可变字符串的内部模块），`strong` 也是对的，还省一次可能的拷贝。

---

## 总结

1. 属性关键字是三个正交问题的答案：所有权归谁、访问器要不要同步、对外暴露什么。分栏选，不要背表。
2. setter 的生成分两条路：nonatomic 的对象属性倾向编译器内联（`objc_storeStrong` 或内联 copy），atomic 和 weak 走运行时函数（`objc_setProperty_*` / `objc_storeWeak`）。
3. `objc_setProperty` 里的 `copyWithZone:` 那一行，是"copy 修饰可变类型必然崩溃"的全部原因。`copy` 只能用于不可变类型的属性。
4. `copy` 不是深拷贝。不可变对象的 copy 直接返回它自己，这是 Foundation 有意为之的优化。实测长字符串、`NSArray`、`NSDictionary` 的 copy 地址与源完全相同。
5. `mutableCopy` 只复制容器这一层，元素仍是共享的。`initWithArray:copyItems:YES` 能做到一层元素深拷贝，再深就只能靠归档。
6. atomic 只保证单次访问器调用不撕裂。实测十万次并发自增，atomic 丢掉一半计数，甚至比 nonatomic 丢得更多；把读写整体加锁才能得到正确结果。
7. atomic 的锁按 ivar 地址分片，同一对象的不同属性之间没有互斥关系，跨属性的一致性它管不了。
8. `weak` 在对象销毁后置 `nil`，`unsafe_unretained` 留下悬垂指针——后者"有时候能跑"正是它危险的地方。
9. delegate 用 weak 的根本原因是所有权方向，循环引用只是用错 strong 的后果。
10. Block 属性的 `copy` 与 `strong` 在当前 ARC 下行为没有观察到差异，选哪个更多是可读性偏好，不是正确性问题。

## 参考资料

### 官方

- [Apple — Encapsulating Data](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/EncapsulatingData/EncapsulatingData.html)：atomicity 与 thread safety 的官方措辞出自这里
- [Apple — NSCopying](https://developer.apple.com/documentation/foundation/nscopying)
- [Apple objc4 源码](https://github.com/apple-oss-distributions/objc4)：`objc-accessors.mm` 里是 `objc_getProperty` / `reallySetProperty` 的完整实现

### 经典

- [objc.io — Value Objects](https://www.objc.io/issues/7-foundation/value-objects/)：解释了为什么不可变类的 `copyWithZone:` 应该 retain 而不是真拷贝
- [Cocoa with Love — Memory and thread-safe custom property methods](https://www.cocoawithlove.com/2009/10/memory-and-thread-safe-custom-property.html)：手写线程安全访问器的正误对照，注意文章写于 MRC 时期

### 拓展

- [pro648 — iOS 中定义属性时几个特性的区别](https://github.com/pro648/tips/wiki/)：转引了 objc4-781 的访问器源码，中文资料里少见的一手引用
- [Adlai-Holler — atomic property 反汇编分析](https://gist.github.com/Adlai-Holler/9e18b5a0f3f30e549b2a2faa54fdaf4f)：各属性组合的真实 ARM64 反汇编对照

### 本地

- [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]]
- [[iOS weak 的实现：SideTable 与置 nil 的时机]]
- [[iOS Block 的结构：ABI、descriptor 与三种类型]]

---

## 附：实验环境

| 项目 | 环境 |
| --- | --- |
| Xcode | 26.6 |
| 运行环境 | iOS Simulator（arm64，Apple Silicon Mac） |
| 构建 | `clang -fobjc-arc -target arm64-apple-ios17.0-simulator` |
| 运行 | `xcrun simctl spawn booted ./binary` |

本文所有输出块均为上述环境真实运行的 stdout。

两点需要注意的偏差：并发实验的具体丢失数量每次运行都不同，只有"大量丢失"这个结论是稳定的，不要引用具体数字；Foundation 内部类名（`__NSSingleObjectArrayI`、`__NSCFString` 等）属于实现细节，不同系统版本可能变化。

> 待补测：`objc_setProperty_atomic` / `objc_storeWeak` 的符号断点验证——在 Xcode 里对这几个符号下断点，赋值不同关键字组合的属性，观察哪些命中、哪些不命中，可以直接验证本文第一节那张"内联 vs 运行时"的分路表。
