---
title: 【iOS】坐标系：frame 是算出来的，bounds 和 transform 才是存的
published: 2026-07-27
description: 给一个旋转了 45° 的 view 写 frame，UIKit 会把 frame.size 当成向量套一次逆变换，bounds 的高度当场塌成 8.2e-15。官方说这时 frame「undefined」，这就是 undefined 的具体长相。
tags:
  - iOS
  - UIKit
  - CALayer
  - 坐标系
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 21
draft: true
---
# 坐标系：frame 是算出来的，bounds 和 transform 才是存的

先看一组数字。一个 `UIView`，`initWithFrame:` 传 `{{100,100},{200,100}}`，然后只做一件事：把 `transform` 设成旋转 45°。

```text
初始        frame={{100, 100}, {200, 100}}                       bounds={{0, 0}, {200, 100}}  center={200, 150}
旋转 45°后  frame={{93.933982, 43.933982}, {212.132034, 212.132034}}  bounds={{0, 0}, {200, 100}}  center={200, 150}
```

`frame` 三个数全变了，`bounds` 和 `center` 一动没动。我没碰过这两个属性，UIKit 也没替我改。这说明 `frame` 根本没被存在任何地方，它是每次读的时候现算的。

**真正被存下来的是 `bounds`、`layer.position` 和 `transform` 这三样，`frame` 是它们的函数。** 这句话是本文的全部内容，下面每一节都是它的一个推论。

---

## 一、frame 的 getter 是什么

先把 212.132 这个数验一遍。一个 200×100 的矩形转 45°，它的外接矩形边长是：

```
W = w·|cosθ| + h·|sinθ| = 200×0.7071 + 100×0.7071 = 212.1320
H = w·|sinθ| + h·|cosθ| = 200×0.7071 + 100×0.7071 = 212.1320
origin = center - size/2 = (200 - 106.066, 150 - 106.066) = (93.9340, 43.9340)
```

和实测逐位吻合。所以旋转之后的 `frame` 是外接矩形，而外接矩形和这个 view「多大」已经没什么关系了。它转到 45° 时最胖，转到 90° 时反而回到 100×200。

外接矩形这个说法只在纯旋转下成立。完整的 getter 我用三组 `anchorPoint` 反推过：把 `bounds` 的四个角相对 `anchorPoint` 做变换，取包围盒，再加上 `position`。

```text
anchor=0.0  实测 frame={{250, 300}, {223.205081, 186.602540}}         手算=({250.0000, 300.0000},{223.2051, 186.6025})
anchor=0.5  实测 frame={{188.397460, 206.698730}, {223.205081, ...}}  手算=({188.3975, 206.6987},{223.2051, 186.6025})
anchor=1.0  实测 frame={{126.794919, 113.397460}, {223.205081, ...}}  手算=({126.7949, 113.3975},{223.2051, 186.6025})
```

三组都对得上。`frame` 里没有一个数字是独立存在的。

顺便把另外三种变换也各跑一遍，好确认变的确实只有 `frame`：

```text
缩放 2x 后      frame={{0, 50}, {400, 200}}      bounds={{0, 0}, {200, 100}}  center={200, 150}
平移(30,40)后   frame={{130, 140}, {200, 100}}   bounds={{0, 0}, {200, 100}}  center={200, 150}
恢复 identity   frame={{100, 100}, {200, 100}}   bounds={{0, 0}, {200, 100}}  center={200, 150}
```

平移那行值得多看一眼。`transform` 的平移分量把 `frame.origin` 推到了 (130,140)，`center` 还停在 (200,150)。此刻这个 view 视觉上在哪，`center` 已经答不上来了，`frame.origin` 也答不上来（它是包围盒的角）。想知道屏幕上那块像素在哪，只能自己算，或者用第五节的换算方法。

## 二、「undefined」的具体长相

官方文档的措辞是这样的：

> **Important:** If a view's `transform` property is not the identity transform, the value of that view's `frame` property is undefined and must be ignored. When applying transforms to a view, you must use the view's `bounds` and `center` properties to get the size and position of the view.

SDK 头文件里也有一句更短的：

```objc
// animatable. do not use frame if view is transformed since it will not correctly reflect the actual location of the view. use bounds + center instead.
@property(nonatomic) CGRect            frame;
```

这两句话我以前读过很多遍，但一直只当成「读出来的值不准」。直到我去试了写。

