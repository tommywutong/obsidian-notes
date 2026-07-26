---
title: 【iOS】Block 循环引用与 weak-strong dance
published: 2026-07-26
description: 只用 weakSelf 的 block 里，第一次读到 A，第二次读到 null——同一个 block，同一个变量。这就是 weak-strong dance 要解决的问题，也是它和「防循环引用」根本不是一回事的原因。
tags:
  - iOS
  - Objective-C
  - Block
  - Memory
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 10
draft: true
---
# Block 循环引用与 weak-strong dance

前两篇讲清楚了两件事：Block 捕获对象时会持有它（[[iOS Block 的结构：ABI、descriptor 与三种类型|靠 BLOCK_HAS_COPY_DISPOSE 对应的辅助函数]]），以及 `__block` 把变量搬到堆上让块内外共享（[[iOS Block 的变量捕获与 __block]]）。

循环引用是这两件事的直接推论，不是一个新知识点。所以这一篇不打算重复"什么是循环引用"，而是回答两个更实际的问题：常见的几种修法各自能不能用，以及 `weak-strong dance` 那个 `strongSelf` 到底在防什么——它防的和"循环引用"根本不是一回事。

---

## 一、五种写法，跑一遍

用一个 weak 探针就能判断有没有泄漏，不需要开 Memory Graph：对象离开作用域后，探针还非 nil 就是泄漏了。

```objc
@interface Node : NSObject
@property (nonatomic, copy) void (^handler)(void);
@end

static void scenario(const char *tag, void (^setup)(Node *)) {
    __weak Node *probe = nil;
    @autoreleasepool {
        Node *n = [Node new];
        probe = n;
        setup(n);
    }
    NSLog(@"    离开作用域后 probe = %@", probe ? @"还活着（泄漏）" : @"已释放");
}
```

五种写法的结果：

```text
① block 里直接用 self          还活着（泄漏）
② 改用 weakSelf                已释放
③ 用 __block 修饰对象          还活着（泄漏）
④ __block + 调用后手动置 nil    已释放
⑤ 只捕获需要的值               已释放
```

第 ① 种是经典环：`n` 持有 `handler`，`handler` 捕获并持有 `n`。两边互相钉住，谁也走不了。

第 ② 种是标准解法。捕获的是一个 `__weak` 变量，不产生持有关系，环断在这一环上。

第 ③ 种值得单独说，因为它是一条被广泛传抄的**过期知识**。

### `__block` 在 ARC 下打不破环

MRC 时代 `__block` 修饰的对象不会被 retain，所以 `__block Node *blockN = n;` 确实能断环。这个结论在很多老文章里，也被很多新文章原样抄了过去。

ARC 改了语义：`__block` 修饰的对象照样被持有。上面第 ③ 行的输出就是证据——换了 `__block` 之后依然泄漏。

第 ④ 种是这条老路在 ARC 下的补救版：`__block` 加上"在 block 内部执行完手动置 nil"。

```objc
__block Node *blockN = n;
n.handler = ^{ NSLog(@"%@", blockN.name); blockN = nil; };
n.handler();   // 必须真的执行
```

它确实能解开，但代价很大：**环的解除依赖 block 被执行**。如果这个 block 是一个可能永远不会触发的回调（网络失败、用户没点、页面提前关掉），环就永远留在那儿。而且它对读代码的人是隐形的——你得读到 block 内部才知道生命周期依赖执行。

我的判断是这种写法基本不该用。它唯一还算合理的场景是"这个 block 一定只执行一次且一定会执行"，但满足这个条件的话，用 `weakSelf` 同样能解决，还不用背这个额外假设。

第 ⑤ 种最省事：如果 block 里只需要某个属性的值，就在外面先取出来捕获那个值，压根别碰 `self`。能用这招的时候优先用它——没有环，也就不需要任何补救。

---

## 二、weakSelf 解决了什么，没解决什么

现在到了这篇真正想讲的部分。

`weakSelf` 断了环，但它引入了一个新问题：**block 执行到一半，对象可能就没了。**

测一下。让 block 在后台队列里读两次 weak 变量，中间人为制造一个空档，期间主线程释放对象：

```objc
__weak Worker *weakW = w;
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"    第一次读 weakW.tag = %@", weakW.tag);
    dispatch_semaphore_signal(sem);   // 通知主线程可以释放了
    usleep(50000);                     // 期间对象被释放
    NSLog(@"    第二次读 weakW.tag = %@", weakW.tag);
});
```

