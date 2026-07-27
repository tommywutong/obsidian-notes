---
title: 【iOS】事件传递与响应者链：hitTest、手势与那些点不动的按钮
published: 2026-07-27
description: 你永远设不出 alpha == 0.01，因为它存的是 32 位 float。从一次 hitTest 的完整递归轨迹讲起，把四个返回 nil 的条件、越界子视图和响应者链逐条测一遍。
tags:
  - iOS
  - UIKit
  - 事件传递
  - 响应者链
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 19
draft: true
---
# 事件传递与响应者链：hitTest、手势与那些点不动的按钮

有个 bug 我在三个项目里各遇到过一次：按钮画在屏幕上，看得见，点不动。

第一次我以为是 `userInteractionEnabled` 忘了打开，查了半天全是 YES。第二次才想明白：那个按钮的一部分伸到了父视图的 `bounds` 外面。父视图先被问"这个点在你身上吗"，它说不在，整条分支就被剪掉了，子视图连被问的机会都没有。

命中测试的规则其实只有几行。麻烦的是每一条都有人记错。

先把本文最硬的一条摆前面。Apple 文档写的是 hitTest 会忽略 "alpha level less than `0.01`"，中文圈普遍转述成"alpha ≤ 0.01 点不到"。两句话矛盾，但两句话在实践中都对——**因为 `UIView.alpha` 存的是 32 位 float，你写 `alpha = 0.01` 时真正存进去的是 0.0099999997764825821，它本来就小于 0.01。** 那个"等于 0.01"的情况根本构造不出来。第三节有逼到 ulp 级别的实测。

前一篇 [[iOS 对象通信：delegate、通知、target-action 与 block 回调]] 结尾欠了一笔账。`targetForAction:withSender:` 怎么沿响应者链找 target，当时在命令行进程里跑不了。这篇还上，见第八节。

---

## 一、先说清楚这些数据是在哪跑的

全文的实验都是 Mac Catalyst 二进制，在我自己的 Mac 上直接 `./labA` 跑出来的。没有模拟器，没有真机：

```shell
SDK=$(xcrun --sdk macosx --show-sdk-path)
clang -fobjc-arc -target arm64-apple-ios17.0-macabi -isysroot "$SDK" \
      -iframework "$SDK/System/iOSSupport/System/Library/Frameworks" \
      -framework Foundation -framework UIKit -framework CoreGraphics -o labA labA.m
./labA
```

编出来是原生 macOS 可执行文件，但链接的是真正的 UIKitCore：

```text
UIKit    = /System/iOSSupport/System/Library/PrivateFrameworks/UIKitCore.framework
UIDevice = iPad / iPadOS 26.5
```

`hitTest:withEvent:` 和 `pointInside:withEvent:` 都是普通方法，`event` 传 `nil` 完全合法。所以命中测试这一段我根本不需要事件，甚至不需要 `UIApplication`。造一棵 `UIView` 树，直接调就行。前七节的所有输出都来自一个没有 window、没有 app 的进程。

这个方法有边界，我把它写在前面而不是藏在末尾：

- 能验的：命中测试算法本身、`pointInside:` 的调用顺序与次数、四个提前返回条件、坐标换算、`nextResponder` 的指向。这些是 UIKitCore 里同一份代码。
- 不能验的：真实触摸的输入路径、`UIApplication sendEvent:` 的上游、手势识别器和 `UIControl` 抢触摸的时序。Catalyst 上鼠标点击走的是 indirect pointer 那条路，和 iOS 的直接触摸不是一回事，我不拿它当 iOS 结论。做不了的部分在第九节如实说明。
- 还有一条容易忽略的：`UIDevice.currentDevice.userInterfaceIdiom` 在这里返回 1（iPad），不是 5（Mac）。凡是结论可能受 idiom 影响的，都得复核。

---

## 二、一次 hitTest 到底走了哪些路

造一棵三层树，每层都子类化，在 `hitTest:` 和 `pointInside:` 里打印自己和入参：

```objc
@implementation V
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    BOOL r = [super pointInside:point withEvent:event];
    TLOG(@"pointInside %@ (%.1f,%.1f) -> %@", self.n, point.x, point.y, r ? @"YES" : @"NO");
    return r;
}
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    TLOG(@"hitTest  IN  %@ (%.1f,%.1f)", self.n, point.x, point.y);
    gDepth++;
    UIView *v = [super hitTest:point withEvent:event];
    gDepth--;
    TLOG(@"hitTest  OUT %@ -> %@", self.n, v ?: @"nil");
    return v;
}
@end
```

树长这样，A 先 `addSubview`，B 后 `addSubview`：

```text
root (0,0,400,800)
├─ A (50,100,200,200)
│   ├─ A1 (20,20,100,100)
│   └─ A2 (150,150,100,100)     ← 右下角伸出 A 的 bounds
└─ B (150,250,200,100)
    └─ deep (10,10,60,60)
```

