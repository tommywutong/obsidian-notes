---
title: 【iOS】架构模式：MVC 到 MVVM，以及它们各自解决不了的问题
published: 2026-07-27
description: 同一个搜索列表写四遍，逐帧对齐行为再数行数。MVC 140 行、MVP 192 行、MVVM 230 行；Apple 文档里那个负责解耦的 mediating controller，iOS SDK 从来没有过。
tags:
  - iOS
  - 架构
  - MVC
  - MVP
  - MVVM
  - 数据绑定
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 36
draft: true
---
# 架构模式：MVC 到 MVVM，以及它们各自解决不了的问题

Apple 那份归档文档里有一句话，中文转述基本都不引：

> Although it is possible to have views directly observe models to detect changes in state, it is best not to do so. A view object should always go through a mediating controller object to learn about changes in an model object.

mediating controller 在同一页有定义，说的是 `NSController` 的子类，配合 Cocoa bindings 用。

然后我去 SDK 里找它。`NSController.h`、`NSObjectController.h`、`NSArrayController.h`、`NSTreeController.h`，四个头文件全部只在 `AppKit.framework` 下。iOS 的 `System/iOSSupport` 里一个都没有。`bind:toObject:withKeyPath:options:` 声明在 `AppKit/NSKeyValueBinding.h`，文件第 15 行是 `APPKIT_API_UNAVAILABLE_BEGIN_MACCATALYST`，连 Catalyst 都不给。

所以那套被当成"苹果推荐的 MVC"讲的东西，用来保证 View 和 Model 解耦的那个零件，iOS 上从来没有过。UIKit 给的只有 target-action、delegate 和 KVO。剩下的自己写。

这一点比"Massive View Controller"更能解释 iOS 的 MVC 为什么变成今天这样。

三个角色分别是什么、胖 Model 瘦 Model、Controller 为什么复用不了，我 2025 年在 [[csdn-151052260-ios-mvc|【iOS】MVC架构]] 和 [[csdn-155750158-ios-mvvm|【iOS】MVVM]] 里写过。这里不重写。

这一篇只做一件事：同一个需求实现三遍，把行为逐帧对齐，然后数行数。

---

## 一、先看数字

需求是一个仓库搜索列表。输入防抖、加载中、空结果、网络失败带重试、磁盘缓存（JSON + 5 分钟 TTL）、单行收藏并持久化、丢弃过期响应。四份实现共用同一套 Model / 网络 / 缓存 / 收藏存储，156 行，只有职责划分不同。

先跑一遍验证它们真的等价。同一段交互脚本，每一步抓一次界面上的可见状态（转菊花没有、占位文案是什么、重试按钮显不显示、几行、第一行长什么样）：

```text
初始       spin=0 ph=输入关键词开始搜索 retry=0 rows=-1 first=-
输入后0.15s spin=0 ph=输入关键词开始搜索 retry=0 rows=-1 first=-
加载完成   spin=0 ph=(hidden) retry=0 rows=4 first=AFNetworking/AFNetworking|Objective-C · ★33.8k · 3 天前|-
收藏第一行 spin=0 ph=(hidden) retry=0 rows=4 first=★ AFNetworking/AFNetworking|Objective-C · ★33.8k · 3 天前|fav
失败       spin=0 ph=加载失败：网络连接已断开 retry=1 rows=-1 first=-
重试成功   spin=0 ph=(hidden) retry=0 rows=2 first=Alamofire/Alamofire|Swift · ★41.2k · 2 天前|-
缓存命中   spin=0 ph=(hidden) retry=0 rows=2 first=Alamofire/Alamofire|Swift · ★41.2k · 2 天前|-
清空       spin=0 ph=输入关键词开始搜索 retry=0 rows=-1 first=-

四份实现的可见状态序列是否逐帧相同：是
```

四份逐帧相同，接下来的数字才有意义。

