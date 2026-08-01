---
title: "【iOS】Runtime - Part 4 && Runtime 应用篇：动态能力如何落到业务代码"
published: 2026-06-16
description: "承接对象结构、消息发送与 Category 加载机制，梳理 Runtime 在业务代码中的典型应用：动态获取类信息、动态添加方法、Method Swizzling、KVO、关联对象、消息转发、自动归档与模型转换。"
tags: ["iOS", "Objective-C", "Runtime", "Method Swizzling", "objc4"]
category: "iOS"
series: "iOS Runtime 系列"
seriesSlug: "ios-runtime"
seriesOrder: 4
draft: true
---

# 前言

前三篇已经把 Runtime 的底层链路铺开了：

- Part 1：对象、类、元类、`isa`、`class_ro_t` / `class_rw_t`。
- Part 2：`objc_msgSend`、方法缓存、慢速查找、动态方法解析、消息转发。
- Part 3：Category 的编译产物、运行时加载、方法列表附加、覆盖假象和关联对象。

Part 4 不再继续只追源码细节，而是把这些机制拉回日常开发：**我们平时说的 Runtime 应用，到底是在使用哪几类动态能力？哪些用法是合理的，哪些用法容易把项目带进不可维护的状态？**

常见 Runtime 用法大致可以分成四类：

| 能力 | 典型 API | 背后的机制 |
| --- | --- | --- |
| 查看类结构 | `class_copyIvarList` / `class_copyPropertyList` / `class_copyMethodList` | 读取类元数据 |
| 动态加方法 | `class_addMethod` / `resolveInstanceMethod:` | 修改 `class_rw_t` / 方法解析 |
| 替换方法实现 | `method_exchangeImplementations` / `method_setImplementation` | 改变 `SEL -> IMP` 关系 |
| 外挂数据 | `objc_setAssociatedObject` / `objc_getAssociatedObject` | 对象外部关联表 |

如果把工程里的常见玩法继续拆开，会发现它们反复依赖的也就几点：

| 底层机制 | 典型应用 |
| --- | --- |
| 消息解析 / 消息转发 | 动态加方法、代理转发、组合对象能力透传、AOP |
| `SEL -> IMP` 映射修改 | Method Swizzling、埋点、异常保护 |
| `isa` 指向变化 | KVO / Isa Swizzling |
| 类元数据遍历 + 外部存储 | 关联对象、自动归档、字典模型互转 |

这几类能力都能在前几篇里找到落点：1111

- “查看类结构”对应 Part 1 的类元数据。
- “动态加方法”对应 Part 2 的动态方法解析。
- “替换方法实现”对应 Part 2 的 `SEL -> IMP` 查找结果。
- “外挂数据”对应 Part 3 的关联对象。

# 1. 动态获取类信息

Runtime 最温和的一类应用，是在运行时读取类的 ivar、property、method、protocol 信息。它不改变程序行为，只是把编译进 Mach-O 的 Objective-C 元数据拿出来看。

常用 API：

```objc
unsigned int count = 0;

Ivar *ivars = class_copyIvarList([Person class], &count);
objc_property_t *properties = class_copyPropertyList([Person class], &count);
Method *methods = class_copyMethodList([Person class], &count);
Protocol *__unsafe_unretained *protocols = class_copyProtocolList([Person class], &count);
```

这些 `copy` 系列 API 返回的是 C 数组，用完要 `free`：

```objc
unsigned int count = 0;
objc_property_t *properties = class_copyPropertyList([Person class], &count);

for (unsigned int i = 0; i < count; i++) {
    const char *name = property_getName(properties[i]);
    const char *attrs = property_getAttributes(properties[i]);
    NSLog(@"%s: %s", name, attrs);
}

free(properties);
```

典型应用：

- 打印对象调试信息。
- 做轻量字段映射。
- 自动归档 / 解档。
- 字典转模型。
- 检查某个类是否实现了某些方法。

这类用法风险较低，但也要注意：**property 和 ivar 不是一回事**。`@property` 是属性元数据和访问器约定，ivar 才是真正的实例存储。Category 可以加 property 元数据，但不能加 ivar，这一点已经在 Part 3 讲过。

# 2. 动态添加方法
   1
动态添加方法的核心 API 是：

```objc
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);
```

它做的事情可以粗略理解为：往类的运行时方法列表里追加一条 `SEL -> IMP` 记录。

示例：

```objc
void dynamicRun(id self, SEL _cmd) {
    NSLog(@"%@ %@", self, NSStringFromSelector(_cmd));
}

BOOL added = class_addMethod([Person class],
                             @selector(run),
                             (IMP)dynamicRun,
                             "v@:");
```

类型编码 `"v@:"` 的意思是：

- `v`：返回值是 `void`。
- `@`：第一个隐藏参数 `self`，类型是对象。
- `:`：第二个隐藏参数 `_cmd`，类型是 selector。
1
动态添加方法最经典的落点，是 Part 2 讲过的动态方法解析：

```objc
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(run)) {
        class_addMethod(self, sel, (IMP)dynamicRun, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```

这条链路是：

```text
objc_msgSend
  -> cache miss
  -> 方法列表没找到
  -> resolveInstanceMethod:
  -> class_addMethod
  -> 重新查找
  -> 命中新 IMP
```

也就是说，动态方法解析不是“直接处理消息”，而是**给类补一条真正的方法记录**。补上之后，后续同样的消息仍然回到正常的消息发送路径。

和方法相关的常用 API 还有一组：

```objc
SEL method_getName(Method m);
IMP method_getImplementation(Method m);
const char *method_getTypeEncoding(Method m);
unsigned int method_getNumberOfArguments(Method m);
char *method_copyReturnType(Method m);
char *method_copyArgumentType(Method m, unsigned int index);
void method_exchangeImplementations(Method m1, Method m2);
IMP method_setImplementation(Method m, IMP imp);
```

它们本质上都是在读写 `method_t` 表达的三件事：selector、类型编码、函数实现。动态加方法时最容易写错的是第三个参数和第四个参数：`IMP` 的真实 C 函数签名必须和 `types` 描述一致，否则消息发送时寄存器里的参数会被错误解释。

# 3. Method Swizzling

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260620212834907.png)

它是 Objective-C 里最有名的"黑魔法",本质上利用了 Runtime 的一个事实:**方法名(SEL)和方法实现(IMP)是分离的、可以在运行时重新绑定的**。上面那张图就是整件事的全部精髓——交换前 `viewWillAppear:` 指向系统的原始实现,交换后它被改指向你写的实现;而你那个方法名反过来指向了原始实现。

## 本质

它利用 Objective-C Runtime 修改方法的 IMP（实现指针），常用于 Hook 系统方法、埋点、AOP、异常保护等场景。核心原理可以直接接回 Part 2：`objc_msgSend(receiver, selector)` 最终要在类的方法表 / 缓存里找到 `SEL -> IMP`，Swizzling 做的就是在运行时改掉这张映射表。

```text
Before:
SEL A -> IMP A
SEL B -> IMP B

After:
SEL A -> IMP B
SEL B -> IMP A
```

常见 API：

```objc
void method_exchangeImplementations(Method m1, Method m2);
IMP method_setImplementation(Method m, IMP imp);
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);
IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types);
```

## 最小实现：`method_exchangeImplementations`

最裸的写法就是直接拿两个 `Method` 做 `method_exchangeImplementations`：

```objc
- (void)tw_swizzle {
    Class cls = [self class];
    Method original = class_getInstanceMethod(cls, @selector(originalFunction));
    Method swizzled = class_getInstanceMethod(cls, @selector(swizzledFunction));

    method_exchangeImplementations(original, swizzled);
}

- (void)originalFunction {
    NSLog(@"originalFunction");
}

- (void)swizzledFunction {
    NSLog(@"swizzledFunction");
}
```

交换后：

```text
originalFunction selector -> swizzledFunction 的 IMP
swizzledFunction selector -> originalFunction 的 IMP
```

`method_exchangeImplementations` 概念上可以理解成两次 `method_setImplementation`：

```objc
IMP impA = method_getImplementation(originalMethod);
IMP impB = method_getImplementation(swizzledMethod);

method_setImplementation(originalMethod, impB);
method_setImplementation(swizzledMethod, impA);
```

这种写法适合在自己定义的类里做演示，两个方法都明确存在于当前类，不涉及继承链和加载时机。它的缺陷也很明显：

- 没有 `dispatch_once`，重复执行会把 IMP 换回去。
- 没有处理“方法来自父类”的情况，可能污染父类方法表。
- 执行时机随调用点而定，不适合做全局 hook。

## 工程使用标准模板：`+load + dispatch_once + class_addMethod`

Category 里通常用 `+load + dispatch_once + class_addMethod` 这一套：

