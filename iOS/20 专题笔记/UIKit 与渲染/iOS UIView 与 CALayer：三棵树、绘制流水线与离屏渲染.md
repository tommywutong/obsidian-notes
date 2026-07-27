---
title: 【iOS】UIView 与 CALayer：三棵树、绘制流水线与离屏渲染
published: 2026-07-27
description: subviews 不是一棵独立的树，它是 layer.sublayers 按 delegate 映射出来的投影。一个什么都不画的空 drawRect:，实测让 100 个 320pt 视图的 footprint 从 2.3 MB 涨到 92 MB。
tags:
  - iOS
  - UIKit
  - CALayer
  - CoreAnimation
  - 渲染
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 22
draft: true
---
# UIView 与 CALayer：三棵树、绘制流水线与离屏渲染

先做一件看起来无害的事。父视图 `p` 加两个子视图，然后对其中一个的 layer 调 `removeFromSuperlayer`：

```objc
[p addSubview:a];        // a.tag = 1
[p addSubview:b];        // b.tag = 2
[a.layer removeFromSuperlayer];
```

```text
                        subviews=[1 2 ] sublayers=[2 ] 
a.superview = 0x0       p.subviews 里还有 a 吗? 1
```

`a.superview` 是 nil，`p.subviews` 里却还有 `a`。两边对同一个父子关系给出了相反的答案。

这不是随手写出来的怪代码。它暴露的是一个被中文资料普遍讲错的东西：**`UIView` 树和 `CALayer` 树不是两棵平行的树，`subviews` 是 `sublayers` 按 `delegate` 映射出来的投影。** 上面那句 `removeFromSuperlayer` 改的是唯一的那棵真树，而 `superview` 是 `UIView` 自己另外记的一份状态，没跟着变。

这篇讲树结构、绘制流水线、图层属性和离屏渲染。几何那部分（`frame` / `bounds` / `transform` / `anchorPoint` / 坐标换算）在 [[iOS 坐标系：frame 是算出来的，bounds 和 transform 才是存的]] 里，本文一个字都不重复。

---

## 一、view 是怎么持有 layer 的

先把 `UIView` 的 ivar 掏出来看。Mac Catalyst 编出来的是原生 macOS 二进制，但加载的是真正的 `UIKitCore`，`class_copyIvarList` 拿到的就是 UIKit 自己的布局：

```text
=== UIView : 37 ivars ===  instanceSize = 408
  [ 4] off=48    _layerRetained                 @"CALayer"
  [ 5] off=56    _subviewCache                  @"NSArray"
  [ 8] off=80    _viewDelegate                  @"UIViewController"
  [13] off=168   _layer                         @"CALayer"
  [14] off=176   _viewBackingAux                @"_UIViewBackingAux"
  ivarLayout   : 06 11 c5 19 11
  weakIvarLayout: (null)
```

两个 layer ivar。名字都不像随手起的。`_layerRetained` 在偏移 48，`_layer` 在偏移 168，读出来是同一个指针：

```text
  _layerRetained(off=48)  = 0x9108481a0
  _layer        (off=168) = 0x9108481a0
  u.layer                 = 0x9108481a0
  三者相同? 1
```

为什么要存两份。`ivarLayout` 给出了答案。这串字节的编码规则是每字节高 4 位表示跳过几个字长、低 4 位表示接下来几个字长是 ARC 强引用。照着解一遍（`UIView` 的 `instanceStart` 是 16，也就是 `UIResponder` 的 `instanceSize`）：

```text
ivarLayout 原始字节: 06 11 c5 19 11
解码出的 strong ivar 偏移: 16 24 32 40 48 56 72 176 184 192 200 208 224 ...

  off=48    _layerRetained          STRONG（ARC 扫描）
  off=64    _window                 不扫描（unsafe_unretained / 非 ARC 管理）
  off=80    _viewDelegate           不扫描（unsafe_unretained / 非 ARC 管理）
  off=168   _layer                  不扫描（unsafe_unretained / 非 ARC 管理）
```

所以 `_layerRetained` 是那根真正的强引用。`_layer` 是同一个对象的一份裸指针缓存，不参与 ARC。

解码器本身可信。它同时判出 `_window` 和 `_viewDelegate`（也就是 view 所属的 `UIViewController`）都不被扫描，而这两条关系恰好是公认的不持有。

`weakIvarLayout` 是 null，`UIView` 一个 weak ivar 都没有。这套 nibble 解码和 [[iOS 对象通信：delegate、通知、target-action 与 block 回调]] 里判定 `UIGestureRecognizerTarget._target` 是 weak 用的是同一个方法。区别在于那一篇从磁盘上的 `class_ro_t` 解，这里直接问运行时要。省掉了处理 chained fixups 指针高位的麻烦。

反过来，layer 对 view 的引用是弱的。头文件写得很直白：

```objc
/* An object that will receive the CALayer delegate methods defined
 * below (for those that it implements). The value of this property is
 * not retained. Default value is nil. */

@property(nullable, weak) id <CALayerDelegate> delegate;
```

"not retained" 只说明不强持有。是 weak 还是 unsafe_unretained，这句话没说。两者的区别在对象销毁后能不能拿到 nil。测一下就知道：

```text
  设完 delegate = 0x102a89290, Probe 引用计数 = 2
  [Probe dealloc]
  Probe 释放后 l.delegate = 0x0  -> nil（weak）
```

置 nil 了，是真 weak。这条对直接用 `CALayer` 的人有意义。你把一个临时对象设成 layer 的 delegate，它析构之后 layer 不会拿着野指针崩，只是安静地不再回调。

### layer 不是懒创建的

有个说法流传很广：`view.layer` 第一次访问时才创建。给 `+layerClass` 加个计数器：

```text
  init 完还没访问 .layer:  _layerRetained=0xa99448700  +layerClass 调用次数=1
  访问 .layer 一次之后:    _layerRetained=0xa99448700  +layerClass 调用次数=1
  再访问 10 次:            +layerClass 调用次数=1
```

`initWithFrame:` 返回时 `_layerRetained` 已经填好了。`+layerClass` 在整个生命周期里只被问一次。

这决定了它的覆写必须是纯函数。它在 `init` 期间被调用，那时候你的实例变量都还没初始化，读 self 上的任何东西都不安全。

### 两个对象之间只有四个方法

`UIView` 在头文件里声明了自己遵守 `CALayerDelegate`：

```objc
@interface UIView : UIResponder <NSCoding, UIAppearance, ..., CALayerDelegate>
```

真正实现了几个，问 runtime：

```text
  displayLayer:              响应=0  IMP=0x0
  drawLayer:inContext:       响应=1  IMP=0x1c6bede6c
  layerWillDraw:             响应=1  IMP=0x1c6bedb98
  layoutSublayersOfLayer:    响应=1  IMP=0x1c6b54038
  actionForLayer:forKey:     响应=1  IMP=0x1c6b0928c
```

四个。`displayLayer:` 没实现。这个缺口是有意义的。这一点后面第五节还要用到：`CALayer` 发现 delegate 不实现它，才会走 `drawLayer:inContext:` 那条给你一个 `CGContext` 的路。