一个 200×100 的 view，转 45°，然后 `v.frame = CGRectMake(0,0,50,50)`：

```text
写之前              frame={{93.933982, 43.933982}, {212.132034, 212.132034}}  bounds={{0, 0}, {200, 100}}  center={200, 150}
frame=(0,0,50,50)后 frame={{-5.8e-15, -3.5e-15}, {50.000000, 50}}             bounds={{0, 0}, {70.710678, 8.2156503822261584e-15}}  center={25, 25}
```

`bounds` 的高度塌成了 8.2e-15。这个 view 现在是一根没有厚度的线。

我最初以为这是某种舍入误差堆出来的，后来发现它有精确的规律。`frame` 的 setter 拿到新的 `size`，**把它当成一个向量，套了一次 `transform` 的逆变换的线性部分，再取绝对值，结果就是新的 `bounds.size`**。逆旋转 45° 作用在向量 (50,50) 上，y 分量恰好是 0。

拿六组不同的变换验一遍：

```text
rot 45°,  (50,50)      实测 bounds.size=(  70.71068,   0.00000)   |M⁻¹·(w,h)|=(  70.71068,   0.00000)
rot 60°,  (120,80)     实测 bounds.size=( 129.28203,  63.92305)   |M⁻¹·(w,h)|=( 129.28203,  63.92305)
rot 90°,  (100,40)     实测 bounds.size=(  40.00000, 100.00000)   |M⁻¹·(w,h)|=(  40.00000, 100.00000)
scale(2,3), (400,300)  实测 bounds.size=( 200.00000, 100.00000)   |M⁻¹·(w,h)|=( 200.00000, 100.00000)
shear c=0.5, (100,100) 实测 bounds.size=(  50.00000, 100.00000)   |M⁻¹·(w,h)|=(  50.00000, 100.00000)
rot30·scale2, (200,150) 实测 bounds.size=( 124.10254,  14.95191)  |M⁻¹·(w,h)|=( 124.10254,  14.95191)
```

六组全中，包括剪切和复合变换。

这个公式在只有缩放的时候是完全正确的：`scale(2,3)` 下写 `frame.size=(400,300)` 得到 `bounds.size=(200,100)`，正是你想要的。一旦矩阵里有旋转分量，它就变成一个几何上毫无意义的运算 —— 把一个「宽高」当成向量去旋转。

所以 undefined 的意思不是「UIKit 拒绝服务」，是它照样给你算，只是算出来的东西没有任何解释。这比直接报错难查得多。

我自己现在的做法很简单：**只要一个 view 有可能被加 `transform`，我就不再对它写 `frame`，一律走 `bounds.size` 加 `center`。** 不判断当前 transform 是不是 identity，因为「当前」这个词在动画里没有意义。

`transform` 本身在写 `frame` 之后是保留的（实测 `a=0.7071 b=0.7071 c=-0.7071 d=0.7071` 原样还在）。被改掉的只有 `bounds` 和 `center`。

## 三、bounds.origin 就是滚动

`bounds.size` 好理解，`bounds.origin` 是这套 API 里最容易被跳过的一个。默认是 (0,0)，很多人写了几年 iOS 也没见它变过。

它的含义是：这个 view 用自己坐标系里的哪一块矩形来摆放子视图。

搭一棵三层树验证。`container` 的 frame 是 (50,100,300,200)，里面装一个 600 高的 `content`，`content` 里有个 `item` 在 y=400 的位置：

```text
container.frame={{50, 100}, {300, 200}} bounds={{0, 0}, {300, 200}}
content.frame  ={{0, 0}, {300, 600}}
item.frame     ={{20, 400}, {100, 40}}
item 在 container 坐标系 = {{20, 400}, {100, 40}}
item 在 root 坐标系      = {{70, 500}, {100, 40}}
```

现在只改一个属性，`container.bounds = CGRectMake(0, 380, 300, 200)`：

```text
container.frame={{50, 100}, {300, 200}}   bounds={{0, 380}, {300, 200}}
content.frame  ={{0, 0}, {300, 600}}      (没动)
item.frame     ={{20, 400}, {100, 40}}    (没动)
container.center={200, 200}               (没动)
item 在 container 坐标系 = {{20, 400}, {100, 40}}     (没动)
item 在 root 坐标系      = {{70, 120}, {100, 40}}     ← 只有这个变了
```

