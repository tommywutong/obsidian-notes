---
title: 【iOS】ARC 的两半：编译器插桩与 runtime 支持
published: 2026-07-26
description: ARC 既不是编译器全包，也不是运行时全包。用 LLVM IR 和两个架构的汇编，看清编译器在哪插桩、运行时做什么决策，以及那条什么都不干的 mov x29, x29 到底在跟谁打暗号。
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

真正的分工是：编译器决定**在哪插、插哪一个函数**，运行时决定**这个函数具体怎么干**。前者是纯粹的静态分析，运行时对此一无所知；后者涉及引用计数存在 isa 的哪几位、要不要加锁、能不能走快速路径，编译器同样一无所知。两边通过一组约定好的 `objc_*` 函数接口对接。

有一个地方能把这种分工看得特别清楚——返回值优化。它需要编译器在调用点摆一条什么都不干的指令，运行时再回过头去读这条指令来做判断。两边隔着机器码互相打暗号。这篇文章大半篇幅都花在这上面。

上一篇 [[iOS 内存：MRC 的所有权规则]] 讲的是规则本身，这篇讲规则怎么被翻译成代码。

---

## 先看编译器插了什么

最直接的办法是把一段普通的 ARC 代码编译成 LLVM IR。测试代码写六个函数，覆盖常见的所有权场景：

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

用 `clang -fobjc-arc -S -emit-llvm -O0` 编出来，把每个函数里的 `objc_*` 调用抽出来：

```text
--- localStrong ---   objc_alloc_init, storeStrong
--- setLabel ---      storeStrong × 4
--- makeBox ---       objc_alloc_init, retain, storeStrong, autoreleaseReturnValue
--- useBox ---        [asm marker], retainAutoreleasedReturnValue, storeStrong
--- weakDance ---     storeStrong, initWeak, loadWeakRetained, storeStrong,
                      destroyWeak, storeStrong, destroyWeak
--- poolScope ---     autoreleasePoolPush, objc_alloc_init, storeStrong, autoreleasePoolPop
```

几件事一眼就能看出来。

`[[Box alloc] init]` 没有变成两次 `objc_msgSend`，而是一个 `objc_alloc_init` 调用。这是 WWDC20 那次运行时改进带来的一组调用优化，同类的还有 `objc_opt_new`（对应 `[Class new]`）、`objc_opt_self`、`objc_opt_isKindOfClass`。它们和所有权无关，纯粹是把高频消息发送替换成直调运行时函数，省掉一次方法查找。

局部强引用出作用域时清理用的是 `objc_storeStrong(&slot, nil)`，不是直接 `objc_release`。这个函数把"retain 新值、赋值、release 旧值"三件事打包成一次调用，传 `nil` 就退化成纯释放。ARC 大量复用它，所以在 IR 里出现得非常频繁。

`weakDance` 那一行把 weak 的三件套完整暴露了：`initWeak` 建槽、`loadWeakRetained` 读、`destroyWeak` 销毁。读一个 weak 变量为什么要 retain，是因为读出来到用它之间对象可能在别的线程被释放——必须在运行时持锁的前提下把它 retain 住，要么拿到一个活着的对象，要么拿到 `nil`，不允许有中间态。这部分留给 [[iOS weak 的实现：SideTable 与置 nil 的时机]]。

---

## 那些桩在优化后大部分会消失

同一份代码换成 `-O1` 再编一次，对比就很有意思了：

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

`setLabel` 从四个 `storeStrong` 变成零。`useBox` 里整套返回值握手直接蒸发——因为 `makeBox` 被内联了，编译器发现对象从创建到销毁全在一个函数里，中间那些配平操作一个都不需要。

这回答了一个常见的疑虑："ARC 插了这么多桩，是不是很浪费？"在未优化构建下确实浪费，但 LLVM 有一个专门的 ObjCARC optimizer pass，会证明哪些 retain/release 对是冗余的然后消掉。发布构建里剩下的调用远比 `-O0` 看到的少。

这也解释了上一篇那个小疑点：为什么刚创建的对象读出来 `extra_rc` 是 2 而不是 0。`-O0` 下把对象传给一个函数，ARC 会插额外的 retain。换成 `-O1` 就没有了。

---

## 返回值优化：一条什么都不干的指令

`useBox` 在 `-O0` 下的完整 IR 是这样的：

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

中间那行 inline asm 就是全文的主角。编译成 arm64 汇编：

```asm
bl	_makeBox
; InlineAsm Start
mov	x29, x29	; marker for objc_retainAutoreleaseReturnValue
; InlineAsm End
bl	_objc_retainAutoreleasedReturnValue
```