这四个方法就是 `UIView` 和 `CALayer` 之间的全部接口。布局走 `layoutSublayersOfLayer:`，绘制走 `drawLayer:inContext:`，动画走 `actionForLayer:forKey:`。三件事，三个入口，下面三节各讲一个。

---

## 二、只有一棵树，`subviews` 是它的投影

回到开头那个现象。把它拆干净。

先看 `subviews` 到底从哪来。`UIView` 有个 `_subviewCache` ivar，名字已经说明问题了。做四组对照：

```text
--- 情形一：removeFromSuperlayer 之前【读过】subviews（建立了缓存）---
  addSubview 后                      subviews=[1 2 ] sublayers=[1 2 ] cache=0x9bac18480
  a.layer removeFromSuperlayer 后    subviews=[1 2 ] sublayers=[2 ]   cache=0x9bac18480
  再 addSubview 一个（强制刷新）      subviews=[2 3 ] sublayers=[2 3 ] cache=0x9bac18480
  此时 a.superview=0x0, subviews 里还有 1 吗? 0

--- 情形二：removeFromSuperlayer 之前【没读过】subviews ---
  直接读                             subviews=[2 ]   sublayers=[2 ]
  x.superview=0x0
```

情形二里 `subviews` 立刻就跟着变了。情形一没变，是因为缓存没被 `removeFromSuperlayer` 作废。等下一次 `addSubview:` 把缓存刷掉，`a` 就消失了。

所以 `subviews` 确实是从 `sublayers` 现算的。中间隔着一层缓存，图层操作不一定作废它。

再看反方向：

```text
--- 情形三：只 addSublayer 一个 view 的 layer ---
  [r.layer addSublayer:z.layer];
  r.subviews.count = 1
  e1.superview     = 0x0 (r=0x104cfc1a0)
  r.subviews 里有 z 吗? 1
```

只把 layer 挂上去，那个 view 就出现在 `subviews` 里了，但它的 `superview` 还是 nil。映射规则就是遍历 `sublayers`、取每个 layer 的 `delegate`。至于 `superview`，它是 `addSubview:` / `removeFromSuperview` 单独维护的一份状态，图层操作碰不到。

顺序也是从图层树来的。直接调 `insertSublayer:above:` 换两个 sublayer 的位置，不碰任何 view API：

```text
   subviews 顺序  = [12 11 ]
   sublayers 顺序 = [12 11 ]
```

`subviews` 跟着换了。我没碰过任何 view API。

我的结论：图层树是唯一的真相，视图树是它的一个视图（view，双关不是故意的）加上 `superview` 这一份旁路状态。这也解释了一件事。给 `UIView` 的 layer 直接 `addSublayer:` 一个裸 `CALayer` 完全合法，画边框、加渐变都这么干。而对某个 view 的 layer 做树结构操作就危险。

具体到写代码，我自己的阈值是：**往 `view.layer` 上挂没有 view 的裸 layer，随便挂；只要那个 layer 是某个 view 的 backing layer，就一律走 view 那套 API。** 这条线以下的所有混用都会产生上面那种自相矛盾的状态，而且不报错、不崩溃，只是行为诡异。

裸 layer 的 `delegate` 是 nil，所以它不会污染 `subviews`：

```text
  只 addSublayer 一个裸 layer 之后:
  parent.subviews.count        = 2（没变）
  parent.layer.sublayers.count = 3
  bare.delegate = 0x0
```

---

## 三、三棵树到底是不是三棵

Core Animation 讲三棵树：model tree、presentation tree、render tree。`CALayer` 上对应两个方法，`presentationLayer` 和 `modelLayer`，头文件的措辞值得抄原文：

```objc
/* Returns a copy of the layer containing all properties as they were
 * at the start of the current transaction, with any active animations
 * applied. This gives a close approximation to the version of the layer
 * that is currently displayed. Returns nil if the layer has not yet
 * been committed. */

- (nullable instancetype)presentationLayer;
```

"Returns nil if the layer has not yet been committed." 这句是实测的起点。

新建一个 `CALayer`，什么都不做：

```text
A. bare layer         = 0xc6e83c0e0
   presentationLayer  = 0x0
   modelLayer         = 0xc6e83c0e0  (== self? 1)
```

`modelLayer` 返回自己，`presentationLayer` 是 nil。这是第一个要澄清的：没上屏的 layer 根本没有 presentation layer，不是"返回一份和 model 一样的拷贝"。

那 "committed" 是什么意思。我先试了最直觉的做法。搭一棵 root + child 的图层树，`[CATransaction flush]` 一下：

```text
before flush           pres=0x0
after flush            pres=0x0
t=0.3s ... t=1.2s      pres=nil
```

动画都跑起来了。`presentationLayer` 还是 nil。这里我卡了一会儿，才想明白"提交"这个词指的是提交进一个渲染上下文，不是提交给 `CATransaction`。一棵没有根的图层树谁也不渲染，自然没有 presentation 副本。

在 iOS 上这一步由 `UIWindow` 完成。Catalyst 上不行，`[[UIWindow alloc] initWithFrame:]` 会抛 `NSApplication has not been created yet`。我不想为了一个实验去起一个真的 App 窗口。

换成 `CAContext` 的 `localContext`（私有 API，仅作实验仪器用），把 root layer 挂上去：

```objc
id ctx = [NSClassFromString(@"CAContext") localContext];
[ctx setLayer:root];
```

```text
after attach+flush: pres = 0x8e30b0200
```

有了。

加一个 2 秒的 `position.x` 动画，每 0.35 秒采样一次：

```text
t=0.35s  model.position.x=350.00 | pres=0x8e30b0200 pres.position.x=103.04 | pres.modelLayer=0x8e30b01a0 (==child:1)
t=0.70s  model.position.x=350.00 | pres=0x8e30b02a0 pres.position.x=156.31 | pres.modelLayer=0x8e30b01a0 (==child:1)
t=1.05s  model.position.x=350.00 | pres=0x8e30b02a0 pres.position.x=209.25 | pres.modelLayer=0x8e30b01a0 (==child:1)
t=1.40s  model.position.x=350.00 | pres=0x8e30b02a0 pres.position.x=262.46 | pres.modelLayer=0x8e30b01a0 (==child:1)
t=1.75s  model.position.x=350.00 | pres=0x8e30b02a0 pres.position.x=315.71 | pres.modelLayer=0x8e30b01a0 (==child:1)

child=0x8e30b01a0  child.modelLayer=0x8e30b01a0 (==child:1)
```

这组数据说了四件事。

`child` 是 `0x...01a0`，`presentationLayer` 是 `0x...0200`，两个不同的对象。`child.modelLayer` 返回 `child` 自己，`pres.modelLayer` 也返回 `child`。所以 model layer 和你手上那个 layer 就是同一个对象，`modelLayer` 这个方法存在只是为了从 presentation 那边找回来。

model 的 `position.x` 从赋值那一刻起就是 350。presentation 的值一路爬升，五次采样的增量是 53.27 / 52.94 / 53.21 / 53.25，对应 150 pt/s × 0.35 s = 52.5。

