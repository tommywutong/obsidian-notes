---
title: 【iOS】UITableView：复用池的真实结构与代理调用顺序
published: 2026-07-27
description: 复用池不是一个池，是一个字典套有序集合，按标识符分开、从末尾取。另外 estimatedRowHeight 的默认值是 UITableViewAutomaticDimension 而不是 0，所以估算默认就开着。
tags:
  - iOS
  - UIKit
  - UITableView
  - 性能优化
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 23
draft: true
---
# UITableView：复用池的真实结构与代理调用顺序

先说一个大部分文章都讲错的地方。

`UITableView` 的复用池，中文资料里画出来几乎都是一个方框，标着"复用池"，cell 滑出屏幕就丢进去，要用就拿出来。这个图不算错，但它把最关键的三件事全省了：池按什么分组、池里是什么容器、取的时候取哪一个。

这三件事都能直接问运行时。`class_copyIvarList` 把 `UITableView` 的 ivar 全打出来，能找到 `_reusableTableCells`，类型是 `NSMutableDictionary`。把它读出来看看装了什么，答案是：**复用池是一个字典，key 是复用标识符，value 是 `NSMutableOrderedSet`，取的时候从末尾拿。**

本文所有数字都是这么来的。第一节是最硬的一节，放在最前面。

---

## 一、复用池的真实结构

### 先把 ivar 列表打出来

```objc
unsigned n = 0;
Ivar *ivars = class_copyIvarList(UITableView.class, &n);
for (unsigned i = 0; i < n; i++) {
    printf("%-46s type=%-20s offset=%ld\n",
           ivar_getName(ivars[i]), ivar_getTypeEncoding(ivars[i]),
           (long)ivar_getOffset(ivars[i]));
}
```

`UITableView` 一共 134 个 ivar。跟复用直接相关的是这几个：

```text
_visibleRows           type={_NSRange="location"Q"length"Q}   offset=2184
_visibleCells          type=@"NSMutableArray"                 offset=2200
_reusableTableCells    type=@"NSMutableDictionary"            offset=2264
_nibMap                type=@"NSMutableDictionary"            offset=2280
_cellClassDict         type=@"NSMutableDictionary"            offset=2800
_tentativeCells        type=@"NSMutableDictionary"            offset=2912
```

`_visibleCells` 是数组，`_reusableTableCells` 是字典。光看类型就已经能推出一件事：池是按复用标识符分开的，不同标识符之间互不相通。注册两个标识符就有两个独立的池，一个池空了不会去另一个池里借。

### 池里装的是有序集合

用 `object_getIvar` 把 `_reusableTableCells` 读出来，在滚动过程中打印：

```objc
Ivar iv = class_getInstanceVariable(UITableView.class, "_reusableTableCells");
NSDictionary *pool = object_getIvar(tableView, iv);
```

300 行、每行高度在 44/66/88 之间循环，表格高 600pt：

```text
== 首屏             offset.y=    0.0  visibleCells=14
     _reusableTableCells      __NSDictionaryM        count=0
     _visibleCells            __NSArrayM             count=14

== 滚 300pt         offset.y=  300.0  visibleCells=14
     _reusableTableCells      __NSDictionaryM        count=1  key=c -> __NSOrderedSetM(5)
     _visibleCells            __NSArrayM             count=14

== 滚 900pt         offset.y=  900.0  visibleCells=14
     _reusableTableCells      __NSDictionaryM        count=1  key=c -> __NSOrderedSetM(14)
     _visibleCells            __NSArrayM             count=14

== 跳到 3000pt      offset.y= 3000.0  visibleCells=14
     _reusableTableCells      __NSDictionaryM        count=1  key=c -> __NSOrderedSetM(15)
     _visibleCells            __NSArrayM             count=15 之后不再涨
```

`__NSOrderedSetM` 就是 `NSMutableOrderedSet`。

选它是有道理的。集合语义保证同一个 cell 不会被放进去两次（放两次就会被 dequeue 出去两次，直接崩），有序则让"取哪一个"有确定答案。用 `NSMutableArray` 得自己查重，用 `NSMutableSet` 又没法控制取出顺序。

还有一点：首屏那一行的 `_reusableTableCells` 是 `count=0`，而在一个从没滚动过、也没回收过任何 cell 的表格上读它，拿到的是 `nil`。这个字典是懒创建的，第一次有 cell 被回收才分配。

### 取的时候取哪一个

有序集合有两头，取头还是取尾，直接决定了复用行为。测一下。

```text
滚到 400 后，池里依次是 cell#1, #2, #3, #4, #5
再往下滚一点，新出场的可见 cell 依次是 #8, #9, #10, #11, #12, #13, #14, #15, #16, #5
此刻池里剩 cell#1, #2, #3, #4, #6
```

出去的是 `#5`（末尾那个），回收进来的 `#6` 也追加在末尾。取尾放尾，就是一个栈。

在完整的调用日志里看得更清楚。六行的表格，可见两行，滚出去四个之后调一次 `reloadData`：

