---
title: 【iOS】ARC 的两半：编译器插桩与 runtime 支持
published: 2026-07-26
description: 编译器和运行时谁都看不见对方的代码，却要在每次方法返回时完成一次握手。用 LLVM IR 和两个架构的汇编，看清这个暗号是怎么对的，以及它在 arm64 上已经换过一代了。
tags:
  - iOS
  - Objective-C
  - Memory
  - ARC
  - LLVM
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 5
draft: true
---
# ARC 的两半：编译器插桩与 runtime 支持

面试里问"ARC 是什么"，最常见的回答是"编译器在编译时自动插入 retain 和 release"。这句话不算错，但它把一半的事实说没了。

真正的分工是：编译器决定在哪插、插哪一个函数，运行时决定这个函数具体怎么干。前者是纯静态分析，运行时对此一无所知；后者涉及引用计数存在 isa 的哪几位、要不要加锁、能不能走快速路径，编译器同样一无所知。

有一个地方能把这种分工看得特别清楚——方法返回值的所有权交接。两边谁都看不见对方的代码，只能隔着一个返回地址对暗号。这篇文章最长的一节花在这上面。

上一篇 [[iOS 内存：MRC 的所有权规则]] 讲规则本身，这篇讲规则怎么被翻译成代码。

---
![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260806211750893.png)


![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260806211757541.png)




## 先看编译器插了什么

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

## 那些桩在优化后大部分会消失

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

这还顺带解释了上一篇那个小疑点：为什么刚创建的对象读出来 `extra_rc` 是 2 而不是 1。`-O0` 下每个 `id` 参数都会被 `objc_storeStrong` 进一个本地槽位，那次多出来的 retain 是**被调方**在函数体开头做的，不在调用点。换 `-O1` 这个槽位直接消失。

---

## 返回值的所有权交接

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

### 要解决的问题

Cocoa 的约定是：方法名不以 `alloc`/`new`/`copy`/`mutableCopy` 开头，就该返回一个调用方不拥有的对象。老实的做法是被调方 `retain` 一次再 `autorelease` 一次，调用方要留住就自己再 `retain`。

一次简单的 getter 调用，为此产生了一次入池、一次出池、两次引用计数操作。而绝大多数情况下调用方拿到值马上就要持有它——中间这趟池完全是白跑的。更糟的是对象存活时间变得不可控：它要等池排空才真正释放，在没有 RunLoop 及时排空的线程上会堆积成内存峰值。

顺带一提，那个"方法名前缀"的约定在编译器里不是约定，是强制的类型信息。我给一个属性起名叫 `copyLabel`，编译直接失败：

```text
error: property follows Cocoa naming convention for returning 'owned' objects
note: explicitly declare getter '-copyLabel' with
      __attribute__((objc_method_family(none))) to return an 'unowned' object
```

编译器把这些前缀识别成 method family，并隐式给它们加上 `ns_returns_retained`。所以上一篇说的"命名约定纯靠自觉"，在 ARC 时代只对方法**实现**成立，方法**声明**这一侧编译器是会管的。

### 暗号怎么对

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

### x86_64 靠指令序列当签名

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

### IR 里的表达变过，但 marker 没消失

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

### 第四种收尾

返回值优化在调用方一侧其实有三个入口，前面只讲了一个。如果拿到的值根本不需要持有：

```objc
void unsafeSink(Box *b) { __unsafe_unretained NSString *s = [b direct]; }
```

生成的是 `objc_unsafeClaimAutoreleasedReturnValue`——握手成功就直接 release，失败就什么都不做。用途是避免"retain 一下马上又 release"这种无谓的往返。

---

## 编译器和运行时各自管什么

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

### 合成访问器有一条专门的捷径

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

## .cxx_destruct 是编译期生成的

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

对象销毁的完整链条在上一篇讲过。这里补一句：如果对象没有弱引用、没有关联对象、没有 C++ 析构、引用计数没溢出过，`rootDealloc` 会走快速路径直接 `free`，`.cxx_destruct` 根本不会被调用。

---

## 几个说法需要纠正

**"ARC 是垃圾回收。"** 不是。GC 要扫描对象图判断可达性，可能有停顿；ARC 是编译期插桩的确定性引用计数，没有扫描、没有停顿。这是两种完全不同的范式，ARC 从来没做过任何图遍历。

**"ARC 全是编译器做的，运行时只是被动执行。"** 返回地址差值的判定、`hasCustomRR` 的分支、`extra_rc` 溢出的迁移策略，全都是运行时主动做的动态决策。编译器只负责把调用点摆对位置。