| | 总代码行 | ViewController | 逻辑层 | 能脱离 UIKit 单测的行 | 视图侧被叫醒次数 |
|---|---|---|---|---|---|
| MVC | 140 | 140 | — | 0 | 9 |
| MVP | 192 | 89 | 103 | 103（54%） | 10 |
| MVVM（4 个输出） | 230 | 87 | 143 | 143（62%） | 35 |
| MVVM（1 个 state） | 215 | 78 | 137 | 137（64%） | 9 |

"代码行"是去掉空行、纯注释行、`#pragma` 之后的数。"视图侧被叫醒次数"数的是同一段脚本跑完，视图那一侧被通知了几次：MVC 数 `render`，MVP 数 `RepoListView` 协议方法，MVVM 数绑定 block。

**MVC 版 140 行，MVVM 版 230 行；变瘦的只有 ViewController 这一个文件，总量涨了 64%。**

这个数字和"MVVM 减少代码量"的流行说法不一致。它减少的是单个文件的行数，不是工程的行数。我早年那篇引过一句"虽然会轻微增加代码量，但是总体上减少了代码的复杂度"，前半句实测是 64%，谈不上轻微。

先说明一句。三份实现是我同一天写的，`viewDidLoad` 里搭界面那 30 行，MVP 和 MVVM 只差最后两行，MVC 多三行。格式化函数一字不差。绝对行数会随人的写法浮动，可比的是三份之间的差。

---

## 二、MVC 那 140 行都在干什么

按 `#pragma mark` 分段数：

```text
（类声明与 ivar）    18 行
视图搭建             31 行
搜索与请求           33 行
状态渲染             22 行
展示格式化           16 行
UITableView         22 行
```

没有一段是废的。这就是 Massive View Controller 的真实成因：每一件事单看都属于"这个页面的事"，加起来就是一个类干了六件事。

请求那段是这样的：

```objc
- (void)fire:(NSString *)query {
    if (query.length == 0) { _repos = nil; _error = nil; _loading = NO; [self render]; return; }

    NSArray<Repo *> *cached = [[RepoCache shared] reposForQuery:query];
    if (cached) { _repos = cached; _error = nil; _loading = NO; [self render]; return; }

    _loading = YES; _error = nil;
    [self render];

    NSInteger gen = ++_generation;
    [[RepoAPI shared] searchRepos:query completion:^(NSArray<Repo *> *repos, NSError *error) {
        if (gen != self->_generation) return;          // 丢弃过期响应
        self->_loading = NO;
        if (error) { self->_error = error; self->_repos = nil; }
        else {
            self->_error = nil; self->_repos = repos;
            [[RepoCache shared] store:repos forQuery:query];
        }
        [self render];
    }];
}
```

渲染那段是一个五分支的 if：

```objc
- (void)render {
    if (_loading) { ... }
    else if (_error) { ... }
    else if (_currentQuery.length == 0) { ... }
    else if (_repos.count == 0) { ... }
    else { ... }
}
```

这个写法有一个常被低估的好处。所有可见状态只在这一个方法里被写入，五个分支互斥，每个分支把四个控件的显隐一次性设完。界面不可能处在"转着菊花同时显示上一次的错误文案"这种状态里，因为根本没有代码路径能造出它。第六节会看到 MVVM 是怎么把这个性质弄丢的。

MVC 真正付不起的代价是另一件事。把这个文件拿去用纯 Foundation 编：

```shell
$ clang -fobjc-arc -framework Foundation -I Shared -c MVC/MVCRepoListViewController.m
MVC/MVCRepoListViewController.m:2:9: fatal error: 'UIKit/UIKit.h' file not found
```

星标数怎么变成 `33.8k`、时间戳怎么变成"3 天前"、失败要不要清空列表、过期响应怎么丢。这些判断全住在一个 `UIViewController` 子类里。想断言它们，先得有一个能跑 UIKit 的环境。140 行里可单测的是 0 行。

---

## 三、MVP：Presenter 和 View 之间要签一份合同

MVP 的做法是把上面那些判断整个搬出去，搬进一个不认识 UIKit 的 `RepoListPresenter`。它对界面的全部认知就是一个协议：

