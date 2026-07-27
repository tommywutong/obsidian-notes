---
title: 【iOS】UIViewController：生命周期的真实顺序与容器控制器
published: 2026-07-27
description: push、present、切 tab，三种容器给出三种不同的回调交错顺序。而且 push 时 viewDidLayoutSubviews 一次都不保证发生在 viewDidAppear 之前——实测 animated:NO 时是 0 次。
tags:
  - iOS
  - UIKit
  - UIViewController
  - 生命周期
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 20
draft: true
---
# UIViewController：生命周期的真实顺序与容器控制器

先看三段日志。同一个 `UIViewController` 子类，同一套回调打印，只换容器。

A push 出 B：

```text
  A  viewWillDisappear:1
  B  viewWillAppear:1
  A  viewDidDisappear:1
  B  viewDidAppear:1
```

A present 出 M：

```text
  A  viewWillDisappear:0
  M  viewWillAppear:0
  M  viewDidAppear:0
  A  viewDidDisappear:0
```

Tab 从 A 切到 T2：

```text
  T2 viewWillAppear:0
  A  viewWillDisappear:0
  A  viewDidDisappear:0
  T2 viewDidAppear:0
```

三种顺序，一种都不重样。push 是「旧的先走、旧的先结束」，present 是「旧的先走、新的先结束」，切 tab 干脆是「新的先来」。**没有一条通用的 UIViewController 生命周期顺序，顺序是容器定的。**

所以这篇不给「万能序列图」。它给的是：怎么把顺序当场测出来，以及自己写容器控制器时哪几步不能漏。

坐标系那套 `frame`/`bounds`/`transform` 的换算在 [[iOS 坐标系：frame 是算出来的，bounds 和 transform 才是存的]]，响应者链在 [[iOS 事件传递与响应者链：hitTest、手势与那些点不动的按钮]]，这里只在必要处引一下。

---

## 一、这些数据是在哪跑的

全文的实验都是 Mac Catalyst 二进制，在我自己的 Mac 上 `./p9.app/Contents/MacOS/p9` 直接跑。一台模拟器都没开。

```shell
SDK=$(xcrun --sdk macosx --show-sdk-path)
clang -fobjc-arc -target arm64-apple-ios17.0-macabi -isysroot "$SDK" \
      -iframework "$SDK/System/iOSSupport/System/Library/Frameworks" \
      -framework Foundation -framework UIKit -framework QuartzCore -o out prog.m
```

和上一篇不同的是，这次需要一个活着的 `UIApplication`。裸的可执行文件跑不起来：

```text
Terminating app due to uncaught exception 'NSInternalInconsistencyException',
reason: 'Invalid parameter not satisfying: bundleIdentifier'
```

`-[UIApplication init]` 会去问 `UISApplicationState initForCurrentApplication`，没有 bundle identifier 就当场抛。

解法是包成一个最小的 `.app`。`Contents/MacOS/<name>` 放二进制，`Info.plist` 只写 `CFBundleIdentifier`、`CFBundleExecutable`、`UIDeviceFamily` 三项，再 `codesign -s -` 签个 ad-hoc 名。之后 `UIApplicationMain` 就正常起来了。它会真的开一个窗口，跑完 `exit(0)` 关掉。

运行环境：

```text
Xcode 26.6 / clang-2100.1.1.101，macOS 26.5.2
UIDevice     = iPad / iPadOS 26.5
UIScreen.mainScreen.bounds = {{0, 0}, {1920, 1080}}  scale=2.0
```

边界写在前面：

- 能验的：回调之间的相对顺序、每个回调触发时 view 的 window / frame / safeAreaInsets 状态、`loadView` 与 `view` getter 的关系、容器 API（`addChildViewController:` 那一套）的行为、控制器的释放时机。这些都是 UIKitCore 里同一份代码。
- 存疑的：动画时长、布局遍数、导航栏高度这些量。Catalyst 上 `UINavigationController` 顶部走的是 AppKit 那一侧的 title bar，安全区数值和 iOS 对不上。凡是涉及数量和时长的结论我都标了「Catalyst 上实测，未在 iOS 复核」。
- `UIScreen.mainScreen.bounds` 是 Mac 屏幕的 1920×1080，不是任何 iOS 设备的尺寸。默认 `loadView` 造出来的 view 就是这个大小，下面第六节会用到。

---

## 二、一次完整的出现，以及 push 为什么是异步的

先看最干净的一次：一个 `UINavigationController` 当 window 的 root，它的 root VC 叫 A。

```text
   257.81  A  viewDidLoad
   257.92  A  viewWillAppear:0
   265.48  A  viewIsAppearing:0
   265.49  A  viewWillLayoutSubviews
   265.49  A  viewDidLayoutSubviews
   364.33  A  viewWillLayoutSubviews
   364.34  A  viewDidLayoutSubviews
   369.91  A  viewDidAppear:0
```

`viewWillLayoutSubviews` 两次。这是常识范围内的，「布局可能跑多遍」几乎每篇文章都会提。

接着 push 出 B，把 A 和 B 的回调按时间戳混排：