`container` 自己在父视图里的位置和大小完全没变，它的所有子视图的 `frame` 也一个没动。变的只有「这些子视图最终落在屏幕的哪里」，整体上移了 380。

这就是滚动。没有任何一个 view 被移动过，被移动的是观察窗口。

`UIScrollView` 就是这么干的，而且它连一层包装都没加：

```text
初始              bounds={{0, 0}, {300, 200}}    offset={0, 0}
offset:=(0,250)   bounds={{0, 250}, {300, 200}}
bounds.origin:=(17,99) -> offset={17, 99}
```

`contentOffset` 和 `bounds.origin` 是同一个存储，双向的。写哪边另一边都跟着变。知道这一点之后，`UIScrollView` 那套 `contentSize` / `contentOffset` / `contentInset` 就没什么神秘的了 —— 前者是内容有多大，中间那个是窗口开在哪，后者是给窗口四周留的余量。

这也解释了一个常见的困惑：为什么在 `scrollView` 里的 subview 上打印 `frame`，滚动前后是一样的？因为它本来就该一样。它相对 `contentView` 的位置从来没变过。

## 四、center、position 和 anchorPoint

头文件里 `center` 的注释是这么写的：

```objc
@property(nonatomic) CGPoint           center;      // center is center of frame, relative to anchorPoint. animatable
```

「center of frame」这半句会误导人。测一下就知道：

```text
anchor=(0,0) 设 center=(500,500) -> layer.position={500, 500}  frame={{500, 500}, {200, 100}}
```

`center` 是 (500,500)，`frame` 的几何中点是 (600,550)。它们不相等。

`center` 真正对应的是 `layer.position`，而且是同一块存储：

```text
默认 anchor(0.5,0.5)  center={200, 150}  layer.position={200, 150}   center == layer.position ? 1
设 layer.position=(10,20) -> view.center={10, 20}
```

写 `layer.position` 读 `center`，写 `center` 读 `layer.position`，两边永远一致。`anchorPoint` 默认在 (0.5,0.5)，也就是 `bounds` 的正中，所以「position 落在中心」这个默认情况让 `center` 这个名字看起来是对的。改掉 `anchorPoint`，名字就不成立了。

那 `anchorPoint` 到底在做什么。固定 `position` 不动，只扫 `anchorPoint`：

```text
anchor=(0.00,0.00)  position={200, 150}  frame.origin={200, 150}   预测 origin=(200.0, 150.0)
anchor=(0.25,0.25)  position={200, 150}  frame.origin={150, 125}   预测 origin=(150.0, 125.0)
anchor=(0.50,0.50)  position={200, 150}  frame.origin={100, 100}   预测 origin=(100.0, 100.0)
anchor=(0.75,0.75)  position={200, 150}  frame.origin={50, 75}     预测 origin=(50.0, 75.0)
anchor=(1.00,1.00)  position={200, 150}  frame.origin={0, 50}      预测 origin=(0.0, 50.0)
```

预测列用的公式是：

```
frame.origin.x = position.x - bounds.size.width  × anchorPoint.x
frame.origin.y = position.y - bounds.size.height × anchorPoint.y
```

五组全中。这条公式在中文圈流传很广，我这次是先跑数据再去对公式的，顺序反过来更有说服力。

`anchorPoint` 是归一化的比例值，因为它要表达「`position` 落在 `bounds` 的百分之几处」，用绝对坐标的话 `bounds` 一变就得跟着改。它同时也是所有 `transform` 的不动点，第一节里旋转 45° 之后 `center` 纹丝不动，就是因为默认锚点在中心。

### 头文件这条注释是错的

```objc
/* Defines the anchor point of the layer's bounds rect, as a point in
 * normalized layer coordinates - '(0, 0)' is the bottom left corner of
 * the bounds rect, '(1, 1)' is the top right corner. Defaults to
 * '(0.5, 0.5)', i.e. the center of the bounds rect. */
@property(nonatomic) CGPoint anchorPoint API_AVAILABLE(ios(16.0)) API_UNAVAILABLE(watchos);
```

这是 `UIView.h` 里的原文，写着 (0,0) 是 bottom left。测一下：

```text
position=(0,0) anchor=(0,0) -> frame={{0, 0}, {200, 100}}
  若 (0,0)=左上，frame 应为 {{0,0},{200,100}}；若 =左下，应为 {{0,-100},{200,100}}
layer.geometryFlipped = 0, contentsAreFlipped = 1
```