点 (100,150)，落在 A1 上。完整轨迹：

```text
hitTest  IN  root (100.0,150.0)
  pointInside root (100.0,150.0) -> YES
  hitTest  IN  B (-50.0,-100.0)
    pointInside B (-50.0,-100.0) -> NO
  hitTest  OUT B -> nil
  hitTest  IN  A (50.0,50.0)
    pointInside A (50.0,50.0) -> YES
    hitTest  IN  A2 (-100.0,-100.0)
      pointInside A2 (-100.0,-100.0) -> NO
    hitTest  OUT A2 -> nil
    hitTest  IN  A1 (30.0,30.0)
      pointInside A1 (30.0,30.0) -> YES
    hitTest  OUT A1 -> <A1>
  hitTest  OUT A -> <A1>
hitTest  OUT root -> <A1>
结果 = <A1>   hitTest 调用 5 次，pointInside 调用 5 次
```

五件事一次说清。

第一，**父视图先被问，`pointInside:` 返回 YES 之后才轮到子视图**。root 的 `pointInside` 先打印，然后才出现 B 和 A。这是个前序遍历。

第二，遍历子视图的顺序是倒的。`root.subviews` 是 `(A, B)`，但先被问的是 B。后添加的视图画在上面，命中测试当然要先问它。`A` 的两个子视图也一样：`(A1, A2)` 里 A2 先被问。

验证一下这个顺序真的跟着 `subviews` 数组走。调一次 `bringSubviewToFront:A`，同一个点重测：

```text
root.subviews = ( <B>, <A> )
hitTest  IN  root (200.0,270.0)
  pointInside root (200.0,270.0) -> YES
  hitTest  IN  A (150.0,170.0)
    pointInside A (150.0,170.0) -> YES
    hitTest  IN  A2 (0.0,20.0)
      pointInside A2 (0.0,20.0) -> YES
    hitTest  OUT A2 -> <A2>
  hitTest  OUT A -> <A2>
hitTest  OUT root -> <A2>
```

同一个点，结果从 `<deep>` 变成 `<A2>`。层级顺序改了，命中结果就跟着改。这就是 `bringSubviewToFront:` 的全部作用。

第三，坐标是逐层换算的。传给 B 的是 (-50,-100)，负数，因为已经转到 B 自己的坐标系了。`hitTest:` 的头文件注释写得很直白：

> recursively calls -pointInside:withEvent:. point is in the receiver's coordinate system

第四，命中之后立刻回溯，不再问剩下的兄弟。A1 返回非 nil，A 就直接把它往上传，A 后面没有别的子视图要试。

第五，没命中任何子视图时返回自己。点空白处 (380,700)，B 和 A 都说不在，root 最后返回 `<root>`。这就是为什么 `hitTest` 极少返回 nil。只有连根视图的 `pointInside` 都是 NO 时才会。我测了 (-10,-10)，那次确实拿到 `(null)`。

### 默认实现等价于什么

把上面五条合起来，`UIView` 的默认实现基本就是这段：

```objc
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    if (self.hidden || !self.userInteractionEnabled || self.alpha < 0.01) return nil;
    if (![self pointInside:point withEvent:event]) return nil;
    for (UIView *sub in [self.subviews reverseObjectEnumerator]) {
        CGPoint p = [self convertPoint:point toView:sub];
        UIView *hit = [sub hitTest:p withEvent:event];
        if (hit) return hit;
    }
    return self;
}
```

三个属性的检查排在 `pointInside:` 前面，这一点可以直接看出来。把 A 的 `userInteractionEnabled` 关掉再跑：

```text
  hitTest  IN  A (50.0,50.0)
  hitTest  OUT A -> nil
```

A 的 `pointInside` 一次都没被调用。三个开关是短路的。代价为零。

`pointInside:` 的默认实现是 `CGRectContainsPoint(self.bounds, point)`，头文件写的是 "default returns YES if point is in bounds"。这是个半开区间，右边界和下边界不算命中：

```text
左上角 (50,100)        -> <A>
右下角 (250,300)       -> <root>
(249.999,299.999)      -> <A>
CGRectContainsPoint(A.bounds, (200,200))         = 0
CGRectContainsPoint(A.bounds, (199.999,199.999)) = 1
```

`transform` 我也测了。给 A 加 45 度旋转，`bounds` 不变而 `frame` 变成 282×282：

```text
点 (150,200) 旋转中心      -> <A1>
点 (60,110)  旋转前在 A 内  -> <root>
点 (150,80)  旋转后才进 A   -> <A>
```

命中测试跟着 `transform` 走，坐标换算用的是 layer 的几何。旋转之后原来能点的地方点不到了。

---

## 三、四个返回 nil 的条件，逐条实测

点 (100,150) 打在 A1 上，基准结果是 `<A1>`。逐个开关拨一次：