```text
  1427.74  >>> [nav pushViewController:B animated:YES]
  1429.55  >>> push 方法已返回，nav.viewControllers.count=2
  1430.27  B  viewDidLoad
  1430.40  A  viewWillDisappear:1
  1430.41  B  viewWillAppear:1
  1438.18  B  viewIsAppearing:1
  1438.18  B  viewWillLayoutSubviews
  1438.18  B  viewDidLayoutSubviews
  1938.72  A  viewDidDisappear:1
  1938.74  B  viewDidAppear:1
```

注意第二行。`pushViewController:animated:` 已经返回了，`viewControllers` 已经是 2 个，而 B 连 `viewDidLoad` 都还没走。这一点我第一次测的时候直接看漏了，因为日志里 push 返回和 `viewDidLoad` 只差 0.7 毫秒，肉眼扫过去像是同步的。是把调用栈打出来才看清的：

```text
--- viewWillAppear: 的调用栈 ---
   -[VC viewWillAppear:]
   -[UIViewController _setViewAppearState:isAnimating:]
   -[UIViewController __viewWillAppear:]
   -[UINavigationController transitionConductor:retargetedToViewControllerForTransitionFromViewController:toViewController:transition:]
   -[_UIViewControllerTransitionConductor _startTransition:fromViewController:toViewController:]
   -[_UIViewControllerTransitionConductor startDeferredTransitionIfNeeded]
   -[UINavigationController __viewWillLayoutSubviews]
   -[UILayoutContainerView layoutSubviews]
```

`startDeferredTransitionIfNeeded`。这个名字说明了一切：`push` 只是记了一笔账，真正的转场等到导航控制器自己那次 `layoutSubviews` 才启动。所以 push 之后立刻读 `B.view.frame` 拿到的是没经过转场的值，在 push 后面紧接着写「等 B 显示完再做某事」的代码是不对的，那段代码会在任何回调之前执行完。

`viewIsAppearing:` 来自完全不同的地方：

```text
--- viewIsAppearing: 的调用栈 ---
   -[VC viewIsAppearing:]
   -[UIViewController __viewIsAppearing:skipWindowCheck:]
   -[UIView(CALayerDelegate) layoutSublayersOfLayer:]
   CA::Layer::perform_update_
   CA::Context::commit_transaction
   CA::Transaction::commit
   __34-[UIApplication _firstCommitBlock]_block_invoke_2
```

它在 CA transaction 提交时、view 的 `layoutSublayersOfLayer:` 里被调。这就是它和 `viewWillAppear:` 最本质的区别：一个在转场开始那一刻，一个在布局遍历里面。下一节展开。

### 布局回调到底跑几遍

「`viewDidLayoutSubviews` 一定在 `viewDidAppear` 之前」这句话我见过很多次，包括中英文都有。测一下。三个控制器，同一个 nav，只改进入方式，统计到 `viewDidAppear` 为止 `viewWillLayoutSubviews` 被调了几次。连跑三遍，结果完全一致：

```text
A(nav root，首次显示):      2 次
B(push, animated:YES):      1 次
C(push, animated:NO):       0 次
```

**`animated:NO` 的 push，到 `viewDidAppear` 为止一次布局回调都没有。** 因为没有动画，转场是一口气做完的，`viewDidAppear` 在同一次 CA transaction 里就发出去了，而这个 view 的 `layoutSubviews` 排在后面。

这条我标一下：Catalyst 上实测，未在 iOS 复核。遍数受布局驱动来源影响。Catalyst 上有一次额外的布局是 AppKit 发起的，调用栈里能看到 `+[CATransaction(NSCATransaction) NS_setFlushesWithDisplayLink]`，iOS 上不会有这一路。但「0 次也可能」这个定性结论我认为站得住。布局遍数从来就没有被 UIKit 承诺过。

所以「在 `viewDidLayoutSubviews` 里做首次布局相关的初始化」这个常见建议，得加个前提：你得自己保证它至少跑过一次。我自己的做法是在 `viewIsAppearing:` 里读几何，需要子视图 frame 的话在 `viewDidLayoutSubviews` 里配一个 `_didInitialLayout` 标志位，两者都不依赖次数。

---

## 三、viewIsAppearing: 到底解决了什么

这个方法是 iOS 17 SDK 新加的，但 API 可用性标的是 iOS 13。SDK 头文件里原文如下（`UIViewController.h`，MacOSX26.5.sdk 的 iOSSupport 版本）：

```objc
/// Called when the view is about to made visible, before it is added to the hierarchy.
/// Because the view is not yet in the hierarchy at the time this method is called, it
/// is too early in the appearance transition for many usages. Prefer -viewIsAppearing:
/// instead of this method when possible.
- (void)viewWillAppear:(BOOL)animated;

/// Called when the view is becoming visible at the beginning of the appearance transition,
/// after it has been added to the hierarchy and been laid out by its superview. This method
/// is very similar to -viewWillAppear: and is always called shortly afterwards (so changes
/// made in either callback will be visible to the user at the same time), but unlike
/// -viewWillAppear:, at the time when -viewIsAppearing: is called all of the following are
/// valid for the view controller and its own view:
///    - View controller and view's trait collection
///    - View's superview chain and window
///    - View's geometry (e.g. frame/bounds, safe area insets, layout margins)
/// Choose this method instead of -viewWillAppear: by default, as it is a direct replacement
/// that provides equivalent or superior behavior in nearly all cases.
- (void)viewIsAppearing:(BOOL)animated API_AVAILABLE(ios(13.0), tvos(13.0)) API_UNAVAILABLE(watchos);
```