```objc
@interface RepoRow : NSObject          // 一行数据，全是可直接显示的字符串
@property (nonatomic, copy) NSString *title;
@property (nonatomic, copy) NSString *subtitle;
@property (nonatomic, copy) NSString *languageTag;
@property (nonatomic, assign) BOOL favorite;
@end

@protocol RepoListView <NSObject>
- (void)showLoading;
- (void)showRows:(NSArray<RepoRow *> *)rows;
- (void)showPlaceholder:(NSString *)text retryEnabled:(BOOL)retry;
- (void)reloadRow:(NSInteger)index with:(RepoRow *)row;
@end
```

VC 那边实现这四个方法，加上三行事件转发：

```objc
- (void)searchBar:(UISearchBar *)sb textDidChange:(NSString *)t { [_presenter searchTextChanged:t]; }
- (void)onRetry { [_presenter retryTapped]; }
- (void)tableView:(UITableView *)tv didSelectRowAtIndexPath:(NSIndexPath *)ip { [_presenter rowTapped:ip.row]; }
```

代价可以直接数出来。协议加 DTO 的头文件 22 行，VC 里实现协议 22 行，事件转发 5 行。合计 49 行，纯粹用来让两个对象互相说话，占 MVP 总量 192 行的 26%。MVC 在这一项上是 0 行。

换来的是 103 行搬进了可单测区。这不是推论，是编译出来的：

```shell
$ clang -fobjc-arc -framework Foundation -I Shared -I MVP -I MVVM \
        -o logic_test Tests/logic_test.m Shared/Shared.m \
        MVP/RepoListPresenter.m MVVM/Observable.m MVVM/RepoListViewModel.m
$ otool -L logic_test
	/System/Library/Frameworks/Foundation.framework/.../Foundation
	/usr/lib/libobjc.A.dylib
	/usr/lib/libSystem.B.dylib
	/System/Library/Frameworks/CoreFoundation.framework/.../CoreFoundation
```

链接的四个库里没有 UIKit。测试文件里的假 View 只是一个记流水账的 `NSObject`：

```objc
@interface StubView : NSObject <RepoListView>
@property (nonatomic, strong) NSMutableArray<NSString *> *calls;
@property (nonatomic, strong) NSArray<RepoRow *> *rows;
@end
```

14 条断言全过。里面有"失败后要求 View 显示重试"、"star 数被格式化成 33.8k"、"空结果走空态文案"，都是原本要开界面才能看的东西。

### Presenter 造不出 UIColor

MVP 的一个具体麻烦藏在 `languageTag` 那个字段里。

Presenter 要格式化，格式化的终点之一是"Swift 显橙色、Objective-C 显蓝色"。`UIColor` 在 UIKit 里，Presenter 不能碰。于是颜色映射只能留在 VC：

```objc
- (UIColor *)colorForLanguage:(NSString *)lang {
    if ([lang isEqualToString:@"Swift"])       return [UIColor colorWithRed:1 green:.35 blue:.2 alpha:1];
    if ([lang isEqualToString:@"Objective-C"]) return [UIColor colorWithRed:.26 green:.5 blue:.9 alpha:1];
    return UIColor.grayColor;
}
```

同一件事（把一条仓库数据变成一行可见内容）被劈成两半，22 行在 Presenter，5 行在 VC，而且这 5 行不在测试覆盖内。这不是我写岔了，是"逻辑层不许 import UIKit"这条规则的直接后果。凡是有值需要落在 UIKit 类型上（颜色、字体、`NSAttributedString`、图片），都会撞上同一堵墙。

绕法有两个，都要付钱。一是让 Presenter 输出语义 token，我选的这个，代价是多一次映射和一份未覆盖代码。二是在逻辑层引一个自己的 `Theme` 抽象，再在 View 侧翻译，代价是多一层类型。

---

## 四、MVVM：ViewModel 不知道 View 存在

MVP 里 Presenter 拿着 `id<RepoListView>` 主动喊人。MVVM 把这根线也剪了：ViewModel 只暴露状态，谁想看谁自己订。