在 UIKit 里 (0,0) 是左上角。这段注释是从 `CALayer` 的 macOS 文档原样搬过来的，macOS 那边 y 轴朝上，左下才对。iOS 的 y 轴朝下，整段描述需要上下颠倒着读。

## 五、CGAffineTransform 的六个数

`CGAffineTransform` 是六个 `CGFloat`：

```c
struct CGAffineTransform {
  CGFloat a, b, c, d;
  CGFloat tx, ty;
};
```

对应的矩阵和点变换公式也在同一个头文件里，是行向量约定：

```c
/* Transform `point' by `t' and return the result:
     p' = p * t
   where p = [ x y 1 ]. */
```

```c
p.x = t.a * point.x + t.c * point.y + t.tx;
p.y = t.b * point.x + t.d * point.y + t.ty;
```

三种基本变换各占哪几个位置，跑一遍就清楚：

```text
identity        a= 1.0000 b= 0.0000 c= 0.0000 d= 1.0000 tx=   0.000 ty=   0.000
平移 (30,40)    a= 1.0000 b= 0.0000 c= 0.0000 d= 1.0000 tx=  30.000 ty=  40.000
缩放 (2,3)      a= 2.0000 b= 0.0000 c= 0.0000 d= 3.0000 tx=   0.000 ty=   0.000
旋转 30°        a= 0.8660 b= 0.5000 c=-0.5000 d= 0.8660 tx=   0.000 ty=   0.000
旋转 90°        a= 0.0000 b= 1.0000 c=-1.0000 d= 0.0000 tx=   0.000 ty=   0.000
```

`tx`/`ty` 只管平移。`a`/`d` 在没有旋转时就是 x/y 的缩放系数。`b`/`c` 一旦非零，里面就掺了旋转或剪切，这时 `a` 已经不等于缩放倍数了（旋转 30° 的 `a` 是 0.866，但没有任何缩放）。

### 顺序：写在左边的先作用

矩阵乘法不交换，这是常识。真正坑人的是 `CGAffineTransformRotate` 这一族函数把新操作放在了哪一边。头文件写得很直白：

```c
/* Translate `t' by `(tx, ty)' and return the result:
     t' = [ 1 0 0 1 tx ty ] * t */
```

```c
/* Concatenate `t2' to `t1' and return the result:
     t' = t1 * t2 */
```

行向量约定下 `p' = p * t1 * t2`，所以 `t1` 先作用。而 `CGAffineTransformTranslate(t, tx, ty)` 把新的平移矩阵放在了 `t` 的左边，意味着新操作先执行。这一族函数全都是前置。

验证：

```text
Concat(T,R)  = 先平移后旋转      a= 0.000 b= 1.000 c=-1.000 d= 0.000 tx=   0.00 ty= 100.00
Translate(R,100,0)               a= 0.000 b= 1.000 c=-1.000 d= 0.000 tx=   0.00 ty= 100.00
  两者相等? 1

Concat(R,T)  = 先旋转后平移      a= 0.000 b= 1.000 c=-1.000 d= 0.000 tx= 100.00 ty=   0.00
Rotate(T,90°)                    a= 0.000 b= 1.000 c=-1.000 d= 0.000 tx= 100.00 ty=   0.00
  两者相等? 1
```

拿一个点走一遍，两条路的差别就是肉眼可见的：

```text
先平移后旋转: (10,0) --T--> (110,0) --R--> (0,110)，实测 {6.7e-15, 110}
先旋转后平移: (10,0) --R--> (0,10)  --T--> (100,10)，实测 {100, 10}
```

挂到 view 上，最终 `frame` 差一个屏幕的距离：

```text
起始         frame={{0, 0}, {200, 100}}      center={100, 50}
先平移后旋转 frame={{50, 50}, {100, 200}}    frame 中点=(100, 150)
先旋转后平移 frame={{150, -50}, {100, 200}}  frame 中点=(200, 50)
```

两者的 `center` 都还是 (100,50)，位移全部体现在 `frame` 上，和第一节一致。

差别的来源可以一句话说清：先平移的那种，位移向量 (100,0) 随后被旋转 90° 变成了 (0,100)；先旋转的那种，位移向量原样保留。所以「先做的操作，会被后做的操作作用一次」。