`API_AVAILABLE(ios(13.0))` 写在这里不是笔误。这个方法是 back-deploy 的，用 Xcode 15 以上编译就能一路部署到 iOS 13。它不需要你把部署目标提到 17。

Apple 自己给的措辞已经很不客气了：`viewWillAppear:` 对很多用途来说 "too early"，`viewIsAppearing:` 是它的 "direct replacement"，默认应该选后者。

那到底差在哪。我造了一个用 Auto Layout 把子视图钉在 `safeAreaLayoutGuide.topAnchor` 上的控制器，在每个回调里打印四个量：

```text
[B] viewDidLoad           view.frame={{0,0},{390,844}}  box.frame={{0,0},{0,0}}     safeAreaTop=0.0   window=(未查)
[B] viewWillAppear:       view.frame={{0,0},{390,844}}  box.frame={{0,0},{0,0}}     safeAreaTop=0.0   window=0x0
[B] viewIsAppearing:      view.frame={{0,0},{390,844}}  box.frame={{0,0},{0,0}}     safeAreaTop=95.0  window=0x105713970
[B] viewDidLayoutSubviews view.frame={{0,0},{390,844}}  box.frame={{0,95},{390,44}}
[B] viewDidAppear:        view.frame={{0,0},{390,844}}  box.frame={{0,95},{390,44}}
```

看 `safeAreaTop` 和 `window` 这两列。`viewWillAppear:` 时 window 是 nil，安全区是 0；到 `viewIsAppearing:` 时 window 有了，安全区是 95。中间隔的正是「加进层级、被父视图布局一遍」这两件事。头文件那三条保证里，「superview chain 和 window」「view 的几何」这两条我验到了。trait collection 那条我没单独测，照抄头文件。

但这里有个我一开始理解错的地方，得单独说。`box.frame` 在 `viewIsAppearing:` 时还是全零，要到 `viewDidLayoutSubviews` 才有值。**`viewIsAppearing:` 保证的是控制器自己那一层 view 的几何，不包括它的子视图。** 头文件写的是 "the view controller and its own view"，"its own view" 是限定语，不是随口一提。我第一次读的时候把它当成「布局已经完成了」，然后拿子视图的 frame 去算东西，全是 0。

还有一列容易看漏。`view.frame` 从头到尾都是 `{{0,0},{390,844}}`，`viewWillAppear:` 时就已经对了。所以如果你只是要 view 自己的宽高，`viewWillAppear:` 常常也够用。这大概就是为什么这个坑能藏这么多年。真正会翻车的是安全区和 window 相关的读取，它们在 `viewWillAppear:` 时是假的，而且假得很安静：不报错，只是给你 0 和 nil。

那 `viewWillAppear:` 还剩什么用。头文件也说了：需要精确卡在转场开始那一刻的场合。比如用 transition coordinator 挂一个 alongside 动画，或者要和 `viewWillDisappear:` 里的代码配对、而那段代码又不依赖几何。除此之外我现在写新代码一律用 `viewIsAppearing:`。

---

## 四、三种容器，三种交错顺序

回到开头那三段日志，把完整版贴出来。

`UINavigationController` push（`animated:YES`）：

```text
  1430.27  B  viewDidLoad
  1430.40  A  viewWillDisappear:1
  1430.41  B  viewWillAppear:1
  1438.18  B  viewIsAppearing:1
  1938.72  A  viewDidDisappear:1
  1938.74  B  viewDidAppear:1
```

pop 回去：

```text
  3834.24  B  viewWillDisappear:1
  3834.25  A  viewWillAppear:1
  3834.92  A  viewIsAppearing:1
  4304.61  B  viewDidDisappear:1
  4304.64  A  viewDidAppear:1
```

modal present / dismiss（`animated:NO`）：

```text
  5186.13  M  viewDidLoad
  5188.83  A  viewWillDisappear:0
  5189.57  M  viewWillAppear:0
  5191.52  M  viewIsAppearing:0
  5193.94  M  viewDidAppear:0
  5193.96  A  viewDidDisappear:0
  5194.07  >>> present completion

  6186.62  M  viewWillDisappear:0
  6186.64  A  viewWillAppear:0
  6188.63  A  viewIsAppearing:0
  6200.42  A  viewDidAppear:0
  6200.44  M  viewDidDisappear:0
```

`UITabBarController` 切 tab：

```text
  8185.71  T2 viewDidLoad
  8185.85  T2 viewWillAppear:0
  8185.85  A  viewWillDisappear:0
  8186.79  >>> selectedIndex 已设
  8190.29  T2 viewIsAppearing:0
  8193.33  A  viewDidDisappear:0
  8193.35  T2 viewDidAppear:0
```

排成一张表：

| 容器动作 | 第 1 个 | 第 2 个 | 第 3 个 | 第 4 个 |
| --- | --- | --- | --- | --- |
| nav push | 旧 willDisappear | 新 willAppear | 旧 didDisappear | 新 didAppear |
| nav pop | 旧 willDisappear | 新 willAppear | 旧 didDisappear | 新 didAppear |
| present | 旧 willDisappear | 新 willAppear | 新 didAppear | 旧 didDisappear |
| dismiss | 旧 willDisappear | 新 willAppear | 新 didAppear | 旧 didDisappear |
| 切 tab | 新 willAppear | 旧 willDisappear | 旧 didDisappear | 新 didAppear |