ObjC 没有现成的绑定原语，先写一个。整个 31 行：

```objc
@interface Observable<__covariant T> : NSObject
@property (nonatomic, strong, nullable) T value;
- (void)observe:(id)owner block:(void (^)(T _Nullable v))block;  // 立即回调一次当前值
@end

@implementation Observable {
    NSMutableArray *_blocks;
    NSHashTable *_owners;
}
- (void)setValue:(id)value {
    _value = value;
    for (void (^b)(id) in [_blocks copy]) b(value);
}
- (void)observe:(id)owner block:(void (^)(id))block {
    [_owners addObject:owner];
    [_blocks addObject:[block copy]];
    block(_value);
}
@end
```

ViewModel 的输出面是四个 Observable：

```objc
@property (nonatomic, readonly) Observable<NSNumber *> *loading;
@property (nonatomic, readonly) Observable<NSString *> *placeholderText;   // nil 表示隐藏
@property (nonatomic, readonly) Observable<NSNumber *> *retryVisible;
@property (nonatomic, readonly) Observable<NSArray<RepoItemVM *> *> *items;
```

VC 那边四段订阅。写第一版的时候我是这么写的：

```objc
__weak typeof(self) ws = self;
[_vm.loading observe:self block:^(NSNumber *on) {
    if (on.boolValue) [ws->_spinner startAnimating]; else [ws->_spinner stopAnimating];
}];
```

四段写完，编译不过。八个错误，每个都是同一句：

```text
error: dereferencing a __weak pointer is not allowed due to possible null value
       caused by race condition, assign it to strong variable first
```

`ws->_ivar` 这种直接访问 ivar 的写法，编译器强制要求先转成 strong。所以每个绑定块的第一行都必须是同一句样板：

```objc
[_vm.placeholderText observe:self block:^(NSString *text) {
    __strong typeof(ws) s = ws; if (!s) return;
    s->_placeholder.text = text;
    s->_placeholder.hidden = (text == nil);
}];
```

四个输出，四遍 weak-strong dance。这个组合为什么必要，[[iOS Block 循环引用与 weak-strong dance]] 里讲过。这里只补一条：在 MVVM 里它每一条绑定都要写一遍，属于固定税。四段绑定一共 22 行，其中 5 行是这个税。

---

## 五、四种绑定机制，胶水各要多少行

绑定是 MVVM 的承重部分，而 iOS 上没有官方绑定。实践里能用的就那么几种。它们各自的运行时基础，[[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]] 和 [[iOS 对象通信：delegate、通知、target-action 与 block 回调]] 已经拆到底了。这里只回答那两篇没回答的一个问题：当成绑定用，每种要写多少胶水。

任务统一：把一个状态对象的三个属性（`NSString`、`NSInteger`、`BOOL`）同步到三个界面更新上。四份都写完，输出序列逐项相同：

```text
KVO          title=hello badge=3 spin=1 spin=0
BLOCK        title=hello badge=3 spin=1 spin=0
DELEGATE     title=hello badge=3 spin=1 spin=0
NOTIFICATION title=hello badge=3 spin=1 spin=0
四份输出是否一致：是

  KVO           胶水 30 行
  BLOCK         胶水 16 行
  DELEGATE      胶水 26 行
  NOTIFICATION  胶水 33 行
```

Combine 没进这张表，因为它进不来。iOS SDK 里 `Combine.framework` 只有 `Combine.tbd` 和 `Modules/`，没有 `Headers/` 目录。它是纯 Swift 框架，Objective-C 侧调不到。ObjC 项目想要响应式，选项只有 ReactiveObjC 这类第三方，或者上面这四种手写。

四个数字要这么读：

- KVO 是唯一发送端零代码的方案。被观察对象什么都不用改，30 行全在接收端：三个 context 常量、三次 `addObserver`、一个五分支的回调分发、三次 `removeObserver`。
- BLOCK 的 16 行是最省的，代价是发送端每个属性要覆写 setter。
- DELEGATE 的 26 行里，协议声明和实现是一对一的死板对应，加一个绑定属性要改三处。
- NOTIFICATION 最贵，33 行里有三个字符串常量和三次 `userInfo` 装箱拆箱，全程没有类型。