```objc
@implementation UIViewController (TWTracking)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class cls = [self class];

        SEL originalSEL = @selector(viewDidAppear:);
        SEL swizzledSEL = @selector(tw_viewDidAppear:);

        Method originalMethod = class_getInstanceMethod(cls, originalSEL);
        Method swizzledMethod = class_getInstanceMethod(cls, swizzledSEL);

        if (!originalMethod || !swizzledMethod) {
            return;
        }

        BOOL didAdd = class_addMethod(cls,
                                      originalSEL,
                                      method_getImplementation(swizzledMethod),
                                      method_getTypeEncoding(swizzledMethod));

        if (didAdd) {
            class_replaceMethod(cls,
                                swizzledSEL,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

- (void)tw_viewDidAppear:(BOOL)animated {
    [self tw_viewDidAppear:animated];
    NSLog(@"track page: %@", NSStringFromClass([self class]));
}

@end
```

这套模板里有三道防护：
- `+load` 保证交换尽早发生
- `dispatch_once` 保证交换只发生一次
- `class_addMethod` 先把原 selector 固化到当前类，避免直接改到父类。

### 为什么经常写在 `+load`

`+load` 在类和 Category 被加载进 Runtime 时直接调用，不依赖消息发送，因此很适合做“尽早生效”的方法替换。 +initialize方法是以懒加载的方式被调用的，如果程序一直没有给某个类或它的子类发送消息，那么这个类的 +initialize方法是永远不会被调用的。所以Swizzling要是写在+initialize方法中，是有可能永远都不被执行。

这也是很多 Swizzling 模板选择 `+load` 而不是 `+initialize` 的原因： Swizzling 改的是全局行为，通常希望类一加载就完成。

配套还有两个规矩：

- 交换动作放进 `dispatch_once`。Swizzling 执行两次就可能把实现换回去，尤其是父类、子类或多个 Category 同时操作同一个 selector 时，很容易出现“看似执行了，实际失效”的结果。
- `+load` 里不要主动调 `[super load]`。Runtime 会按加载流程调用各自的 `+load`，手动调 super 可能让父类交换逻辑重复执行，导致交换两次后等于没交换。

但这也意味着它有明显代价：
- 调用时机很早，很多业务环境还没准备好。
- 所有 Category 的 `+load` 都会执行，启动期成本容易累积。
- 多个库同时 swizzle 同一个方法时，顺序不应该被业务依赖。

### 为什么要 `class_addMethod`

如果当前类没有实现某个方法，而是继承自父类，直接 exchange 可能会影响父类的方法实现关系，所以在实际中通常是：

1. 先尝试 `class_addMethod`，把原 selector 加到当前类。
2. 如果添加成功，再用 `class_replaceMethod` 把 swizzled selector 替换成原实现。
3. 如果添加失败，说明当前类本身已有实现，再直接 exchange。

设想你要 swizzle 的 `originalSelector`,本类自己并没有重写,它的实现是从父类继承来的。如果你直接调 `method_exchangeImplementations`,`class_getInstanceMethod` 会沿继承链找到**父类**那个 Method 并把它交换了——结果就是你污染了父类的方法实现,所有兄弟子类都会受牵连,这是灾难性的。

所以模板先用 `class_addMethod` 试着把"自定义实现"添加到**本类**的 `originalSelector` 上:

- 如果返回 `YES`(添加成功),说明本类原本没有自己的实现(实现在父类)。此时本类已经有了一份指向自定义实现的 `originalSelector`,接着再用 `class_replaceMethod` 把 `swizzledSelector` 指向那份继承来的原始 IMP 即可。整个过程只在本类新增/改写,**不碰父类**。
- 如果返回 `NO`(添加失败),说明本类已经有自己的 `originalSelector` 实现,那就放心直接 `method_exchangeImplementations` 交换两者。

这一步保证了 swizzling 的影响**严格锁定在当前类**,不会顺着继承链向上扩散。


## 调用原实现的两种方式

Swizzling 后最容易让人困惑的是：替换方法内部如何调用原实现？常见有两种写法。

### 方式 A：`[self swizzled_xxx]`

这是 Category 模板里最常见的写法：

```objc
- (void)tw_viewDidAppear:(BOOL)animated {
    [self tw_viewDidAppear:animated];
    NSLog(@"track page: %@", NSStringFromClass([self class]));
}
```

交换后，`tw_viewDidAppear:` 这个 selector 指向了原来的 `viewDidAppear:` IMP，所以在 `tw_viewDidAppear:` 里调用 `[self tw_viewDidAppear:animated]`，并不是递归调用自己，而是在调用原实现。它的优点是模板短；缺点是读起来像递归，并且原实现看到的 `_cmd` 可能变成 swizzled selector。

每个 OC 方法实现都有两个隐式参数：`self` 和 `_cmd`，其中 `_cmd` 是"**这次调用是通过哪个 selector 进来的**"。问题就出在方式 A 是靠"再发一次消息"回到原实现的，而这次消息用的 selector 是 `tw_viewDidAppear:`，不是 `viewDidAppear:`。

你的自定义实现是被 `viewDidAppear:` 这个 SEL 触发的，所以它内部的 `_cmd` 是**正确的** `viewDidAppear:`；但当它转手用 `tw_viewDidAppear:` 把球传给原实现时，原实现内部拿到的 `_cmd` 就变成了 `tw_viewDidAppear:`。也就是说，**被坑的是原实现，不是你的实现**。这是方式 A 无法回避的副作用——只要你是"通过第二个 selector 再派发一次"回到原实现的，原实现看到的 `_cmd` 就一定是那个冒牌名字。

老实说，**绝大多数情况下不会出事**，这也是方式 A 能在业务代码里横行多年的原因。像 `viewDidAppear:` 这种生命周期方法，UIKit 的实现根本不读自己的 `_cmd`，`_cmd` 是张三还是李四它都照跑不误，但下面这几类场景，`_cmd` 被改就会埋下隐蔽的 bug：

- 第一类是**原实现内部依赖 `NSStringFromSelector(_cmd)` 做日志或埋点**的，它会记录成错误的方法名
- 第二类、也是最危险的——**一个 IMP 被多个 selector 共用、并靠 `_cmd` 区分自己该干哪件事**的情况，比如 `@dynamic` 属性的动态访问器、KVO 生成的 setter、或者通过 `resolveInstanceMethod:` 给一批 selector 注册了同一个函数再用 `_cmd` 分流的实现。这类实现一旦拿到错的 `_cmd`，就会去操作错误的属性，行为彻底错乱。第三类是某些 Apple 框架内部机制会反射 `_cmd`。正因为你写"基础库"时**无法假设**目标方法属不属于以上几类，所以通用 swizzling 工具必须保证 `_cmd` 的正确性——这就引出了方式 B。

### 方式 B：保存原 IMP 的函数指针

互相发 swizzled selector 虽然是经典模板，但阅读时容易误以为递归，而且 `_cmd` 会变成 swizzled selector。基础库里可以改用“保存原 IMP + C 函数指针”的方式：

```objc
static void (*original_setFrame)(id self, SEL _cmd, CGRect frame);

static void tw_setFrame(id self, SEL _cmd, CGRect frame) {
    NSLog(@"%@", NSStringFromCGRect(frame));
    original_setFrame(self, @selector(setFrame:), frame);
}

static BOOL tw_replaceMethodAndStore(Class cls,
                                     SEL originalSEL,
                                     IMP replacementIMP,
                                     IMP *store) {
    Method method = class_getInstanceMethod(cls, originalSEL);
    if (!method) return NO;

    const char *types = method_getTypeEncoding(method);
    IMP oldIMP = class_replaceMethod(cls, originalSEL, replacementIMP, types);
    if (!oldIMP) {
        oldIMP = method_getImplementation(method);
    }

    if (oldIMP && store) {
        *store = oldIMP;
    }
    return oldIMP != NULL;
}
```

这种写法的优点是意图更明确：新实现是一个 C 函数，原实现存在 `original_setFrame` 里，调用原逻辑时不再依赖“交换后的 selector 指向原 IMP”这个隐含约定。

代价是每个被替换的方法都需要独立保存原 IMP。如果多个库都替换同一个 selector，后替换者可能拿到的是前一个替换实现，调用链顺序仍然要小心维护。



## 复杂对象：class cluster / 真实类探测

有些系统对象不是你看到的公开类，而是 class cluster 或私有子类。AFNetworking 早期为了 hook `NSURLSessionTask` 的 `resume` / `suspend`，做过一套运行时探测：先创建一个真实 task，拿到它的实际 class，再沿继承链寻找每一层自己实现过 `resume` 的类并分别 swizzle。这个例子适合理解复杂 hook 的思路，但它依赖系统内部类名和继承结构；这些细节会随系统版本变化，不适合直接照搬到现代业务代码里。

核心判断是这两个 IMP 比较：

```objc
IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));
IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));

if (classResumeIMP != superclassResumeIMP &&
    classResumeIMP != originalAFResumeIMP) {
    // 当前类自己实现了 resume，且还没被替换过
    swizzle(currentClass, @selector(resume), @selector(af_resume));
}
```