`will` 那一组：导航和模态都是旧的先，tab 是新的先。`did` 那一组：导航和 tab 都是旧的先，模态是新的先。三个容器把这两个维度的四种组合占了三种。

我的判断是这张表不值得背。值得记的只有一件事：任何依赖「另一个控制器此刻处于什么状态」的代码都是错的。典型的坑是在 `viewWillAppear:` 里读一个全局的「当前页面」变量。push 的时候旧控制器已经声明要走了，present 的时候旧控制器还没声明结束，切 tab 的时候新控制器抢在旧控制器前面。同一行代码，三个容器三个结果。

tab 那段日志里还有个容易看漏的地方：`selectedIndex 已设` 这行打印在 `T2 viewWillAppear` 后面。也就是说 `self.tab.selectedIndex = 1` 这个赋值语句里就同步发出了两个回调，和 push 的异步行为正好相反。容器之间连同步/异步都不统一。

> Catalyst 上实测，未在 iOS 复核。相对顺序我认为是 UIKitCore 的同一份逻辑，但 present 的转场在 Catalyst 上由 AppKit 的窗口层参与，`animated:YES` 时的时序值得在 iOS 上再验一次。

---

## 五、自己写一个容器控制器

这一节是全文最实用的部分。标准三步几乎每篇文章都会列，但很少有人真去掉一步看会怎样。我逐个试了。

### 头文件怎么说

先抄一段一手依据，`UIViewController.h` 里关于 `addChildViewController:` 的注释：

```objc
/*
  addChildViewController: will call [child willMoveToParentViewController:self] before adding the
  child. However, it will not call didMoveToParentViewController:. It is expected that a container view
  controller subclass will make this call after a transition to the new child has completed or, in the
  case of no transition, immediately after the call to addChildViewController:. Similarly,
  removeFromParentViewController does not call [self willMoveToParentViewController:nil] before removing the
  child. This is also the responsibility of the container subclass.
*/
```

规则是对称的：加的时候 UIKit 帮你调 `willMove`，你补 `didMove`；移除的时候 UIKit 帮你调 `didMove(nil)`，你补 `willMove(nil)`。

所以标准写法是：

```objc
// 添加
[self addChildViewController:child];      // UIKit 内部调 willMoveToParentViewController:self
child.view.frame = self.view.bounds;
[self.view addSubview:child.view];
[child didMoveToParentViewController:self];

// 移除
[child willMoveToParentViewController:nil];
[child.view removeFromSuperview];
[child removeFromParentViewController];   // UIKit 内部调 didMoveToParentViewController:nil
```

### 完整三步的真实日志

```text
[完整流程] addChildViewController:
  C1 willMoveToParent:container  (此刻 parent=container)
  → 此刻 c.parentViewController=self  childVCs.count=1
  C1 viewDidLoad
[完整流程] [self.view addSubview:c.view]
  C1 viewWillAppear
[完整流程] didMoveToParentViewController:
  C1 didMoveToParent:container
  C1 viewIsAppearing   frame={{0, 0}, {390, 844}}
  C1 viewDidAppear
  C1 didMoveToParent:container      ← 第二次
```

两处和文档对不上。

第一处：头文件说 `willMoveToParentViewController:` 在「adding the child」之前调，但回调里读到的 `self.parentViewController` 已经是 container 了。我把 log 挪到 `[super willMoveToParentViewController:]` 之前再测一遍，还是 container。所以父子关系在 `willMove` 发出之前就建立了，`willMove` 的语义更接近「view 层级即将变化」而不是「parent 即将变化」。

第二处更实际：`didMoveToParentViewController:` 收到了两次。我只写了一次。第二次的调用栈是：

```text
   -[Child didMoveToParentViewController:]
   -[UIViewController _setViewAppearState:isAnimating:]
   -[UIViewController __viewDidAppear:]
   __64-[UIViewController viewDidMoveToWindow:shouldAppearOrDisappear:]_block_invoke
   -[UIViewController _executeAfterAppearanceBlock]
```

UIKit 在出现转场结束时又补发了一次。连跑两遍，次数稳定是 2。而如果我干脆不调 `didMove`，这个子控制器一次都收不到。

移除那一侧也是 2 次，而我一次都没手写。调用栈第一次来自 `-[UIViewController removeChildViewController:notifyDidMove:]`，第二次来自 `__viewDidDisappear:`。

结论很直接：**`didMoveToParentViewController:` 不是一次性回调，别在里面放非幂等的代码。** 我见过在这里发网络请求的，两次。

> Catalyst 上实测，未在 iOS 复核。第二次调用挂在出现转场的收尾逻辑上，iOS 的转场实现不同，次数可能不一样。但「不保证只有一次」这个定性结论足够指导写法了。

### 漏掉 didMove 会怎样

```text
[漏 didMove] addChildViewController:
  C2 willMoveToParent:container
  C2 viewDidLoad
[漏 didMove] addSubview
  C2 viewWillAppear
[漏 didMove] 故意不调 didMoveToParentViewController:
  C2 viewIsAppearing   frame={{0, 0}, {390, 844}}
  C2 viewDidAppear
```