动画期间"属性已经是终值、屏幕上还没走到"这件事，这组数据是它的直接证据。

presentation layer 的地址在 t=0.35 和 t=0.70 之间变了一次。它是每次提交重新生成的临时对象。别缓存它。

还有一个细节：`pres.presentationLayer` 返回它自己，不是再套一层。

至于第三棵树。render tree 在公开 API 里没有任何入口，我没有办法观测它，本文不给它编任何行为。能说的只有两句。它在渲染服务进程那边。`presentationLayer` 是 CA 给你的一个近似，头文件的原词是 "a close approximation to the version of the layer that is currently displayed"。approximation 这个词是 Apple 自己选的。

> 待真机补测：以上全部在 Catalyst + `CAContext localContext` 下测得。iOS 上正确的做法是把 view 加进 `UIWindow` 并 `makeKeyAndVisible`，然后在 `CADisplayLink` 回调里打印 `layer.presentationLayer.position` 与 `layer.position`。需要确认的是三点：入 window 前 `presentationLayer` 是否同样为 nil；动画期间两者是否同样分叉；presentation layer 的地址是否同样逐帧变化。

---

## 四、UIKit 是怎么把隐式动画关掉的

给一个裸 `CALayer` 改 `position`，它自己会动起来。给 `view.layer` 改，不会。这个差别的实现位置很具体，`actionForKey:` 的文档注释把查找顺序完整列了出来：

```objc
/* Returns the action object associated with the event named by the
 * string 'event'. The default implementation searches for an action
 * object in the following places:
 *
 * 1. if defined, call the delegate method -actionForLayer:forKey:
 * 2. look in the layer's `actions' dictionary
 * 3. look in any `actions' dictionaries in the `style' hierarchy
 * 4. call +defaultActionForKey: on the layer's class
 *
 * If any of these steps results in a non-nil action object, the
 * following steps are ignored. If the final result is an instance of
 * NSNull, it is converted to `nil'. */

- (nullable id<CAAction>)actionForKey:(NSString *)event;
```

第一步就问 delegate。而 `UIView` 恰好实现了 `actionForLayer:forKey:`。拦截点就在这。

对着八个 key 各问一遍，裸 layer 和 `view.layer` 并排：

```text
--- key: position
  bare CALayer   actionForKey:@"position" -> CABasicAnimation  (0x81944c9e0)
  UIView.layer   actionForKey:@"position" -> nil  (0x0)
--- key: bounds
  bare CALayer   -> CABasicAnimation      UIView.layer -> nil
--- key: opacity
  bare CALayer   -> CABasicAnimation      UIView.layer -> nil
--- key: backgroundColor
  bare CALayer   -> CABasicAnimation      UIView.layer -> nil
--- key: transform
  bare CALayer   -> CABasicAnimation      UIView.layer -> nil
--- key: contents
  bare CALayer   -> CABasicAnimation      UIView.layer -> nil
--- key: hidden
  bare CALayer   -> CATransition          UIView.layer -> nil
--- key: onOrderIn
  bare CALayer   -> nil                   UIView.layer -> nil
```

改完属性数一下动画：

```text
  bare.animationKeys       = ( position )
  v.layer.animationKeys    = (null)
  bare 的隐式动画: CABasicAnimation duration=0.250
```

隐式动画默认 0.25 秒。这个值我以前一直记成 0.3。`hidden` 拿到的是 `CATransition` 而不是 `CABasicAnimation`，这个差别我没在任何中文文章里见人提过。

现在纠正一个到处都在传的说法。很多文章写"`UIView.layer` 的 `actionForKey:` 返回 `NSNull`，所以隐式动画被禁用"。上面这组数据显示它返回的是 nil。`NSNull` 出现在更里面一层：

```text
  [v actionForLayer:v.layer forKey:@"position"] = NSNull (0x1fb9c8c00)  是 NSNull? 1
  ... forKey:@"opacity"                         = NSNull                 是 NSNull? 1
```

`UIView` 作为 delegate 返回 `NSNull`，`actionForKey:` 按文档最后那句把 `NSNull` 归一成 nil 再返回给调用方。链条是 delegate 给 `NSNull`、layer 转成 nil、CA 看到 nil 就不加动画。说"返回 NSNull"的人大概率没自己打印过，只是把机制的中间态当成了结果。

更漂亮的是 `UIView` 动画 block 内部：

```text
  inside animateWithDuration: v.layer actionForKey:@"position" -> _UIViewAdditiveAnimationAction
  inside: [v actionForLayer:forKey:] -> _UIViewAdditiveAnimationAction  是 NSNull? 0
```

同一个 delegate 方法，在动画 block 里换成了返回一个真的 action 对象。`UIView` 动画和 Core Animation 的隐式动画走的是同一条通道，UIKit 只是平时把开关拨到 `NSNull`，进 block 时拨回来。类名里的 Additive 还解释了另一件事：为什么连续两次 `UIView` 动画会叠加，而不是后一次打断前一次。

用 `UIView` 的 setter（`v2.center = ...`）改，结果和用 `v2.layer.position` 改一样，都没有动画：

```text
  v2.layer.animationKeys   = (null)
```

因为拦截点在 layer 的 action 查询上，跟你从哪个 setter 进来无关。

---

## 五、drawRect: 与那块 backing store

`UIView` 的 `_viewFlags` 是一个巨大的位域。里面第二位就叫 `implementsDrawRect`：

```text
{?="userInteractionDisabled"b1"implementsDrawRect"b1"implementsDidScroll"b1 ... }
```

UIKit 在初始化时就把"这个子类实现没实现 `drawRect:`"记成了一个 bit。直接读它：

```text
=== PlainView（不实现 drawRect:） ===
  implementsDrawRect(ivar bit) = 0
  contents (after display) = 0x0

=== EmptyDrawView（空实现 drawRect:） ===
  implementsDrawRect(ivar bit) = 1
  contents (after display) = <CABackingStore 0x100d308d0 (buffer [200 200] BGRX8888)>

=== RealDrawView（真的画） ===
  implementsDrawRect(ivar bit) = 1
  contents (after display) = <CABackingStore 0x100d10730 (buffer [200 200] BGRX8888)>
```

不实现 `drawRect:` 的 view，`layer.contents` 走完整个显示流程之后仍然是 NULL，一个字节的位图都没有。只要方法存在，哪怕方法体是空的,一块 `CABackingStore` 就被分配出来。100pt 的 view、`contentsScale` 2，buffer 是 200×200。

尺寸怎么算。扫一遍就清楚了：

```text
   50pt @1x -> buffer [50 50]        100pt @1x -> buffer [100 100]      320pt @1x -> buffer [320 320]
   50pt @2x -> buffer [100 100]      100pt @2x -> buffer [200 200]      320pt @2x -> buffer [640 640]
   50pt @3x -> buffer [150 150]      100pt @3x -> buffer [300 300]      320pt @3x -> buffer [960 960]
```

就是 `点 × contentsScale`，没有别的规则。BGRX8888 是每像素 4 字节，所以一个 320pt 的 view 在 3 倍屏上标称 960×960×4 = 3.5 MB。