第一条过滤掉“只是继承父类实现”的中间层；第二条避免重复 swizzle。这个例子说明：当目标类不稳定、真实类来自系统内部时，Swizzling 的复杂度会迅速上升。普通业务不应该轻易照搬这种级别的 hook。

## 应用场景：典型用法

**Method Swizzling 的核心应用场景，都指向同一类需求：想给现有方法（尤其是改不了源码的系统方法或第三方库方法）统一插入行为，又不愿用继承或逐个修改去实现——最典型的是无痕埋点、崩溃防护、AOP 切面、全局 UI/多语言定制和系统 Bug 修复这五大类。**

理解了前面讲的“交换 IMP”机制后，场景这块其实就是同一个问题的不同变体：**在不碰原代码的前提下，往一个方法的执行链路里“插一脚”**。下面这张图先把全貌铺开，再逐类深入。


### 监控统计：无痕埋点

设想产品要统计“每个页面被访问了多少次”。
最笨的办法是在每个 `ViewController` 的 `viewDidAppear:` 里手动加一行统计代码；
稍好一点是写个 `BaseViewController` 让大家继承，但 `UITableViewController`、`UICollectionViewController` 这些不同基类各自又得写一遍。
Swizzling 给出的是**一次性、全覆盖、零侵入**的方案——在 `UIViewController` 的 Category 里 swizzle 一次 `viewDidAppear:`，所有控制器（包括第三方库里的）自动全部生效。 

```objc
@implementation UIViewController (Analytics)
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // ... 标准交换模板 ...
    });
}
- (void)tw_viewDidAppear:(BOOL)animated {
    [self tw_viewDidAppear:animated];                       // 先跑系统原实现
    [Analytics track:NSStringFromClass([self class])];      // 再统一上报
}
@end
```

同一思路还能扩展到**点击事件统计**（swizzle `UIControl` 的 `sendAction:to:forEvent:`，所有按钮点击自动捕获）和**性能监控**（在原实现前后打时间戳，统计每个方法的耗时，定位卡顿）。这类场景的共同点是：埋点逻辑与业务逻辑完全无关，散落在每个角落，却又必须“无处不在”

### 防护容错

OC 的集合类对越界、传 nil 极其敏感，`NSArray` 越界、`NSMutableDictionary` 传 nil value 都会直接抛异常崩溃，而苹果的 API 本身不做这层保护。逐个调用点去加 `if` 判断既不现实也容易漏。Swizzling 可以一次性给这些“危险方法”套上统一的边界检查与异常捕获，把“崩溃”降级成“打条日志 + 安全返回”。

```objc
@implementation NSArray (SafeAccess)
+ (void)load {
    // 注意真实类名：不可变数组是 __NSArrayI
    Method from = class_getInstanceMethod(objc_getClass("__NSArrayI"), @selector(objectAtIndex:));
    Method to   = class_getInstanceMethod(objc_getClass("__NSArrayI"), @selector(tw_objectAtIndex:));
    method_exchangeImplementations(from, to);
}
- (id)tw_objectAtIndex:(NSUInteger)index {
    if (index >= self.count) {        // 越界拦截
        NSLog(@" NSArray 越界: index %lu, count %lu", index, self.count);
        return nil;                   // 不崩，返回 nil
    }
    return [self tw_objectAtIndex:index];
}
@end
```

这里有个**专属于这类场景的坑**值得记住：集合类是“类簇（class cluster）”，平时用的 `NSArray`/`NSMutableArray` 只是抽象的对外门面，真正干活的是内部私有类——不可变数组是 `__NSArrayI`、可变数组是 `__NSArrayM`、字典则是 `__NSDictionaryI`/`__NSDictionaryM`。所以 swizzle 时必须用 `objc_getClass("__NSArrayI")` 拿到真实类，直接对 `NSArray` 下手是无效的。

不过这类“全局兜底”是把双刃剑：它能救线上崩溃，但也会**掩盖代码里真实的逻辑错误**（越界本该在开发期暴露并修掉，被静默吞掉后问题会潜伏更久）。业界常见的折中是只在 Release 包启用兜底、Debug 包照常崩溃以便及早发现，或者兜底的同时把异常上报到监控平台。

### 全局定制：换肤、多语言、统一样式

这类场景的诉求是“**让某个全局行为在运行时被整体替换**”，而不想为此重启 App 或改遍代码。

最经典的是**多语言热切换**。常规做法是切语言后必须重启 App 才能让 `NSLocalizedString` 重新读取对应语言包。通过 swizzle `NSBundle` 的 `localizedStringForKey:value:table:`，可以让它在运行时根据当前选择的语言去读不同的 bundle，从而做到**不重启即时切换**。同理，**换肤/夜间模式**可以 hook 颜色或图片的读取方法，让同一套 `imageNamed:` 在不同主题下返回不同资源；**统一 UI 样式**（比如全 App 导航栏字体、返回按钮）也能通过 swizzle 相应的初始化方法一次性铺开。这类用法的特点是“一处开关、全局变脸”，是 `UIAppearance` 难以覆盖的深度定制的补充手段。

### 修复与改造

有时系统某个 API 存在 Bug，或某个第三方库的行为不符合你的需求，但你**拿不到、也不该改它的源码**。Swizzling 这时就像一把手术刀，让你在外部“打补丁”。比如某个版本的系统控件在特定条件下会崩，你可以 swizzle 它的相关方法，在调用原实现前先规避掉触发条件；又比如第三方 SDK 的某个方法日志太吵或行为不合预期，你可以 hook 它做拦截或改写。这是“无法修改源码时的最后手段”，威力大，但也最容易引入隐蔽问题——一旦对方在新版本里改了实现，你的补丁可能失效甚至冲突。


### 使用场景：

凡是能用**继承、组合、协议、通知**这些显式手段解决的，都优先用它们，因为它们是编译期可见、可追溯、不改变全局状态的。只有当“既改不了源码、又必须全局生效、还没有别的路”三个条件同时成立时，Swizzling 才是那个最优解。而且无论用在哪个场景，前面标准模板那节强调的几条铁律——**写在 `+load`、包进 `dispatch_once`、用带 `class_addMethod` 判断的安全模板、写清楚注释**——都必须照做，否则它带来的麻烦会远大于便利。
使用 Swizzling 时要记住几条边界：

- **只做小而确定的替换**：不要把复杂业务逻辑塞进 `+load`。
- **必须幂等**：用 `dispatch_once` 防止重复交换。
- **不要在 `+load` 里调 `[super load]`**：Runtime 会自己调用父类和分类的 `+load`。
- **尽量保留原实现**：除非非常确定可以完全替代，否则要调用原 IMP。
- **避免命名冲突**：Category 方法必须加明确前缀，基础库更适合用函数指针方案。
- **注意 `_cmd` 改变**：互相调用 swizzled selector 时，原实现看到的 `_cmd` 可能不是原 selector。
- **不要依赖多个 swizzle 的顺序**：不同库、不同 Category 的加载顺序不应成为业务前提。
- **谨慎修改系统类**：Swizzling 改的是全局类行为，会影响所有实例和子类。


# 4. 关联对象
![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260622175757334.png)

## 4.1 是什么，解决什么问题

Associated Objects 是 Objective-C 2.0 Runtime 的特性之一。它要补的，是 Category 一个先天缺陷：**Category 不能添加实例变量**。往一个已经存在的类上写 `@property`，编译器既不会合成 ivar，也不会生成存取方法——`@property` 在 Category 里退化成了一句空声明，背后什么都没有。关联对象就是用来补上这块“存储”的：把值挂进 Runtime 维护的一张外部表，让对象看起来多了一个属性。

官方给出的典型用途有三类：

- 给已有的类添加私有变量；
- 给已有的类添加公有属性（也就是“Category 加属性”的标准做法）；
- **为 KVO 创建关联的观察者**——把观察者对象直接挂在被观察对象上，省去自己单独维护一张“对象 → 观察者”的映射表（正好呼应前面第 7 章的 KVO）。

## 4.2 怎么用

整套机制对外只有三个 API：

```objc
// 设置 / 更新关联值（value 传 nil 表示删除这个 key）
void objc_setAssociatedObject(id object, const void *key,
                              id value, objc_AssociationPolicy policy);
// 读取关联值
id   objc_getAssociatedObject(id object, const void *key);
// 移除该对象上的全部关联值
void objc_removeAssociatedObjects(id object);
```

最常见的用法，就是在 Category 里手写一对存取方法，把“属性”落到关联表上：

```objc
@interface UIViewController (TWTracking)
@property (nonatomic, copy) NSString *tw_pageName;
@end

@implementation UIViewController (TWTracking)

static const void *TWPageNameKey = &TWPageNameKey;

- (void)setTw_pageName:(NSString *)tw_pageName {
    objc_setAssociatedObject(self,
                             TWPageNameKey,
                             tw_pageName,
                             OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)tw_pageName {
    return objc_getAssociatedObject(self, TWPageNameKey);
}

@end
```