我踩过的坑是这个：以为 `CGAffineTransformTranslate(view.transform, 100, 0)` 是「在当前基础上往右挪 100」。它其实是「在当前坐标系里往 x 正方向挪 100」，如果当前已经转了 90°，这个 x 正方向在屏幕上是朝下的。想要「在屏幕坐标系里往右挪 100」，得写 `CGAffineTransformConcat(view.transform, CGAffineTransformMakeTranslation(100, 0))`。

## 六、坐标换算四件套

```objc
- (CGPoint)convertPoint:(CGPoint)point toView:(nullable UIView *)view;
- (CGPoint)convertPoint:(CGPoint)point fromView:(nullable UIView *)view;
- (CGRect)convertRect:(CGRect)rect toView:(nullable UIView *)view;
- (CGRect)convertRect:(CGRect)rect fromView:(nullable UIView *)view;
```

先建一棵树：`A` 是根 (0,0,400,800)，`B` 是 A 的子 (50,100,300,400)，`C` 是 B 的子 (20,30,100,100)，`D` 是 A 的另一个子 (10,10,50,50)。

无变换的基线：

```text
C.bounds 原点在 A 里    = {70, 130}      手算 = (50+20, 100+30)
A 里的 (70,130) 回到 C  = {0, 0}
C 的整块矩形到 D 坐标系 = {{60, 120}, {100, 100}}   手算 = (70-10, 130-10)
```

加上 `bounds.origin` 偏移，公式里多一项减法：

```text
B.bounds := {{15, 25}, {300, 400}}
C 原点在 A 里 = {55, 105}   预测 (50+20-15, 100+30-25)=(55,105)
```

所以父到子的换算是 `子.frame.origin - 父.bounds.origin` 逐层累加，这和第三节说的「`bounds.origin` 决定子视图落在哪」是同一件事的两种表述。

给 `B` 加 90° 旋转之后，就不能再用 `frame.origin` 了：

```text
B.center={200, 300}  B.bounds={{0, 0}, {300, 400}}
C 原点在 A 里 = {370, 170}
手算：center=(200,300) + R90·((20,30)-(150,200)) = (370.0000, 170.0000)
```

手算用的公式是 `A 坐标 = B.center + M·(p - anchorPoint × B.bounds.size)`，和第一节 getter 的公式是同一个。逐位吻合。

`convertRect:` 在有旋转时返回的是外接矩形，跟 `frame` 的逻辑一样：

```text
C 的 bounds 换到 A = {{270, 170}, {100, 100}}
```

C 是个 100×100 的正方形，转 90° 之后外接矩形还是 100×100，所以这里看不出来。旋转 45° 的话就会看到尺寸变大。

### 传 nil 是什么意思

文档说 nil 表示 window 坐标系。我测了两种情况。

有 window 的时候，`toView:nil` 和 `toView:window` 结果完全相同：

```text
[B convertPoint:zero toView:nil] = {40, 60}
[B convertPoint:zero toView:win] = {40, 60}
[B convertPoint:(0,0) fromView:nil] = {-40, -60}
给 A 加平移(1000,2000)后 toView:nil = {1040, 2060}
```

没有 window 的时候（比如一棵还没被 `addSubview:` 进去的视图树），它原样返回输入：

```text
[det convertPoint:zero toView:nil] = {0, 0}  (det.window = 无)
```

这一条比较危险。你在 `viewDidLoad` 里对一个还没上屏的 view 调 `convertPoint:toView:nil`，拿到的是没有意义的原值，而且不会报任何错。我的习惯是能写具体的目标 view 就不写 nil。

还有一个没写在文档里的情况我也测了：两个没有共同祖先的 view 之间换算，UIKit 会把两棵树的根当成在同一个坐标空间里。

```text
[C convertPoint:zero toView:lonely] = {363, 161}
```

C 在 A 里是 (370,170)，`lonely.frame.origin` 是 (7,9)，363 = 370−7。这个行为我没在任何文档里见到，别依赖它。

## 七、transform 之后，hitTest 认哪套坐标

把一个 (100,100,200,50) 的横条转 90°：

```text
旋转 90° 后 box.frame={{175, 25}, {50, 200}}  center={200, 125}  bounds={{0, 0}, {200, 50}}

点 (110,110) 旋转前的左上角区域（视觉上现在是空的）  -> 没命中
点 (200,110) 旋转后视觉上盖住的区域（竖条上部）      -> box 命中
点 (200,200) 旋转后中心                              -> box 命中
点 (180,220) 旋转后竖条内偏左                        -> box 命中
点 (290,125) 旋转前矩形右端（视觉上现在是空的）      -> 没命中
```