出现回调一个不少。`parentViewController` 正确，`childViewControllers` 正确，view 正常显示。唯一的损失是 `didMoveToParentViewController:` 这个通知没发出去。

这和流传的说法不一样。很多文章写「漏掉 `didMove` 会导致子控制器收不到生命周期回调」，实测不成立。它就是一个通知点，UIKit 自己不依赖它。真正的影响面是：如果你或者你用的第三方库在子类里重写了 `didMoveToParentViewController:` 来做初始化（这是个常见写法），那段代码不会执行。

我的判断是这一步照写，但理由不是「不写会出问题」，而是它是容器契约的一部分，而且你不知道子控制器的实现有没有依赖它。

### 漏掉 addChildViewController: 会怎样

这一步才是真的会出事，只是出事的方式和多数人预期的不同。

```text
[漏 addChild] 只 addSubview，不 addChildViewController:
  C3 viewWillAppear
  → c.parentViewController=nil  childVCs.count=0
  C3 viewIsAppearing   frame={{0, 0}, {390, 844}}
  C3 viewDidAppear
```

出现回调照样全都有。因为 UIKit 的出现回调是由 view 进入 window 层级驱动的，跟父子关系无关。反向验证：只调 `addChildViewController:` 而不把 view 加进层级，一个出现回调都不会有。

那漏掉它的代价是什么。父控制器不再持有子控制器。看这段：

```objc
@autoreleasepool {
    Child *z = [Child new];
    z.view.frame = CGRectMake(0, 0, 100, 100);
    [self.root.view addSubview:z.view];   // 只加 view
    orphanView = z.view;
    wOrphan = z;                          // weak
}
```

```text
 2726.56  Z-orphan viewWillAppear
 2728.22  addSubview 完成，z.view.nextResponder=Child
 2728.22  局部作用域结束 → weak child = 0x70f3fdc00     ← 还活着
 2728.22  view 还在层级里吗？superview=UIView
 2728.64  Z-orphan viewIsAppearing
 2731.48  Z-orphan viewDidAppear
 2731.49  Z-orphan dealloc                             ← 转场一结束就没了
```

view 还在屏幕上，控制器已经析构。UIKit 在出现转场期间临时保住了它，转场一结束就撒手。

那时候责任链上还剩什么。换一个程序，等控制器真的没了之后再读一次：

```text
 2760.02  加完，weak child=0x7028a5c00
 2825.75    Child dealloc（view 还挂在 window 上）
 3152.73  过了几轮 runloop，weak child=0x0
 3152.94  orphan.superview=UIView            ← view 仍然在窗口上
 3152.95  现在读 orphan.nextResponder（child 已经没了）↓
 3152.95  nextResponder=UIView               ← 控制器从链上消失了
```

`nextResponder` 直接跳过控制器指向了父 view，不崩溃，也不报错。挂在这个控制器上的 target-action、delegate、`IBAction` 全部失效。界面看起来完全正常，就是点什么都没反应。这个 bug 我认为比崩溃难查，因为它一句报错都没有。

把 `addChildViewController:` 加回去，同一段代码：

```text
  Z-managed viewWillAppear
  局部作用域结束 → weak child = 0x70f3fdc00（容器持有它）
  Z-managed ping（还活着）
  Z-managed viewIsAppearing
  Z-managed viewDidAppear
```

所以 `addChildViewController:` 的核心作用是所有权和关系图，出现回调只是搭便车。

### 关掉自动转发

容器如果重写 `shouldAutomaticallyForwardAppearanceMethods` 返回 NO，子控制器就彻底收不到出现回调了，得自己发：

```text
=== D. shouldAutomaticallyForwardAppearanceMethods = NO ===
  子控制器已加进不转发的容器（上面没有任何 appear 回调）
  手动 beginAppearanceTransition:YES / endAppearanceTransition ↓
  D-child viewWillAppear
  D-child viewIsAppearing
  D-child viewDidAppear
```

头文件对这两个方法的说明是：

```objc
// If a custom container controller manually forwards its appearance callbacks, then rather than calling
// viewWillAppear:, viewDidAppear: viewWillDisappear:, or viewDidDisappear: on the children these methods
// should be used instead. This will ensure that descendent child controllers appearance methods will be
// invoked.
```

关键在 "descendent"。直接给子控制器发 `viewWillAppear:` 只影响它一个，`beginAppearanceTransition:animated:` 会往下递归整棵子树。除非你在做自定义转场并且需要精确控制回调时机，否则别关自动转发，默认值是 YES。

---

## 六、loadView、viewDidLoad 和 view 这个 getter

`view` 是个属性，但它的 getter 有副作用。头文件写得很明确：

```objc
@property(null_resettable, nonatomic,strong) UIView *view; // The getter first invokes [self loadView] if the view hasn't been set yet.
- (void)loadView; // ... Should never be called directly.
- (void)viewDidLoad; // Called after the view has been loaded.
@property(nonatomic, readonly, getter=isViewLoaded) BOOL viewLoaded API_AVAILABLE(ios(3.0));
@property(nullable, nonatomic, readonly, strong) UIView *viewIfLoaded API_AVAILABLE(ios(9.0));
```