```text
    didEndDisplayingCell[3] #4
    didEndDisplayingCell[2] #3
    didEndDisplayingCell[1] #2
    didEndDisplayingCell[0] #1     ← 逆序入池，栈顶是 #4
  cellForRow[0]  → 返回 #4
  cellForRow[1]  → 返回 #3
  cellForRow[2]  → 返回 #2
  cellForRow[3]  → 返回 #1         ← 逆序出池
```

`didEndDisplaying` 是从下往上逆序发的，所以 `#4` 最后进池、也最先出池。这就是为什么 `reloadData` 之后 cell 和 row 的对应关系会整个翻过来。如果你在 cell 上挂了什么按 index 缓存的东西，这里就是它错位的地方。

### 池会不会自己变小

不会。滚 40 步之后池稳定在 5 个，再 `reloadData` 一次还是 5 个。

但有一件事会清空它：

```text
滚 40 步后：池 5 个 / 可见 10 个 / 累计 new 16 个
再 reloadData 一次：池 5 个 / 可见 10 个 / 累计 new 16 个
发一次内存警告通知后：池 0 个
```

`UIApplicationDidReceiveMemoryWarningNotification` 一到，池直接清空。这个行为在文档里没写，但很好理解：池里的 cell 是纯粹的缓存，丢掉不影响正确性，只是下次要重新 alloc。

### 到底会有几个 cell 活着

这里要纠正一个流传很广的说法。我自己早年写 TableView 优化那篇的时候，也照抄了这句：

> `UITableView` 只会创建比一个屏幕所有显示的 cell + 1 个单元格。

它有条件。500 行、每行 60pt、表格高 600pt，可见 10 行，滚 3000pt，用三种方式：

| 滚动方式 | 可见 | 池里 | 活着的 cell | 累计创建 |
|---|---|---|---|---|
| 每步 30pt（半个 cell），滚 100 步 | 10 | 1 | 11 | 12 |
| 每步 600pt（整屏），滚 5 步 | 10 | 10 | 20 | 21 |
| 一步跳到 3000pt | 10 | 10 | 20 | 20 |

慢滚的时候"一屏 + 1"完全成立。整屏跳一次就翻倍，而且翻上去之后不会再降回来。

原因在调用顺序里。跨越整屏时，新可见的行和旧可见的行没有交集，UIKit 先向 `cellForRowAtIndexPath:` 要新 cell、要完了才把旧 cell 放进池。要的那一刻池是空的，只能新建一屏。这一屏新建完，旧的一屏就永久留在池里了。

**慢滚的稳态是一屏 + 1，整屏跳一次就变成两屏，并且不再回落。**对于会用 `scrollToRow` 长距离跳转、或者带 section index 快速定位的列表，内存里的 cell 数量要按两屏算。

---

## 二、代理方法真正被调了几次

六行的表格，行高固定 100，表格高 250，`estimatedRowHeight = 0`。下面是没有裁剪的完整序列。

### reloadData 自己就跑了一遍

```text
===== ① reloadData 本身（还没 layout） =====
  1  numberOfSectionsInTableView
  2  numberOfRowsInSection -> 6
  3    heightForRow[0]
  4    heightForRow[1]
  5    heightForRow[2]
  6    heightForRow[3]
  7    heightForRow[4]
  8    heightForRow[5]
```

`reloadData` 返回的时候，六行的高度已经全问过一遍了，但一个 `cellForRow` 都还没调。

### layout 时又跑了一遍

```text
===== ② 第一次 layout =====
  1  numberOfSectionsInTableView
  2  numberOfRowsInSection -> 6
  3    heightForRow[0]  ... heightForRow[5]      ← 又来一遍
  9    cellForRow[0]  ← 进入
 10        « PCell 新建 #1 »
 11    heightForRow[0]
 12    cellForRow[0]  → 返回 #1
 13    heightForRow[0]
 14      willDisplayCell[0] #1
 15    cellForRow[1]  ← 进入
 ...
 21  heightForHeaderInSection
 22  viewForHeaderInSection
 23  titleForHeaderInSection
 24      willDisplayHeaderView
 25        « layoutSubviews #1 h=100 »
 26        « layoutSubviews #2 h=100 »
```

三件事值得记住：

第一，`heightForRow` 对可见行是问三次。一次在批量高度那一轮，一次在 `dequeue` 内部（新 API 要按高度给 cell 定尺寸），一次在 `cellForRow` 返回之后。加上 `reloadData` 自己那一轮，一个可见行在首屏总共被问四次高度。

第二，header 相关的回调排在所有 cell 后面。`heightForHeaderInSection` → `viewForHeaderInSection` → `titleForHeaderInSection` → `willDisplayHeaderView`，这个顺序稳定。

第三，cell 的 `layoutSubviews` 排在最后，所有代理回调都跑完了才轮到它。所以在 `layoutSubviews` 里读数据、在 `cellForRow` 里赋值，时序上是安全的。

`heightForRow` 的总次数可以算出来。100 行、10 行可见、估算关闭时实测 220 次：

```text
100（reloadData 那轮）+ 100（layout 那轮）+ 10 × 2（可见行的额外两次）= 220
```