```text
基准                                 -> <A1>
A1.hidden = YES                      -> <A>
A1.alpha = 0.0                       -> <A>
A1.userInteractionEnabled = NO       -> <A>
A.hidden = YES（父）                 -> <root>
A.userInteractionEnabled = NO（父）  -> <root>
点 (-10,-10)（pointInside 全 NO）    -> (null)
```

前三条只让那一个视图退出，命中落回它的父视图。作用在父视图上时整棵子树一起消失。

`hidden` 和 `alpha` 有个细节值得单说：它们根本不存在 `UIView` 上。我在 `UIView` 的 ivar 列表里一个 alpha 相关的字段都没找到，两边其实是同一份数据：

```text
设 view.alpha = 0.35
  view.alpha        = 0.34999999403953552
  layer.opacity     = 0.34999999403953552
反过来设 layer.opacity = 0.75
  view.alpha        = 0.75
设 view.hidden=YES -> layer.hidden=1
设 layer.hidden=NO -> view.hidden=0
```

绕过 view 直接改 layer，命中测试照样认。0.35 读回来变成 0.34999999403953552，原因也在这里。`CALayer.opacity` 的返回类型是 `f`，`UIView.alpha` 的返回类型是 `d`。中间过了一道 float。

### alpha 的边界到底在哪

这就要说回开头那条。先扫一遍：

```text
alpha 设 0.0099990  读回 0.009999000  -> <A>
alpha 设 0.0100000  读回 0.010000000  -> <A>
alpha 设 0.0100001  读回 0.010000100  -> <A1>
alpha 设 0.0110000  读回 0.011000000  -> <A1>
```

看起来就是流传的"≤ 0.01 不响应"。我一开始也是这么记的。但把精度打满，故事完全变了：

```text
你写 view.alpha = 0.01，真正存进去的是 0.0099999997764825821
它比十进制的 0.01 小 2.24e-10
存 0.0099999997764825821 -> hitTest <A>
存 0.010000000707805157 -> hitTest <A1> （比 0.01 大 7.08e-10）
传 double 0.01（0.01）读回 0.0099999997764825821，相等吗 0
```

用 `nextafterf` 逐个 ulp 走一遍，边界干干净净地卡在 float 表示的 0.01 上：

```text
  float 0.0099999997764825821 (第 0 个 ulp) -> <A>
  float 0.010000000707805157 (第 1 个 ulp) -> <A1>
  float 0.0099999988451600075 (下 1 ulp)   -> <A>
```

所以这件事是这样的：Apple 文档说的 "less than `0.01`" 是准确的，中文圈说的"≤ 0.01"在可观测行为上也永远正确。原因不在 UIKit 的比较符号，在于 `alpha` 存成 float 之后，十进制 0.01 落在了 0.01 的下方。你没有办法让 `alpha` 真的等于 0.01，所以 `<` 和 `<=` 这两种实现在外部完全无法区分。我试过用 `nextafterf` 卡在正中间，卡不出来。float 里 0.01 和它的下一个数之间没有别的数。

能被点到的最小 alpha 是 0.010000000707805157。这个数我没在任何文章里见过。

它是个实现细节，别拿去写业务代码。有价值的是过程。

实用价值在哪？想要一个"看不见但能点"的视图，别用 `alpha = 0.01` 去踩线，那是错的一侧。用 `backgroundColor = clearColor`，`alpha` 保持 1：

```text
A1.backgroundColor = clearColor, alpha=1 -> <A1>
```

透明和不可交互是两件事。`clearColor` 只管绘制。参与命中测试的是 `alpha`。

### alpha 是逐层判断的

一个连带问题：父子视图各 0.1，视觉上叠加成 0.01，命中测试会不会认这个叠加值？

```text
A=0.02 A1=0.02（视觉上叠加 0.0004）-> <A1>
A=0.009 A1=1.0                     -> <root>
```

不认。每一层只看自己。A 和 A1 各 0.02 时视觉上已经几乎全透明，照样能点到 A1。反过来 A 自己是 0.009，A1 是完全不透明的 1.0，整棵子树一起出局。

### 几个默认值

```text
UIView                   userInteractionEnabled = YES
UILabel                  userInteractionEnabled = NO
UIImageView              userInteractionEnabled = NO
UIButton                 userInteractionEnabled = YES
UIScrollView             userInteractionEnabled = YES
UIStackView              userInteractionEnabled = YES
UIActivityIndicatorView  userInteractionEnabled = YES
UIProgressView           userInteractionEnabled = YES
```

`UILabel` 和 `UIImageView` 默认是 NO，其余都是 YES。给 label 加手势不生效是新手最常见的一个坑，原因就在这一行。我猜 `UIActivityIndicatorView` 也是 NO，测出来是 YES，猜错了。

---

## 四、越界的子视图为什么点不到