key 的常见写法：

```objc
static const void *Key = &Key;
```

它的优点是地址稳定、唯一性强、不依赖字符串内容。

还有两种常见写法：

```objc
static char TWPageNameKey;
objc_setAssociatedObject(self, &TWPageNameKey, value, policy);

objc_setAssociatedObject(self, @selector(tw_pageName), value, policy);
```

`@selector(getter)` 这种写法也很常见，优点是不用额外定义 key，并且天然和属性 getter 对应。

## 4.3 关联策略与内存语义

第四个参数 `policy` 决定关联值的内存语义，按你想要的属性语义对号入座即可：

| 属性语义 | 关联策略 | 线程安全 |
| --- | --- | --- |
| `assign` | `OBJC_ASSOCIATION_ASSIGN` | 否 |
| `strong, nonatomic` | `OBJC_ASSOCIATION_RETAIN_NONATOMIC` | 否 |
| `copy, nonatomic` | `OBJC_ASSOCIATION_COPY_NONATOMIC` | 否 |
| `strong, atomic` | `OBJC_ASSOCIATION_RETAIN` | 是 |
| `copy, atomic` | `OBJC_ASSOCIATION_COPY` | 是 |

这里有一个**必须强调**的坑：`OBJC_ASSOCIATION_ASSIGN` 不是 `weak`，它更接近 `unsafe_unretained`。被关联的对象释放后，关联表里不会自动把它置 nil，再读就是野指针。换句话说，关联对象本身**给不了 `weak` 那种“对象没了自动变 nil”的语义**——真要这个效果，得自己在合适时机清理，或者关联一个内部持 `weak` 引用的包装对象。

## 4.4 底层实现：两层哈希 + 全局锁

关键认知：**关联值不存在对象自己身上**，而是由 Runtime 用一张全局表统一管理。这张表可以拆成四个角色来理解（名字来自源码思路，不同版本实现细节有调整，但这个模型一直成立）：

- `AssociationsManager`：总管，持有一把锁（早期是自旋锁 `spinlock`，新版已换成别的锁，本质都是给整张表加锁、保证多线程读写安全）；
- `AssociationsHashMap`：第一层哈希，**以对象地址为 key**（地址会先做一层伪装 `disguised_ptr_t`），用来定位某个对象专属的关联表；
- `ObjectAssociationMap`：第二层哈希，**以你传入的 key 为 key**，定位到具体那条关联记录；
- `ObjcAssociation`：最终的记录，存着 `{ policy, value }`。

串起来是这样一条两层查找链：

```text
AssociationsManager（持锁）
└─ AssociationsHashMap
      对象地址 → ObjectAssociationMap
                     你的 key → ObjcAssociation { policy, value }
```

set / get / remove 都建立在这两层查找之上：

- **set，且 value 非 nil**：加锁 → 第一层用对象地址找到（或新建）`ObjectAssociationMap` → 第二层按 key 插入或更新 `{policy, value}`。如果这是该对象**第一次**被关联，还会调用 `setHasAssociatedObjects()`，把对象 isa 里的 `has_assoc` 标志位置 1。
- **set，且 value 为 nil**：等价于“删除这个 key”，在两层表里找到对应记录并擦除。
- **get**：加锁 → 两层查找拿到 `ObjcAssociation` → 按 policy 决定是否对返回值做 retain / autorelease。
- **removeAssociatedObjects**：先看 `has_assoc` 标志，没有就直接返回；有则通过 `_object_remove_assocations()` 清空该对象的整张 `ObjectAssociationMap`，再从第一层表里抹掉这个对象的条目。

两点要记住：

1. 关联对象**不是 ivar**。它不改变对象大小，也不改变 ivar 偏移，对象内部结构原封不动——值全在那张外部全局表里。
2. 这张表是全局共享的，每次读写都要过锁。在高频热路径上大量使用关联对象，锁竞争是要考虑的隐性成本。

## 4.5 生命周期：对象释放时自动清理

最容易让人犯嘀咕的问题是：我挂上去的关联值，要不要手动释放？

**不用。** 对象销毁时，Runtime 会自动回收它的全部关联值，链路大致是：

```text
[obj dealloc]
  → object_dispose()
     → objc_destructInstance()
        → _object_remove_assocations()   // 若 has_assoc 为 1，清空该对象的关联表
```

`has_assoc` 标志位（对应 Part 1 的 `isa_t.has_assoc`）在这里就是一次性能优化：释放任何对象时，Runtime 先瞄一眼这个标志，为 0 就整套关联清理逻辑都跳过，省掉无谓的哈希查找。这也是为什么“第一次设置关联值”时要顺手把它置 1。

按 WWDC 2011 Session 322 的说法，关联对象的擦除时机其实**比很多人直觉的要晚**——它发生在 `object_dispose()` 阶段，由 `NSObject` 的 `-dealloc` 间接触发，而不是对象一进入 `dealloc` 就立刻清。对业务代码的含义很简单：**你不需要在自己的 `dealloc` 里手动清关联对象，Runtime 已经包办了。**

最后区分两个容易混的“删除”：

- 删**单个** key：对同一个 key 重新 `set` 一个 `nil` 值即可；
- 删**全部**：`objc_removeAssociatedObjects(obj)` 会清掉这个对象上的**所有**关联值，因此**不适合只删自己那一个**——别人（比如某个第三方库）挂在同一对象上的关联值会被一起抹掉。日常基本只用前者。

# 5. 消息转发应用

消息转发可以用于代理转发、容错、多播等场景。它承接 Part 2 的转发链。严格说，完整兜底管线前面还有一次**动态方法解析**：如果 `resolveInstanceMethod:` / `resolveClassMethod:` 成功补上方法，就不会进入转发；只有动态解析也放弃，才会继续走下面三步。

```text
objc_msgSend 查缓存/方法列表
  -> resolveInstanceMethod: / resolveClassMethod:
  -> forwardingTargetForSelector:
  -> methodSignatureForSelector:
  -> forwardInvocation:
  -> doesNotRecognizeSelector:
```

如果只是把消息交给另一个对象，优先用快速转发：

```objc
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if ([self.helper respondsToSelector:aSelector]) {
        return self.helper;
    }
    return [super forwardingTargetForSelector:aSelector];
}
```

这里还有一个反面边界：`forwardingTargetForSelector:` 可以返回另一个对象，也可以返回 `nil` / `[super forwardingTargetForSelector:]` 让 Runtime 继续走完整转发，但**不要返回 `self`**。返回 `self` 等于把同一条消息重新发给自己，下一轮又进 `forwardingTargetForSelector:`，最终无限递归。

## 组合对象能力透传（模拟多继承）

Objective-C 在语言层面只支持单继承。这里讨论的不是让一个类真的继承多个父类，而是借助**消息转发（Message Forwarding）​**做组合对象能力透传：当宿主对象自己没有某个方法实现时，把这条消息在运行时"透传"给它持有的其他对象去执行。把多个能力提供方组合在一起、再通过转发让宿主对象统一对外暴露它们的方法，对外看像一个聚合了多方能力的对象，实际执行者仍是各自的真实对象。

### forwardingTargetForSelector 做"快速转发"​

```objc
// Father 类有方法 work；Mother 类有方法 cook
@interface Son : Father
@property (nonatomic, strong) Mother *mother;
@end

@implementation Son
- (instancetype)init {
    if (self = [super init]) {
        _mother = [Mother new];
    }
    return self;
}

// 自己处理不了的消息，问问 mother 能不能处理
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if ([_mother respondsToSelector:aSelector]) {
        return _mother;   // 把 receiver 替换成 mother
    }
    return nil;           // 返回 nil 则继续走完整转发
}
@end

// 使用
Son *son = [[Son alloc] init];
[son work];                              // 继承自 Father → I'm working.
[son performSelector:@selector(cook)];   // 透传给 Mother → I'm cooking.

```

从调用者视角看， `Son` 同时"拥有"了 `Father`（真继承）和 `Mother`（透传借来）的能力。它的局限也很明确：**只能把消息整体转给一个对象，无法对参数和返回值做任何加工，也无法同时分发给多个对象**。 这不是真正的多继承，只是消息被转手了。`NSObject` 的很多查询默认也不理解这种“能力透传”：

```objc
[son respondsToSelector:@selector(cook)] // 默认可能仍是 NO
[son isKindOfClass:[Mother class]]       // 仍是 NO
```

如果你希望这个透传行为更像“这个对象真的支持该能力”，就要同步覆写 `respondsToSelector:`、`methodSignatureForSelector:`、`conformsToProtocol:`，甚至类方法侧的 `instancesRespondToSelector:`。这也是为什么消息转发不应该被当作继承的常规替代品：它适合做组合和代理，不适合伪造类型体系。

### forwardInvocation 做"完整转发"​