行数之外，出错方式差别更大。四个失败当场跑出来：

```text
KVO 键路径拼错：编译通过，运行时不报错，就是收不到回调
KVO 观察者没实现回调方法：抛 NSInternalInconsistencyException
KVO 重复 removeObserver：抛 NSRangeException
通知不带 object 过滤：两个无关实例各触发一次，共 2 次
```

第一条最要命。我把 `@"title"` 写成 `@"titel"`。`addObserver:` 照常返回，设值照常执行，没有异常，没有日志。界面就是不更新。查这个 bug 只能靠盯着字符串看。

**键路径拼错这件事，编译器和运行时都不会告诉你。**

所以在 MVVM 的绑定这个位置上，我不用 KVO。它多出的 14 行胶水买到的是"发送端不用改"，而 ViewModel 本来就是我自己写的，改 setter 没有任何障碍。我早年那篇写"KVO 比较难用容易出问题，最好还是用 RAC 封装"，方向是对的，结论应该收在 block 上，不是引一个框架。

---

## 六、MVVM 解决不了的两件事

### 中间态

MVC 的 `render` 是一次性把四个控件设完。MVVM 的四个 Observable 是分四次通知的，中间那几刻界面处在什么状态，没人管。

在 VC 的绑定之后再挂一组观察者，每次有输出变化就抓一次界面。从"错误态"点重试：

```text
错误态：            spin=0 ph=加载失败：网络连接已断开 retry=1 rows=-1
#1 loading         变化后： spin=1 ph=加载失败：网络连接已断开 retry=1 rows=-1
#2 retryVisible    变化后： spin=1 ph=加载失败：网络连接已断开 retry=0 rows=-1
#3 placeholderText 变化后： spin=1 ph=(hidden)               retry=0 rows=-1
最终：              spin=1 ph=(hidden)               retry=0 rows=-1
```

第 1 帧和第 2 帧，菊花已经转起来了，上一次的错误文案还在。这两个状态在 MVC 和 MVP 里造不出来。

要先说清楚它有多严重。UIKit 一轮 RunLoop 只提交一次，这三次写入落在同一帧里，用户在屏幕上看不到闪烁。所以它不是显示 bug，是不变量被破坏了。一旦绑定块里的事带有立即副作用，它就会读到那个不该存在的组合状态。起动画、`layoutIfNeeded`、`becomeFirstResponder`、埋一个曝光点，都算。曝光埋点最容易中招：错误提示的曝光会在菊花已经转起来的那一刻再上报一次。

修法是把四个输出合成一个不可变快照：

```objc
@interface ListState : NSObject
@property (nonatomic, readonly) BOOL loading;
@property (nonatomic, readonly, nullable) NSString *placeholderText;
@property (nonatomic, readonly) BOOL retryVisible;
@property (nonatomic, readonly) NSArray<RepoItemVM *> *items;
+ (instancetype)loading;
+ (instancetype)placeholder:(NSString *)t retry:(BOOL)r;
+ (instancetype)list:(NSArray<RepoItemVM *> *)items;
@end
```

VC 那边从四段绑定收成一段，`bind` 从 22 行降到 13 行，总代码从 230 降到 215。

**四个 Observable 输出让视图侧被叫醒 35 次，合并成一个 state 之后是 9 次，和 MVC 的 `render` 次数完全一致。**

35 比 9，差 3.9 倍。每一次多余的唤醒都是一次 `reloadData` 或者一次控件属性写入。这个页面只有四个输出，输出多的页面比例只会更差。

我的结论是：ViewModel 只应该有一个输出。多个独立 Observable 看起来更"细粒度"，实际是把状态机的互斥关系从代码里删掉了，然后指望订阅方自己拼回来。

### 导航

ViewModel 不能 import UIKit，那"点一行进详情页"这条指令怎么发出去。

流行做法是把它也当成一个输出：