回到开头那个 bug。A2 的 frame 是 `(150,150,100,100)`，而 A 的 bounds 只有 200×200，所以 A2 右下角那一块伸到了 A 外面：

```text
A2 在 root 坐标系里的矩形 = {{200, 250}, {100, 100}}
A  在 root 坐标系里的矩形 = {{50, 100}, {200, 200}}
```

点 (270,320)。这个点在 A2 的可见范围内，屏幕上明明画着东西：

```text
hitTest  IN  root (270.0,320.0)
  pointInside root (270.0,320.0) -> YES
  hitTest  IN  A (220.0,220.0)
    pointInside A (220.0,220.0) -> NO
  hitTest  OUT A -> nil
hitTest  OUT root -> <root>
```

A 说"不在我身上"，A2 根本没被问过。Apple 文档把这个行为写得很清楚：

> This method doesn't report points that lie outside the view's bounds as hits, even if they actually lie within one of the view's subviews.

改法是重写 A 的 `pointInside:`，把子视图占的区域也算进来：

```objc
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    if ([super pointInside:point withEvent:event]) return YES;
    for (UIView *sub in self.subviews) {
        if (!sub.hidden && sub.alpha > 0.01 && CGRectContainsPoint(sub.frame, point)) return YES;
    }
    return NO;
}
```

同一个点再跑：

```text
  hitTest  IN  A (220.0,220.0)
    pointInside A (220.0,220.0) -> YES
    hitTest  IN  A2 (70.0,70.0)
      pointInside A2 (70.0,70.0) -> YES
    hitTest  OUT A2 -> <A2>
  hitTest  OUT A -> <A2>
结果 = <A2>
```

通了。

注意循环里那两个判断。不加的话，一个已经藏起来的子视图也会把它的区域放行，制造出一块"什么都点不到但吞掉了触摸"的死区。这个洞我自己踩过。

### clipsToBounds 是个障眼法

Apple 那句话后面还跟了半句，说这种情况会在 `clipsToBounds` 为 false 时发生。很多中文文章据此写成"`clipsToBounds` 会影响 hitTest"。我把两个值都测了：

```text
A.clipsToBounds=YES, 重写了 pointInside -> <A2>
A.layer.masksToBounds=YES               -> <A2>
```

**`clipsToBounds` 对命中测试没有任何影响。** 它只管绘制要不要裁剪。开着它，A2 超出去的部分在屏幕上被裁掉了，但只要 A 的 `pointInside:` 放行，那块看不见的区域照样可点。

这个组合能造出一种很难查的 bug：视图被裁得看不见，触摸却被它接走了。Apple 原文提 `clipsToBounds`，是在描述你什么时候会注意到这个现象。不裁剪时你能看见那块伸出去的内容，于是才会奇怪它为什么点不动。那句话讲的是可见性，不是命中规则。

---

## 五、扩大点击区域，三种写法

设计给的按钮 32×32，能点的范围太小。三种常见方案，我都跑了一遍。

方案一，重写 `pointInside:`。

```objc
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    return CGRectContainsPoint(CGRectInset(self.bounds, -20, -20), point);
}
```

```text
padding = 0   -> <A>
padding = 20  -> <B>
B.bounds 没变：{{0, 0}, {200, 100}}，B.frame 没变：{{150, 250}, {200, 100}}
```

`bounds` 和 `frame` 一个字节都没动。布局、约束、绘制全不受影响。

方案二，套一个更大的透明容器。 把按钮放进一个四周各大 20pt 的父视图里：

```text
hitTest  IN  wrap (70.0,10.0)
    pointInside wrap (70.0,10.0) -> YES
    hitTest  IN  B (50.0,-10.0)
      pointInside B (50.0,-10.0) -> NO
    hitTest  OUT B -> nil
  hitTest  OUT wrap -> <wrap>
结果 = <wrap>（点到的是容器，不是 B 本身）
```

这就是这个方案最大的问题：命中的是容器，不是按钮。`UIControl` 的 target-action 收不到，你得把手势挂到容器上再转发一层。而且多一层视图、多一层布局。

方案三，直接把 frame 撑大，内容用 inset 往回缩。

```text
B.frame 撑大到 {{130, 230}, {240, 140}} -> <B>
代价：B 的 bounds 变成 {{0, 0}, {240, 140}}，内部布局全部要跟着改
```

`UIButton` 的 `contentEdgeInsets`（iOS 15 起是 `UIButton.Configuration` 的 `contentInsets`）走的就是这条路。按钮自己知道怎么把内容缩回去，所以用在 `UIButton` 上是干净的。自定义视图用这招，所有子视图的位置都得重算。

我自己的选择：**默认用方案一，写成一个 `UIView` 的分类，只在 `pointInside:` 里加一个 `hitTestInsets` 属性。** 理由是它唯一不改变几何。方案三只在 `UIButton` 上用，因为那是官方 API 支持的路。方案二我基本不用，多一层视图换来一个"点到的不是自己"的麻烦，不划算。