### 这块内存实际要多少

标称值不等于 footprint。这是 [[iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint]] 那篇的主线之一：申请了地址范围，不代表页面立刻变 dirty。

所以我用同一套仪器实际量了一遍，`task_info(TASK_VM_INFO)` 读 `phys_footprint`。每个配置单独起一个进程，避免上一批的释放污染读数：

```text
模式=nodraw  100 个 320pt @3x   理论 backing store 合计 = 351.6 MB
  before     internal=   3.34 MB  compressed=0.00 MB  phys_footprint=   3.72 MB
  after      internal=   5.64 MB  compressed=0.00 MB  phys_footprint=   6.03 MB

模式=empty   100 个 320pt @3x   理论 backing store 合计 = 351.6 MB
  before     internal=   3.34 MB  compressed=0.00 MB  phys_footprint=   3.72 MB
  after      internal=  95.34 MB  compressed=0.00 MB  phys_footprint=  95.80 MB

模式=red     100 个 320pt @3x   理论 backing store 合计 = 351.6 MB
  before     internal=   3.31 MB  compressed=0.00 MB  phys_footprint=   3.69 MB
  after      internal= 359.52 MB  compressed=0.00 MB  phys_footprint= 360.09 MB
```

三行的差值是 2.3 MB、92.1 MB、356.4 MB。差得不是一点。

`red` 那行的 `drawRect:` 是一句 `UIRectFill(rect)`，把每个像素都写了一遍，实测 356.4 MB 和标称的 351.6 MB 几乎重合。这验证了 `点 × scale × 4` 这个公式本身是对的。

`empty` 那行只有标称值的 26%。合理的解释是缓冲区按全尺寸预留了地址范围，但空的 `drawRect:` 什么都没写，大部分页面从没变 dirty。这个解释和上面那篇文章里 "`malloc(32 * 1024)` 之后 footprint 没有立刻涨 32 KB" 是同一个机制，但我要说清楚：这是我对数据的解读，我没有验证过这个机制。我一开始的猜测是内存压缩，结果 `compressed` 一栏三行全是 0.00 MB，猜错了。先怀疑仪器再怀疑结论，这次是先怀疑了自己。

不管解释是什么，可以直接用的数字是这个比值：**一个什么都不画的空 `drawRect:`，让同一批视图的 footprint 从 2.3 MB 变成 92 MB，40 倍。** "不要留空的 `drawRect:`"这条建议我以前只当成一句口号，量完之后不会再这么想。

`opaque` 在这组实验里没有影响。我试过把 `view.opaque` 设成 NO 再跑一遍，buffer 格式仍然是 BGRX8888，footprint 也几乎一样（95.75 vs 95.69 MB）。

> 待真机补测：`opaque` 在 iOS 上应当影响 backing store 的像素格式（不透明用 BGRX / 32bpp 无 alpha，半透明用 BGRA），进而影响混合开销。Catalyst 上我两种情况都只拿到 BGRX8888，这一条我没能验证。真机复现方法：在 iPhone 上分别用 `opaque = YES/NO` 建同样的 view，打印 `layer.contents` 的 description 看 buffer 格式，再用 Instruments 的 Color Blended Layers 看是否有红色混合区。

最后补一条容易忽略的。只把一张现成的图赋给 `layer.contents`，不走 `drawRect:`，`contents` 就是那个 `CGImage` 本身，没有额外的 `CABackingStore`：

```text
  contents = <CGImage 0x102b38390> ... width = 200, height = 200, bpc = 8, bpp = 32
```

所以"用图片而不是 `drawRect:` 画"省下来的正是那一块中间缓冲。

---

## 六、攒到一次提交

`setNeedsDisplay` 连着调 100 次，`drawRect:` 被调几次。直接数：

```text
--- 连续调用 setNeedsDisplay 100 次 ---
  调用完、还没回 RunLoop:  drawRect 被调 0 次
  layer.needsDisplay = 1
  跑一轮 RunLoop 之后:      drawRect 被调 1 次

--- setNeedsLayout 100 次 ---
  同步:                   layoutSubviews=0
  跑一轮 RunLoop:         layoutSubviews=1
```

100 次变 1 次。而且在循环还没跑完的时候，一次都没执行。这两个方法做的只有一件事：打一个脏标记。

`layoutIfNeeded` 是那个"现在就做"的开关，但它认脏标记：

```text
--- setNeedsLayout 100 次 + layoutIfNeeded ---
  layoutIfNeeded 后:      layoutSubviews=1
  再调一次 layoutIfNeeded: layoutSubviews=1（没有脏标记就不干活）
```

### 是谁在跑一轮 RunLoop 的时候来收账

把主 RunLoop 的 observer 打出来，用 `dladdr` 反查回调：

```text
<CFRunLoopObserver 0x101687d70>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000,
  callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x198259d60)}
```

`activities = 0xa0` 是 `kCFRunLoopBeforeWaiting`（0x20）加 `kCFRunLoopExit`（0x80）。`order = 2000000` 是个很大的数，意味着它在同一时机的其他 observer 之后才跑。回调名字 demangle 出来是 `CA::Transaction::observer_callback`。

也就是说：主线程忙完这一轮所有的 source 和 timer，即将进入休眠。就在这个点上，Core Animation 的 observer 被叫醒，把这一轮攒下的修改一次性提交。这就是"攒到一次"的执行者。RunLoop 的 observer 机制本身在 [[iOS RunLoop：mode、source 与那张流程图今天还对不对]]。

光看 observer 还不够，我在 `drawRect:` 和 `layoutSubviews` 里各抓了一次调用栈（已 demangle）：

```text
---- layoutSubviews 的调用栈 ----
0   -[BTView layoutSubviews]
3   UIKitCore   -[UIView(CALayerDelegate) layoutSublayersOfLayer:]
4   QuartzCore  CA::Layer::perform_update_(CA::Layer*, CALayer*, unsigned int, CA::LayerUpdateReason, CA::Transaction*)
5   QuartzCore  CA::Layer::update_if_needed_(CA::Transaction*, CA::LayerUpdateReason)
6   QuartzCore  CA::Context::commit_transaction(CA::Transaction*, double, double*) + 608
7   QuartzCore  CA::Transaction::commit()
8   QuartzCore  CA::Transaction::flush_as_runloop_observer(bool)
9   CoreFoundation  __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__
10  CoreFoundation  __CFRunLoopDoObservers

---- drawRect: 的调用栈 ----
0   -[BTView drawRect:]
3   UIKitCore   -[UIView(CALayerDelegate) drawLayer:inContext:]
4   QuartzCore  CABackingStoreUpdate_
5   QuartzCore  invocation function for block in CA::Layer::display_()
6   QuartzCore  -[CALayer _display]
7   QuartzCore  CA::Layer::display_if_needed(CA::Transaction*)
8   QuartzCore  CA::Context::commit_transaction(CA::Transaction*, double, double*) + 620
9   QuartzCore  CA::Transaction::commit()
10  QuartzCore  CA::Transaction::flush_as_runloop_observer(bool)
11  CoreFoundation  __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__
```