所以判断「view 加载了没有」绝不能写 `self.view != nil`。实测：

```text
    x.isViewLoaded = 0   （没触发加载）
    x.viewIfLoaded = 0x0 （没触发加载）
    现在写 x.view != nil ↓
  DefaultVC loadView 进入
  DefaultVC loadView 退出，view=UIView
  DefaultVC viewDidLoad
    x.view != nil 结果 = 1，但此刻 isViewLoaded 已经变成 1
```

这个判断把自己判成立了。`isViewLoaded` 和 `viewIfLoaded` 是只读的，不会触发加载，要用它们。

同一个副作用还有个更常见的形式：在 `init` 里碰 view。我自己写第一版实验代码时就随手写了 `A.view.backgroundColor = UIColor.systemBlueColor;`，日志立刻变成这样：

```text
  776.92  --- alloc/init A ---
  777.03  A  loadView  进入
  777.30  A  viewDidLoad
  793.60  --- init 完成，isViewLoaded=1 ---
  795.09  --- 设 rootViewController ---
```

`viewDidLoad` 跑在设置 `rootViewController` 之前。对单个控制器无所谓，但如果是 `UITabBarController` 一次配五个 tab，五个控制器的视图会在启动时全部建起来，用户只看得到第一个。Jesse Squires 那篇文章讲的就是这个，他给的排查手段很好用：在 `-[UIViewController viewDidLoad]` 上下符号断点，加个 log action，启动后不做任何操作，看谁不该出现在列表里。

### loadView 不调 super

```text
  CustomVC loadView：自己建，不调 super
  CustomVC viewDidLoad, view=UIScrollView frame={{0, 0}, {100, 100}}
```

正常。`loadView` 的契约就是「造一个 view 并赋给 `self.view`」，调不调 super 取决于你想不想要那个默认的空 `UIView`。想要就调 super 再往上加东西，不想要就自己 `self.view = ...`。两条路都对，混着写才有问题（调了 super 又整个替换掉，白造一个）。

真正有意思的是什么都不做：

```text
E1: loadView 1 次, viewDidLoad 1 次, view=0x0, isViewLoaded=0
```

`viewDidLoad` 照样被调了，而 `self.view` 是 nil，`isViewLoaded` 还是 NO。UIKit 没有兜底给你造一个。这个状态很危险，因为它一声不吭。如果 `viewDidLoad` 里再碰一次 `self.view`，getter 发现还是 nil，又去调 `loadView`：

```text
E2: viewDidLoad 里访问 self.view ↓
  重入 2000 层
  ...
  重入 12000 层
（进程被 SIGSEGV 干掉，exit=139）
```

栈溢出。一万两千多层，然后 SIGSEGV。

崩溃栈里会是一万多帧重复的 `loadView`，看不出任何原因。所以自己重写 `loadView` 时，要保证每条分支都给 `self.view` 赋了值——有 `if` 就一定要有 `else`。这是我在 `loadView` 里唯一坚持的规矩。

### viewDidLoad 会不会调两次

会。

```text
(1) 第一次访问 view
     loadView #1 -> 0xc21092140
     viewDidLoad #1, view=0xc21092140
(2) setView: 一个新的 UIScrollView
    isViewLoaded=1  view=UIScrollView
(3) v.view = nil
    isViewLoaded=0
(4) 再访问 v.view
     loadView #2 -> 0xc21091dc0
     viewDidLoad #2, view=0xc21091dc0
=> loadView 共 2 次，viewDidLoad 共 2 次
```

`view` 是 `null_resettable` 的，置 nil 合法，`isViewLoaded` 跟着变回 NO，下次访问重新走一遍完整的加载流程。没有任何「只调一次」的标志位在挡着。

正常代码里当然碰不到这个，`view = nil` 没人会写。但 iOS 6 之前的 `viewDidUnload` 机制走的就是这条路，而「`viewDidLoad` 保证只调一次」这个说法在那个年代本来就不成立。今天它成立是因为没人再卸载 view 了，不是因为有机制保证。第八节接着说。

---

## 七、控制器什么时候被释放

pop 之后 `dealloc` 什么时候调，取决于还有谁持有。

有外部强引用时：

```text
  3828.32  >>> [nav popViewControllerAnimated:YES]
  3833.78  >>> pop 已返回，nav.viewControllers.count=1，B 仍被局部强引用
  3834.24  B  viewWillDisappear:1
  4304.61  B  viewDidDisappear:1
  5027.85  >>> 现在把局部强引用置 nil
  5027.86  B  dealloc  <<<<
```

`viewControllers` 在 pop 返回时就已经变成 1 个了，但 B 活到我松手为止。

没有外部强引用时（`animated:NO`）：

```text
  3184.01  >>> pop 前，weak B=0x73e0e2000
  3190.52  >>> pop 刚返回，weak B=0x73e0e2000
  3191.65  B  viewWillDisappear:0
  3193.51  B  viewDidDisappear:0
  3193.53  A  viewDidAppear:0
  3193.63  B  dealloc  <<<<
```

pop 返回时 B 还在。`popViewControllerAnimated:` 返回的就是被弹出的控制器，它是 autoreleased 的，转场期间 UIKit 也临时持有。整个转场走完才析构。

模态：