10000 行时实测 20020 次，同一个公式对得上。

### 滚动时不会重问行数

```text
===== ③ 滚 120pt（露出下一行） =====
  1      scrollViewDidScroll y=120
  2    cellForRow[2]  ← 进入
  4    heightForRow[2]
  5    cellForRow[2]  → 返回 #3
  6    heightForRow[2]
  7      willDisplayCell[2] #3
```

`numberOfRowsInSection:` 一次都没有。行数和每行的位置在 `reloadData` 时算好，存进 `_rowData`，滚动只是按 `contentOffset` 查这张表。"`numberOfRowsInSection:` 会被调很多次"这个印象，来源是它在每次 `reloadData` / `beginUpdates` 里都要被问，而不是滚动。

还有一个本系列前面提过的问题，一并回答。把 `delegate` 完全不设，表格照样显示。

```text
OnlyDS 遵守 UITableViewDelegate 吗？ 否
tableView.delegate     = nil
visibleCells.count     = 10
contentSize            = {390, 8238.8142013549805}
第一个可见 cell frame  = {{0, 0}, {390, 63}}
第一个可见 cell 文本   = 第 0 行 / delegate 是 nil
滚到 3000 后仍可见 10 行，首行是「第 69 行」
```

`UITableViewDelegate` 里没有一个 `@required` 方法。真正必需的是 `UITableViewDataSource` 的 `numberOfRowsInSection:` 和 `cellForRowAtIndexPath:` 两个。

---

## 三、estimatedRowHeight：先看默认值

我一开始设计这个实验的时候，对照组写的是"`estimatedRowHeight = 0`（关）"和"`estimatedRowHeight = 66`（开）"，跑完发现两份输出一个字节都不差。

原因有两层，第一层在 SDK 头文件里：

```objc
@property (nonatomic) CGFloat rowHeight;             // default is UITableViewAutomaticDimension
@property (nonatomic) CGFloat estimatedRowHeight API_AVAILABLE(ios(7.0)); // default is UITableViewAutomaticDimension, set to 0 to disable
@property (nonatomic) CGFloat estimatedSectionHeaderHeight API_AVAILABLE(ios(7.0)); // default is UITableViewAutomaticDimension, set to 0 to disable
```

`UITableViewAutomaticDimension` 的值是 `-1`。刚 `new` 出来的表格打印一下：

```text
UITableViewAutomaticDimension = -1.0
刚 new 出来： rowHeight=-1.0  estimatedRowHeight=-1.0  estSectionHeader=-1.0
```

**`estimatedRowHeight` 的默认值是 `UITableViewAutomaticDimension`，估算默认就是开着的；把它设成 0 才是关。**说"默认是 0，你要设一个值才能开启估算"的文章，说反了。注释里 `set to 0 to disable` 六个词说得很清楚。

第二层原因是：那两份输出一样，是因为我在两组里都实现了 `estimatedHeightForRowAtIndexPath:`。

### 五组对照

100 行，高度按 44/66/88 循环，真实总高 6578.00：

| | 实现 estimatedHeightForRow: | estimatedRowHeight | heightForRow 次数 | 涉及行数 | contentSize.height |
|---|---|---|---|---|---|
| A | 否 | 默认 (-1) | 30 | 10 | 4242.18 |
| B | 否 | 0 | 220 | 100 | 6578.00 |
| C | 否 | 66 | 33 | 10 | 6578.00 |
| D | 是 | 0 | 33 | 10 | 6578.00 |
| E | 是 | 66 | 33 | 10 | 6578.00 |

D 和 E 完全一样。**只要实现了 `estimatedHeightForRowAtIndexPath:`，`estimatedRowHeight` 属性写多少都不再影响估算的开关。**想真正关掉估算，两件事都要做：属性设 0，并且不实现那个代理方法。

B 是唯一一组把 100 行全问了一遍的。这一列就是所谓"自适应高度卡顿"的根。

### 我差点写下一条错误结论

看 C 那行：只问了 10 行的真实高度，`contentSize` 却精确等于 6578.00。我当时的第一反应是"估算开着也能算准总高，UIKit 有什么补偿机制"，差点就这么写了。

幸好换了一组参数重跑。改成 100 行、高度 `37 + (row % 7) * 13`、真实总高 7535.00、估算值给 500：

```text
真实总高 = 7535.00    全按 500 估 = 50000.00

默认(-1) 无 est 回调                  heightForRow= 27  contentSize.h=  4263.22
estimatedRowHeight=0 无 est 回调      heightForRow=218  contentSize.h=  7535.00
estimatedRowHeight=500 无 est 回调    heightForRow= 27  contentSize.h= 46119.00
estimatedRowHeight=500 + est 返回 500 heightForRow= 27  contentSize.h= 46119.00
estimatedRowHeight=500 + est 返回 9999 heightForRow=27  contentSize.h=910528.00
```

46119 拆开：已经量过真实高度的 9 行加起来是 619，剩下 91 行按 500 估是 45500，加起来正好 46119。规则很简单：