这两张栈把整条链焊死了，几乎不需要再引用任何二手图：

`__CFRunLoopDoObservers` → `CA::Transaction::flush_as_runloop_observer` → `CA::Transaction::commit` → `CA::Context::commit_transaction`，然后在 `commit_transaction` 内部分岔。布局在偏移 +608 处调 `update_if_needed_`，绘制在偏移 +620 处调 `display_if_needed`。同一个函数里相差 12 字节，**布局阶段在绘制阶段之前，而且是硬编码的先后，不取决于你先打哪个脏标记**：

```text
A. 先 setNeedsLayout 再 setNeedsDisplay：  1. layoutSubviews  2. layerWillDraw:  3. drawRect:
B. 先 setNeedsDisplay 再 setNeedsLayout：  1. layoutSubviews  2. layerWillDraw:  3. drawRect:
```

反过来 `layoutIfNeeded` 只做布局那一半：

```text
C. 两个脏标记都打上，调 layoutIfNeeded：
  调 layoutIfNeeded：      1. layoutSubviews
  再跑 RunLoop：           2. layerWillDraw:  3. drawRect:
```

这条顺序能解释一个常见困惑。在 `layoutSubviews` 里改子视图的 frame 立刻生效，改一个需要重绘的属性却要等下一帧。原因就是你正站在 `commit_transaction` 的布局阶段里，绘制阶段还在后面。

还有一条：显式的 `CATransaction` 会绕开 RunLoop：

```text
D. [CATransaction begin]; [v setNeedsDisplay];
  begin 之后、commit 之前：seq=0
  [CATransaction commit];  ->  1. layerWillDraw:  2. drawRect:
  commit 之后：seq=2
```

`commit` 当场就把绘制做了。所以"必须等下一轮 RunLoop"这句话准确的说法是"等下一次事务提交"，只不过默认那次提交是 RunLoop 替你发起的。

至于 `drawLayer:inContext:` 为什么会在栈上出现而不是 `displayLayer:`：第一节数过，`UIView` 没实现 `displayLayer:`。`CALayer` 先问 delegate 有没有 `displayLayer:`。有就把整个 contents 的生产权交出去，你自己给一张图。没有才分配 backing store、开一个 `CGContext`、调 `drawLayer:inContext:`。栈里 `CABackingStoreUpdate_` 就压在 `drawLayer:inContext:` 上面一层，和第五节量到的那块 buffer 是同一个东西。


---

## 七、从图层树到屏幕

"App 提交 → Render Server → GPU → 显示"这张图网上到处都是，画法各不相同，细节互相打架。我的处理办法是把它劈成两半：我自己能证明的一半，和我只能引用 Apple 的一半。

### 我能证明的一半

就是上一节那两张调用栈。它们覆盖的是这段：

```text
你的代码改属性 / setNeedsLayout / setNeedsDisplay
        │  只打脏标记，什么都不做
        ▼
RunLoop 跑完这一轮的 source 和 timer，准备休眠
        │  kCFRunLoopBeforeWaiting（0xa0 里的 0x20），order = 2000000
        ▼
CA::Transaction::observer_callback
        └─ CA::Transaction::flush_as_runloop_observer
             └─ CA::Transaction::commit
                  └─ CA::Context::commit_transaction
                       ├─ +608  CA::Layer::update_if_needed_  → layoutSublayersOfLayer: → layoutSubviews
                       └─ +620  CA::Layer::display_if_needed  → -[CALayer _display]
                                    └─ CABackingStoreUpdate_  → drawLayer:inContext: → drawRect:
```

符号名、偏移量、observer 的 activity 掩码和 order，全是这台机器上打出来的。这一段不用引用任何人。

### 引用的那一半

提交之后图层树被送去渲染服务进程，那边生成绘制指令交给 GPU，GPU 渲染完等下一个 VSync 换帧上屏。这几步发生在另一个进程和另一块芯片上，我在 Catalyst 上进不去也量不到。

好在这部分 Apple 自己讲过，而且有讲义 PDF 可查，不必依赖任何转述。下面的引文全部来自 WWDC 2014 Session 419「Advanced Graphics and Animations for iOS Apps」的官方讲义（190 页，我把 PDF 下下来逐页核对过）。

管线本身在第 10 到 19 页逐步搭出来，四个方框加两个标注：

```text
Application  ──▶  Render Server  ──▶  GPU  ──▶  Display
     ↑                  ↑
Core Animation     Core Animation

16.67 ms      Commit Transaction / Decode / Draw Calls
```

`Commit Transaction` 这个词就是上面那张调用栈里的 `CA::Transaction::commit`。两边对上了。

第 20 到 29 页把 Commit Transaction 拆成四个子阶段，标题逐字是 `Layout / Display / Prepare / Commit`：

> Layout — Set up the views
> `layoutSubviews` overrides are invoked / View creation, `addSubview:` / Populate content, database lookups / Usually CPU bound or I/O bound
>
> Display — Draw contents via `drawRect:` if it is overridden / String drawing / Usually CPU or memory bound
>
> Prepare — Image decoding / Image conversion
>
> Commit — Package up layers and send to render server / Recursive / Expensive if layer tree is complex

这四个阶段在中文圈被引用了上千次，我原本打算把它标成"社区结论"，查到讲义原文之后改了：它是 Apple 自己的分法，可以直接用。

我实测只直接观测到前两个（`update_if_needed_` 和 `display_if_needed`）。Prepare 和 Commit 我没有观测到，这不说明它们不存在，只说明我的实验里没有图片解码、也没有真正的远端渲染宿主可提交。

第 37 页讲 GPU 侧的 Tile Based Rendering：

> Screen is split into tiles of NxN pixels / Each tile fits into the SoC cache / Geometry is split in tile buckets / Rasterization can begin after all geometry is submitted

这条对理解离屏渲染的代价很关键。tile 要放进 SoC 缓存才快，一旦需要切到另一块缓冲区渲染再切回来，这个局部性就被打断了。

### 一条要修正的归属

很多中文图里，管线的第一格写着 "Handle Events"。Session 419 的讲义里没有这个阶段名。190 页我翻完了。

它的真实出处是 Apple Tech Talks 2020 的 #10855「Explore UI animation hitches and the render loop」。那里把 render loop 拆成五个 phase：event、commit、render prepare、render execute、display。

这个模型更新，也更贴近现在。但它和 2014 年那张四格图是两套划分，不该拼成一张。把 "Handle Events" 塞进 Session 419 的流程图，是两个模型被缝在一起的结果。

（Tech Talk 的这部分我只有视频口述，没有讲义原文，所以阶段名之外的细节我不往下写。）

---

## 八、离屏渲染：先说清楚每条依据是什么级别

这一节最容易写成抄一份清单。我先说清楚自己站在哪：离屏渲染是不是被触发、代价多大，这两件事我在 Catalyst 上一个都测不了。我试过，下面讲怎么失败的。

所以这一节的每条结论都带一个标签：我实测的、Apple 原文的、社区的。

### 我实测的：cornerRadius 到底圆了什么