**"ARC 会追踪对象之间的引用关系。"** 强引用不追踪。ARC 只维护每个对象自身的一个计数值，谁持有它、持有了几次，全部不记。唯一的例外是 `__weak`——运行时确实在 SideTable 里记着哪些槽指向我，好在销毁时逐个置 `nil`，但这份账不参与计数，也不构成可达性图。所以循环引用无解：检测环需要遍历强引用图，而这张图从来没被建出来过。

**"用了 ARC 就不会内存泄漏。"** 强引用环照样泄漏。Block 捕获 `self` 和 delegate 忘了标 weak 是两个最常见的来源，这个话题在 [[iOS Block 循环引用与 weak-strong dance]] 展开。

**"`.cxx_destruct` 是运行时动态生成的。"** 是编译期生成的真实方法，能在编译产物的方法列表里找到。

**"返回值优化靠一条固定的 `mov fp, fp` 实现。"** 只有 x86_64 是真的在读指令，而它读的还不是这条；arm64 早就换成返回地址比对了，marker 在那边只是兜底。

---

## 总结

ARC 的分工可以压缩成一句话：编译器做静态的所有权分析并决定插桩位置，运行时做动态的计数与快速路径判定，两边靠一组 `objc_*` 函数对接。

这句话有一个容易被误读的推论——插桩多不等于开销大。`-O0` 下 `setLabel` 那四个 `storeStrong`，到 `-O1` 一个不剩。但 ARC optimizer 只能成对消除，跨函数的所有权转移它不敢动，这才是发布构建里 ARC 仍有开销的真正原因。

分工最锋利的地方在返回值交接上。两边谁都看不见对方的代码，只能靠一个返回地址对暗号：arm64 把对象扣在 TLS 里等调用方来认领，靠返回地址差值是 4 还是 8 来判断；x86_64 走的仍是老路，从返回地址读机器指令、跳三次确认符号。暗号对不上就退回传统的 autorelease + retain，只赔性能不赔正确性——这个"失败也不会错"的设计，才是它敢在每一次方法调用上生效的原因。

`.cxx_destruct` 是同一套分工的另一个样本，方向反过来：编译器按类生成，运行时按继承链调用，而且在对象满足快速销毁条件时压根不调。

最后一条方法论，和上一篇一样：这篇里几处"网上说的不对"，包括我自己第一版写错的两处，都不是靠读更多文章发现的，而是换个 `-fobjc-runtime` 版本、换个 target、把手写 getter 和合成 getter 摆在一起编一次。凡是来自记忆的断言都出过错，凡是来自编译产物的断言都对。

再往下走就是 weak 的实现细节和 Block 的所有权语义，是这个系列接下来两篇的内容。

## 参考资料

### 规范与源码

- [Clang ARC Specification](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)：运行时函数清单、method family、unretained return values 的语义定义
- [objc4 — objc-object.h / NSObject.mm](https://github.com/apple-oss-distributions/objc4)：`callerAcceptsOptimizedReturn`、`prepareOptimizedReturn`、四个返回值优化函数
- [objc4 — objc-config.h](https://github.com/apple-oss-distributions/objc4)：`HAS_RETURNADDR_AUTORELEASE_ELISION` 这个开关决定了 arm64 走哪条路
- [llvm-project PR #138696](https://github.com/llvm/llvm-project/pull/138696)：上游 LLVM 支持生成 `objc_claimAutoreleasedReturnValue`，2025 年 5 月

### 文章

- [Mike Ash — Automatic Reference Counting](https://www.mikeash.com/pyblog/friday-qa-2011-09-30-automatic-reference-counting.html)：ARC 早期最清晰的整体讲解，但文中关于 getter 用哪个返回值优化函数的描述，和现在编译器的实际行为已经对不上
- [Mike Ash — When an Autorelease Isn't](https://www.mikeash.com/pyblog/friday-qa-2014-05-09-when-an-autorelease-isnt.html)：返回值优化误伤导致过早释放的真实调试案例
- [sunnyxx — ARC 下 dealloc 过程及 .cxx_destruct 的探究](https://blog.sunnyxx.com/2014/04/02/objc_dig_arc_dealloc/)：用 lldb watchpoint 实测 `.cxx_destruct` 的执行时机，但没提 `rootDealloc` 的快速路径会完全跳过它
- [Always Processing — objc_retain](https://alwaysprocessing.blog/2023/07/22/objc-retain)：跟着当前 objc4 走读引用计数实现

### 本地

- [[iOS 内存：MRC 的所有权规则]]
- [[iOS 属性关键字：从所有权推导，而不是从类型名猜]]
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