有一条边界要记住：方案一扩出去的区域仍然受制于父视图。父视图的 `pointInside:` 说不在，你在子视图里怎么扩都没用。这和第四节是同一件事。

---

## 六、hitTest 被调用多少次

一次自顶向下的命中测试，每个被访问到的视图各调一次 `hitTest:`，路径被剪掉的分支不再往下走：

```text
命中最深的 A1   -> <A1>   hitTest 5 次   pointInside 5 次
命中 deep       -> <deep> hitTest 3 次   pointInside 3 次
落在 root 空白  -> <root> hitTest 3 次   pointInside 3 次
完全在 root 外  -> nil    hitTest 1 次   pointInside 1 次
```

次数跟树的形状走，不跟深度单独走。给 root 再挂 50 个兄弟视图，点一个谁都不覆盖的位置：

```text
root 有 52 个子视图，点空白处：hitTest 53 次  pointInside 53 次
```

53 次。每个兄弟都得问一遍。同理，往下嵌 30 层，那条链上每层各一次。

所以在这两个方法里写重逻辑是要付代价的。读文件、查数据库、做正则匹配，或者随手 `[self.subviews sortedArrayUsingComparator:]` 一下。代价会按视图树的规模放大。我见过一份代码在 `pointInside:` 里调 `layoutIfNeeded`，滚动时直接卡住。

一次真实触摸里 UIKit 会调几次，我在 Catalyst 上没测出来。计数需要真实触摸事件。而 Catalyst 上的鼠标点击走的是 indirect pointer 那条路，时序和 iOS 不一样。测出来的数字我不敢往 iOS 上搬。

> 待真机补测：在真机上给一个自定义 `UIWindow` 覆写 `hitTest:withEvent:`，内部用静态计数器记次数，同时在 `sendEvent:` 里按 `UITouch.phase` 分段打印"本次 sendEvent 期间计数增加了多少"。一次完整的按下-抬起需要观察三个数：整个手势周期里 `hitTest:` 的总次数、`phase == UITouchPhaseBegan` 那一次事件内部的次数、以及移动过程中还会不会重新做命中测试。预期是 Began 阶段至少一次；手势识别器参与判定时可能追加。这个数字我不猜。

---

## 七、响应者链的真实走向

命中测试结束，第一阶段就完了。它只回答一个问题：这次触摸交给谁。接下来是第二阶段，谁真正处理它。Apple 把两件事分得很清楚：

> UIKit uses view-based hit-testing to determine where touch events occur. ... The `hitTest(_:with:)` method of `UIView` traverses the view hierarchy, looking for the deepest subview that contains the specified touch

> Unhandled events are passed from responder to responder in the active responder chain

从最深的视图一路 `nextResponder` 打到底，这是我在 Catalyst 上实测的输出：

```text
起点：deep（A 的子视图）
   0. V  (deep)
   1. V  (A)
   2. V  (root)
   3. RootVC
   4. UIDropShadowView
   5. UITransitionView
   6. UIWindow
   7. UIWindowScene
   8. TracingApp
   9. AppDelegate
```

和几乎所有中文文章画的那张图对不上。流传最广的版本是这个：`view → superview → ... → UIViewController → UIWindow → UIApplication → AppDelegate`。我引的那篇 SwiftRocks 写的是 `MyView → MyViewController → UIWindow → UIApplication → AppDelegate`。实际链条上多了三个节点。

分开说，因为它们的可信度不一样。

`UIWindowScene` 出现在 `UIWindow` 和 `UIApplication` 之间，我认为在 iOS 上也是这样。`UIScene` 是 `UIResponder` 的子类，这一点可以直接查到：

```text
UIWindowScene  isResponder=1  UIWindowScene -> UIScene -> UIResponder -> NSObject
```

iOS 13 引入 scene 之后链条就该多这一节。Apple 那份文档还写着"The window's next responder is the `UIApplication` object"，没跟上 scene。

`UIDropShadowView` 和 `UITransitionView` 这两层我不敢下结论。它们是窗口托管视图，Catalyst 把 iOS 界面装进 macOS 窗口时会插入自己的一套包装。我只能说：在我这个 Catalyst 进程里，`RootVC.nextResponder` 是 `UIDropShadowView` 而不是 `UIWindow`。

> 待真机补测：同一份 `printChain:` 代码原样拿到真机跑一遍，重点看 `rootViewController.view.superview` 是不是 `UIWindow` 本身。如果真机上也有 `UIDropShadowView` / `UITransitionView`，那这两层就和 Catalyst 无关，是现代 UIKit 窗口托管的常态；如果没有，就是桥接层引入的。我倾向于前者，但这是倾向，不是结论。