先把一个基础语义弄准，后面所有讨论都建立在它上面。头文件对 `cornerRadius` 只说了两句：

```objc
/* When positive, the background of the layer will be drawn with
 * rounded corners. Also effects the mask generated by the
 * `masksToBounds' property. Defaults to zero. Animatable. */

@property CGFloat cornerRadius;
```

"the background of the layer"。只说了背景。没提 contents，也没提子层。拿三种内容各测一遍，白底上放一个 100×100 的层，采样左上角 (2,2) 那个像素：

```text
A. 只有 backgroundColor：
  无圆角                                 角落=B FF G 00 R 00   （蓝）
  cornerRadius=30（未开 masksToBounds）  角落=B FF G FF R FF   （白，切掉了）

B. backgroundColor + 一个铺满的红色子层：
  cornerRadius=30，masksToBounds=NO      角落=B 00 G 00 R FF   （红，没切）
  cornerRadius=30，masksToBounds=YES     角落=B FF G FF R FF   （白，切了）

C. contents 是一张绿色图：
  cornerRadius=30，masksToBounds=NO      角落=B 00 G FF R 00   （绿，没切）
  cornerRadius=30，masksToBounds=YES     角落=B FF G FF R FF   （白，切了）
```

和头文件一字不差。`cornerRadius` 单独作用时只圆背景色，`contents` 和子图层原样露在外面。要裁掉它们必须开 `masksToBounds`。

（这组数据走的是 `renderInContext:` 的 CPU 路径，验的是裁剪语义。GPU 合成时是否另开缓冲区，它一个字都说明不了。）

### Apple 原文：四段讲义和几段文档

以下引文分两个来源。头文件的部分抄自当前 SDK 的 `CALayer.h`，讲义的部分抄自上一节那份 Session 419 PDF。

Group opacity（讲义第 93 页），触发条件写得极死：

> Will introduce offscreen passes:
> • If layer is not opaque (opacity != 1.0)
> • And if layer has nontrivial content (child layers or background image)
> Sub view hierarchy needs to be composited before being blended
> Always turn it off if not needed

两个条件是 AND。这一点很多清单里丢了。头文件那边的措辞对得上，说的是 "when true, and the layer's opacity property is less than one"。默认值实测：

```text
  属性                       UIView.layer     裸 CALayer
  allowsGroupOpacity           1                1
  allowsEdgeAntialiasing       0                0
  masksToBounds                0                0
  shouldRasterize              0                0
  drawsAsynchronously          0                0
```

`allowsGroupOpacity` 默认就是 YES。所以只要 `view.alpha < 1` 且它有子层，这条就成立。这大概是日常代码里最容易无意触发的一条，而且很少有人往这上面想。

Mask（讲义第 53 到 55 页）直接画出了三个 pass：

> Pass 1 — Render layer mask to texture
> Pass 2 — Render layer content to texture
> Compositing pass — Apply mask to content texture

mask 要多少次额外渲染，Apple 数给你看了。三次。

Shadow（讲义第 157、158 页），Apple 自己给的修法和头文件一致：

```objc
// 让 Core Animation 自己算阴影形状
imageViewLayer.shadowOpacity = 1.0;
imageViewLayer.shadowRadius = 2.0;
// Perhaps there is a more efficient way
imageViewLayer.shadowPath = CGPathCreateWithRect(imageRect, NULL);
```

头文件解释了为什么。不给 path，阴影形状得从 "the layer's composited alpha channel" 里推出来，也就是先把图层合成到某个地方。

第 167 页给了一张柱状图，四台设备（iPhone 5s / iPhone 5 / iPhone 4s / iPod touch）各测 `CA Shadow` 和 `shadowPath` 两种写法，目标 60 fps。`shadowPath` 那一侧四台全是 60，让 Core Animation 自己算形状的那一侧最低掉到 33。

我只敢说到这。这张图的柱子和设备名在 PDF 文字层里对不上号，具体哪台掉到 33 我读不出来，就不写。这也是 2014 年那几台设备上的数，和今天的硬件没关系，引它只为说明量级。

Rasterization（讲义第 91 页），这页解决了我原本准备标成"来源不明"的两个数字：

> Use to composite to image once with GPU
> Enable with `shouldRasterize` property on CALayer
> Extra offscreen passes when updating content
> Do not overuse, cache size is limited to 2.5x of screen size
> Rasterized images evicted from cache if unused for more than 100ms

2.5 倍屏幕和 100 ms 都是 Apple 官方讲义上的原文。我原本打算写"社区流传的数字，不采信"，查到这一页之后收回。

但要补两句。一是这两个数字发布于 2014 年，属于 OpenGL ES 时代的实现细节，Apple 此后再没重申过，现行文档里一个字都没有，Metal 时代是否还成立我不知道。二是这个数字变过。2012 年 Session 238 里 Apple 说的还是 "roughly twice the size of the screen"，objc.io 2013 年那篇继承的就是这个 2 倍。同一个数字 Apple 自己两年内改过一次，这本身就说明它是实现细节而不是契约。

还有一句被普遍忽略：`shouldRasterize` 的缓存在头文件里被称为 "an implementation detail"，用词是 "may attempt to cache"。Apple 从来没承诺过一定缓存。加上讲义第一行的 "Extra offscreen passes when updating content"，结论很清楚：内容会变的图层开 `shouldRasterize` 是纯亏。

### 那条"iOS 9 之后就不触发了"的说法

这条我查了来龙去脉。结果和我预期不一样，值得单独写。

"cornerRadius + masksToBounds 必定触发离屏"的绝对化说法，源头是 Apple 自己。WWDC 2012 Session 238 里的原话没有加任何条件。而条件式的说法也是 Apple 自己。2011 年 Session 121 的原话是 "can require an off-screen rendering pass. And it will in this case because this view actually has a bunch of subviews"。注意那个 in this case。2020 年的 Tech Talk 10857 又回到条件式，说圆角 "can sometimes require an offscreen"。条件是两条：渲染器有没有拿到足够的信息，有没有需要被裁剪的 sublayer。

至于"iOS 9 起纯色背景不再触发"这个版本号叙事：我没有找到任何 Apple 的声明，release notes、WWDC、文档里都没有。社区里能追溯到的、较早一份带完整测试环境和数据的报告是 seedante 的 OptimizationForOffscreenRender（iPad mini 一代 / iOS 9.3.1）。但它的主张和中文圈转述的版本不是一回事。它说的是：`contents` 为 nil 时你根本不需要设 `masksToBounds`，只设 `backgroundColor` + `cornerRadius` 就有圆角，离屏自然绕开了。它没说"设了 `masksToBounds` 也不离屏"。

这一点和我上面那组实测正好撞上。纯色背景的 view 压根不需要 `masksToBounds`，我的 A 组数据就是证明。所以这条流传的说法真正该讲的不是版本变化，是很多人给纯色 view 加了一个用不着的 `masksToBounds`。

我的建议是干脆放弃版本号叙事，改用 Apple 2020 那个框架：圆角是否离屏，取决于渲染器有没有足够信息、以及有没有需要裁剪的子层。这个说法有一手背书，而且能同时解释两派实测结果。

### 我怎么试的，怎么失败的

QuartzCore 里有 `CA_COLOR_OFFSCREEN` 这个环境变量，就是那个"离屏染黄"选项的底层开关。我的想法是：既然本地 `CAContext` 能在进程内渲染，那用 `CARenderer` 把图层树渲染进一张 Metal 纹理，再把像素读回来，就能看出哪些配置被染了黄。

写完跑了两版，第二版老老实实按头文件说的传了自己的 `kCARendererMetalCommandQueue` 并 `waitUntilCompleted`：

```text
device=Apple M1 Pro  CA_COLOR_OFFSCREEN=(未设)