```text
contentSize.height = Σ(已量过的行的真实高度) + 估算值 × 剩下的行数
```

回头看 C 那组：可见 10 行真实高度合计 638，剩下 90 行按 66 估是 5940，加起来 6578。而 44/66/88 的平均值恰好就是 66，所以真实总高也是 6578。那个"精确"是巧合。

这也解释了自适应高度列表里滚动条乱跳的现象：每量到一批真实高度，`contentSize` 就被修一次，滚动条的比例跟着变。A 那组能直接看到，首屏 4242.18，滚一屏之后变成 4475.76。

A 组里 UIKit 自己用的默认估算值可以反推。`(4242.18 - 638) / 90 = 40.0464`，在 3000pt 那组里反推也是 40.0464。这个数是 Mac Catalyst 上的，iOS 上多半不同。

### 用数字讲代价

10000 行，`heightForRow` 里做一次 `boundingRectWithSize:` 文本排版（这是动态高度最常见的写法）：

```text
-- heightForRow 只做算术 --
estimatedRowHeight = 0（关）    heightForRow 20020 次   首屏  22.94 ms
estimatedRowHeight = 66（开）   heightForRow    33 次   首屏   4.56 ms
-- heightForRow 里做一次 boundingRect 文本排版 --
estimatedRowHeight = 0（关）    heightForRow 20020 次   首屏 3805.78 ms
estimatedRowHeight = 66（开）   heightForRow    30 次   首屏   9.40 ms
```

**10000 行、`heightForRow` 里排一次版：估算关是 3.8 秒，估算开是 9 毫秒。**

跑了三遍，排版那两行分别是 3902 / 5323 / 3806 毫秒和 124 / 13 / 9 毫秒。124 那次是第一遍的冷启动，字体和排版引擎都还没热。差三个数量级这个结论是稳的，具体倍数别当精确值用。

结论不是"永远开估算"。估算开着，`contentSize` 就是错的，滚动条会跳，`scrollToRow` 的落点会偏。我自己的取舍是：定高列表直接设 `rowHeight`、不实现 `heightForRowAtIndexPath:`；动态高度的列表开估算，同时自己缓存算过的高度，`heightForRow` 里只做一次字典查找。行数上千还坚持关估算，首屏就是秒级的。

---

## 四、注册与 dequeue：两个 API 的差别

### 什么都不注册

```text
A. 老 API dequeueReusableCellWithIdentifier:        返回 nil
B. 新 API forIndexPath:  抛 NSInternalInconsistencyException
   reason: unable to dequeue a cell with identifier none - must register a nib
           or a class for the identifier or connect a prototype cell in a storyboard
```

头文件里两个方法的注释就摆着这个区别：

```objc
- (nullable __kindof UITableViewCell *)dequeueReusableCellWithIdentifier:(NSString *)identifier;  // Used by the delegate to acquire an already allocated cell, in lieu of allocating a new one.
- (__kindof UITableViewCell *)dequeueReusableCellWithIdentifier:(NSString *)identifier forIndexPath:(NSIndexPath *)indexPath API_AVAILABLE(ios(6.0)); // newer dequeue method guarantees a cell is returned and resized properly, assuming identifier is registered
```

返回类型上老 API 是 `nullable`，新 API 不是。所以老 API 必须配 `if (!cell) cell = [[XXX alloc] init...]`，新 API 一定不为 nil，那句判空是死代码。

注释里还有一句常被忽略：`guarantees a cell is returned and resized properly`。"resized properly"有前提。在 `cellForRow` 里打印新 API 返回的 cell：

```text
[cellForRow 内部 dequeue 到的 frame = {{0, 0}, {390, 40.046391601562497}}]
```

宽度已经是表格宽度 390，高度是当前的估算值。但在 `cellForRow` 之外、脱离布局流程直接调用时，两个 API 拿到的都是 `{{0, 0}, {320, 44}}`，宽度还是 iPhone 时代那个 320。

```text
C. 老 API  返回 非 nil  cell#1  frame={{0, 0}, {320, 44}}
D. 新 API  返回 cell#2  frame={{0, 0}, {320, 44}}
```

所以"新 API 会帮你把 cell 调好尺寸"这句话，成立的条件是它被 UIKit 的布局流程调用。自己在别处 dequeue 一个来量尺寸，量到的是没有意义的数。

### registerClass: 存在哪

```text
_cellClassDict         = { byclass = PCell; }
_nibMap                = { }
_reusableTableCells    = nil
```

一个标识符到类的字典，加一个标识符到 nib 的字典。两条注册路径互相独立，注册的时候后来的覆盖先来的。

`registerClass:nil` 可以撤销注册：

```text
F. 撤销后新 API 抛 NSInternalInconsistencyException:
   unable to dequeue a cell with identifier reg - must register a nib or a class ...
```

注意撤销的那一刻池子里其实还躺着之前建的 cell，但新 API 照样抛异常。它检查的是 `_cellClassDict` / `_nibMap`，不是池。