链条的其余部分和文档完全吻合。Apple 的规则是这么定的：

> If the view is the root view of a view controller, the next responder is the view controller; otherwise, the next responder is the view's superview.

计划里问了一个问题：为什么 `UIViewController` 会插在 root view 和它的 superview 中间。上面这句就是答案，而且我用一个子 VC 验证了它：

```text
起点：childVC.view（子 VC 的根视图）
   0. V  (childVC.view)
   1. ChildVC          ← 它是 ChildVC 的根视图，所以下一个是 ChildVC
   2. V  (root)        ← ChildVC 的下一个是它 view 的 superview
   3. RootVC
   ...
```

`childVC.view` 的下一个不是 `root`，而是 `ChildVC`。VC 处理完再接回 `root`。加一个子 VC 就在链上插一个节点，这是 view controller containment 能工作的基础。

对照一下 `UIButton` 那条，它不是任何 VC 的根视图，所以直接接 superview：

```text
   0. Btn
   1. V  (root)
   2. RootVC
```

`UIApplication` 的下一个是 AppDelegate，文档给的条件也验上了：

```text
UIApplication.nextResponder = <AppDelegate: 0xb25442200>
app.delegate                = <AppDelegate: 0xb25442200>
```

> The next responder is the app delegate, but only if the app delegate is an instance of `UIResponder` and isn't a view, view controller, or the app object itself.

所以 AppDelegate 继承 `UIResponder` 不是模板代码写着好看。它继承别的类，链条就在 `UIApplication` 断掉。

最后一件事：`UIGestureRecognizer` 根本不在这条链上。

```text
UIGestureRecognizer  isResponder=0  UIGestureRecognizer -> NSObject
```

它直接继承 `NSObject`。手势识别器不是响应者。它挂在响应者旁边，是另一套机制，走的是平行的一条路。这一点在第九节还要用。

---

## 八、target 传 nil 时，action 沿链找人

前一篇欠的账在这里还。`sendAction:to:nil:from:` 的 `to` 传 nil 时，UIKit 会沿响应者链找第一个实现了这个 selector 的对象。这次有 `UIApplication` 了，能直接测：

```text
nearAction: RootVC 实现；farAction: 只有 AppDelegate 实现
    >>> nearAction 被 RootVC 接住
sendAction:nearAction: from:deep     -> 1
    >>> farAction 被 AppDelegate 接住 sender=<deep>
sendAction:farAction:  from:deep     -> 1
sendAction:nobody:     from:deep     -> 0
targetForAction:nearAction: from deep = <RootVC: 0xb24d4ca00>
targetForAction:farAction:  from deep = <AppDelegate: 0xb25442200>
targetForAction:(无人实现)            = nil
```

三种结果都对上了。近的被 `RootVC` 接住，远的一路走到 AppDelegate。没人实现就返回 0，`targetForAction:` 给 nil。整个过程一次异常都没有。找不到人只是安静地什么都不做。

`targetForAction:withSender:` 的默认实现，头文件注释说得比文档清楚：

> Allows an action to be forwarded to another target. By default checks -canPerformAction:withSender: to either return self, or go up the responder chain.

所以这是个递归：问自己能不能干，能就返回 self，不能就问 `nextResponder`。想让某个环节拦下来，覆写 `canPerformAction:withSender:` 就行，不用碰 `targetForAction:`。Interface Builder 里那个 First Responder 占位对象，连的就是这套机制。它不代表任何具体对象，只代表"运行时沿链找"。

`UIMenuController` 的菜单项能不能点、`copy:` / `paste:` 灰不灰，全是同一套判定。

---

## 九、手势和 UIControl 抢触摸：这一节我没测成

这一节本来要测三件事：手势识别器和 `UIControl` 谁先响应，`cancelsTouchesInView` / `delaysTouchesBegan` 的实际效果，以及 `shouldRecognizeSimultaneouslyWithGestureRecognizer:` 管不管用。我没做成。说清楚为什么。

这三件事全部发生在触摸分发过程中，需要真实的 `UITouch` 序列。Catalyst 上我确实能起 app，也能拿到窗口。但鼠标点击进来的是 indirect pointer 类型的触摸。而且 `UIButton` 上还被系统塞了两个额外的手势识别器：

```text
btn 上的手势 = (
    "<UITapGestureRecognizer: ...; cancelsTouchesInView = NO; target= <(action=selectGestureHandler:, target=<_UISelectionInteraction ...>)>>",
    "<_UIFocusSelectObserverGestureRecognizer: ...; cancelsTouchesInView = NO; delaysTouchesEnded = NO; ...>",
    "<UITapGestureRecognizer: ...; target= <(action=btnTap:, target=<RootVC ...>)>>"
)
```

前两个是 `_UISelectionInteraction` 装的，iOS 上的普通 `UIButton` 没有它们。在这样一个被动过手脚的环境里测出来的优先级，搬到 iOS 上就是错的。合成触摸我也不打算做：手工造 `UITouch` 要调一串私有 setter，观察手段本身会改变被观察的对象。