baseline 纯蓝方块                  非零字节=0  中心=B00 G00 R00 A00
cornerRadius+masksToBounds         非零字节=0  中心=B00 G00 R00 A00
shadow 无 shadowPath               非零字节=0  中心=B00 G00 R00 A00
mask                               非零字节=0  中心=B00 G00 R00 A00
shouldRasterize                    非零字节=0  中心=B00 G00 R00 A00
```

整张纹理一个非零字节都没有。`CARenderer` 在 Catalyst 下根本没往里写东西，染色开不开已经无所谓了，我连 baseline 都渲染不出来。

在这上面花的时间比预期多，收获是一条负面结论：**Catalyst 能验 UIKit 的结构和 API 契约，验不了渲染。** 这和这个系列前面几篇的经验一致，只是边界比我想象中更靠前。

### 汇总：每条依据是什么级别

| 说法 | 级别 |
|---|---|
| `cornerRadius` 只圆背景色，裁 `contents` / 子层要 `masksToBounds` | 头文件原文 + 本文实测 |
| 不给 `shadowPath` 时阴影形状来自合成后的 alpha 通道 | 头文件原文 |
| filters 会强制栅格化 | 头文件原文 |
| group opacity 需要 `opacity != 1.0` 且有 nontrivial content 两个条件 | Session 419 讲义第 93 页 |
| mask 是 2 个渲染 pass 加 1 个合成 pass | Session 419 讲义第 53–55 页 |
| `shouldRasterize` 缓存上限 2.5 倍屏幕、100 ms 未用即逐出 | Session 419 讲义第 91 页。2014 年数据，此后未更新，Metal 时代是否成立未知 |
| 圆角是否离屏取决于信息是否充分、有无待裁剪子层 | Tech Talk 10857 口述，我只有视频没有讲义 |
| edge antialiasing 触发离屏 | Apple 只在 2011 年 Session 121 说过，之后未再确认；社区在 iOS 8/9 上实测复现不出来 |
| "iOS 9 之后纯色背景 + 圆角 + 裁剪不再离屏" | 社区结论，无任何 Apple 依据。且转述失真，原报告讲的是压根不要设 `masksToBounds` |
| 上述各条在今天的 GPU 上具体开销几次外缓冲 | 我没有观测手段，本文未验证 |
| 圆角、阴影、mask 的相对性能排序 | 必须自己实测。结论依赖内容、动画和系统版本，任何写死的排序都别信 |

### 今天该怎么在真机上测

先修正一个几乎所有中文文章都还在教的做法。Instruments 的 Core Animation 模板早就没了。

Xcode 9.3（2018 年 4 月）的 release notes 写着它和 template 一起废弃，Debug Options 里的功能搬进了 Xcode 的 Debug > View Debugging。我在本机 Instruments 26.6 里列了全部 `.tracetemplate`。确实找不到 Core Animation。

有点讽刺的是，上面那份 2014 年的讲义自己在第 169 页写的就是 "Use Core Animation instrument to find them"。四年之后这个工具就没了。引一手材料也得看年份。

正确的位置在这里。我在本机 Xcode 26.6 的菜单定义数据里核对过：

```text
Debug > View Debugging > Rendering >
    Color Blended Layers
    Color Copied Images
    Color Misaligned Images
    Color Offscreen-Rendered Yellow      ← 离屏渲染染黄
    Color Hits Green and Misses Red
    Color Compositing Fast-Path Blue     ← 老文章里叫 Color OpenGL Fast Path Blue
    Color Layer Formats
    Color Immediately
    Flash Updated Regions