答案是旋转后的位置。看得见的地方点得到，看不见的地方点不到。

机制在头文件里写着：

```objc
- (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event;   // default returns YES if point is in bounds
```

`pointInside:` 判的是 `bounds`，`bounds` 是不受 `transform` 影响的。所以命中判断这一步始终在未变换的坐标系里做。`transform` 的作用发生在上一步：父视图把点换算到子视图坐标系时，套的是 `transform` 的逆。

把中间量打出来能看得更清楚：

```text
root(110,110) -> box 局部 {85, 115}      pointInside=0    (115 > bounds 高 50)
root(200,110) -> box 局部 {85, 25.0}     pointInside=1
root(200,200) -> box 局部 {175, 25.0}    pointInside=1
root(180,220) -> box 局部 {195, 45.0}    pointInside=1
root(290,125) -> box 局部 {100, -65.0}   pointInside=0
```

所以「hitTest 用变换后的坐标」和「pointInside 用 bounds」两句话不矛盾，它们说的是流水线上的两个不同环节。这条链完整走一遍在 [[iOS 事件传递与响应者链：hitTest、手势与那些点不动的按钮]]。

### 一个可以当 bug 用的边界

`transform` 设成 `scale(0,0)` 是隐藏 view 的一种常见写法（尤其在动画的起止状态里）。它的 `frame` 会塌成一个点：

```text
scale(0,0) frame={{50, 50}, {0, 0}}
hitTest(50,50) 命中 z? 1
z 局部坐标 = {50, 50}
```

看不见，但点得到。原因在 CoreGraphics 的头文件里：

```c
/* Invert `t' and return the result. If `t' has zero determinant, then `t'
   is returned unchanged. */
```

行列式为 0 时 `CGAffineTransformInvert` 原样返回输入。于是父到子的坐标换算变成了一次身份映射，(50,50) 原封不动传给 `pointInside:`，落在 `bounds` 里面，命中。

用 `alpha = 0` 或者 `hidden = YES` 隐藏就没这个问题，`hitTest:` 对这两者有显式判断。缩放到 0 没有。

## 八、layer.frame、transform3D 和几个别名

`view.frame` 和 `layer.frame` 是同一个东西，连浮点误差都一样：

```text
view.frame  = {{-1.602540378443871, -23.301270189221924}, {223.20508075688775, 186.60254037844385}}
layer.frame = {{-1.602540378443871, -23.301270189221924}, {223.20508075688775, 186.60254037844385}}
view.bounds = {{0, 0}, {200, 100}} / layer.bounds = {{0, 0}, {200, 100}}
view.transform        a= 0.8660 b= 0.5000 c=-0.5000 d= 0.8660
layer.affineTransform a= 0.8660 b= 0.5000 c=-0.5000 d= 0.8660
```

`UIView` 在几何上就是 `CALayer` 的一层转发。`center` 转发到 `position`，`transform` 转发到 `layer.transform` 的仿射部分，`bounds` 和 `frame` 直接就是 layer 的。这也解释了为什么 `frame` 是算出来的：真正的存储在 layer 那边，而 layer 存的也是 `bounds` + `position` + `transform` + `anchorPoint`。

### transform3D 和 transform 会互相清空

`transform3D` 是 iOS 13 加的属性。头文件说：

```objc
@property(nonatomic) CATransform3D     transform3D API_AVAILABLE(ios(13.0), tvos(13.0)) API_UNAVAILABLE(watchos); // default is CATransform3DIdentity. animatable. Please use this property instead of the transform property on the layer
```

2D 变换会如实反映到 3D 上：

```text
设 2D scale2 后 transform3D = [2.00 0.00 0.00 0.00 / 0.00 2.00 0.00 0.00 / 0.00 0.00 1.00 0.00 / 0.00 0.00 0.00 1.00]
```

反过来就不行了。设一个绕 Y 轴 45° 的 3D 旋转，然后读 `view.transform`：