> 待真机补测：`registerNib:` 这条路本文没能实测。`xcrun ibtool --compile` 要 GUI 服务，无头环境下一直挂住，编不出 nib。补的方法是在 Xcode 里建一个 `UITableViewCell` 的 xib，用 `registerNib:` 注册，然后看三件事：`_nibMap` 里存的是什么类型的对象；先 `registerClass:` 再 `registerNib:` 同一个标识符时哪个生效；nib 里设的 `rowHeight` 会不会影响 dequeue 出来的 frame。

---

## 五、造一个复用错乱的 bug，再修好

经典场景：cell 上有张异步加载的图，用户快速滑动。

有 bug 的写法，回调里直接写 `self`：

```objc
@implementation BadCell
- (void)loadRow:(NSInteger)row {
    self.textLabel.text = @"";                       // 只清了文字
    FakeDownload(key(row), ^(NSString *img) {
        self.textLabel.text = img;                   // 回来就写，不问自己现在是谁
    });
}
@end
```

`FakeDownload` 不是真的网络请求，是一个把回调攒起来、由我手动触发的桩。这样"什么时候回来、按什么顺序回来"完全可控，结果是确定的。

关键在回调顺序。真实世界里请求不是先发先回：首屏那几张图大、慢，用户滑过去之后新请求命中了缓存、秒回。所以桩按"后发的先回"来触发。

200 行，可见 10 行，分 6 步滑到 offset=1800：

```text
######## BadCell：回调里直接写 self.textLabel ########
首屏 10 行，在途请求 10 个（图都还没下完）
分 6 步滑到 offset=1800，累计在途请求 40 个，期间只 new 了 16 个 cell 实例
      row 30 （cell#1）该显示 IMG(30)   实际显示 IMG(0)
      row 31 （cell#2）该显示 IMG(31)   实际显示 IMG(1)
      row 32 （cell#3）该显示 IMG(32)   实际显示 IMG(2)
      row 33 （cell#4）该显示 IMG(33)   实际显示 IMG(3)
      row 34 （cell#15）该显示 IMG(34)  实际显示 IMG(14)
      row 35 （cell#16）该显示 IMG(35)  实际显示 IMG(19)
      row 36 （cell#8）该显示 IMG(36)   实际显示 IMG(7)
      row 37 （cell#9）该显示 IMG(37)   实际显示 IMG(8)
      row 38 （cell#5）该显示 IMG(38)   实际显示 IMG(4)
      row 39 （cell#6）该显示 IMG(39)   实际显示 IMG(5)
   可见 10 行 → 正确 0，串图 10，空白 0
```

10 行全错。`cell#1` 现在服务 row 30，但 row 0 那个回调还攥着对它的强引用，回来就把 `IMG(0)` 写了上去。

我第一次跑这个实验的时候没复现出来。当时是一步跳到 offset=1800，结果 10 行全对。原因在第一节：跨越三屏的跳转会新建一整屏 cell，服务 row 30~39 的根本不是原来那批实例，自然不串。要复现复用 bug，得让复用真的发生，也就是小步滚动。这个坑值得记一下，很多人写 demo 复现不出来就是这个原因。

修的办法是给 cell 加一个令牌，复用时翻一次：

```objc
@implementation GoodCell
- (void)prepareForReuse {
    [super prepareForReuse];
    self.textLabel.text = @"";
    self.token++;                 // 令牌一变，在途回调全部作废
}
- (void)loadRow:(NSInteger)row {
    self.textLabel.text = @"";
    NSInteger my = self.token;
    __weak GoodCell *weakSelf = self;
    FakeDownload(key(row), ^(NSString *img) {
        GoodCell *s = weakSelf;
        if (!s) return;
        if (s.token != my) return;   // 已经被复用给别人了
        s.textLabel.text = img;
    });
}
@end
```

```text
######## GoodCell：prepareForReuse 翻令牌 + 回调查令牌 ########
分 6 步滑到 offset=1800，累计在途请求 40 个，期间只 new 了 16 个 cell 实例
   可见 10 行 → 正确 10，串图 0，空白 0；被令牌挡掉的过期回调 24 个
```

40 个回调里有 24 个是过期的，全被挡下。

### prepareForReuse 里该写什么

从上面的调用日志能看到，`prepareForReuse` 是在 `dequeue` 内部触发的，早于 `heightForRow`，也早于你在 `cellForRow` 里赋值。它的定位是"把 cell 恢复到刚出厂的状态"。

该做：清空文字、图片、`accessoryView`；把 `selected` / `highlighted` 复位；作废在途的异步回调（令牌、`SDWebImage` 的 `sd_cancelCurrentImageLoad`、`NSURLSessionTask` 的 `cancel`）；停掉动画。

不该做：读数据源。这里没有 `indexPath`，也没有任何上下文，`cell.superview` 可能已经是 nil。见过有人在 `prepareForReuse` 里去查 model，那是在猜自己接下来会被派给谁。

不该做：重建子视图。`prepareForReuse` 的存在意义就是省下重建成本，在里面 `removeFromSuperview` 再 `addSubview` 等于把复用的收益还回去。