```

染色只是入门手段。Apple 现在推荐的两个工具都在 View Debugger 里，比染色好用得多。

一个是 `Editor > Show Layers`。选中一个 layer，inspector 直接给出 offscreen count 和 offscreen flags（比如 `offscreen mask`）。这个图层离屏了几次、为什么，一眼就有。

另一个是 `Editor > Show Optimization Opportunities`，它会在 Runtime Issue Navigator 里给出具体的修改建议。

帧率和卡顿用 Instruments 的 Animation Hitches 模板。它的 render count 一列就是 GPU 的离屏 pass 次数。

> 待真机补测：以下这条实验阶梯我没有跑，数字一个都没有。
>
> 1. 真机，不要用模拟器，模拟器的 GPU 路径不同。
> 2. 跑一个列表，每 cell 一个头像视图，按阶梯改一步测一次：纯色背景 + `cornerRadius`（不开 `masksToBounds`）→ 加 `masksToBounds` → 换成图片内容 + `masksToBounds` → 加动态阴影（无 `shadowPath`）→ 补上 `shadowPath` → 开 `shouldRasterize`。
> 3. 每步记三个数：`Show Layers` 里该 layer 的 offscreen count、Animation Hitches 的 render count、FPS 曲线。
> 4. 同时开 Color Blended Layers（红色是混合区）区分混合和离屏，这是两回事。
>
> 要回答的问题：第 2 步哪几级的 offscreen count 真的大于 0；`shouldRasterize` 那一级在内容静止和内容每帧变化两种情况下各是什么表现。这两个问题的答案我现在不知道，本文不猜。

---

## 总结

`UIView` 用 `_layerRetained` 这个 ARC 强引用持有 layer，另外拿 `_layer` 存一份不参与 ARC 的裸指针；layer 反过来 weak 引用 view 当 delegate。两个对象之间只有四个方法：`layoutSublayersOfLayer:`、`drawLayer:inContext:`、`layerWillDraw:`、`actionForLayer:forKey:`。layer 在 `init` 里就建好了，`+layerClass` 只被问一次。

`subviews` 是 `layer.sublayers` 按 `delegate` 映射出来的，中间隔一层缓存；`superview` 是另一份独立状态。混着用图层 API 和视图 API，能造出 `superview` 为 nil 而 `subviews` 里还有它的自相矛盾状态，且不报错。

model layer 就是你手上那个对象，presentation layer 是提交之后才存在的另一个对象，动画期间两者的值分叉。render tree 没有公开入口，本文没写它的任何行为。

隐式动画的关闭点在 `actionForKey:` 的第一步：`UIView` 作为 delegate 返回 `NSNull`，layer 按文档把 `NSNull` 归一成 nil。所以 `actionForKey:` 返回的是 nil，不是 `NSNull`。同一个方法在 `UIView` 动画 block 里会改口返回 `_UIViewAdditiveAnimationAction`。

只要 `drawRect:` 这个方法存在，哪怕方法体是空的，backing store 就会被分配。100 个 320pt 视图在 3 倍 scale 下，不实现是 2.3 MB，空实现是 92 MB，真画满是 356 MB。

`setNeedsDisplay` 调 100 次只换来一次 `drawRect:`，收账的是 `CA::Transaction` 注册在 `kCFRunLoopBeforeWaiting` 上、order 为 2000000 的那个 observer。同一次提交里布局永远先于绘制，顺序是硬编码的。

离屏渲染那一节我自己一个数字都没测出来。写在里面的 2.5 倍屏幕、100 ms、mask 的三个 pass，全是 Apple 讲义上的原文，我做的只是把 PDF 下下来逐页核对，再把每条依据是什么级别标清楚。

最后一条是方法论。这篇里最硬的证据是两张 demangle 之后的调用栈，它们把 RunLoop、事务提交、布局、绘制、backing store 一次串完，不需要相信任何人。而我原本准备把两件事写成"社区结论不采信"，查完讲义发现两条都是 Apple 自己说的。往哪个方向偷懒都会错。

下一篇 [[iOS 事件传递与响应者链：hitTest、手势与那些点不动的按钮]]。

## 参考资料

### 一手：可以逐字引用的

- 当前 macOS SDK 的 `QuartzCore.framework/Headers/CALayer.h`。本文关于 `presentationLayer` / `modelLayer` / `actionForKey:` / `cornerRadius` / `shadowPath` / `shouldRasterize` / `allowsGroupOpacity` / `mask` 的引文全抄自这里
- 当前 iOS SDK 的 `UIKit.framework/Headers/UIView.h`。`layer` 属性那行注释里的 "view is layer's delegate" 是官方对这层关系的原话
- [WWDC 2014 Session 419 — Advanced Graphics and Animations for iOS Apps 官方讲义 PDF](https://devstreaming-cdn.apple.com/videos/wwdc/2014/419xxli6f60a6bs/419/419_advanced_graphics_and_animation_performance.pdf)（190 页）。第七、第八节所有讲义引文的出处：管线四格图在 p10–19，Commit Transaction 四阶段在 p20–29，tile based rendering 在 p37，mask 的三个 pass 在 p53–55，rasterization 的 2.5x / 100ms 在 p91，group opacity 的两个条件在 p93，shadowPath 在 p157–158 与 p167。[对应视频](https://developer.apple.com/videos/play/wwdc2014/419/)
- [Core Animation Basics](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/CoreAnimationBasics/CoreAnimationBasics.html)：三棵树这个说法的官方出处
- [Setting Up Layer Objects](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/SettingUpLayerObjects/SettingUpLayerObjects.html)

### 一手但只有视频口述，我没有讲义原文

引用时我只用了阶段名和定性说法，没有引任何数字。

- [Tech Talks 10855 — Explore UI animation hitches and the render loop](https://developer.apple.com/videos/play/tech-talks/10855/)：render loop 五阶段模型，也是 "Handle Events" 的真实出处
- [Tech Talks 10857 — Demystify and eliminate hitches in the render phase](https://developer.apple.com/videos/play/tech-talks/10857/)：Apple 目前最系统的离屏渲染论述，以及 View Debugger 的 offscreen count 用法
- [WWDC 2011 Session 121 — Understanding UIKit Rendering](https://developer.apple.com/videos/play/wwdc2011/121/)：圆角离屏最早的条件式表述
- [WWDC 2012 Session 238 — iOS App Performance: Graphics and Animations](https://developer.apple.com/videos/play/wwdc2012/238/)：绝对化说法的源头，也是"2 倍屏幕大小"那个旧数字的出处
- [WWDC 2012 Session 506 — Optimizing 2D Graphics and Animation Performance](https://developer.apple.com/videos/play/wwdc2012/506/)

### 二手，注明成文时间

- [objc.io — Getting Pixels onto the Screen](https://www.objc.io/issues/3-views/moving-pixels-onto-the-screen/)（Daniel Eggert，2013 年 8 月）。讲合成原理的部分仍然值得读。但它以 OpenGL ES 为基础，说的缓存大小是 2 倍（Apple 2014 年改成了 2.5 倍），把圆角说成"一种特殊的 mask"（Apple 2020 年明确说 `masksToBounds` 比自建 mask layer 快得多），教的 Instruments Core Animation 模板已于 2018 年废弃。这三处都别照抄
- [seedante/OptimizationForOffscreenRender](https://github.com/seedante/OptimizationForOffscreenRender)（iPad mini 一代 / iOS 9.3.1）。"纯色背景不必设 `masksToBounds`"这条社区结论我能追溯到的较早一份带完整数据的报告，注意它的主张和中文圈的转述有出入
- [Advanced UIView shadow effects using shadowPath](https://www.hackingwithswift.com/articles/155/advanced-uiview-shadow-effects-using-shadowpath)
- [Apple Developer Forums thread 100507](https://developer.apple.com/forums/thread/100507)：Core Animation instrument 废弃于 Xcode 9.3 的出处

### 本地

- [[iOS 坐标系：frame 是算出来的，bounds 和 transform 才是存的]] —— 几何属性全部在那一篇
- [[iOS RunLoop：mode、source 与那张流程图今天还对不对]] —— 第六节那个 observer 的机制背景
- [[iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint]] —— 第五节 backing store 的内存账用的是那一篇的仪器和结论
- [[iOS 对象通信：delegate、通知、target-action 与 block 回调]] —— ivarLayout 解码方法的出处

---

实验环境：Xcode 26.6（Build 17F113），macOS 26.5，Apple M1 Pro（arm64）。全部实验编译为 Mac Catalyst target：

```shell
SDK=$(xcrun --sdk macosx --show-sdk-path)
clang -fobjc-arc -target arm64-apple-ios17.0-macabi -isysroot "$SDK" \
      -iframework "$SDK/System/iOSSupport/System/Library/Frameworks" \
      -framework Foundation -framework UIKit -framework QuartzCore -framework CoreGraphics \
      -o out prog.m && ./out
```

产物是原生 macOS 二进制，直接 `./out` 运行。没有启动任何模拟器，也不是真机。它链接并加载的是真正的 `UIKitCore`，所以 ivar 布局、类结构、delegate 契约、action 查找、事务提交时序这些跑的都是 UIKit 自己的代码。

适用边界要说死：结构与 API 契约可信，渲染相关的一切不可信。backing store 的存在性、尺寸公式和 footprint 数量级是在 macOS 的内存子系统上量的。`contentsScale` 默认 1.0，iOS 上会被设成屏幕 scale。`opaque` 对像素格式没表现出影响。`CARenderer` 完全不工作。凡是涉及 GPU、合成、离屏 pass、帧率的结论，本文一律没有自己给数字，只给了真机复现步骤和 Apple 的原文。

另外，第三节用了私有类 `CAContext` 的 `+localContext` 作为实验仪器，只为了让图层树进入一个渲染上下文以观测 `presentationLayer`。它不是可以写进产品代码的东西，正式验证请按第三节末尾的方法在 `UIWindow` 里做。