能确定的只有静态部分。默认值来自头文件原文，不是我记的：

> `cancelsTouchesInView` — default is YES. causes touchesCancelled:withEvent: or pressesCancelled:withEvent: to be sent to the view for all touches or presses recognized as part of this gesture immediately before the action method is called.

> `delaysTouchesBegan` — default is NO. causes all touch or press events to be delivered to the target view only after this gesture has failed recognition.

> `delaysTouchesEnded` — default is YES. causes touchesEnded or pressesEnded events to be delivered to the target view only after this gesture has failed recognition.

我新建的 `UITapGestureRecognizer` 读出来是 `cancelsTouchesInView=1 delaysTouchesBegan=0 delaysTouchesEnded=1`，和文档一致。

还有一句头文件注释是理解全局的关键：

> a UIGestureRecognizer receives touches hit-tested to its view and any of that view's subviews

手势识别器拿到触摸的前提，仍然是命中测试选中了它的视图或者视图的后代。第二节那套算法在这里一点没变。手势不绕过 hit-test，它只是在 hit-test 之后多插了一层裁决。

配合第七节那条"`UIGestureRecognizer` 不是 `UIResponder`"，机制的形状就清楚了。命中的 view 会正常收到 `touchesBegan:`。同时这些触摸也被送给挂在链路上的手势识别器。识别器一旦进入 Recognized，`cancelsTouchesInView` 决定要不要给 view 补一个 `touchesCancelled:`。

按钮"按下去高亮了又弹回去、action 没触发"，就是这么来的。某个祖先视图上的手势把它取消了。

> 待真机补测（这一节全部）：真机上建一个 `UIButton`，父视图加一个 `UITapGestureRecognizer`，在按钮的 `touchesBegan/Ended/Cancelled`、`TouchDown`/`TouchUpInside`、手势 action 里各打一行时间戳，跑四个组合。
> ① 默认配置：预期按钮先收到 `touchesBegan` 和 `TouchDown`，手势识别成功后按钮收到 `touchesCancelled`，`TouchUpInside` 不触发。
> ② 手势 `cancelsTouchesInView = NO`：预期两边都触发，按钮不再收到 `touchesCancelled`。
> ③ 手势 `delaysTouchesBegan = YES`：预期按钮的 `touchesBegan` 被推迟到手势失败之后，高亮明显变慢。
> ④ 实现 `gestureRecognizer:shouldRecognizeSimultaneouslyWithGestureRecognizer:` 返回 YES，观察它对"手势 vs UIControl"这一对有没有效果——我的预期是没有，因为它只协调手势之间的互斥，而 `UIControl` 的 target-action 不是手势。这一条我特别想验，因为网上有文章把它当成解决按钮和手势冲突的办法。
> 另外顺便记一下 iOS 上一个全新 `UIButton` 的 `gestureRecognizers` 是不是 nil，用来确认上面那两个 `_UISelectionInteraction` 手势确实只是 Catalyst 的产物。

有个静态事实可以先记下。`UIControl` 覆写了 `hitTest:withEvent:`，`UIScrollView` 也覆写了。`UIWindow` 两个都没覆写。

```text
UIWindow 覆写了 hitTest: 吗       0
UIWindow 覆写了 pointInside: 吗   0
UIControl 覆写了 hitTest: 吗      1
UIScrollView 覆写了 hitTest: 吗   1
```

`UIWindow` 那两个 0 挺有用。网上有说法认为窗口对命中测试有特殊处理。至少在方法这一层没有，它走的就是 `UIView` 的默认实现。

target-action 的所有权语义、那两个 target 类的 weak ivar，前一篇已经解剖过。这里不重复。

---

## 十、几个不准的说法

- **"`clipsToBounds` 会影响 hitTest。"** 不影响。两个值我都测了，结果一样。它只管绘制裁剪。子视图点不到的原因是父视图的 `pointInside:` 返回 NO，和裁不裁剪无关。
- **"alpha ≤ 0.01 不响应触摸。"** 可观测行为上对，理由通常是错的。文档写的是 less than 0.01，而 `alpha` 存成 float 让十进制 0.01 落到了 0.01 下方，两种说法因此无法区分。能被点到的最小 alpha 是 0.010000000707805157。
- **"响应者链是 view → VC → window → application → delegate。"** 现代 UIKit 里 `UIWindow` 和 `UIApplication` 之间还有 `UIWindowScene`。窗口托管视图可能还会再插几层，我在 Catalyst 上量到 `UIDropShadowView` 和 `UITransitionView`。
- **"hitTest 返回 nil 表示没点中。"** 绝大多数情况下它返回的是根视图自己。只有连根视图的 `pointInside:` 都是 NO 时才返回 nil。
- **"父子视图的 alpha 会叠加起来参与判断。"** 逐层独立判断。父 0.02、子 0.02 照样能点到子视图。
- **"手势识别器在响应者链上。"** `UIGestureRecognizer` 直接继承 `NSObject`。它和响应者链是两套机制。
- **"设成透明就点不到了。"** `backgroundColor = clearColor` 完全不影响命中测试。管事的是 `alpha`。
- **"`hitTest:` 一次触摸只调一次。"** 一次自顶向下的遍历里，每个被访问的视图各调一次；root 挂 52 个兄弟就是 53 次。一次真实触摸里 UIKit 调几遍我没测，见第六节。