```objc
@interface Son : Father
@property (nonatomic, strong) Mother *mother;
@property (nonatomic, strong) NSMutableArray<id> *forwardTargets;
@end

@implementation Son

- (instancetype)init {
    if (self = [super init]) {
        _mother = [Mother new];
        _forwardTargets = [NSMutableArray arrayWithObject:_mother];
    }
    return self;
}

// 第一步：给找不到的方法补一份方法签名。
// 返回 nil，Runtime 就会进入 doesNotRecognizeSelector: 崩溃。
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *signature = [super methodSignatureForSelector:aSelector];
    if (signature) {
        return signature;
    }

    for (id target in self.forwardTargets) {
        signature = [target methodSignatureForSelector:aSelector];
        if (signature) {
            return signature;
        }
    }

    return nil;
}

// 第二步：Runtime 把原始调用包装成 NSInvocation 交给这里。
- (void)forwardInvocation:(NSInvocation *)invocation {
    SEL selector = invocation.selector;
    BOOL handled = NO;
    BOOL rewroteObjectArgument = NO;

    // 示例：读取并改写参数。注意 self 和 _cmd 占 index 0、1，
    // 业务参数从 index 2 开始。
    if (selector == @selector(cookWithFood:)) {
        __unsafe_unretained NSString *food = nil;
        [invocation getArgument:&food atIndex:2];

        if (food.length == 0) {
            NSString *defaultFood = [NSString stringWithFormat:@"%@", @"rice"];
            [invocation setArgument:&defaultFood atIndex:2];
            rewroteObjectArgument = YES;
        }
    }

    // NSInvocation 默认不持有参数。只要改写了对象参数，或者 invocation
    // 可能被缓存、延迟、跨线程执行，就应该让它持有当前参数。
    if (rewroteObjectArgument) {
        [invocation retainArguments];
    }

    for (id target in self.forwardTargets) {
        if (![target respondsToSelector:selector]) {
            continue;
        }

        [invocation invokeWithTarget:target];
        handled = YES;

        // 有返回值的方法只能选择一个目标的返回值；
        // void 方法才适合继续分发给多个对象。
        const char *returnType = invocation.methodSignature.methodReturnType;
        if (strcmp(returnType, @encode(void)) != 0) {
            if (returnType[0] == '@') {
                __unsafe_unretained id returnValue = nil;
                [invocation getReturnValue:&returnValue];
                NSLog(@"forward %@ return: %@",
                      NSStringFromSelector(selector),
                      returnValue);
            }
            return;
        }
    }

    if (handled) {
        return;
    }

    [super forwardInvocation:invocation];
}

@end
```

`forwardInvocation:` 的强大之处在于 `NSInvocation` 把"消息调用的全部细节"都封装好了，你可以在调用前后插入逻辑、改写参数、读取返回值。这也是 AOP（面向切面编程）和很多 Hook 框架的实现基础——业务方法被替换成 `_objc_msgForward`，最终都汇聚到 `forwardInvocation:` 里统一加工。

上面 `retainArguments` 是一个很重要的边界：`NSInvocation` 默认只保存参数字节，不 retain 对象参数，也不 copy C 字符串。这个 demo 是同步调用，即使不加也大概率能跑；但只要你改写的是动态生成的对象，或者把 invocation 存起来稍后执行，就可能留下悬垂指针。实际写框架时，只要有“改写对象参数 / 延迟调用 / 跨线程调用”的可能，就应该显式调用 `retainArguments`。

#### **​能力透传的边界：内省要诚实**

透传出来的能力是"借来"的，`NSObject` 的内省方法只认真正的继承体系，**不认转发链**。也就是说，即便 `son` 能执行 `cook`，`[son respondsToSelector:@selector(cook)]` 默认仍可能返回 `NO`。如果你的透传对象要对外表现成"能响应这些消息"，可以把转发算法补进部分内省方法里：

```objc
- (BOOL)respondsToSelector:(SEL)aSelector {
    if ([super respondsToSelector:aSelector]) {
        return YES;
    }

    for (id target in self.forwardTargets) {
        if ([target respondsToSelector:aSelector]) {
            return YES;
        }
    }

    return NO;
}

- (BOOL)conformsToProtocol:(Protocol *)aProtocol {
    if ([super conformsToProtocol:aProtocol]) {
        return YES;
    }

    for (id target in self.forwardTargets) {
        if ([target conformsToProtocol:aProtocol]) {
            return YES;
        }
    }

    return NO;
}

// 类方法只能回答静态能力。如果转发目标是每个实例动态配置的，
// 这里就不能完整表达，只能列出固定会透传的能力提供方。
+ (BOOL)instancesRespondToSelector:(SEL)aSelector {
    return [super instancesRespondToSelector:aSelector] ||
           [Mother instancesRespondToSelector:aSelector];
}
```



这类覆写只影响运行时内省，比如 `[obj respondsToSelector:]`。它不会改变编译期类型检查：形参写成 `id<SomeProtocol>`、变量静态类型、编译器警告，都不会因为你覆写了 `conformsToProtocol:` 而改变。

另外，伪造 `respondsToSelector:` 也不是零成本。一旦调用方相信它返回 YES 并真的发消息，这条消息在宿主类里仍然找不到实现，每次都要走转发路径。快速转发只是多一跳，完整转发则要创建 `NSInvocation`、匹配签名、包装参数，开销明显高于直接调用。高频热路径上，优先用显式组合、协议方法或快速转发，不要把所有能力都压到 `forwardInvocation:`。

所以，"组合对象能力透传"不是让 Objective-C 真的拥有多继承，而是用 Runtime 把组合对象的能力在消息层面暴露出去。


# 6.  AOP

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260621164825648.png)


`forwardInvocation:` 更完整的应用，是用 `NSProxy` 做一层动态代理，把一次消息调用包起来，在目标方法前后插入额外逻辑。这就是 AOP（Aspect Oriented Programming，面向切面编程）在 Objective-C 里的经典做法之一。

`NSProxy` 和 `NSObject` 一样，都是 Objective-C 根层级里的抽象基类。不同的是，`NSProxy` 天生就是为了“转发消息”设计的：它本身不像 `NSObject` 那样带大量默认行为，而是要求子类重点实现：

```objc
- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel;
- (void)forwardInvocation:(NSInvocation *)invocation;
```

系统里常见的 `NSDistantObject`、`NSProtocolChecker` 都是基于 `NSProxy` 思路做消息代理或协议检查。

下面搭一个最小 AOP 代理：调用目标方法前后，分别执行 `preInvoke:` 和 `postInvoke:`。

```objc
@protocol Invoker <NSObject>
- (void)preInvoke:(NSInvocation *)invocation;
- (void)postInvoke:(NSInvocation *)invocation;
@end

@interface AuditingInvoker : NSObject <Invoker>
@end

@implementation AuditingInvoker

- (void)preInvoke:(NSInvocation *)invocation {
    NSLog(@"before %@", NSStringFromSelector(invocation.selector));
}

- (void)postInvoke:(NSInvocation *)invocation {
    NSLog(@"after %@", NSStringFromSelector(invocation.selector));
}

@end
```

代理对象保存三个东西：

- `proxyTarget`：真正接收消息的对象。
- `invoker`：切面逻辑执行者。
- `selectors`：哪些 selector 需要切面增强。

```objc
@interface AspectProxy : NSProxy

@property (nonatomic, strong, readonly) id proxyTarget;
@property (nonatomic, strong, readonly) id<Invoker> invoker;
@property (nonatomic, strong, readonly) NSMutableSet<NSString *> *selectors;

+ (instancetype)proxyWithTarget:(id)target invoker:(id<Invoker>)invoker;
- (void)registerSelector:(SEL)selector;

@end

@implementation AspectProxy

+ (instancetype)proxyWithTarget:(id)target invoker:(id<Invoker>)invoker {
    AspectProxy *proxy = [AspectProxy alloc];
    proxy->_proxyTarget = target;
    proxy->_invoker = invoker;
    proxy->_selectors = [NSMutableSet set];
    return proxy;
}

- (void)registerSelector:(SEL)selector {
    [self.selectors addObject:NSStringFromSelector(selector)];
}

- (BOOL)shouldInterceptSelector:(SEL)selector {
    return [self.selectors containsObject:NSStringFromSelector(selector)];
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector {
    return [self.proxyTarget methodSignatureForSelector:selector];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    SEL selector = invocation.selector;

    if ([self shouldInterceptSelector:selector]) {
        [self.invoker preInvoke:invocation];
        [invocation invokeWithTarget:self.proxyTarget];
        [self.invoker postInvoke:invocation];
        return;
    }

    [invocation invokeWithTarget:self.proxyTarget];
}

- (BOOL)respondsToSelector:(SEL)aSelector {
    return [self.proxyTarget respondsToSelector:aSelector];
}

@end
```

测试对象：