```text
view.transform        a= 1.0000 b= 0.0000 c= 0.0000 d= 1.0000 tx=0 ty=0
CGAffineTransformIsIdentity = 1
layer.affineTransform a= 1.0000 b= 0.0000 c= 0.0000 d= 1.0000
CATransform3DIsAffine(layer.transform) = 0
view.transform3D.m13  = -0.7071
frame = {{29.289321881345245, 0}, {141.42135623730951, 100}}
```

`view.transform` 读出来是 identity，而这个 view 明明被转了 45°（`frame` 宽度 141.42 = 200×cos45° 就是证据）。绕 Y 轴的旋转在 2D 仿射矩阵里根本表达不出来，所以降维时被整个丢掉了。

**如果你的代码里有 `if (CGAffineTransformIsIdentity(view.transform))` 这样的判断，它会在有 3D 变换的 view 上给出错误答案。**

更狠的是两个属性互相覆盖，没有任何合成：

```text
先设 2D 平移(50,0)，再设 3D 绕 Y 45°  -> frame={{29.289322, 0}, {141.421356, 100}}   平移没了
先设 3D 绕 Y 45°，再设 2D 平移(50,0)  -> frame={{50, 0}, {200, 100}}  m13=0.0000     3D 没了
```

它们是同一块状态的两个投影，后写的那次赋值把整块状态换掉。想在 3D 基础上叠加，只能自己构造完整的 `CATransform3D`。

## 九、非整数 frame 与 contentsScale

UIKit 对小数点不做任何处理：

```text
frame  = {{10.299999999999997, 20.699999999999999}, {100.40000000000001, 50.600000000000001}}
bounds = {{0, 0}, {100.40000000000001, 50.600000000000001}}  center = {60.5, 46}
```

加进一个真实的 window 之后依然原样保留，`layer.frame` 也一样。取整这件事发生在光栅化那一层，几何属性上看不到。

后果是像素采样。屏幕 scale 为 2 时，一个逻辑点对应 2 个物理像素，只有 0.5 的整数倍能落在像素边界上：

```text
pt=10.00 -> px=20.00  是否整像素=1
pt=10.25 -> px=20.50  是否整像素=0
pt=10.30 -> px=20.60  是否整像素=0
pt=10.50 -> px=21.00  是否整像素=1
pt=10.75 -> px=21.50  是否整像素=0
```

落不到边界上，渲染时就得做抗锯齿插值，视觉上表现为发虚的边和文字。这是「1 像素分割线」和「label 文字模糊」这两类问题的共同来源。`CGRectIntegral` 只能做到点级别的取整，scale=3 的设备上真正该对齐的是 1/3 点。我自己的做法是需要对齐时按 `round(value * scale) / scale` 算，不用 `CGRectIntegral`。

> **这一节的 `contentsScale` 数据在 Mac Catalyst 上不可信。** 我实测到的是：window 已经 `makeKeyAndVisible` 并显示出来，`layer.contentsScale` 依然是 1.00，而同一时刻 `window.screen.scale` 和 `traitCollection.displayScale` 都是 2.00。
>
> ```text
> makeKeyAndVisible 前: win.layer.contentsScale=1.00  v=1.00
> 显示后:               win.layer.contentsScale=1.00  v=1.00
> win.screen.scale=2.00  traitCollection.displayScale=2.00
> ```
>
> 在 iOS 上，view 加入 window 层级后 `contentsScale` 应当被设为屏幕 scale。Catalyst 这里多了一层 AppKit 桥接，实际的高分屏放大不由这个 layer 承担。
>
> 待真机补测：在 iPhone 上打印一个已上屏 view 的 `layer.contentsScale`，确认它等于 `UIScreen.main.scale`；以及 `contentScaleFactor` 手动设为 3 之后 `drawRect:` 拿到的 context 分辨率。复现方法就是本文的实验代码，改成 iOS target 跑。

同理，`UIScreen.mainScreen.bounds` 在 Catalyst 上返回的是 Mac 显示器的尺寸（我两次运行分别拿到 960×600 和 1920×1080，取决于当时接的哪块屏），`nativeScale` 是 2.00。凡是和屏幕尺寸、安全区有关的数字，本文一个都没用。

---

## 总结

`frame` 每次读都是现算：`bounds` 的四角相对 `anchorPoint` 过一遍 `transform`，取包围盒，加上 `position`。真正的存储是 `bounds`、`position`、`transform`、`anchorPoint` 这四样，全在 `CALayer` 上。