```text
  1611.99  present 后 weak M=0xc9c571c00  presentedVC=0xc9c571c00
  2410.57  dismiss
  2415.98    completion 里 weak M=0xc9c571c00
  2416.04    M dealloc
```

`presentedViewController` 持有它，dismiss 的 completion block 执行完就析构。注意 completion 里 M 还活着，在这里面读 modal 的属性是安全的。

`setViewControllers:` 整体替换：

```text
  6410.57  setViewControllers:@[A]
  6417.85    B dealloc
```

同一次调用里就放手了。

一条实践建议：查「pop 了但没释放」的问题，别看 `viewControllers.count`，那个数字降下去只说明容器松手了。用 `__weak` 引用配一个延迟检查，或者干脆在 `dealloc` 里打日志。我自己的习惯是每个 VC 都留一行 `dealloc` 打印，成本几乎为零，省下的排查时间不止一次。

还有个容易漏的：A 从头到尾只走了一次 `viewDidLoad`。push 出去再 pop 回来，A 的 view 一直在，`viewDidLoad` 不会重来。所以放在 `viewDidLoad` 里的数据刷新代码，用户 pop 回来时不会执行。这类代码该放 `viewIsAppearing:`。

---

## 八、didReceiveMemoryWarning 今天还剩什么

头文件里那行注释写了十几年了：

```objc
- (void)didReceiveMemoryWarning; // Called when the parent application receives a memory warning. On iOS 6.0 it will no longer clear the view by default.
```

`viewWillUnload` 和 `viewDidUnload` 的可用性标记也写得明明白白：

```objc
- (void)viewWillUnload API_DEPRECATED("", ios(5.0, 6.0));
- (void)viewDidUnload  API_DEPRECATED("", ios(3.0, 6.0));
```

实测。场景是 nav 里 A 被 B 盖住（A 的 view 已经离屏），B 在屏幕上：

```text
--- 途径1：postNotification UIApplicationDidReceiveMemoryWarningNotification ---
（什么都没发生）

--- 途径2：私有 -[UIApplication _performMemoryWarning] ---
  AppDelegate applicationDidReceiveMemoryWarning
  A  didReceiveMemoryWarning   isViewLoaded=0  view=0x0  window=0x0
  B  didReceiveMemoryWarning   isViewLoaded=1  view=0xb936e0000  window=0x1028b6390

--- 最终 ---
  A.isViewLoaded=0  B.isViewLoaded=1  view 指针没变
  viewDidUnload 一次都没被调
```

两条：

第一，手动 `postNotificationName:UIApplicationDidReceiveMemoryWarningNotification` 触达不到任何 view controller。网上教「测试内存警告就发个通知」的写法是无效的，只有走 `_performMemoryWarning` 这条私有路径（或者 Xcode Debug 菜单里的 Simulate Memory Warning）才会真正逐个下发。这个坑挺隐蔽的，因为通知发出去了，注册了这个通知的其他代码也确实收到了，只有 VC 那一层没反应。

第二，`didReceiveMemoryWarning` 发给了树上的所有控制器，不管在不在屏幕上，view 一个都没被卸载，指针原封不动，`viewDidUnload` 不存在。

所以这个回调今天还剩什么？它是个纯通知，一件事都不替你做。有意义的用法只有一种：你自己在里面释放可重建的缓存。

```objc
- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    [self.thumbnailCache removeAllObjects];   // 能重新算出来的，扔
}
```

不能写的：卸载 view，或者置空任何 `viewDidLoad` 里建立的东西。因为 `viewDidLoad` 不会再跑第二遍（除非你自己 `view = nil`，见上一节），置空了就永远回不来了。

另外，`NSCache` 自己就会响应内存压力，图片缓存这类东西交给它比自己在这个回调里管更省事。真要监控内存压力，`DISPATCH_SOURCE_TYPE_MEMORYPRESSURE` 的粒度比这个回调细得多，能区分 warn 和 critical。

---

## 九、几个不准的说法

- **「生命周期顺序是固定的。」** 第四节那张表，三个容器三种交错。押注任何跨控制器的顺序都会翻车。
- **「`viewDidLayoutSubviews` 一定在 `viewDidAppear` 之前。」** `animated:NO` 的 push 实测是 0 次。布局遍数 UIKit 从来没承诺过。
- **「`viewIsAppearing:` 需要 iOS 17。」** SDK 里标的是 `API_AVAILABLE(ios(13.0))`，back-deploy 到 iOS 13，只要用 Xcode 15 以上编译。
- **「`viewIsAppearing:` 时布局已经完成。」** 完成的是控制器自己那层 view 的几何。子视图的 frame 还是 0，要等 `viewDidLayoutSubviews`。
- **「漏掉 `didMoveToParentViewController:` 子控制器会收不到生命周期回调。」** 实测出现回调一个不少。漏掉它只是少发一个通知。
- **「漏掉 `addChildViewController:` 子控制器收不到出现回调。」** 也照样收得到。真正的后果是没人持有它，转场一结束就析构，view 还在屏幕上而责任链已经断了。
- **「移除时要手动调 `didMoveToParentViewController:nil`。」** `removeFromParentViewController` 已经替你调了。要手动补的是 `willMoveToParentViewController:nil`，加和移除这一对是反的。
- **「`didMoveToParentViewController:` 只调一次。」** 标准三步流程里实测收到 2 次，第二次由 UIKit 在转场收尾时补发。
- **「`viewDidLoad` 保证只调一次。」** 没有标志位挡着。`view = nil` 之后再访问就会再走一遍。今天碰不到只是因为没人卸载 view 了。
- **「用 `self.view != nil` 判断视图是否加载。」** 这个判断会把自己判成立。用 `isViewLoaded` 或 `viewIfLoaded`。
- **「发个 `UIApplicationDidReceiveMemoryWarningNotification` 就能测内存警告。」** 通知发得出去，但 view controller 收不到。
- **「`push` 之后控制器就出现了。」** `push` 只是排了个队，转场要等容器下一次 `layoutSubviews`。切 tab 反而是同步的。