也不该做：清那些每次 `cellForRow` 都会重新赋值的字段。这类字段清不清没有区别，只是白跑一遍。真正需要清的，是那些"某些行有、某些行没有"的字段。判断标准就一条：如果某个属性只在部分行被赋值，它必须在 `prepareForReuse` 里清掉。

异步图片这条线的完整实现在 [[iOS SDWebImage：下载、解码与两级缓存的完整链路]]，那边讲了它是怎么用 `UIView` 的关联对象记 operation key 来做取消的。

---

## 六、reloadData、reloadRows 与 batch updates

### reloadData 不会重建 cell

20 行的表格，可见 15 行：

```text
首屏可见 15 个，累计 new 18 个
首屏 cell 编号 4,5,6,7,8,9,10,11,12,13,14,15,16,17,18
reloadData 之后 cell 编号 4,17,15,13,11,9,7,5,18,16,14,12,10,8,6   新建 0 个
reloadRows([0]) 之后 cell 编号 19,17,15,13,11,9,7,5,18,16,14,12,10,8,6   新建 1 个
```

`reloadData` 一个新 cell 都没建，15 个全部走了"入池再出池"。编号顺序被打乱，正是第一节那个栈的效果。

`reloadRows` 反而建了一个新的。为什么？

### reloadRows 就是 delete + insert

证据来自异常信息。故意让数据源在 `reloadRowsAtIndexPaths:` 期间从 20 行变成 3 行：

```text
Invalid batch updates detected: the number of sections and/or rows returned by the
data source before and after performing the batch updates are inconsistent with the updates.
Data source before updates = { 1 section with row counts: [20] }
Data source after updates = { 1 section with row counts: [3] }
Updates = [
	Delete row (0 - 0),
	Insert row (0 - 0)
]
```

我只调了 `reloadRowsAtIndexPaths:`，UIKit 打印出来的 `Updates` 却是一条删除加一条插入。所以那个新 cell 是这么来的：insert 先向 `cellForRow` 要新 cell，此时 delete 还没把旧 cell 放回池，池里没货，只能新建。

这条也解释了 `reloadRows` 为什么会打断 cell 上正在跑的动画、为什么会让 `UITextField` 丢焦点。它把那一行整个换掉了。只想改内容的话，直接拿 `cellForRowAtIndexPath:` 取到 cell 改属性更省，代价是要自己保证数据源一致。

### 数据源不一致会抛什么

20 行的表格，每种情况用一个全新的进程跑。

为什么要开新进程，是被逼出来的。我第一次把七种情况写在一个 `main` 里，第一个异常抛完，第二个 case 直接段错误。异常是从 `endUpdates` 内部抛出来的，UIKit 的 update 状态没机会回滚，后面的结果全不可信。

| 操作 | 数据源改成 | 结果 |
|---|---|---|
| 删 1 行 | 还是 20 | `Invalid update: invalid number of rows in section 0.` |
| 删 1 行 | 15 | `Invalid batch updates detected: ...` 并打印前后行数与 Updates 列表 |
| 删下标 99 | 19 | `attempt to delete row 99 from section 0 which only contains 20 rows before the update` |
| 删 1 行 | 19 | 通过，可见 15 行，新建 0 个 cell |
| 同批删 0 又插 0 | 还是 20 | 通过 |
| 插到下标 100 | 21 | `attempt to insert row 100 into section 0, but there are only 21 rows in section 0 after the update` |
| 删整个 section 0 | 还是 20 | `Invalid update: invalid number of sections.` |

第一条的完整原文，就是线上崩溃日志里最常见的那句：

```text
Invalid update: invalid number of rows in section 0. The number of rows contained in
an existing section after the update (20) must be equal to the number of rows contained
in that section before the update (20), plus or minus the number of rows inserted or
deleted from that section (0 inserted, 1 deleted) and plus or minus the number of rows
moved into or out of that section (0 moved in, 0 moved out).
```

注意有两套不同的报错：越界类的（`attempt to delete row 99 ...`）在 `endUpdates` 早期就拦住，报的是下标；数量对不上的报的是等式。看到"attempt to"开头，找的是 indexPath 越界；看到"must be equal to"，找的是数据源改没改、改了几个。

还有一条实践上的：`beginUpdates` / `endUpdates` 之间不要读数据源。UIKit 会在这一对调用的前后各问一次行数，中间你如果又改了数组，两次的答案就对不上。正确的顺序是先把数据源改完，再进 `beginUpdates`。

---

## 七、卡顿排查：能测的和测不了的

这一节要诚实。本文的实验环境是 Mac Catalyst，跑的是原生 macOS 进程，加载真正的 `UIKitCore`，但没有 iOS 的显示链路。

测不了的：

- 真实滚动帧率、掉帧数、hitch 时长。没有屏幕刷新，没有 CADisplayLink 的真实节拍。
- 离屏渲染。渲染在 macOS 的合成器上，和 iOS 的 GPU 路径不是一回事。
- Instruments 的 Time Profiler / Animation Hitches。这些要 attach 到 iOS 进程。
- 图片解码耗时。解码走的是 macOS 的 ImageIO，和 iOS 的硬解不可比。