官方说 transform 非 identity 时 `frame` 是 undefined。undefined 的具体行为是：setter 把 `frame.size` 当向量套一次逆变换取绝对值，写进 `bounds.size`。旋转 45° 时能把高度算成 8.2e-15，而且不报错。

`bounds.origin` 是滚动的全部实现。`UIScrollView.contentOffset` 和它是同一块存储，双向同步。

`CGAffineTransformRotate` 这一族函数把新操作前置，也就是新操作先执行、然后被已有变换作用一次。要在屏幕坐标系里追加操作，用 `CGAffineTransformConcat(旧, 新)`。

`hitTest:` 落在旋转后的位置，因为坐标换算走了逆变换、而 `pointInside:` 判的是不受影响的 `bounds`。`scale(0,0)` 是这条链上的一个洞：矩阵不可逆时 `CGAffineTransformInvert` 原样返回，于是看不见的 view 照样能被点中。

下一篇 [[iOS 事件传递与响应者链：hitTest、手势与那些点不动的按钮]]，把这条链从 `UIApplication` 一路走到 `touchesBegan:`。几何属性背后的 `CALayer` 树和绘制流水线在 [[iOS UIView 与 CALayer：三棵树、绘制流水线与离屏渲染]]。

## 参考资料

### 官方

- [View Programming Guide — View Geometry and Coordinate Systems](https://developer.apple.com/library/archive/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/WindowsandViews/WindowsandViews.html)：`frame` undefined 那段原文的出处，也是 frame/bounds/center 三者互相牵动关系的权威描述
- 当前 SDK 的 `UIKit.framework/Headers/UIView.h`：`frame` / `bounds` / `center` / `anchorPoint` / `pointInside:` 的注释，本文的引用全部抄自这里
- 当前 SDK 的 `CoreGraphics.framework/Headers/CGAffineTransform.h`：矩阵布局、`p' = p * t` 的行向量约定、`Concat` 与 `Translate` 的乘法顺序定义、零行列式求逆的行为
- [objc.io — Animations Explained](https://www.objc.io/issues/12-animations/animations-explained/)：model layer tree 与 presentation layer tree 的分工

### 拓展

- [理解 anchorPoint，position，frame 的关系](https://joeshang.github.io/2014-12-19-understand-anchorpoint-position-frame/)：第四节那条 origin 公式的出处，从「为什么 CALayer 要叫 position 而 UIView 要叫 center」这个角度切入，思路很顺
- [iOS 中的图形变换](http://www.samirchen.com/graphic-transform-in-ios/)：Quartz 2D 的 CTM 与 UIKit transform 的坐标系差异讲得清楚，也引了 `frame` undefined 的官方原文
- [iOS 图层几何学](https://zhangbuhuai.com/post/layer-geometry-in-ios.html)：写作时该站返回 403，未能核对内容，列在这里仅作线索

### 本地

- [[iOS 事件传递与响应者链：hitTest、手势与那些点不动的按钮]]
- [[iOS UIView 与 CALayer：三棵树、绘制流水线与离屏渲染]]

---

实验环境：Xcode 26.6，macOS arm64（Apple Silicon）。全部实验用 Mac Catalyst target 编译成原生 macOS 二进制直接运行，不是模拟器，也不是真机：

```shell
SDK=$(xcrun --sdk macosx --show-sdk-path)
clang -fobjc-arc -target arm64-apple-ios17.0-macabi -isysroot "$SDK" \
      -iframework "$SDK/System/iOSSupport/System/Library/Frameworks" \
      -framework Foundation -framework UIKit -framework CoreGraphics \
      -framework QuartzCore -o out prog.m && ./out
```

这样编出来的二进制链接和加载的是真正的 `UIKitCore`，几何计算走的是同一份代码。本文一到八节全是纯几何运算，不需要屏幕、不需要触摸、不需要 run loop，适用边界和 iOS 一致。

需要 `UIWindow` 的那两个实验（`toView:nil` 的语义、`contentsScale`）走不了这条路：直接 `[[UIWindow alloc] initWithFrame:]` 会抛 `NSApplication has not been created yet`。我另外做了一个带 `Info.plist` 的最小 `.app` bundle，用 `UIApplicationMain` 起进程，才拿到真实的 window。这也是本文唯一需要签名和 bundle identifier 的部分。

第九节的 `contentsScale` 和所有 `UIScreen` 数值受 Catalyst 桥接影响，已在正文里单独标注，不要当成 iOS 上的结论。