---

## 总结

顺序是容器定的。push、present、切 tab 三种动作，两个控制器四个回调，实测出三种不同的交错方式。真正该记的不是哪个在前，而是别写依赖对方状态的代码。

`viewIsAppearing:` 是这几年 UIKit 最值得换过去的一个 API。头文件承诺控制器自己那层 view 的 window、trait collection 和几何在这一刻都有效。我实测的是 window 和安全区：`viewWillAppear:` 里前者是 nil、后者是 0，而且不报错。它 back-deploy 到 iOS 13，没有升级门槛。子视图的布局仍然要等 `viewDidLayoutSubviews`。

容器控制器那三步里，`addChildViewController:` 是唯一漏了会真出事的。它管的是所有权：漏掉它，子控制器在转场结束时就析构了，view 留在屏幕上，责任链断掉，界面正常但点不动。`didMove` / `willMove(nil)` 是通知，加的时候补 `didMove`，移除的时候补 `willMove(nil)`，方向是反的。而且 `didMove` 会被调不止一次，别在里面放非幂等的代码。

`view` 这个 getter 有副作用，`self.view != nil` 是个会自我实现的判断。`didReceiveMemoryWarning` 今天什么都不替你做，view 不会被卸载，`viewDidUnload` 早就没了。

最后一条方法论，和这个系列前面几篇是同一条：这一整篇里所有「和通说不一样」的地方，没有一条是靠读更多文章得出来的。都是把回调打上时间戳、把调用栈打出来、故意漏掉一步看会怎样测出来的。UIKit 的行为文档写得比 runtime 模糊得多，**能跑的实验一次比读十篇文章可靠。**

并行的两篇：图层树和渲染在 [[iOS UIView 与 CALayer：三棵树、绘制流水线与离屏渲染]]，事件与响应者链在 [[iOS 事件传递与响应者链：hitTest、手势与那些点不动的按钮]]。

## 参考资料

### 官方

- `UIViewController.h`（MacOSX26.5.sdk 的 `System/iOSSupport` 版本）：`viewIsAppearing:` 三条保证的原文、containment 那段对称契约、`didReceiveMemoryWarning` 那行 iOS 6 注释，全文最硬的依据都在这个文件里
- [UIViewController](https://developer.apple.com/documentation/uikit/uiviewcontroller)：Overview、Managing the View、Responding to View-Related Events 三节

`viewIsAppearing:` 是 WWDC23 的 "What's new in UIKit" 发布的，但那场 session 我没看，本文引的全部是 SDK 头文件原文。

### 经典

- [Use Your Loaf — UIKit View Lifecycle: viewIsAppearing](https://useyourloaf.com/blog/uikit-view-lifecycle-viewisappearing/)：把 back-deploy 到 iOS 13 这件事讲清楚了，我是从这里知道不用等 iOS 17 的
- [Jesse Squires — How to find and fix premature view controller loading on iOS](https://www.jessesquires.com/blog/2023/02/20/ios-view-controller-loading/)：第六节那个符号断点排查法出自这里

### 本地

- [[iOS 坐标系：frame 是算出来的，bounds 和 transform 才是存的]]
- [[iOS 事件传递与响应者链：hitTest、手势与那些点不动的按钮]]
- [[iOS UIView 与 CALayer：三棵树、绘制流水线与离屏渲染]]
- [[iOS RunLoop：mode、source 与那张流程图今天还对不对]]：第二节那些「等下一次 layoutSubviews」的调用栈落在哪一段 RunLoop 里，那篇有图

---

实验环境：Xcode 26.6（clang-2100.1.1.101），macOS 26.5.2，Mac Catalyst 二进制，`-target arm64-apple-ios17.0-macabi`，打成最小 `.app` 后直接运行。`UIDevice` 报 iPad / iPadOS 26.5，`UIScreen.mainScreen.bounds` 是 Mac 屏幕的 1920×1080。全程没有 boot 任何模拟器。

回调之间的相对顺序、几何有效性、容器 API 的行为我认为可以直接迁移到 iOS，它们是 UIKitCore 里同一份代码。数量和时长类的结论（布局遍数、`didMove` 的次数、安全区数值、动画时长）受 Catalyst 的 AppKit 桥接影响，正文里逐条标了。

> 待 iOS 复核：把第四节那三组交错顺序、第五节 `didMove` 的次数、第二节的布局遍数在 iPhone 15 / iOS 26.5 上重跑一遍。代码原样可用，只需要换成 iOS target 编译并 `xcrun simctl spawn` 或装进模拟器。