`mov x29, x29` 把帧指针赋值给自己，是一条彻底的空操作。它存在的唯一目的是当暗号。

### 要解决的问题

Cocoa 的约定是：方法名不以 `alloc`/`new`/`copy`/`mutableCopy` 开头，就该返回一个调用方不拥有的对象。老实的做法是被调方 `retain` 一次再 `autorelease` 一次，调用方要留住的话自己再 `retain`。

一次简单的 getter 调用，为此产生了一次入池、一次出池、两次引用计数操作。而绝大多数情况下调用方拿到值马上就要持有它——中间这趟自动释放池完全是白跑的。更糟的是对象存活时间变得不可控：它要等到池排空才真正释放，在没有 RunLoop 及时排空的线程上会堆积成内存峰值。

### 怎么绕过去

被调方的 `objc_autoreleaseReturnValue` 不会立刻决定要不要 autorelease。它先做一件很反常的事：**读自己的返回地址，看调用方的下一条指令长什么样**。

```c
// arm64
static ALWAYS_INLINE bool
callerAcceptsOptimizedReturn(const void *ra)
{
    // fd 03 1d aa    mov fp, fp
    if (*(uint32_t *)ra == 0xaa1d03fd) {
        return true;
    }
    return false;
}
```

如果下一条指令正是那个 marker，说明调用方马上就要调 `objc_retainAutoreleasedReturnValue`。于是被调方在线程本地存储里设一个标记，把对象以 +1 的状态原样返回，**完全不进池**。调用方那边的 `objc_retainAutoreleasedReturnValue` 检查到这个标记，也跳过 retain，直接用。

一来一回，两次引用计数操作和一次入池全部省掉。

握手失败也不会有任何问题：被调方老实 `objc_autorelease`，调用方老实 `objc_retain`，退回到没有优化时的行为。这个机制在正确性上是完全安全的，只是性能红利吃不到。

### x86_64 不需要 marker

这一点值得单独说，因为它证明了 marker 不是什么魔法，只是一个"识别调用方意图"的手段——手段可以换。

同一份代码换成 x86_64 target，`-O0` 编译：

```asm
callq	_makeBox
movq	%rax, %rdi
callq	_objc_retainAutoreleasedReturnValue
```

没有 marker。因为 `movq %rax, %rdi` 紧跟一个 `callq` 到 `objc_retainAutoreleasedReturnValue`，这个指令序列本身就足够独特了。objc4 在 x86_64 上的判定就是直接匹配这串字节，顺带还要确认那个 call 的目标符号确实是 `objc_retainAutoreleasedReturnValue` 或 `objc_unsafeClaimAutoreleasedReturnValue`：

```c
// x86_64
// 48 89 c7    movq  %rax,%rdi
// e8          callq symbol
if (*ra4 != 0xe8c78948) return false;
...
if (*sym != objc_retainAutoreleasedReturnValue &&
    *sym != objc_unsafeClaimAutoreleasedReturnValue)
{
    return false;
}
return true;
```

arm64 的指令是定长的，`bl` 之后紧跟什么没有这么强的特征，所以需要额外插一条标记指令。x86_64 省下了这一条。

这个对照也顺手证伪了一个常见说法："ARC 的返回值优化靠一条固定的 `mov fp, fp` 实现"。只有 arm 系列需要它。

### 优化在 IR 里的表达变过

`-O1` 及以上，marker 不再以 inline asm 的形式出现在 IR 里，而是变成挂在调用指令上的一个 operand bundle：

```llvm
%5 = tail call ptr @"objc_msgSend$name"(...)
       [ "clang.arc.attachedcall"(ptr @llvm.objc.retainAutoreleasedReturnValue) ]
```

marker 该不该发射，决策从前端下沉到了后端，按目标架构决定。这个改动本身是为了给一个更新的函数铺路：`objc_claimAutoreleasedReturnValue` 完全不需要 marker，靠这个 operand bundle 就能完成绑定。

时间线上有个挺有意思的落差：这个免 marker 的机制在 Apple 自己的工具链里已经用了几年，但真正合并进开源上游 LLVM 是 2025 年的事。所以"这是不是新特性"这个问题，取决于你问的是运行时入口点什么时候出现，还是编译器什么时候真的开始生成它——两条时间线差了好几年。

---

## 编译器和运行时各自管什么

把上面的观察整理一下，边界其实相当清晰。