```objc
@interface Student : NSObject
- (void)study:(NSString *)subject andRead:(NSString *)book;
- (void)sleep;
@end

@implementation Student

- (void)study:(NSString *)subject andRead:(NSString *)book {
    NSLog(@"study %@, read %@", subject, book);
}

- (void)sleep {
    NSLog(@"sleep");
}

@end
```

使用方式：

```objc
Student *student = [Student new];
AuditingInvoker *invoker = [AuditingInvoker new];
AspectProxy *proxy = [AspectProxy proxyWithTarget:student invoker:invoker];

[proxy registerSelector:@selector(study:andRead:)];

[(id)proxy study:@"Runtime" andRead:@"objc4"];
[(id)proxy sleep];
```

输出逻辑可以理解成：

```text
before study:andRead:
study Runtime, read objc4
after study:andRead:
sleep
```

`study:andRead:` 被注册进 `selectors`，所以一次调用会触发 `pre -> target -> post` 三步；`sleep` 没注册，就只透传给目标对象。

## 6.1 方案一：NSProxy 代理式 AOP

上面这种写法的核心是**不改原类**，而是让调用方拿到一个 proxy：

```objc
Student *student = [Student new];
AspectProxy *proxy = [AspectProxy proxyWithTarget:student invoker:invoker];

[(id)proxy study:@"Runtime" andRead:@"objc4"];
```

消息先发给 `AspectProxy`，再由 `forwardInvocation:` 转给 `student`。切面逻辑也只发生在经过这个 proxy 的调用上：

```text
caller -> proxy -> forwardInvocation: -> target
```

所以它更像一种“对象级代理增强”：适合日志审计、权限校验、延迟执行、远程代理、弱代理、多播代理这类场景。优点是边界清楚、影响范围小；缺点是调用方必须持有并使用这个 proxy。如果某个地方绕过 proxy 直接调用原对象，切面逻辑就不会执行。

## 6.2 方案二：Aspects 风格 AOP

halfrost 那篇文章里讲的 AOP，更接近经典 Aspects 库的思路：**不要求调用方换成 proxy，而是直接改目标类的方法入口**。

它的大致流程是：

```text
注册阶段：
target selector 原 IMP -> 保存起来
target selector 新 IMP -> 替换成 _objc_msgForward
切面 block -> 按 before / instead / after 保存

执行阶段：
objc_msgSend 找到的 IMP 是 _objc_msgForward
消息进入完整转发
methodSignatureForSelector: 返回原方法签名
forwardInvocation: 拿到 NSInvocation
执行 before / instead / after
必要时再调用原 IMP
```

伪代码可以理解成这样：

```objc
static void TWInstallAspect(Class cls, SEL selector, TWAspectInfo *info) {
    Method method = class_getInstanceMethod(cls, selector);
    if (!method) {
        return;
    }

    info->originalIMP = method_getImplementation(method);
    info->typeEncoding = method_getTypeEncoding(method);

    // 关键点：把原方法入口改成消息转发入口。
    // 后续调用这个 selector 时，不再直接执行原 IMP，而是进入 forwardInvocation:。
    method_setImplementation(method, _objc_msgForward);
}
```

真正执行时，框架会在 `forwardInvocation:` 里根据 selector 找到对应的切面信息：

```objc
- (void)forwardInvocation:(NSInvocation *)invocation {
    TWAspectInfo *info = TWAspectInfoForSelector(invocation.selector);
    if (!info) {
        [self tw_forwardInvocation:invocation];
        return;
    }

    for (TWAspectBlock block in info.beforeBlocks) {
        block(invocation);
    }

    if (info.insteadBlocks.count > 0) {
        for (TWAspectBlock block in info.insteadBlocks) {
            block(invocation);
        }
    } else {
        // 伪代码：真实框架不能这么粗暴地写。
        // 它需要按方法签名处理参数、返回值、结构体返回、stret/fpret 等细节，
        // 再安全地调用之前保存的 originalIMP。
        TWInvokeOriginalIMP(info.originalIMP, invocation);
    }

    for (TWAspectBlock block in info.afterBlocks) {
        block(invocation);
    }
}
```

这条路和 `NSProxy` 最大的区别是：调用方仍然调用原对象，完全感知不到 proxy 的存在。

```text
caller -> target selector -> _objc_msgForward -> forwardInvocation: -> aspect/original IMP
```

所以它更适合“对已有方法统一插入行为”的场景，比如埋点、统计、调试探针、SDK 级 Hook。代价也更高：它同时碰了 Method Swizzling、消息转发、`NSInvocation` 和原 IMP 调用，对方法签名、返回值、结构体参数、多次 hook 顺序都非常敏感。尤其是调用原 IMP 这一步，不能只靠简单的 C 函数强转糊过去，真实框架必须严肃处理参数 ABI。

## 6.3 两种 AOP 方案怎么选

| 方案 | 是否修改原类 IMP | 调用方是否要换对象 | 影响范围 | 更适合 |
| --- | --- | --- | --- | --- |
| `NSProxy + forwardInvocation:` | 否 | 是，必须经过 proxy | 单个对象 / 单条调用链 | 对象级代理增强、权限、日志、弱代理、多播代理 |
| Aspects 风格 `_objc_msgForward` | 是 | 否，仍然调用原对象 | 类级方法入口，通常影响所有实例 | 埋点、Hook、SDK 级横切逻辑 |
| 普通 Method Swizzling | 是 | 否 | 类级方法入口，通常影响所有实例 | 替换系统方法、异常保护、全局行为修正 |

所以这两种 AOP 不是互相替代，而是边界不同：

- 如果你能控制调用链，并且只想增强某个对象，优先用 `NSProxy`，因为它局部、显式、可控。
- 如果你要对现有类的现有方法统一插入逻辑，调用方又不能改成 proxy，才考虑 Aspects 风格。
- 如果只是简单地把一个方法实现换成另一个实现，不需要 `NSInvocation` 读取和改写参数，普通 Method Swizzling 更直接。

日常业务代码里，不建议一上来就用 Aspects 风格。它能力强，但隐蔽性也强，调试成本高；更适合作为框架能力，而不是普通业务分支。

# 7. Isa Swizzling 与 KVO

Method Swizzling 改的是 `SEL -> IMP` 映射；Isa Swizzling 改的是对象的 `isa` 指向。最典型的系统级应用就是 KVO。