---

## 总结

命中测试是个前序遍历：父视图先答 `pointInside:`，通过了才倒序问子视图，命中就立刻回溯。三个开关短路在 `pointInside:` 之前。被它们挡住的视图，连 `pointInside:` 都不会被调用。

alpha 的阈值是个 float 精度问题，跟 UIKit 的比较符号无关。你设不出 0.01。

越界子视图点不到，是因为父视图的 `pointInside:` 先说了不在。`clipsToBounds` 和这件事无关，它只决定你能不能看见那块区域——被裁掉的部分照样可点，这是个很难查的组合。扩大点击区域我推荐重写 `pointInside:`，它是唯一不动几何的方案。

响应者链的实际形状比流传的那张图长。`UIWindowScene` 是 iOS 13 之后的常态，Apple 自己的文档都还没更新。VC 插在 root view 和 superview 之间是有明文规定的，子 VC 也一样插一个节点。

最后一条是方法上的。这一篇里我最想验的手势冲突那节，恰恰是 Catalyst 做不到的那部分，我把它留成了空。写清楚"这个我没测"，比补一段看起来很像的输出有价值得多。

下一篇 [[iOS RunLoop：mode、source 与那张流程图今天还对不对]]。

## 参考资料

### 官方

- [Using responders and the responder chain to handle events](https://developer.apple.com/documentation/uikit/using-responders-and-the-responder-chain-to-handle-events)：`nextResponder` 的四条规则、hit-testing 与响应者链的分工都出自这里
- [UIView.hitTest(\_:with:)](https://developer.apple.com/documentation/uikit/uiview/hittest(_:with:))："ignores view objects that are hidden, that have disabled user interactions, or that have an alpha level less than 0.01"，以及越界子视图那段
- [UIResponder](https://developer.apple.com/documentation/uikit/uiresponder)
- SDK 头文件：`UIView.h`、`UIResponder.h`、`UIGestureRecognizer.h`。第九节那三条默认值和"a UIGestureRecognizer receives touches hit-tested to its view and any of that view's subviews"都是头文件原文

### 经典

- [SwiftRocks — Understanding the iOS Responder Chain](https://swiftrocks.com/understanding-the-ios-responder-chain)：讲清了链条的意义，但给的链是 `MyView → MyViewController → UIWindow → UIApplication → AppDelegate`，缺 `UIWindowScene`
- [The Amazing Responder Chain](https://www.cocoanetics.com/2012/09/the-amazing-responder-chain/)：First Responder 占位对象的原理
- [Understanding cocoa and cocoa touch responder chain](https://medium.com/ios-os-x-development/understanding-cocoa-and-cocoa-touch-responder-chain-12fe558ebe97)

### 本地

- [[iOS 对象通信：delegate、通知、target-action 与 block 回调]]
- [[iOS RunLoop：mode、source 与那张流程图今天还对不对]]

---

实验环境：macOS（Apple Silicon，arm64），Mac Catalyst 目标，`clang -fobjc-arc -target arm64-apple-ios17.0-macabi`，链接 `/System/iOSSupport/.../UIKitCore.framework`（iPadOS 26.5）。编出来的是原生 macOS 可执行文件，直接 `./labA` 运行，不是模拟器，也不是真机。开工前和收工后各查一次 `xcrun simctl list devices booted`，两次都是空。

第二到第六节、第十节的数据来自一个不含 `UIApplication` 的纯视图树进程：手工搭 `UIView` 树，直接调 `hitTest:withEvent:`，`event` 传 nil。第七、八节需要 `UIApplication` 和 window，用 `UIApplicationMain` 起了一个 Catalyst app，只读 `nextResponder` 和调 `sendAction:to:nil:`，没有模拟任何输入。

这套方法的适用边界：命中测试算法、坐标换算、`nextResponder` 指向是 UIKitCore 里同一份代码，可以直接采信。触摸的输入路径、手势与控件的抢夺时序、屏幕 scale 与安全区受 AppKit 桥接影响，本文凡涉及这些的地方都标了 `> 待真机补测`，并写清了复现方法。`userInterfaceIdiom` 在这个进程里是 iPad 而非 Mac，跟 idiom 相关的结论也需要复核。