编译器管的是静态的部分：分析每个指针表达式的所有权限定符（`__strong` / `__weak` / `__unsafe_unretained` / `__autoreleasing`），这是纯类型系统的推导；决定在哪些语法位置插入哪个 `objc_*` 调用；生成 `.cxx_destruct` 方法；在 `-dealloc` 末尾自动补上对父类的转发；判断某个调用点是否满足 `objc_alloc` 这类模式替换的条件。

运行时管的是动态的部分：引用计数到底存在 isa 的哪几位、CAS 循环怎么写、溢出了往 SideTable 挪多少；类有没有重写 `retain`/`release`（`hasCustomRR()`），决定走快速路径还是发消息；弱引用表的哈希桶管理和对象销毁时的批量置 `nil`；以及上面那个读机器指令做判断的返回值优化。

一句话切开：**"这里要不要调 `objc_retain`"是编译器的问题，"`objc_retain` 怎么让计数加一"是运行时的问题。** 编译器从来不直接操作引用计数存储，它只会生成函数调用。

Clang ARC 规范里列出的运行时函数不算多，按用途归一下：

| 用途 | 函数 |
| --- | --- |
| 基本增减 | `objc_retain` / `objc_release` / `objc_autorelease` |
| 强引用赋值 | `objc_storeStrong` |
| 弱引用 | `objc_initWeak` / `objc_storeWeak` / `objc_loadWeak` / `objc_loadWeakRetained` / `objc_destroyWeak` / `objc_copyWeak` / `objc_moveWeak` |
| 返回值优化（被调方） | `objc_autoreleaseReturnValue` / `objc_retainAutoreleaseReturnValue` |
| 返回值优化（调用方） | `objc_retainAutoreleasedReturnValue` / `objc_unsafeClaimAutoreleasedReturnValue` / `objc_claimAutoreleasedReturnValue` |
| 自动释放池 | `objc_autoreleasePoolPush` / `objc_autoreleasePoolPop` |
| Block | `objc_retainBlock` |

上表里返回值优化那两栏，实际生成频率差别很大。`objc_autoreleaseReturnValue` 和 `objc_retainAutoreleasedReturnValue` 是绝对主力；`objc_retainAutoreleaseReturnValue`（注意少了个 "d"，是给"返回一个当前 +0 的值"用的）在我这次的实验里一次都没出现过。

顺带一个反常的发现：合成的 `nonatomic strong` 属性 getter，在 `-O0` 下**一个 `objc_*` 调用都不生成**，就是裸的取地址、load、返回。因为编译器判定，调用方持有 `self` 期间那个 ivar 不可能被释放，retain/autorelease 整套都可以省掉。所以"getter 会做 retain + autorelease"这个流传很广的说法，至少在这种最常见的情况下是不成立的。

---

## .cxx_destruct 是编译期生成的

ARC 下不用手写释放 ivar 的代码，那这活谁干的？

答案是一个叫 `.cxx_destruct` 的方法。名字来自 C++——它本来是为带 C++ 成员对象的类做析构用的，ARC 把它借来承载 ivar 的自动释放。

编译产物里能直接看到它：

```llvm
define internal void @"\01-[Box .cxx_destruct]"(ptr noundef %0, ptr noundef %1) #1 {
  ...
  call void @llvm.objc.storeStrong(ptr %6, ptr null) #2
  ...
}
```

就是对每个 ARC 托管的 ivar 调一次 `objc_storeStrong(&ivar, nil)`。这个方法由 Clang 在编译期生成，挂进类的实例方法列表；运行时在对象销毁流程里通过普通的方法查找把它调起来。生成方和调用方分属两侧——这个小细节本身就是"编译器插桩 + 运行时支持"这套分工的缩影。

对象销毁的完整链条在上一篇讲过，这里只补一句：如果对象没有弱引用、没有关联对象、没有 ARC 托管的 ivar、引用计数没溢出过，`rootDealloc` 会走快速路径直接 `free`，`.cxx_destruct` 根本不会被调用。

---

## 几个说法需要纠正

**"ARC 是垃圾回收。"** 不是。GC 要扫描对象图判断可达性，可能有停顿；ARC 是编译期插桩的确定性引用计数，没有扫描、没有停顿，对象在最后一次 release 后立即销毁（除非它在自动释放池里等着）。这是两种完全不同的范式，ARC 从来没做过任何图遍历。

**"ARC 全是编译器做的，运行时只是被动执行。"** 返回值优化的判定逻辑、`hasCustomRR` 的分支、`extra_rc` 溢出的迁移策略，全都是运行时主动做的动态决策。编译器只负责把调用点摆对位置。

**"ARC 会追踪对象之间的引用关系。"** 不会。ARC 只维护每个对象自身的一个计数值，它不知道也不需要知道是谁持有它。正因为如此，循环引用它无能为力——检测环需要遍历引用图，而这恰恰是引用计数不做的事。