```objc
@property (nonatomic, readonly) Observable<NSString *> *route;
- (void)rowTapped:(NSString *)name { self.route.value = name; }
```

VC 订阅，收到非 nil 就 push。这个写法有一个当场能触发的 bug。`Observable` 在 `observe:` 时会立即回调一次当前值（这是它能正确初始化界面的原因），而 ViewModel 通常活得比 VC 长。VC 被重建（旋转、pop 后再 push、状态恢复）重新订阅时：

```text
方案 A（route 不清空）：用户点了 1 次，实际 push 2 次  apple/swift,apple/swift
方案 B（发完置 nil）： 用户点了 1 次，实际 push 1 次  apple/swift
```

修法是发完立刻置 nil，一行：

```objc
- (void)rowTapped:(NSString *)name { self.route.value = name; self.route.value = nil; }
```

这一行是补丁不是设计。它说明状态和事件是两种东西，而 `Observable` 只表达前者。真正干净的做法是给事件另开一套只推不存的通道，或者把导航从 ViewModel 里整个拿走交给 Coordinator。前者是自己再写一个原语，后者是再加一层。

对照一下 MVC 的成本。`didSelectRow` 里一行 `pushViewController:animated:`，没有任何可以出错的地方。

---

## 七、把 Fowler 那句话摆在这

MVVM 的祖宗是 Fowler 的 Presentation Model。他在原文里对这套东西的最大抱怨写得很直接：

> Probably the most annoying part of Presentation Model is the synchronization between Presentation Model and view. It's simple code to write, but I always like to minimize this kind of boring repetitive code.

同一页还有一句，正好命中我上面量到的东西：

> If you put the synchronization in the view, it won't get picked up by tests on the Presentation Model. If you put it in the Presentation Model you add a dependency to the view in the Presentation Model which means more coupling and stubbing.

我的 MVVM 版本选了前者，同步代码在 VC 里。后果是准确的：VC 那 87 行一行都没被那 14 条断言覆盖，其中就包括第六节那个中间态问题所在的绑定段。**测试覆盖率的数字会因为架构调整而变好看，但没被覆盖的那部分恰好是最容易出状态 bug 的那部分。**

Apple 当年给 Cocoa 的答案就是 mediating controller。同步代码既不在 view 也不在 model，在一族现成的 `NSController` 里，由框架维护。这个方案在 macOS 上确实成立。iOS 没有拿到它。于是每个项目都要自己写一遍那 22 行同步代码，写完还测不到。

回头看 Apple 那份 MVC 文档，落差就清楚了。文档里写：

> View objects learn about changes in model data through the application's controller objects and communicate user-initiated changes—for example, text entered in a text field—through controller objects to an application's model objects.

而 iOS 实践中，`UITableViewCell` 拿到一个 model 对象自己配置自己是绝对主流。我早年那篇也写到了这一点，说典型的 MVC 已经被违背，而人们一直不觉得这有哪些不对。补一句当时没写的：这不是开发者集体偷懒。严格按文档做，cell 的每个字段都要 Controller 逐个赋值。`cellForRowAtIndexPath:` 会膨胀。而 Apple 给 macOS 的那个消肿工具没给 iOS。

文档里还有一处更细的区分被普遍忽略了。Apple 把 controller 分成两类：coordinating controller 负责 "oversee the functioning of the entire application"，view controller 则 "owns the interface (the views)"。iOS 的 `UIViewController` 顶着后一个名字，两份活全干了。

---

## 八、我的阈值

写到这里该给一个能执行的判断，不是"看情况"。

**我的线是：一个页面的 ViewController 超过 250 行，或者出现第 4 个互斥状态，就拆。** 两个条件满足其一就动手。

250 这个数是从上面量出来的往上推的。我的 MVC 版 140 行覆盖了搜索、防抖、缓存、网络、五态渲染、格式化、收藏、竞态处理。在真实项目里这算中等偏上。留到 250 大概能再吃下一次分页和一个筛选面板。再往上，`render` 的分支数会超过我一眼能读完的量。