>Automatic key-value observing is implemented using a technique called _isa-swizzling_.  
The isa pointer, as the name suggests, points to the object's class which maintains a dispatch table. This dispatch table essentially contains pointers to the methods the class implements, among other data.  
When an observer is registered for an attribute of an object the isa pointer of the observed object is modified, pointing to an intermediate class rather than at the true class. As a result the value of the isa pointer does not necessarily reflect the actual class of the instance.  
You should never rely on the isa pointer to determine class membership. Instead, you should use the [class](https://developer.apple.com/reference/objectivec/1418956-nsobject/1571949-class) method to determine the class of an object instance.

当你对一个对象添加观察者：

```objc
Student *stu = [Student new];
[stu addObserver:self
      forKeyPath:@"name"
         options:NSKeyValueObservingOptionNew
         context:nil];
```

系统不会直接修改 `Student` 这个类，而是动态创建一个中间子类，常见名字类似：

```text
NSKVONotifying_Student
```

然后把这个具体对象的 `isa` 从 `Student` 指向 `NSKVONotifying_Student`。所以 KVO 改变的是“这个对象运行时属于哪个类”，而不是“Student 类本身长什么样”。

可以用 `object_getClass` 和方法列表直接观察这个变化：

```objc
static void PrintClassInfo(id obj) {
    NSLog(@"[obj class]       = %@", [obj class]);
    NSLog(@"object_getClass   = %@", object_getClass(obj));

    unsigned int count = 0;
    Method *methods = class_copyMethodList(object_getClass(obj), &count);
    for (unsigned int i = 0; i < count; i++) {
        SEL sel = method_getName(methods[i]);
        NSLog(@"method: %@", NSStringFromSelector(sel));
    }
    free(methods);
}

Student *stu = [Student new];
PrintClassInfo(stu);

[stu addObserver:self
      forKeyPath:@"name"
         options:NSKeyValueObservingOptionNew
         context:nil];

PrintClassInfo(stu);
```

添加观察者之前，`[stu class]` 和 `object_getClass(stu)` 都指向 `Student`。添加观察者之后，`[stu class]` 仍然返回 `Student`，但 `object_getClass(stu)` 会变成类似 `NSKVONotifying_Student` 的动态子类。

打印方法列表时，会看到动态子类里通常多出几类方法。不同系统版本的私有实现细节可能不完全一致，但这个观察结果能说明 KVO 的基本策略：

```text
添加观察者之前：
[stu class]       = Student
object_getClass   = Student
methods           = name, setName:, .cxx_destruct ...

添加观察者之后：
[stu class]       = Student
object_getClass   = NSKVONotifying_Student
methods           = setName:, class, dealloc, _isKVOA ...
```

也就是说，KVO 不是直接改 `Student`，而是临时插入了一个 `Student` 的动态子类。这个动态子类通常会覆写四类方法：


## 7.1 重写 `class`：隐藏动态子类

中间类之所以要重写 `class` 方法，是为了**在你调用它时返回与改写继承关系之前同样的内容**，也就是依旧返回 `Student`，让外界察觉不到真实类已经被偷换成了 `NSKVONotifying_Student`。

```objc
NSLog(@"[stu class]     = %@", [stu class]);
NSLog(@"object_getClass = %@", object_getClass(stu));
```

加 KVO 之后，两行输出可能分别是：

```text
[stu class]     = Student
object_getClass = NSKVONotifying_Student
```

原因是二者不在同一层面：
- `[stu class]` 是一次普通消息发送，最终会走到对象当前类里的 `class` 方法。KVO 动态子类覆写了这个方法，让它返回原来的 `Student`。
- `object_getClass(stu)` 是 Runtime API，直接读取对象当前的 `isa` 指向，所以能看到真实的 `NSKVONotifying_Student`。

KVO 这么做是为了不破坏外部代码对对象类型的直觉。你仍然可以把它当成 `Student` 用，但如果你绕开方法调用直接问 Runtime，它的真实类已经变了。

## 7.2 重写 setter：插入 will / did 通知

KVO 真正监听的是“值发生变化”。对普通属性来说，最自然的切入点就是 setter。动态子类会覆写被观察属性对应的 setter，在赋值前后插入：

```objc
- (void)willChangeValueForKey:(NSString *)key;
- (void)didChangeValueForKey:(NSString *)key;
```

最终在 `didChangeValueForKey:` 之后，观察者会收到：

```objc
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary<NSKeyValueChangeKey, id> *)change
                       context:(void *)context;
```

用伪代码表示，这个动态子类大概像这样。注意这不是 Apple 私有实现源码，只是把 KVO 的关键行为摊开：

```objc
@interface NSKVONotifying_Student : Student
@end

@implementation NSKVONotifying_Student

- (void)setName:(NSString *)name {
    [self willChangeValueForKey:@"name"];

    // 调用原 Student 的 setter。真实实现可能直接调原 IMP，
    // 也可能走 Foundation 内部的 __NSSet...ValueAndNotify 辅助函数。
    // 这里用 super 表达“回到原类逻辑”。
    [super setName:name];

    [self didChangeValueForKey:@"name"];
}

- (Class)class {
    return [Student class];
}

- (void)dealloc {
    // 做 KVO 相关清理，比如移除观察状态、恢复 isa 等。
    // ARC 下源码里不能手写 [super dealloc]；这里只表达“释放阶段会有清理逻辑”。
}

- (BOOL)_isKVOA {
    return YES;
}

@end
```

自动通知能覆盖几条常见路径：

```objc
stu.name = @"Tom";                    // 走 setter
[stu setName:@"Tom"];                 // 走 setter
[stu setValue:@"Tom" forKey:@"name"]; // KVC，NSObject 的自动通知也会覆盖
```

如果你绕过 setter 直接改 ivar，KVO 就没有自动切入点。这时要么不要这么写，要么手动包一层通知：

```objc
[stu willChangeValueForKey:@"name"];
_name = @"Tom";                    // 直接改 ivar 时，需要手动包 will/did
[stu didChangeValueForKey:@"name"];
```

把上面这几条路径并排看，会发现它们其实在回答同一个问题：`will/didChangeValueForKey:` 这对通知，到底是在哪一层被插进去的。这条判据是理解 KVO 自动通知的总钥匙，值得多停留一会儿。

**先看这对通知各自在做什么。** 它们不是可有可无的两声招呼，而是各自承担一半工作。`willChangeValueForKey:` 在值真正改变*之前*调用，KVO 在这一刻把当前值记下来，作为将来 change 字典里的 `NSKeyValueChangeOldKey`，并标记这个 key“正处于变化中”；

`didChangeValueForKey:` 在值改变*之后*调用，KVO 这时才去读新值填进 `NSKeyValueChangeNewKey`，组装好 change 字典，最终回调 `observeValueForKeyPath:ofObject:change:context:`。

所以顺序是硬约束：必须 will 在前、写值在中、did 在后。少了 will，旧值就抓不到；顺序颠倒，新旧值就会对调或错位。

**再回到三条落点，它们只是把这对通知插在了不同的层。**

- *有访问器方法时*，运行时重写 setter 把通知织进去。真实实现通常不是简单地 `[super setName:]`，而是走 Foundation 的 `_NSSetObjectValueAndNotify`（以及对应基础类型、结构体的 `_NSSet<Type>ValueAndNotify`）这类辅助函数——它内部正是“willChange → 原始赋值 → didChange”的封装。这也解释了 7.4 节用 `nm` 能在 Foundation 里翻出一整排这种符号：对象、整型、浮点、结构体各有一条专门的通知路径，而不是所有类型共用一个。

- *没有访问器方法时*，KVC 兜底。`setValue:forKey:` 在找不到 setter、退回直接写 ivar（按 `_key`、`_isKey`、`key`、`isKey` 的顺序查找）之前，会自己补上 will/did。所以哪怕一个属性压根没声明 setter、只能靠 KVC 读写，KVO 依然监听得到——这次通知是 KVC 替你发的。

- *手动调用时*，是你自己接管了这件事。要么你用 `automaticallyNotifiesObserversForKey:` 关掉了某个 key 的自动通知，要么你绕过 setter 直接改了 ivar，这时就得亲手把这对通知摆在赋值前后，做运行时本该替你做的事。

**反过来，失效的边界也由这同一条判据划定。** 任何“值变了、但这对通知没按规矩发出”的路径，KVO 都是瞎的：直接 `_name = @"Tom"` 改 ivar 又不手动包通知、在 C 层面直接写底层内存、或者通过别的渠道改了背后的存储而绕开了所有 setter 和 KVC。换句话说，KVO 监听的从来不是“值”，而是“通知”——问题不在值有没有变，而在通知有没有发。

`keyPathsForValuesAffectingValueForKey:`。它能让一个派生属性（比如 `fullName`）在 `firstName`、`lastName` 变化时也对外发出通知，本质上就是把“别人的通知”接到自己头上。KVO 之所以能这么玩，正是因为它认的始终是通知这件事，而不是某一句具体的赋值语句。

如果你想完全接管某个 key 的通知时机，可以覆写：

```objc
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    if ([key isEqualToString:@"name"]) {
        return NO;
    }
    return [super automaticallyNotifiesObserversForKey:key];
}
```

这样系统不会为 `name` 自动发送通知，你需要自己在合适的位置调用 `willChangeValueForKey:` / `didChangeValueForKey:`。

## 7.3 重写 `dealloc`：清理观察关系和动态类状态

KVO 动态子类还会覆写 `dealloc`。这里不是为了改变业务释放逻辑，而是为了处理观察机制自己的尾巴，比如观察状态、动态子类、isa 恢复等内部清理。

这也是 KVO 老接口容易出问题的原因之一：观察关系本身有生命周期。如果观察者和被观察对象释放顺序处理不好，就容易出现重复移除、未移除、释放后回调等问题。现代代码里如果能用 block token 或 Swift 的 `NSKeyValueObservation`，生命周期会更容易管理；在 Objective-C 老接口里，至少要保证 add/remove 成对出现。

## 7.4 重写 `_isKVOA`：标记 KVO 动态子类

`_isKVOA` 是私有方法，不能在业务代码里依赖它。它的意义更像一个内部标记：告诉 Foundation 这个类是 KVO 机制生成的动态子类。

halfrost 文章里还提到可以用 `nm -a Foundation` 观察到一批 KVO 辅助函数，例如对象、整型、浮点、结构体相关的 `__NSSet...ValueAndNotify`。这个点说明 KVO 的 setter 通知并不只服务对象类型，基础类型和常见结构体也有对应处理路径。放到现代文章里可以作为理解线索，但不要把这些私有符号当成稳定 API。

这也解释了一个常见差异：

```objc
[stu class]              // Student
object_getClass(stu)     // NSKVONotifying_Student
```

`class` 是方法，可以被 KVO 子类覆写；`object_getClass` 是 Runtime API，会直接沿对象的 isa 取真实类。也正因为 isa 可能被系统换掉，工程里不要用“直接读 isa”判断类型关系，应该用 `class`、`isKindOfClass:`、协议或明确的业务字段。

## 7.5 到底什么时候能监听，什么时候不能

前面四节都在讲 KVO 怎么用动态子类“偷梁换柱”。落到工程里，最常被问的其实是另一个问题：我这么改值，到底能不能被监听到？把前面的机制收成一条判据，就能直接对号入座：

> **改值前后，那对 `willChangeValueForKey:` / `didChangeValueForKey:` 有没有被发出。发了就能监听，没发就不能。**

下面所有“能 / 不能”，本质都是在套这一条——KVO 认的从来不是“值变了”，而是“通知发了”。

**能监听**——改值路径经过了某个会发通知的写入点：

| 写法 | 为什么能 |
|---|---|
| `obj.name = x` | 编译成 `[obj setName:]`，走被重写的 setter，发通知 |
| `[obj setName:x]` | 同上，走 setter |
| `[obj setValue:x forKey:@"name"]`（**有** setter） | KVC 最终也调 setter，发通知 |
| `[obj setValue:x forKey:@"name"]`（**没** setter、有 ivar） | KVC 直接写 ivar，并**自己补发**通知 |
| 手动 `willChange…` → 改值 → `didChange…` | 你亲手发了通知 |
| `[[obj mutableArrayValueForKey:@"items"] addObject:x]` | 集合代理会替你发集合变更通知 |
| 派生属性（配了 `keyPathsForValuesAffectingValueForKey:`），其依赖的源属性按上面方式变化 | 源属性的通知被转发到派生属性 |

**不能监听**——改值绕开了所有会发通知的写入点：

| 写法 | 为什么不能 |
|---|---|
| `_name = x`（直接改 ivar，又不手动包通知） | 绕过 setter，没人发通知 |
| `[_items addObject:x]` / `_dict[k] = v`（直接改可变对象内容） | 对象指针没变、setter 没被调，**最经典的坑** |
| 关掉自动通知（`automaticallyNotifiesObserversForKey:` 返回 `NO`）又忘了手动调 will/did | 自动的没了，手动的也没补 |
| C 层 / 底层直接写内存 | 完全绕开 ObjC 方法体系 |

归纳一句：**能不能监听，只取决于“改值的那条路径会不会经过一个发通知的写入点”**。经过被重写的 setter、经过 KVC 的 `setValue:forKey:`、或你手动补，都能；直接动底层存储、或只改可变对象的内容而不替换对象，都不能。

实战里九成的“KVO 不触发”都出在两处：一是直接改 ivar（`_name = x`），改成 `self.name = x` 或走 KVC 即可；二是可变集合“改内容”而非“换对象”（`[_items addObject:]` 监听不到），要么用 `mutableArrayValueForKey:` 操作，要么整体替换 `self.items = newArray`。

即便是苹果官方实现的 KVO 也并非完美。它的主要问题集中在回调机制上：不能传一个 selector 或者 block 作为回调，而必须重写 `-addObserver:forKeyPath:options:context:` 所引发的一系列连锁问题。如果只监听一两个属性还好，一旦监听的属性变多，或者同时监听多个对象的属性，就会比较麻烦，往往需要在回调方法里写大量 `if-else` 判断来区分。
最后，官方文档上对于KVO的实现的最后，给出了需要我们注意的一点是，**永远不要用用isa来判断一个类的继承关系，而是应该用class方法来判断类的实例。**

总的来说，isa-swizzling 是一套"用动态子类偷梁换柱"的优雅设计：通过改写 `isa` 指针让对象在运行时指向中间类，再靠 `setter` 注入变更通知、靠 `class` 维持伪装、靠 `dealloc` 善后、靠 `_isKVOA` 自我标识，从而在完全不侵入原类代码的前提下实现了属性变化的自动监听。


# 8. 基于遍历的自动归档与模型转换

`class_copyIvarList` / `class_copyPropertyList` 还有两个很常见的工程用途：自动归档和字典模型互转。它们的共同点是：先遍历类元数据，再通过 KVC、getter/setter 或 `objc_msgSend` 读写值。

## NSCoding 自动归档 / 解档

手写 `encodeWithCoder:` 和 `initWithCoder:` 时，属性一多就会出现大量重复代码：

```objc
- (void)encodeWithCoder:(NSCoder *)coder {
    [coder encodeObject:self.name forKey:@"name"];
    [coder encodeInteger:self.age forKey:@"age"];
}
```

可以用 ivar 遍历减少样板代码：

```objc
- (void)encodeWithCoder:(NSCoder *)coder {
    unsigned int count = 0;
    Ivar *ivars = class_copyIvarList([self class], &count);

    for (unsigned int i = 0; i < count; i++) {
        const char *name = ivar_getName(ivars[i]);
        NSString *key = [NSString stringWithUTF8String:name];
        id value = [self valueForKey:key];
        [coder encodeObject:value forKey:key];
    }

    free(ivars);
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super init];
    if (!self) return nil;

    unsigned int count = 0;
    Ivar *ivars = class_copyIvarList([self class], &count);

    for (unsigned int i = 0; i < count; i++) {
        const char *name = ivar_getName(ivars[i]);
        NSString *key = [NSString stringWithUTF8String:name];
        id value = [coder decodeObjectForKey:key];
        [self setValue:value forKey:key];
    }

    free(ivars);
    return self;
}
```

实际项目里还要处理父类 ivar、基本类型、忽略字段、字段重命名和安全归档等问题；但核心思路就是“遍历元数据 + 按 key 读写”。

## 字典与模型互转

字典转模型也是同一套路：

1. 用 `class_copyPropertyList` 拿到属性列表。
2. 用 `property_getName` 得到属性名。
3. 从字典里取同名 value。
4. 通过 KVC 或 setter 赋值。
5. 如果属性本身还是模型，再递归转换。

模型转字典则反过来：遍历属性，生成 getter，取出值后写进字典。

MJExtension、JSONModel 这类库都建立在这条思路上，只是它们额外处理了容器泛型、字段映射、类型转换、嵌套模型、黑白名单等大量工程细节。

# 9. Runtime 应用边界

Runtime 能力强，但边界也要清楚。

**第一，签名必须一致。**

Swizzling 或动态添加方法时，IMP 的真实函数签名必须和 selector 对应的方法签名一致。签名错了，参数寄存器和返回值解释都会错。

**第二，不要依赖多个 swizzle 的顺序。**

多个库都替换同一个方法时，最终调用链取决于加载和交换顺序。这个顺序可以观察，但不应该成为业务前提。

**第三，Swizzling 改的是全局行为。**

你交换的不是某个对象的方法，而是类的方法映射。一个 Category 里的交换，可能影响整个 App 中所有这个类及其子类的实例。改动非自己拥有的类时，必须默认它会和系统、第三方库、未来版本发生交叉影响。

**第四，命名冲突要提前规避。**

Category 方法没有命名空间。如果两个库都加了 `xxx_viewDidAppear:`，后加载的一方可能覆盖前一方。基础库里更稳的做法，是使用足够明确的前缀，或者直接保存原 IMP，用 C 函数作为新实现，减少 selector 命名冲突。

**第五，`_cmd` 可能改变。**

这一点前面已经提过：交换后，方法体里看到的 `_cmd` 不一定还是原始 selector。如果原方法依赖 `_cmd`，用“互相调用 swizzled selector”的模板就可能埋坑；保存原 IMP 并显式传入原 selector 更可控。

**第六，少碰系统私有 API。**

Runtime 可以看见很多东西，但能看见不代表应该调用。私有 API 有审核风险，也有系统升级后的兼容风险。

**第七，优先保留原实现。**

无论是 `method_setImplementation` 还是 `class_replaceMethod`，如果后续还需要调用旧逻辑，应当提前保存原 IMP。

**第八，调试成本要算进去。**

Swizzling 和消息转发都会让堆栈不再直观：你看到的 selector、实际执行的 IMP、调用栈上的方法名可能不是同一件事。做 SDK 或团队基础设施时，凡是改过 Runtime 行为的地方，都应该写清楚注释、文档和开关。

**第九，应用层不要把 Runtime 当作架构手段。**

Runtime 更适合框架层、基础设施层、调试工具、兼容补丁。业务代码如果大量依赖 Runtime 隐式改行为，长期维护成本会非常高。

# 小结

Part 4 真正想说明的是：Runtime 应用并不是独立于底层机制之外的一套“技巧”，它们都能落回前三篇讲过的结构和流程。

- 动态获取类信息：读类元数据。
- 动态添加方法：改运行期方法列表。
- Method Swizzling：改 `SEL -> IMP` 映射。
- 关联对象：在对象外部挂值。
- 消息转发：改变找不到方法之后的处理路径。
- AOP：可以用 `NSProxy + forwardInvocation:` 做对象级代理增强，也可以用 `_objc_msgForward` 做 Aspects 风格的类级 Hook。
- KVO / Isa Swizzling：改变单个对象的 `isa` 指向。
- 自动归档 / 模型转换：遍历 ivar / property 元数据并读写值。

理解这些应用，关键不是背 API，而是知道每个 API 最终动的是哪一层结构、会影响哪一段消息发送路径。

# 参考方向

- objc4 源码：`objc-runtime-new.mm`、`objc-cache.mm`、`message.h`
- Apple Runtime Reference
- Effective Objective-C 2.0：消息发送、关联对象、Method Swizzling 相关章节
- sunnyxx / halfrost Runtime 实战类文章：组合转发、KVO、Swizzling 风险点