能测的，也是这一节全部的数据来源：

- `heightForRow` 的调用次数。这是纯逻辑，UIKitCore 里同一份代码。
- `heightForRow` 的总耗时。绝对值不可比，数量级差异可比（3.8 秒 vs 9 毫秒这种）。
- cell 实例的创建次数。给 `initWithStyle:reuseIdentifier:` 加计数器即可。
- 复用命中率。在 `cellForRow` 里统计 dequeue 次数和"拿到的是旧 cell"的次数。

命中率这个指标很好加，代码就几行：

```objc
static int gDequeued = 0, gReuseHit = 0;
// cell 上加一个 BOOL fresh，init 里置 YES
gDequeued++;
PCell *c = [t dequeueReusableCellWithIdentifier:@"c" forIndexPath:ip];
if (!c.fresh) gReuseHit++;
c.fresh = NO;
```

实测一份 100 行、可见 10 行的列表，滚 20 步 × 300pt：

```text
【dequeue 调用】  100 次  命中 80 次 (80%)
【cell 实例 new】  20 个
```

命中率低于 90% 基本只有三个原因：复用标识符按 index 拼了字符串（每行一个池）、滚动步长太大、或者根本没用 dequeue。这三个用上面这几行计数器都能当场分辨。

> 待真机补测：帧率与离屏渲染部分。复现方法是把本文的 `BadCell` / `GoodCell` 两份代码放进一个 iOS App，测同一段滚动路径的 hitch time ratio。工具用 Instruments 的 Animation Hitches 模板。圆角 cell 的离屏区域用下面那一小节说的着色开关看。数字不在这里编。

### 填一个坑：离屏渲染到底用什么工具看