```text
    第一次读 weakW.tag = A
    [dealloc] A
    第二次读 weakW.tag = (null)   ← 同一个 block 内，值变了
```

同一个 block、同一个变量，前后两次读到不同的东西。

这不是并发的边角情况，它是 `__weak` 的正常语义——对象销毁时弱引用就该变 `nil`。问题在于 block 里的代码通常假设"我读到的这个对象在我这段逻辑里一直有效"，而这个假设不成立。

后果分两种。轻的是逻辑错乱：前半段基于对象状态做了判断，后半段对象没了，走进了不该走的分支。重的是空指针语义带来的静默失败——Objective-C 给 `nil` 发消息不崩，于是后半段的所有操作全部变成空操作，你什么都看不到。

### strongSelf 做的就是把对象钉住

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    __strong Worker *strongW = weakW;      // 就这一行
    if (!strongW) { return; }               // 拿不到就直接返回
    NSLog(@"    第一次读 strongW.tag = %@", strongW.tag);
    usleep(50000);
    NSLog(@"    第二次读 strongW.tag = %@", strongW.tag);
});
```

```text
    第一次读 strongW.tag = B
    第二次读 strongW.tag = B   ← 全程有效
    [dealloc] B
    block 结束后 weakW = 已释放
```

两次都读到 `B`，而且 dealloc 发生在 block 执行完之后。

注意最后一行：block 结束后弱引用显示已释放。这说明那个强引用是**临时**的——它只在 block 这一次执行期间存在，随局部变量出作用域而消失，不会形成新的环。

这是整个 dance 的关键，也是最容易被讲混的地方：

| | 解决的问题 | 存在时长 |
| --- | --- | --- |
| `weakSelf` | 断开 block 与对象之间的**永久**持有 | 与 block 同寿 |
| `strongSelf` | 保证对象在**这一次执行**期间不消失 | 一次调用 |

两者不是"一个操作的两个步骤"，是两个不同问题的两个答案。只是它们经常一起出现，被打包成了一个叫 "weak-strong dance" 的模板。

理解了这一点，就能回答一个常见困惑：为什么加了 `strongSelf` 不会重新造成循环引用？因为环的定义是"永久互相持有"，而 `strongSelf` 是个局部变量，block 一执行完就没了。对象在这期间多活了几十微秒，仅此而已。

### 什么时候可以不写 strongSelf

也不是每次都要写。判据是：**block 里对 self 的访问是不是只有一次。**

```objc
// 只用一次，weakSelf 足够
__weak typeof(self) weakSelf = self;
self.handler = ^{ [weakSelf reload]; };
```

单次访问不存在"读到一半变了"的问题——要么在那一刻有效，要么已经是 `nil`，`nil` 发消息安全。

需要 dance 的是这种：

```objc
__weak typeof(self) weakSelf = self;
self.handler = ^{
    if (weakSelf.isLoading) return;     // 第一次读
    [weakSelf startTask];                // 第二次读，此时可能已经是 nil
    weakSelf.state = Done;               // 第三次读
};
```

三次访问之间，对象随时可能消失，逻辑就散了。

不过说实话，判断"是不是只有一次"本身也是负担，而且代码会演化——今天一次，明天有人加了一行就变两次。所以工程上直接一律写 dance 也是合理的选择，成本只有两行。我自己的做法是：只要 block 里有分支或者多于一条语句，就写。

---

## 三、Block 判空解决什么，不解决什么

Objective-C 给 `nil` 发消息是安全的，但**调用一个空的 Block 指针不是**——那是直接跳到地址 0，必崩。所以到处能看到这个写法：

```objc
if (self.completion) {
    self.completion(result);
}
```

它解决的是"这个 block 属性从来没被赋值过"。这个场景很常见，判空是必要的。

但它不解决并发。上面这两行之间有个窗口：判空之后、调用之前，另一个线程可能把 `self.completion` 置成了 `nil`，而属性的旧值已经被 release。这时候调用的就是一个悬垂指针。

严格的写法是先取到本地：

```objc
void (^completion)(id) = self.completion;   // 取一次，拿到强引用
if (completion) {
    completion(result);
}
```

这样即使属性在中途被改，本地这份强引用保证被调用的 block 在整个调用期间有效。这跟上一节 `strongSelf` 是完全相同的手法——**把一个可能变化的引用先固定成一个不变的局部强引用**，只不过一个作用于 `self`，一个作用于 block 本身。

值得说明的是，这个竞态在实践中触发概率很低，很多代码库一直用简单的判空写法也没出过事。但它是真实存在的，而修复成本只是多一个局部变量。

---

## 四、几个说法需要辨析

**"`__block` 可以打破循环引用。"** MRC 下成立，ARC 下不成立。上面第 ① 节第 ③ 组实测泄漏。这条老知识在 ARC 项目里会让人写出自以为安全的泄漏代码，是本文最危险的一条。

**"加了 strongSelf 又形成了循环引用。"** 不会。环的定义是永久互相持有，`strongSelf` 是局部变量，block 执行完就释放。实测 block 结束后弱引用即显示已释放。

**"weak-strong dance 是为了防循环引用。"** 防环的是 `weakSelf` 那一半。`strongSelf` 那一半防的是执行期间对象消失，是另一个问题。把两者混为一谈，就解释不了"为什么 strongSelf 不会造成新的环"。

**"用了 weakSelf 就万事大吉。"** 见第二节。单次访问确实够用，多次访问会读到不一致的状态。

**"`if (block) block();` 是线程安全的。"** 不是。判空和调用之间存在窗口，见第三节。

**"所有 block 属性都要担心循环引用。"** 只有 self 持有的 block 才需要。像 `[UIView animateWithDuration:animations:]` 这种一次性执行完就释放的，或者 `enumerateObjectsUsingBlock:` 这类同步执行的，block 根本不会被长期持有，写 `weakSelf` 反而是噪音。判据是这个 block 有没有被存成属性或者被某个长寿对象持有。

---

## 总结

循环引用是"Block 持有对象、对象持有 Block"的直接推论，前两篇讲的捕获机制已经把原理说完了，这一篇只关心修法。

五种修法里，`weakSelf` 是标准解法；只捕获需要的值最省事，能用就用；`__block` 修饰对象在 ARC 下**打不破环**，那是 MRC 时代的结论；`__block` 加手动置 nil 能解开，但把生命周期押在"block 一定会被执行"上，不建议。

`weakSelf` 和 `strongSelf` 解决的是两个不同的问题：前者断开永久持有，后者保证单次执行期间对象不消失。实测同一个 block 里两次读 weak 变量能读到 `A` 和 `null`，就是后者存在的理由。而 `strongSelf` 是局部变量，执行完即释放，不会造成新的环。

`if (block) block()` 解决"从没赋值过"，不解决并发。要严格就先取到本地强引用——跟 `strongSelf` 是同一个手法。

第二周到这里就收完了。下一篇进入第三周：[[iOS Runtime 应用：Method Swizzling 与 KVO 的实现]]。

## 参考资料

### 官方

- [Apple — Working with Blocks](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html)：Avoid Strong Reference Cycles 一节
- [Clang ARC Specification](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)：`__weak` / `__strong` 的所有权语义

### 拓展

- [深入理解 weak-strong dance](https://luohs.github.io/2017/05/31/20170531/)
- [iOS 中的 block 是如何持有对象的](https://github.com/draveness/analyze/blob/master/contents/FBRetainCycleDetector/iOS%20%E4%B8%AD%E7%9A%84%20block%20%E6%98%AF%E5%A6%82%E4%BD%95%E6%8C%81%E6%9C%89%E5%AF%B9%E8%B1%A1%E7%9A%84.md)：从 FBRetainCycleDetector 的角度看 block 的持有关系
- [weak-strong dance 的注意事项](https://lvv.me/posts/2022/08/13_weak_strong_dance/)

### 本地

- [[iOS Block 的结构：ABI、descriptor 与三种类型]]
- [[iOS Block 的变量捕获与 __block]]
- [[iOS weak 的实现：SideTable 与置 nil 的时机]]

---

实验环境：Xcode 26.6，iOS 模拟器（arm64，Apple Silicon Mac），`clang -fobjc-arc -target arm64-apple-ios17.0-simulator`。所有输出块都是真实运行结果。

第二节那个实验用信号量和 `usleep` 人为拉开了时间窗口，目的是让一个真实存在但概率很低的竞态稳定复现。真实代码里这个窗口通常只有几微秒，触发概率低，但不等于不存在——这类 bug 的特征正是"线上偶现、本地永远复现不了"。

用 weak 探针判断泄漏是个轻量办法，适合写单元测试。但它只能判断"这一个对象有没有被释放"，查不出整条引用链。真要定位复杂的环，还是得用 Memory Graph Debugger 或 Instruments。