"第 4 个互斥状态"是另一个更早的信号。空态、加载、错误、有数据是四个，我这个页面正好卡在线上（我把"未搜索"和"空结果"算成同一个占位分支，所以是四个分支五个 case）。再加一个（比如"无网络离线可读缓存"），我就不再相信自己能在 `render` 里把互斥关系写全。

拆的时候我选 ViewModel + 单一 state 输出，不选 MVP。理由是上面第三节量的 49 行协议样板。MVP 的协议在页面稳定之后是资产，它是一份显式的界面契约。在页面还在改的时候是负债：加一个显示项要改三处，协议、Presenter、VC。ViewModel 那边加一个字段只改一处。我自己的页面多数属于后一种。

几条更细的，都是我实际会照着做的：

- 格式化逻辑一律进 ViewModel，哪怕只有一个 `stringWithFormat:`。它是纯函数，最好测，搬出去的边际成本最低。
- ViewModel 一个页面一个，不给每个 cell 配 ViewModel。cell 的数据用一个无行为的 DTO（我这里的 `RepoItemVM`，四个属性，零方法）。
- 导航不进 ViewModel。我宁可让 VC 在事件转发里直接 push，也不为它引一个 Coordinator。第六节那个 push 两次的 bug，我不想在每个页面上防一遍。
- 绑定用 block，不用 KVO。理由在第五节。
- 样板代码占比超过 30% 就退回上一档。MVP 在我这个页面上是 26%，已经接近。VIPER 把 Interactor 和 Router 再切出去，这个比例只会更高，而它买到的东西（Router 可替换、Interactor 可复用）在一个只会被这一个页面用到的模块上等于没买。

### 两个我不同意的说法

一是"MVVM 的双向绑定"。我早年那篇引过 model-view-binder 那套说法：Model 变 ViewModel 自动变，ViewModel 变 View 自动变。那是 WPF 的语境，那边有框架级的 binding。iOS 上没有。硬做双向要么 KVO 对绑，第五节量过，30 行胶水还要自己防循环通知。要么上第三方。我做出来的四份实现全是单向：ViewModel 出状态，View 调方法。这个方向是清楚的、可断点的、能逐帧对齐的。"双向绑定"在 iOS 的 MVVM 讨论里更多是个转述过来的词，不是一个能落地的机制。

二是"引入架构模式提升可测试性"这句话本身。它是对的，但省略了一半。MVP 把 103 行搬进可测区，同时新造了 49 行永远测不到的协议实现。MVVM 把 143 行搬进可测区，同时新造了 22 行永远测不到的绑定代码，而这 22 行正是中间态 bug 的产地。净收益是正的。但"可测试性"这个词遮住了新增的不可测部分。看架构收益要看两个数：搬进去多少，新造了多少。

---

## 总结

Apple 文档里的 MVC 靠一个叫 mediating controller 的角色保证 View 不认识 Model。那个角色是 `NSController` 一族加 bindings，只在 AppKit 里，iOS SDK 从来没有过。iOS 的 MVC 缺了零件。Massive View Controller 是这个缺口的下游结果。

同一个搜索列表写四遍、逐帧对齐之后。MVC 140 行全在 VC 里，可单测 0 行。MVP 192 行，协议样板 49 行，可单测 103 行。MVVM 230 行，可单测 143 行。把四个输出合并成一个 state 之后是 215 行，视图侧唤醒次数从 35 次降回 9 次，和 MVC 持平。

绑定这一步，block 16 行、delegate 26 行、KVO 30 行、通知 33 行。KVO 是唯一发送端零成本的，但键路径拼错没有任何编译期或运行期信号。Combine 在 ObjC 里根本调不到，SDK 里那个 framework 没有头文件。

MVVM 的两个坑都能当场触发：多输出造出中间态，实测两个不该存在的中间帧，把导航当输出会在 VC 重建时 push 两次。前者的修法是单一 state，后者的修法是发完置 nil，两个修法都是补丁。

我的阈值：VC 超过 250 行或出现第 4 个互斥状态就拆 ViewModel，样板占比超过 30% 就退回上一档，导航不进 ViewModel。