我在 2025 年那篇 [TableView 的优化](https://blog.csdn.net/2402_86720949/article/details/155425916) 里写过一句：

> 对于离屏渲染的检测，据说苹果为我们提供了一个测试工具，这里先埋一个坑，以后再来填。

现在填上。工具是模拟器的 Debug 菜单 → Color Off-screen Rendered，打开之后被离屏渲染的区域会被整片染色。从 `Simulator.app` 的 `MainMenu.nib` 里 `strings` 一下，这一组菜单项和它们背后的 action 都在：

```text
菜单项                          selector
Color Blended Layers            toggleCADebugColorCopy:
Color Copied Images             toggleCADebugColorOffscreen:
Color Misaligned Images         toggleCADebugColorOpaque:
Color Off-screen Rendered       toggleCADebugColorSubpixel:
```

（左右两列都是从 nib 里按字母序抓出来的，一一对应关系我没有验，只能确定是这四个菜单项对这四个 selector。名字本身已经说明它们各管一件事：混合、拷贝、离屏、亚像素对齐。）

四个开关要配合着看。查 cell 卡顿时我一般先开 Color Blended Layers，因为"忘了给 cell 和它的子视图设不透明背景"是最常见、也最容易改的一条，改完立竿见影。离屏那个开关放第二个开。

真机那条路变了。老文章说的"Instruments 的 Core Animation 模板勾 Color Offscreen-Rendered Yellow"，在 Xcode 26.6 上已经找不到了：

```shell
$ xcrun xctrace list templates
Activity Monitor / Allocations / Animation Hitches / App Launch / ...
Metal System Trace / Network / ... / Time Profiler
```

模板列表里没有 Core Animation。这个能力现在归 Animation Hitches 模板。

至于离屏渲染本身是怎么回事、哪些属性会触发、`shouldRasterize` 到底是省了还是亏了，那是 [[iOS UIView 与 CALayer：三棵树、绘制流水线与离屏渲染]] 的地盘，本文不重复。我只补一句和 TableView 直接相关的：cell 上的圆角要不要担心，取决于圆角的数量和是否跟着滚动，一两个头像的 `cornerRadius` 在今天的设备上通常不是瓶颈，先测再改。

---

## 八、几个需要修正的说法

- "复用池是一个池。" 是一个字典，按复用标识符分组，每组是一个 `NSMutableOrderedSet`，取尾放尾。不同标识符的池完全隔离。
- "UITableView 只会创建一屏 + 1 个 cell。" 慢滚成立。跨整屏跳一次就变成两屏，而且不再回落。
- "estimatedRowHeight 默认是 0，要手动设才开启估算。" 默认是 `UITableViewAutomaticDimension`（-1），估算默认开着，设成 0 才是关。头文件注释原文是 `set to 0 to disable`。
- "设了 estimatedRowHeight 就能减少 heightForRow 的调用。" 前半句对，但只要实现了 `estimatedHeightForRowAtIndexPath:`，这个属性设多少都不影响开关。
- "开了估算 contentSize 就不准。" 更准确的说法是 `contentSize = 已量过的行的真实高度之和 + 估算值 × 剩余行数`，会随着滚动被不断修正。
- "reloadData 会把所有 cell 重建一遍。" 实测新建 0 个，全部走池。真正会新建的是 `reloadRowsAtIndexPaths:`，因为它在内部被展开成 delete + insert。
- "numberOfRowsInSection: 滚动时会被反复调用。" 滚动时一次都不调。行数在 `reloadData` 时算进 `_rowData`。
- "没有 delegate 表格就不显示。" `UITableViewDelegate` 没有 `@required` 方法，`delegate` 为 nil 时表格照常渲染和滚动。

---

## 总结

复用池的真实形状是 `NSMutableDictionary<标识符, NSMutableOrderedSet<cell>>`，按标识符隔离，按栈的方式取放。`didEndDisplaying` 逆序入池，所以 `reloadData` 之后 cell 与 row 的对应关系会整个翻转。内存警告会把池清空。

`heightForRow` 的调用次数有公式：估算关闭时是 `行数 × 2 + 可见行数 × 2`，10000 行实测 20020 次；估算开启时只问可见行，实测 30 次左右。这个差别在 `heightForRow` 里做文本排版时会放大成三个数量级。

`estimatedRowHeight` 默认就是开着的，这一条颠覆了很多"优化建议"的前提。真要关，属性设 0 且不实现 `estimatedHeightForRowAtIndexPath:`，两件事都要做。

复用错乱的复现有个前提：得让复用真的发生。一步跳三屏会新建一整屏 cell，反而复现不出来。

最后是方法论。这一篇里所有和通说不一致的结论，没有一条是靠读更多文章得来的，都是 `class_copyIvarList` 打一遍、`object_getIvar` 读一遍、加个计数器数一遍。UIKit 不开源，但它的 ivar、它抛的异常、它的调用序列，全都是可观测的。

下一篇 [[iOS Foundation 集合：类簇、真实实现与选型]]。

## 参考资料

### 官方

- [Filling a Table with Data](https://developer.apple.com/documentation/uikit/filling-a-table-with-data)：dataSource 契约的官方说法
- [UITableViewDataSource](https://developer.apple.com/documentation/uikit/uitableviewdatasource)：`@required` 只有两个方法
- [UITableViewDelegate](https://developer.apple.com/documentation/uikit/uitableviewdelegate)：全部 optional
- [prepareForReuse](https://developer.apple.com/documentation/uikit/uitableviewcell/prepareforreuse())：官方明确说不要在这里做重建
- SDK 头文件 `UIKit.framework/Headers/UITableView.h`：`estimatedRowHeight` 的默认值、两个 dequeue 方法的注释，本文引用的都是原文

### 经典

- [objc.io — Smooth Scrolling in UITableView and UICollectionView](https://www.objc.io/issues/1-view-controllers/table-views/)：把复用、布局和主线程成本串起来的一篇，概念部分今天仍然成立
- [Facebook — Delivering High Scroll Performance](https://engineering.fb.com/2015/06/25/ios/delivering-high-scroll-performance/)：按需加载与预取的思路来源

### 本地

- [[iOS 事件传递与响应者链：hitTest、手势与那些点不动的按钮]]
- [[iOS 坐标系：frame 是算出来的，bounds 和 transform 才是存的]]
- [[iOS UIView 与 CALayer：三棵树、绘制流水线与离屏渲染]]
- [[iOS UIViewController：生命周期的真实顺序与容器控制器]]
- [[iOS SDWebImage：下载、解码与两级缓存的完整链路]]
- [[iOS RunLoop：mode、source 与那张流程图今天还对不对]]
- [[iOS 对象通信：delegate、通知、target-action 与 block 回调]]

---

实验环境：Xcode 26.6（17F113），macOS arm64，Mac Catalyst。编译方式：

```shell
SDK=$(xcrun --sdk macosx --show-sdk-path)
clang -fobjc-arc -target arm64-apple-ios17.0-macabi -isysroot "$SDK" \
      -iframework "$SDK/System/iOSSupport/System/Library/Frameworks" \
      -framework Foundation -framework UIKit -o out prog.m && ./out
```

编出来是原生 macOS 二进制，加载的是真正的 `UIKitCore`。全程没有 boot 任何模拟器，也没有窗口：表格直接 `alloc/init` 加 `layoutIfNeeded` 驱动，滚动靠直接改 `contentOffset`。

边界要说清楚。复用池结构、代理调用顺序、`estimatedRowHeight` 的开关逻辑、异常文本，这些是 `UIKitCore` 里同一份代码，Catalyst 和 iOS 走的是一条路。但有几个数是 Catalyst 特有的，不要直接搬到 iOS：

- 默认行高。Catalyst 上 `UITableViewCell` 自适应出来是 45.0pt（subtitle 样式 63.0pt），iOS 上是另一套 metrics。
- UIKit 内部的默认估算高度。Catalyst 上反推是 40.0464pt。
- 老 API 在 `cellForRow` 之外返回的 `{320, 44}`。

> 待真机补测：把本文的 e1（ivar dump）和 e13（完整调用序列）两个程序改成 iOS target 跑一遍，确认 `_reusableTableCells` 的类型、有序集合的取放端、以及 `didEndDisplaying` 的逆序在 iOS 上一致。我的判断是这三条都会一致，但没验过就是没验过。