**"用了 ARC 就不会内存泄漏。"** 强引用环照样泄漏，Block 捕获 `self` 和 delegate 忘了标 weak 是两个最常见的来源。这个话题在第二周的 [[iOS Block 循环引用与 weak-strong dance]] 展开。

**"`.cxx_destruct` 是运行时动态生成的。"** 是编译期生成的真实方法，能在编译产物的方法列表里找到它。

---

## 总结

ARC 的分工可以压缩成一句话：编译器做静态的所有权分析并决定插桩位置，运行时做动态的计数与快速路径判定，两边靠一组 `objc_*` 函数对接。

围绕这句话有四个值得记住的具体事实。第一，`-O0` 下看到的插桩密度不代表发布构建，LLVM 的 ARC optimizer 会消掉大部分冗余对，`setLabel` 那个从四个 `storeStrong` 变成零的例子最能说明问题。第二，返回值优化靠"调用方在调用点摆一条特定指令、被调方回读这条指令"完成握手，arm64 需要 `mov x29, x29` 这条空操作，x86_64 靠指令序列本身当签名，不需要额外指令。第三，握手失败会安全退化成传统的 autorelease + retain，这个优化只影响性能不影响正确性。第四，`.cxx_destruct` 是编译器生成、运行时调用的，而且在对象满足快速销毁条件时压根不会被调用。

再往下走就是 weak 的实现细节和 Block 的所有权语义，分别是这个系列接下来两篇的内容。

## 参考资料

### 规范与源码

- [Clang ARC Specification](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)：运行时函数清单和语义定义的唯一权威出处
- [objc4 — objc-object.h](https://github.com/apple-oss-distributions/objc4)：`callerAcceptsOptimizedReturn`、`prepareOptimizedReturn`、`rootDealloc` 都在这里
- [objc4 — NSObject.mm](https://github.com/apple-oss-distributions/objc4)：四个返回值优化函数的实现

### 文章

- [Mike Ash — Automatic Reference Counting](https://www.mikeash.com/pyblog/friday-qa-2011-09-30-automatic-reference-counting.html)：ARC 早期最清晰的整体讲解，但文中关于 getter 用哪个返回值优化函数的描述，和现在编译器的实际行为已经对不上
- [Mike Ash — When an Autorelease Isn't](https://www.mikeash.com/pyblog/friday-qa-2014-05-09-when-an-autorelease-isnt.html)：一个返回值优化误伤导致过早释放的真实调试案例
- [sunnyxx — ARC 下 dealloc 过程及 .cxx_destruct 的探究](https://blog.sunnyxx.com/2014/04/02/objc_dig_arc_dealloc/)：用 lldb watchpoint 实测 `.cxx_destruct` 的执行时机
- [Always Processing — objc_retain](https://alwaysprocessing.blog/2023/07/22/objc-retain)：跟着当前 objc4 走读引用计数实现，不涉及返回值优化

### 本地

- [[iOS 内存：MRC 的所有权规则]]
- [[iOS 属性关键字：从所有权推导，而不是从类型名猜]]
- [[iOS weak 的实现：SideTable 与置 nil 的时机]]

---

## 附：实验环境与复现

| 项目 | 环境 |
| --- | --- |
| Xcode | 26.6（Apple clang 21） |
| 目标 | `arm64-apple-ios17.0-simulator` 与 `x86_64-apple-ios17.0-simulator` |
| 命令 | `clang -fobjc-arc -S -emit-llvm -O0/-O1` 看 IR；去掉 `-emit-llvm` 看汇编 |

文中所有 IR 和汇编片段都是上述命令的真实输出。想自己复现，把测试代码存成 `.m` 文件，然后：

```shell
SDK=$(xcrun --sdk iphonesimulator --show-sdk-path)
clang -fobjc-arc -S -emit-llvm -O0 -isysroot "$SDK" \
      -target arm64-apple-ios17.0-simulator -o out_O0.ll own.m
clang -fobjc-arc -S -O0 -isysroot "$SDK" \
      -target x86_64-apple-ios17.0-simulator -o out_x86.s own.m
```

换 `-O1` 再编一遍做对照，差异最明显的是 `setLabel` 和 `useBox` 两个函数。

需要注意的是，IR 里的函数名带 `llvm.objc.` 前缀（如 `llvm.objc.storeStrong`），这是 LLVM 的 intrinsic 形式，最终会降级成对同名运行时函数的调用；汇编里看到的才是真实的符号名 `_objc_storeStrong`。