下一篇 [[iOS 网络分层：URLSession 之上该有几层]]。

## 参考资料

### 官方（均已归档）

- [Cocoa Encyclopedia — Model-View-Controller](https://developer.apple.com/library/archive/documentation/General/Conceptual/CocoaEncyclopedia/Model-View-Controller/Model-View-Controller.html)：mediating controller / coordinating controller / view controller 的三分，以及"A view object should always go through a mediating controller object"那句原话
- [Cocoa Core Competencies — Model-View-Controller](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/MVC.html)：三个角色的 Communication 段落
- Xcode 26.6 附带的 MacOSX26.5 SDK 头文件：`AppKit/NSController.h`、`AppKit/NSKeyValueBinding.h`（第 15 行 `APPKIT_API_UNAVAILABLE_BEGIN_MACCATALYST`）、`System/iOSSupport/.../Combine.framework`（无 `Headers/`）

### 经典

- [Martin Fowler — Presentation Model](https://martinfowler.com/eaaDev/PresentationModel.html)：MVVM 的直接来源。同步代码放哪、放哪都要付什么代价，原文说得比后来的转述清楚
- [objc.io — Introduction to MVVM](https://www.objc.io/issues/13-architecture/mvvm/)：ObjC 语境下 MVVM 的标准介绍。"MVVM works best with a binding mechanism" 这句在 iOS 上是整篇文章的前提，也是它最大的空缺

### 本地

- [[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]]
- [[iOS 对象通信：delegate、通知、target-action 与 block 回调]]
- [[iOS Block 循环引用与 weak-strong dance]]
- [[iOS UIViewController：生命周期的真实顺序与容器控制器]]
- [[iOS 网络分层：URLSession 之上该有几层]]
- 我自己 2025 年那两篇：[[csdn-151052260-ios-mvc|【iOS】MVC架构]]（三个角色、胖瘦 Model、Controller 臃肿的定性描述，本文不重复）· [[csdn-155750158-ios-mvvm|【iOS】MVVM]]（Apple MVC 与传统 MVC 的差别、Controller 消肿的四种手法、绑定方式的罗列。本文第五节把那份罗列换成了行数，第八节对其中"双向绑定"和"KVO 做绑定"两条给了不同结论）

---

实验环境：macOS 26.5.2，arm64 Apple Silicon，Xcode 26.6，Apple clang。全程没有 boot 任何模拟器。

四份 UI 实现编成 Mac Catalyst target 跑，用的是真正的 `UIKitCore`：

```shell
SDK=$(xcrun --sdk macosx --show-sdk-path)
clang -fobjc-arc -target arm64-apple-ios17.0-macabi -isysroot "$SDK" \
      -iframework "$SDK/System/iOSSupport/System/Library/Frameworks" \
      -framework Foundation -framework UIKit -o out main.m ...
```

没有 `UIWindow`，用 `CFRunLoopRunInMode` 推进异步，界面状态用 KVC 读 VC 的私有 ivar 抓。`UITableView` 的行数和 cell 内容是直接问 `dataSource` 拿的。有一处必须说明。`UIApplication` 不存在时 `sendActionsForControlEvents:` 不派发事件。重试按钮是照 `actionsForTarget:forControlEvent:` 拿到 selector 直接调的。

网络层是打桩的：固定 50 ms 延迟，失败次数可注入。这是为了让四份实现跑出逐帧可比的结果，不是真实网络。缓存写在 `NSTemporaryDirectory()` 下，每轮清空。

行数统计口径：去掉空行、纯注释行、`#pragma` 之后的行数，脚本对四份用同一套规则。绝对值和写法强相关，四份之间的差才是结论。

> 待真机补测：第六节那个中间态在真机上是否真的不产生可见闪烁。Catalyst 的合成时机与 iOS 不完全相同。复现方法：在 `placeholderText` 的绑定块里加一个 `[UIView animateWithDuration:0.25 ...]`，从错误态点重试，用 Instruments 的 Core Animation 抓一下有没有两段重叠的动画。
