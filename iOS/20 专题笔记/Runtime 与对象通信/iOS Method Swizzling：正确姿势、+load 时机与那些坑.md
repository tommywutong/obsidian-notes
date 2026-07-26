---
title: 【iOS】Method Swizzling：正确姿势、+load 时机与那些坑
published: 2026-07-27
description: 人人都在抄的 class_addMethod 模板，在"子类先 swizzle、父类后 swizzle"时会把父类的 hook 静默吃掉。而现在的链接器默认合并 category，正好把这个 bug 藏起来了。
tags:
  - iOS
  - Objective-C
  - Runtime
  - Swizzling
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 13
draft: true
---
# Method Swizzling：正确姿势、+load 时机与那些坑

这篇有个前提要先说清楚。

Swizzling 的原理和常规写法，我自己已经写过两遍了。[[Runtime/Swizzling]] 那篇整理了四种写法：裸的 `method_exchangeImplementations`、`+load` + `dispatch_once` + `class_addMethod` 的标准模板、保存原 IMP 的函数指针式、以及 AFNetworking 遍历继承链探测类簇真实类的写法。[[Runtime/应用篇]] 第 3 节把同一套东西接回消息发送机制，多讲了 `_cmd` 被篡改的后果、类簇的坑和五类典型应用场景，第 6 节还讲了 `NSProxy` 和 Aspects 风格两种 AOP 的分野。这两篇里的结论我基本都还认，不重复。

问题是那两篇里没有一行是跑出来的。全是"会污染父类方法表"、"应该在 `+load` 里做"、"要用 `dispatch_once`"这类断言。这次我把它们逐条搬到模拟器上跑，跑出来五个结果和我原来写的不一样：

1. 直接 exchange 一个继承来的方法，后果不是"父类行为被改了"这么温和，是父类和兄弟类当场 `unrecognized selector` 崩溃。
2. 那个人人都在抄的 `class_addMethod` 模板并没有解决全部问题。子类先 swizzle、父类后 swizzle，父类的 hook 会被静默跳过。
3. 现在的链接器（我这台机器上是 `ld-1267`）默认把 category 合并进类，这件事直接改变了 `+load` 的调用顺序，也就改变了上面那个 bug 出不出现。换回 `-Wl,-ld_classic` 结果就不一样。
4. `+initialize` 不能用来做 swizzling 的真正理由，不是流传最广的"可能永远不被调用"，而是它会被调用多次，两次交换正好等于没换。
5. "`+load` 在 `main()` 之前、在主线程调用"这条只对启动期成立。运行期 `dlopen` 能把 `+load` 推到任意子线程上，我跑出来 `pthread_main_np()` 返回 0。

外加两块两篇笔记完全没覆盖的：交换类方法时 `class_addMethod` 必须打在元类上，打错了不报错，静默失效；以及不塞额外 selector 的那条路，把 IMP 直接换成 `_objc_msgForward`。

下面全部有输出。

---

## 一、先把"污染父类"跑成崩溃

三个类。`Parent` 实现 `-greet`，`Child` 和 `Sibling` 都只继承、不重写。只在 `Child` 的 category 里 swizzle 这个继承来的方法：

```objc
@interface Parent : NSObject
- (void)greet;
@end
@implementation Parent
- (void)greet { printf("      >> Parent 的原始 greet 实现\n"); }
@end

@interface Child : Parent @end
@implementation Child @end          // 故意不实现 greet

@interface Sibling : Parent @end
@implementation Sibling @end        // 故意不实现 greet

@interface Child (Hook) @end
@implementation Child (Hook)

+ (void)load {
    Method origM = class_getInstanceMethod(self, @selector(greet));
    Method swzM  = class_getInstanceMethod(self, @selector(hook_greet));
    method_exchangeImplementations(origM, swzM);       // 错误写法
}

- (void)hook_greet {
    printf("      >> [hook] before\n");
    [self hook_greet];
    printf("      >> [hook] after\n");
}
@end
```

跑出来：

```text
[swizzle] 写法：直接 method_exchangeImplementations
  方法表归属（自己的方法列表里有没有 greet）：
    Parent  自己有 greet? 是   IMP=0x104e64a5c
    Child   自己有 greet? 否   IMP=0x104e64a5c
    Sibling 自己有 greet? 否   IMP=0x104e64a5c

  [Parent  greet]
      >> [hook] before
      !! 抛异常：-[Parent hook_greet]: unrecognized selector sent to instance 0x60000000c030
  [Child   greet]
      >> [hook] before
      >> Parent 的原始 greet 实现
      >> [hook] after
  [Sibling greet]
      >> [hook] before
      !! 抛异常：-[Sibling hook_greet]: unrecognized selector sent to instance 0x60000000c030
```

我第一次跑这个实验的时候是准备看"Sibling 的行为被改了"，结果直接吃了两个崩溃。原因很直白：`class_getInstanceMethod(Child, @selector(greet))` 沿继承链找上去，返回的是 `Parent` 的那个 `method_t`。交换之后，`Parent` 的 `greet` 指向了 hook 实现，而 hook 实现第一件事是 `[self hook_greet]` —— `hook_greet` 只挂在 `Child` 上，`Parent` 和 `Sibling` 根本没有这个方法。

所以这个 bug 的真实杀伤力比"父类被污染"大得多。它甚至不是隐蔽的，是当场爆炸。真正隐蔽的情况是 hook 实现不回调原实现，那才会变成一个安静的全局行为改变。

换成 `class_addMethod` 先行的模板，同一份代码：

```text
[swizzle] 写法：class_addMethod 先行
[swizzle] class_addMethod 返回 YES（本类原本没有 greet）
  方法表归属（自己的方法列表里有没有 greet）：
    Parent  自己有 greet? 是   IMP=0x100a2c9b8
    Child   自己有 greet? 是   IMP=0x100a2cb18   ← 新增的
    Sibling 自己有 greet? 否   IMP=0x100a2c9b8

  [Parent  greet]
      >> Parent 的原始 greet 实现
  [Child   greet]
      >> [hook] before
      >> Parent 的原始 greet 实现
      >> [hook] after
  [Sibling greet]
      >> Parent 的原始 greet 实现
```

`Child` 的方法列表里多了一条自己的 `greet`。影响锁死在 `Child` 这一层。

### 这个 YES / NO 是怎么算出来的

objc4 里 `class_addMethod` 和 `class_replaceMethod` 是同一个函数的两个入口：

```cpp
BOOL
class_addMethod(Class cls, SEL name, IMP imp, const char *types)
{
    if (!cls) return NO;
    mutex_locker_t lock(runtimeLock);
    return ! addMethod(cls, name, imp, types ?: "", NO);
}

IMP
class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)
{
    if (!cls) return nil;
    mutex_locker_t lock(runtimeLock);
    return addMethod(cls, name, imp, types ?: "", YES);
}
```

`addMethod` 内部只有一句是关键：

```cpp
    method_t *m;
    if ((m = getMethodNoSuper_nolock(cls, name))) {
        // already exists
        if (!replace) {
            result = m->imp(false);
        } else {
            result = _method_setImplementation(cls, m, imp);
        }
    } else {
        ... 新建一条 method_list_t 挂上去 ...
        result = nil;
    }
```

`getMethodNoSuper_nolock`，名字已经说明一切：只查本类，不上溯继承链。所以 `class_addMethod` 返回 `YES` 严格等价于"本类自己的方法列表里没有这个 selector"，跟父类有没有无关。这正是整个模板的支点。

`class_replaceMethod` 的返回值也是同一套逻辑的产物，实测：

```text
class_getInstanceMethod(C, a) 的 IMP  = 0x1025c48b8  （其实是 P 的）
class_getMethodImplementation(P, a)   = 0x1025c48b8

class_replaceMethod(C, a)  返回 0x0        → NULL（C 原本没有 a，等价于 class_addMethod）
class_replaceMethod(C, b)  返回 0x1025c4910 → 旧 IMP（C 自己的 b）

替换后：
替换实现 _cmd=a
替换实现 _cmd=b
父类 P 有没有被动到：
P.a
P.b

class_addMethod(C, a, ...) 再来一次 = NO（现在 C 自己有了）
```

一个函数两种语义：本类没有就当 `class_addMethod` 用并返回 `NULL`，本类有就当 `method_setImplementation` 用并返回旧 IMP。方式三那套"保存原 IMP 的函数指针"写法之所以要写 `if (!imp) imp = method_getImplementation(method);` 这句兜底，就是在补第一种情况。

记住这个 `NULL`。下一节整篇都在讲它。

---

## 二、经典模板自己也有一个 bug

这一节是全文最有价值的部分，所以放在第二节。

把场景稍微改一下：`AP` 有 `-run`，`AC` 继承它。两边各写一个 category 去 hook 同一个 `run`，两边都老老实实用 `class_addMethod` 模板。唯一的变量是谁先执行。

先让子类先跑：

```objc
classicSwizzle([AC class], @selector(run), @selector(ac_run));   // 子类先
classicSwizzle([AP class], @selector(run), @selector(ap_run));   // 父类后
[[AC new] run];
```

```text
场景一：经典 class_addMethod 模板，子类先 swizzle、父类后 swizzle
  [AC run]:
  子类 hook in
      [AP 原始 run]
  子类 hook out
```

父类那个 hook 一行都没打。它被完整地跳过了，而且没有任何报错。

原因就是上一节那个 `NULL`。子类走的是 `class_addMethod` 返回 `YES` 的分支：

```objc
class_replaceMethod(cls, swzSEL,
                    method_getImplementation(originalMethod),   // ← 就是这里
                    method_getTypeEncoding(originalMethod));
```

`method_getImplementation(originalMethod)` 在这一刻读出父类的 IMP，然后把这个裸函数指针焊死在子类的 `ac_run` 上。之后父类再怎么改自己的方法表，子类手里那份拷贝都不会跟着变。子类调 `[self ac_run]`，走到的是父类被 hook 之前的那个实现。

这不是继承链的问题，是快照的问题。**子类手里那份 IMP 是拷贝，父类之后再被 hook，它看不见。**

### 谁决定了先后顺序

上面那个实验是我手动排的顺序，现实里顺序由 `+load` 决定。把这套东西拆成四个文件重跑：`base.m`（两个类）、`childhook.m`、`parenthook.m`、`main.m`，然后调换链接顺序。

结果第一次跑出来是这样的，两种链接顺序完全一样：

```text
######## 链接顺序：childhook 在前 ########
  +[DParent(Hook) load] 执行 swizzle
  +[DChild(Hook) load] 执行 swizzle

[DChild run]:
  子类 hook in
    父类 hook in
      [DParent 原始 run]
    父类 hook out
  子类 hook out
```

父类总是先跑，bug 不出现。我以为是自己实验设计错了，去翻二进制才发现问题：

```console
$ otool -l m1 | grep sectname | grep -i cat
（空）
```

`__objc_catlist` 和 `__objc_nlcatlist` 两个段都不见了。category 在链接期就被折进类里了。`man ld` 里写得明明白白：

> -no_objc_category_merging
> By default when producing final linked image, the linker will optimize Objective-C classes by merging any categories on a class into the class. Both the class and its categories must be defined in the image being linked for the optimization to occur. Using this option disables that behavior.

这是新链接器的默认行为，我这台机器上是 `ld-1267`。加 `-Wl,-ld_classic` 换回旧链接器，`__objc_catlist` 立刻回来，可以确认这是新链接器带来的变化。`DParent(Hook)` 的 `+load` 被合并成了 `DParent` 这个类自己的 `+load`，于是它不再走 category 的调度路径，而是走 `schedule_class_load` 的父类优先规则。bug 被链接器掩盖掉了。

加上开关关掉合并，`__objc_catlist` 回来了，两种链接顺序立刻分道扬镳：

```text
######## -Wl,-no_objc_category_merging / childhook 在前 ########
  +[DChild(Hook) load] 执行 swizzle
  +[DParent(Hook) load] 执行 swizzle

[DChild run]:
  子类 hook in
      [DParent 原始 run]        ← 父类 hook 消失
  子类 hook out

######## -Wl,-no_objc_category_merging / parenthook 在前 ########
  +[DParent(Hook) load] 执行 swizzle
  +[DChild(Hook) load] 执行 swizzle

[DChild run]:
  子类 hook in
    父类 hook in
      [DParent 原始 run]
    父类 hook out
  子类 hook out
```

调换两个 `.o` 的顺序就调换了 category 的 `+load` 顺序，`+load` 顺序决定这个 bug 出不出现。而这个顺序在真实工程里是 Xcode 的 Compile Sources 列表、CocoaPods 的排序、静态库的合并方式共同决定的，谁都不会去审。

再补一个数据点：同样是 childhook 在前，换成 `-Wl,-ld_classic` 编出来的顺序又变回了父类在先，bug 消失。所以准确的说法是这个顺序由工具链决定，链接器版本、合并开关、目标文件顺序三个变量都能改写它，而它们没有一个是你在写 category 时能看见的。

补一个反例说明合并的边界：如果 `DParent` 自己也写了 `+load`，链接器就没法把 category 的 `+load` 合并进去（一个类只能有一个 `+load`），`__objc_catlist` 就会重新出现，bug 也跟着回来。我实测过这一版，输出和上面关掉开关的第一种一样。

**所以"我们线上跑了三年没出过事"这句话，在这件事上不构成证据。** 它可能只说明你的 category 一直被链接器合并着。

还有一个边界要说清楚：这个优化要求类和 category 在同一个二进制里。你 hook `UIViewController` 那种系统类，永远不会被合并，走的是标准 category 路径。

### 三步之间还有一个窗口

同一个模板还有第二笔隐藏成本，和顺序无关。它的成功路径是两次独立的运行时调用，第一次跑完、第二次还没跑的那一瞬间，方法表长这样：

- `run` → hook 的函数体（`class_addMethod` 刚写进去的）
- `ac_run` → 还是 hook 的函数体（`class_replaceMethod` 还没执行）

此刻调用 `run`，hook 里那句 `[self ac_run]` 会打回它自己。我在两步中间插了一次调用，加了个深度计数器免得真炸栈：

```text
步骤 0（还没动手）：
      原实现 work

步骤 1 完成，class_addMethod 返回 YES。此刻调用 work：
      hook_work（第 1 层）
      hook_work（第 2 层）
      hook_work（第 3 层）
      hook_work（第 4 层）
      hook_work（第 5 层）
      递归深度已到 6，停止

步骤 2 完成。此刻调用 work：
      hook_work（第 1 层）
      原实现 work
```

没有计数器就是栈溢出。

在 `+load` 里这个窗口通常摸不到，业务代码那会儿还没跑。但只要 swizzle 入口被暴露成 `+[SDK start]` 这种可以在任意时刻调用的 API，它就是真的。`dispatch_once` 在这里帮不上忙：它保证 block 只跑一次，不保证 block 跑到一半时别的线程看不到中间状态。

`method_exchangeImplementations` 反而没有这个窗口，两次 `setImp` 都在同一把 `runtimeLock` 里。于是模板陷进一个尴尬：安全的那条路不原子，原子的那条路不安全。保存原 IMP 的写法两头都占——它只有一次运行时调用，而且 `class_replaceMethod` 天生只碰当前类。

### 别人是怎么绕过去的

RSSwizzle 的做法我觉得是这几个库里最漂亮的。它压根不在 swizzle 的时刻去存父类 IMP：

```objc
__block IMP originalIMP = NULL;

RSSWizzleImpProvider originalImpProvider = ^IMP{
    OSSpinLockLock(&lock);
    IMP imp = originalIMP;
    OSSpinLockUnlock(&lock);

    if (NULL == imp){
        // If the class does not implement the method
        // we need to find an implementation in one of the superclasses.
        Class superclass = class_getSuperclass(classToSwizzle);
        imp = method_getImplementation(class_getInstanceMethod(superclass,selector));
    }
    return imp;
};
```

注释里写的是"到父类里找一个实现"，但真正值钱的是这段代码所处的位置：它是一个 block，在每次调用原实现的时候才执行，不是在 swizzle 的时候。`class_replaceMethod` 返回 `NULL` 时 `originalIMP` 保持为空，于是每次都现查一遍父类的当前实现。父类后来被谁 hook 了，这里立刻就能看见。

照这个思路写一份最小实现验证：

```objc
static void lazySwizzle(Class cls, SEL sel, const char *tag) {
    Method m = class_getInstanceMethod(cls, sel);
    const char *types = method_getTypeEncoding(m);

    __block IMP originalIMP = NULL;
    Class swizzledClass = cls;
    IMP (^impProvider)(void) = ^IMP {
        IMP imp = originalIMP;
        if (imp == NULL) {
            Class sup = class_getSuperclass(swizzledClass);
            imp = method_getImplementation(class_getInstanceMethod(sup, sel));
        }
        return imp;
    };

    void (^hook)(id) = ^(id self) {
        printf("  %s hook in\n", tag);
        ((VoidIMP)impProvider())(self, sel);
        printf("  %s hook out\n", tag);
    };
    originalIMP = class_replaceMethod(cls, sel, imp_implementationWithBlock(hook), types);
}
```

同样是子类先、父类后：

```text
场景二：换成调用时才查父类实现（RSSwizzle 的做法），同样是子类先、父类后
  [BC run]:
  子类 hook in
  父类 hook in
      [BP 原始 run]
  父类 hook out
  子类 hook out
```

链子接上了。

Aspects 走的是另一条路，它不修，它禁止。`aspect_isSelectorAllowedAndTrack` 会沿着继承链两个方向扫一遍，只要同一个继承链上已经有人 hook 过这个 selector 就直接报错返回：

```objc
NSString *errorDescription = [NSString stringWithFormat:
    @"Error: %@ already hooked in %@. A method can only be hooked once per class hierarchy.",
    selectorName, NSStringFromClass(currentClass)];
```

它还维护了一个 `AspectTracker`，往上给每个祖先类都打上"我有个子类改了这个方法"的标记，所以父类后来想 hook 也会被拦下。这条规则读起来很霸道，但看完上面那组实验就知道它在防什么。

同一个函数里还有一份黑名单，也值得抄：`retain` / `release` / `autorelease` / `forwardInvocation:` 一律拒绝，`dealloc` 只允许 before 切面。前四个动了会把内存管理和转发机制本身弄坏，最后一个是因为对象已经在析构途中，after 里拿到的 `self` 已经不能用了。

我自己的判断：业务代码里如果你 hook 的是自己项目里的类，先问一句"这个方法是本类实现的还是继承来的"。是本类实现的，经典模板足够安全；是继承来的，要么改成延迟查找，要么干脆别用 category 而是直接重写。基础库另说，基础库应该直接抄 RSSwizzle 的 impProvider 结构。

---

## 三、`+load` 不是消息发送

这一条是后面所有 `+load` 行为的总解释。objc4 里 `+load` 是这么调的：

```cpp
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue;
        (*load_method)(cls, @selector(load));
    }
```

IMP 是在 `add_class_to_loadable_list` 里通过 `cls->getLoadMethod()` 提前取好存进数组的，调用时直接解函数指针，`objc_msgSend` 全程不参与。

`+initialize` 恰好相反：

```cpp
void callInitialize(Class cls)
{
    ((void(*)(Class, SEL))objc_msgSend)(cls, @selector(initialize));
    asm("");
}
```

**一个是函数指针直调，一个是 objc_msgSend。** 两者所有行为差异都是从这一行推出来的：直调不查方法表，所以不存在覆盖，父类的、子类的、每个 category 的 `+load` 各调各的；消息发送要查方法表，所以有覆盖，也有继承。

跑一组来对：

```objc
@interface LP : NSObject @end
@implementation LP
+ (void)load { printf("  +[LP load]\n"); }
@end
@interface LC : LP @end
@implementation LC
+ (void)load { printf("  +[LC load]\n"); }
@end
@interface LG : LC @end
@implementation LG @end                 // 不实现 +load
// LP 有 CatA / CatB 两个分类，LC 有 CatC，都只写 +load
```

```text
  +[LP load]
  +[LC load]
  +[LP(CatA) load]
  +[LP(CatB) load]
  +[LC(CatC) load]
```

三件事：类的 `+load` 全部排在 category 前面；类之间父类优先；`LG` 没实现就一次都不调，它不会继承到 `LC` 的那个。

有了第二节的教训，这里我特意先 `otool -l` 查了一下，`__objc_catlist` 确实在，category 没有被合并掉。原因是 `LP` 和 `LC` 都写了自己的 `+load`，一个类塞不下两个，链接器只好放弃合并。这组输出是真的 category 调度顺序。

父类优先来自 `schedule_class_load` 的递归：

```cpp
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    ASSERT(cls->isRealized());
    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    schedule_class_load(cls->getSuperclass());

    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED);
}
```

先递归安排父类再安排自己，队列天然是父类在前。注意它跨 image 也成立，父类在别的库里也一样先安排。这是 `+load` 时机里唯一被运行时保证的顺序。

类和 category 的先后来自 `call_load_methods` 的循环结构：

```cpp
    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }
        // 2. Call category +loads ONCE
        more_categories = call_category_loads();
    } while (loadable_classes_used > 0  ||  more_categories);
```

类的队列要清空才轮到 category，而且 category 那一轮只跑一次就回头再检查类队列。category 之间的顺序则完全没有规则可言，就是 `__objc_nlcatlist` 这张表里的排列顺序，而这张表是链接器写出来的。

### 一个能直接看见"直调"的实验

在 `main()` 里手动补一句 `[LP load]`：

```text
⑥ 手动再调一次 +[LP load]：
  +[LP(CatB) load]
```

只有 `CatB` 那一份被调到。手写的 `[LP load]` 是一次正常的消息发送，走方法表查找，而 category 的方法会盖住主类的同名方法，两个 category 里后附加的又盖住先附加的，最后只有一个赢家。

启动时那三行 `+[LP load]` / `+[LP(CatA) load]` / `+[LP(CatB) load]` 全都打印过，手动调却只剩一行。同一个 selector，两种调用方式，结果不同。这就是"直调 vs 消息发送"最直接的证据。

`[super load]` 也一起验了：

```text
  +[SP load] 被调用
  +[SC load] 开始，下面调 [super load]
  +[SP load] 被调用
  +[SC load] 结束
```

父类的 `+load` 确实被执行了第二次。如果父类那份里是一段 swizzle，第二次执行正好把它换回去。老文章里"不要在 `+load` 里调 `[super load]`"这条是对的。

---

## 四、`+load` 里能碰什么

这是我这次跑出来最意外的一组。

三个文件：`early.m` 里的 `Early` 在 `+load` 里去访问 `Late`，`late.m` 里的 `Late` 有自己的 `+load` 和 `+initialize`，链接顺序 `early.m` 在前。`Early` 的 `+load` 里还额外 `dlopen` 了一个带 `+load` 的 dylib。

```text
  +[Early load] 开始
    主线程? 是
    NSClassFromString(@"Late") = 0x10279c1c8
  +[Late initialize] 被触发
    [Late ping] = 42   （类已 realize，消息能发）
    但 gLateLoadRan = 0   ← +[Late load] 还没跑
    dlopen 返回 0x2c7620，注意上面有没有出现 +[Plugin load]
  +[Early load] 结束
  +[Late load]  （此刻 gLateLoadRan 置 1）
      +[Plugin load]（来自 dlopen 的 dylib）

---- main ----
  gLateLoadRan = 1
```

三个观察点。

第一，`Late` 这个类是完全可用的。`NSClassFromString` 拿得到，消息发得出去，返回值正确。`_read_images` 阶段就已经把类 realize 好了，`+load` 只是排在后面的一道额外工序。所以"`+load` 里别碰其他类"这句话如果理解成"会崩"，是错的。

第二，`+[Late initialize]` 在 `+[Late load]` 之前就跑了。这是我完全没预料到的顺序。发消息触发了初始化，而它自己的 `+load` 还老老实实排在队列里。一个类的 `+initialize` 早于它自己的 `+load`，听起来违反直觉，但从机制上看很自然。`+load` 是 dyld 通知驱动的批处理，`+initialize` 是消息驱动的懒加载，两条独立的路径。如果你在某个类的 `+load` 里做初始化、又在 `+initialize` 里假设 `+load` 已经跑过，这里就有一个真实的空窗期。

第三，`gLateLoadRan` 读出来是 0。类能用不代表它的 `+load` 跑过。你在 `+load` 里 hook 一个别的库的类，而那个库的 `+load` 里也在 hook 同一个方法，谁先谁后取决于链接顺序，跟第二节是同一个问题。

至于 `dlopen`：它成功返回了句柄，但 `+[Plugin load]` 是等 `+[Early load]` 结束之后才打印的。源码里的原因是这一句：

```cpp
void call_load_methods(void)
{
    static bool loading = NO;
    ...
    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;
```

新 image 的类被 `prepare_load_methods` 排进队列了，但内层的 `call_load_methods` 直接返回，要等最外层那一圈 `do-while` 转到下一轮才真正执行。所以 `dlopen` 返回之后立刻用插件类的 `+load` 副作用，一定拿到空。

### 一个能稳定复现的死锁

`load_images` 是这样加锁的：

```cpp
    recursive_mutex_locker_t lock(loadMethodLock);
    {
        mutex_locker_t lock2(runtimeLock);
        loadAllCategoriesIfNeeded();
        prepare_load_methods(...);
    }
    // Call +load methods (without runtimeLock - re-entrant)
    call_load_methods();
```

`loadMethodLock` 是递归锁，整个 `+load` 执行期间一直被持有。递归锁只对同一个线程免疫，换个线程就是普通互斥锁。所以下面这段是死锁：

```objc
+ (void)load {
    dispatch_semaphore_t sem = dispatch_semaphore_create(0);
    [NSThread detachNewThreadWithBlock:^{
        dlopen("/tmp/swz/libplugin.dylib", RTLD_NOW);   // 子线程要 loadMethodLock
        dispatch_semaphore_signal(sem);
    }];
    dispatch_semaphore_wait(sem, ...);                   // 主线程等子线程
}
```

我把等待时间设成 5 秒好让程序能跑完：

```text
  +[Boom load] 开始，当前持有 loadMethodLock
    子线程：准备 dlopen
  +[Boom load] 等待结果：超时 —— 死锁
      +[Plugin load]（来自 dlopen 的 dylib）
    子线程：dlopen 返回 0x53f620
```

超时之后 `+load` 返回、锁释放，子线程的 `dlopen` 才走通。换成 `DISPATCH_TIME_FOREVER` 就是永久卡死在启动阶段。

现实里没人会这么直白地写。但"`+load` 里同步等一个后台任务"这个形状很常见。同步读配置、同步初始化某个 SDK、`dispatch_sync` 到别的队列，都是。只要那条路径上任何一环碰到镜像加载，就是这个死锁。

### 反过来：`+load` 也可能跑在子线程、跑在 `main()` 之后

上面那个死锁是"`+load` 里等别人"，把方向调过来同样值得跑一次。让 `main()` 先跑起来，再从一个 global queue 上 `dlopen`：

```objc
printf("  main() 已经在跑了\n");
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    printf("  后台线程即将 dlopen\n");
    void *h = dlopen("/tmp/ios-notes-lab/w3d2/libplugin.dylib", RTLD_NOW);
    printf("  dlopen 返回 %p\n", h);
});
```

```text
  main() 已经在跑了
  后台线程即将 dlopen
  [load] Plugin —— 主线程? 否   时间点：main() 之后
  dlopen 返回 0x6ab620 (成功)
  main() 结束
```

`pthread_main_np()` 返回 0。所以"`+load` 在 `main()` 之前、在主线程调用"这两条只在启动期成立，它们描述的是启动期那一批镜像，不是 `+load` 这个方法的性质。加载 Bundle、`NSBundle` 的 `load`、某些 SDK 的延迟初始化，都会把 `+load` 推到运行期的任意线程上。

在 `+load` 里碰 UI、或者假设自己拿得到主线程的 RunLoop，先确认这个类不可能出现在动态加载的镜像里。

我给自己定的规矩是：`+load` 里只允许出现纯计算和对自己这个类的 runtime 操作。不发起 IO，不起线程，不等任何东西，不碰 `dlopen`。要跟别的类打交道，先想清楚你依赖的是它的存在（安全）还是它的 `+load` 副作用（不安全）。

---

## 五、`+initialize` 为什么不能拿来 swizzle

我在 [[Runtime/应用篇]] 里给的理由是"可能永远不被调用"，在 [[Runtime/Swizzling]] 里写的是"可能被子类覆盖导致多次执行"。前者没错但不是最致命的，后者的措辞是错的：出问题的时候子类恰恰什么都没覆盖。

```objc
@implementation KP
+ (void)initialize {
    printf("  +[KP initialize] self=%s → 执行一次 swizzle\n", class_getName(self));
    sw([KP class], @selector(go), @selector(k_go));
}
- (void)go { printf("      [原始 go]\n"); }
- (void)k_go { printf("    hook in\n"); [self k_go]; printf("    hook out\n"); }
@end

@interface KC : KP @end
@implementation KC @end        // 不实现 +initialize
```

```text
① 只碰 KP：
  +[KP initialize] self=KP → 执行一次 swizzle
    hook in
      [原始 go]
    hook out

② 再碰子类 KC（会把 +[KP initialize] 再触发一次）：
  +[KP initialize] self=KC → 执行一次 swizzle
      [原始 go]

③ 回头再看 KP：
      [原始 go]
```

第一次 hook 生效。碰一下子类，`+[KP initialize]` 又被调了一次，`self` 变成 `KC`，第二次交换把方法换了回去，hook 彻底消失，连 `KP` 自己都不再有 hook。

有多少个不实现 `+initialize` 的子类，这段代码就会跑多少遍。奇数次生效，偶数次白干。所以准确的说法是：子类没有实现 `+initialize`，父类那份就会被当作子类的实现再执行一遍，`self` 换成子类。源码里对这个行为有一句注释，还挂着一个很老的 radar 号：

```cpp
    // Send the +initialize message.
    // Note that +initialize is sent to the superclass (again) if
    // this class doesn't implement +initialize. 2157218
```

换成三层继承（`IP` → `IC` → `IG`，只有 `IP` 实现 `+initialize`）再跑一遍：

```text
① 第一次 [IC new]：
  +[IP initialize]  self = IP
  +[IP initialize]  self = IC
② 第二次 [IC new]：
③ 第一次 [IG new]：
  +[IP initialize]  self = IG
④ 直接 [IP new]（前面已经被间接初始化过了）：
```

一份实现被调了三次。运行时不会记住"我调过 `IP` 的这段代码了"，它只记录每个类自己有没有初始化过，而 `IC` 和 `IG` 各算一个类。

还有一个 category 相关的差异，正好和第三节呼应：主类和 category 都写 `+initialize` 时，只有 category 的会执行。

```text
⑤ [JP new]（主类和 category 都写了 +initialize）：
  +[JP(Cat) initialize]（category 实现）self = JP
```

`+load` 两份都跑，`+initialize` 只跑一份。同样是那一行 `objc_msgSend` 的推论。

至于 `dispatch_once` 能不能救 `+initialize` 里的 swizzle：能，但那时候你只是在用 `+initialize` 当一个"第一次被使用时"的触发器，收益仅剩启动期不做无用功，风险是 hook 生效时机不确定。我自己不会这么写。

---

## 六、`dispatch_once` 到底防住了什么

先说一句可能有点冒犯的：**在 `+load` 里写 `dispatch_once`，绝大多数时候是仪式，不是防护。**

`+load` 由 `call_class_loads` / `call_category_loads` 各调一次，每个 `+load` 方法对应一条 `loadable_class` 或 `loadable_category` 记录，调完就从队列里摘掉。运行时本来就只调一次，而且全程在 `loadMethodLock` 保护下串行执行。启动期这批还是在主线程、在 `main()` 之前跑完的（上一节实测过）。这个位置上并发和重入都不存在。

例外是运行期 `dlopen` 进来的镜像，它的 `+load` 在谁调 `dlopen` 就在谁那个线程上跑。但那时候 `loadMethodLock` 还是串行的，多个 `+load` 之间依然不会并发。

真正会让 `+load` 跑两次的只有两种情况：有人手写 `[super load]`（第三节验过），或者有人手写 `[SomeClass load]`。这两种 `dispatch_once` 确实拦得住。

拦不住的是这个：

```objc
@implementation Target3 (Copy1)
+ (void)load {
    static dispatch_once_t t1;
    dispatch_once(&t1, ^{ swizzle(self, @selector(work), @selector(cp_work)); });
}
@end

@implementation Target3 (Copy2)
+ (void)load {
    // 模拟"同一份 hook 代码被打进两个二进制"：另一个编译单元、另一个 onceToken
    static dispatch_once_t t2;
    dispatch_once(&t2, ^{ swizzle(self, @selector(work), @selector(cp_work)); });
}
@end
```

```text
C. 同一个 hook 被两个 onceToken 各执行一次：
      [原始 work]
```

hook 消失了。`dispatch_once` 守的是一个 token 变量，不是"这个方法有没有被 swizzle 过"这件事。同一份 category 代码被静态库和主工程各链一遍、或者被两个 pod 各自 vendored 一份，就会有两个独立的 token，各自都认为自己是第一次。两次交换互相抵消。

这类问题在真实工程里的样子通常是：某个 SDK 的埋点在集成了另一个也 vendored 了同一份代码的 SDK 之后突然全体失效，而两边的日志都显示"swizzle 已执行"。

要做真正的幂等，得把状态挂在被 swizzle 的目标上，而不是挂在执行 swizzle 的代码上。RSSwizzle 的接口就是这么设计的，它要求调用方传一个 `key`，再选一种 mode：

```objc
typedef NS_ENUM(NSUInteger, RSSwizzleMode) {
    RSSwizzleModeAlways = 0,
    RSSwizzleModeOncePerClass = 1,
    RSSwizzleModeOncePerClassAndSuperclasses = 2
};
```

`OncePerClassAndSuperclasses` 会沿继承链往上查一遍有没有人用同一个 key 动过，这同时也是第二节那个问题的另一半答案。

同一批实验里还有一个更难查的变体：

```objc
@implementation Target2 (One)
+ (void)load { swizzle(self, @selector(work), @selector(tw_work)); }
- (void)tw_work { printf("    One-in\n"); [self tw_work]; printf("    One-out\n"); }
@end

@implementation Target2 (Two)
+ (void)load { swizzle(self, @selector(work), @selector(tw_work)); }
- (void)tw_work { printf("    Two-in\n"); [self tw_work]; printf("    Two-out\n"); }
@end
```

```text
B. 两个 category、同一个 tw_work 名字：
      [原始 work]
```

两个 category 用了同一个 `tw_work` 名字。category 之间同名方法后附加的盖住先附加的，两次 swizzle 拿到的是同一对 `method_t`，交换两次等于没换。两个 hook 一个都没生效，编译器一个警告都不给。

把它拆开只留一个 category 做 swizzle，另一个只是碰巧定义了同名方法：

```text
B'. 一个 category swizzle，另一个 category 抢走了同名方法：
    Two-in
      [原始 work]
    Two-out
```

`One` 写的 swizzle，跑起来的是 `Two` 的实现。这就是所有文章都在念叨"给 swizzled 方法加前缀"的具体后果。`af_` 这种前缀不是洁癖。

---

## 七、多个 hook 叠加起来是什么顺序

```objc
@implementation Target (Log)
+ (void)load { swizzle(self, @selector(work), @selector(log_work)); }
- (void)log_work { printf("    Log-in\n"); [self log_work]; printf("    Log-out\n"); }
@end

@implementation Target (Track)
+ (void)load { swizzle(self, @selector(work), @selector(track_work)); }
- (void)track_work { printf("  Track-in\n"); [self track_work]; printf("  Track-out\n"); }
@end
```

```text
  Log 完成 swizzle
  Track 完成 swizzle

A. 两个 category、不同 swizzled selector：
  Track-in
    Log-in
      [原始 work]
    Log-out
  Track-out
```

后 swizzle 的在外层，先 swizzle 的在内层。洋葱结构，后来居上，跟中间件的注册顺序是一个感觉。

前提是 selector 不重名、目标方法是本类自己实现的、每一层都老老实实回调原实现。三个前提有一个不成立，前面几节的坑就轮着来。

这也解释了为什么"不要依赖多个 swizzle 的顺序"这条建议是对的：顺序由工具链决定，而工具链那一侧的变量（Compile Sources 排序、Pod 顺序、链接器版本、合并开关）没有一个会出现在 code review 里。

---

## 八、类方法必须打在元类上

已有两篇的例子清一色是实例方法。类方法有个额外的坑，我第一次动手就掉进去了，而且它的失败方式是所有坑里最安静的一个。

`class_getClassMethod` 拿 `Method` 没问题，但 `class_addMethod` 的第一个参数得换成元类：

```objc
Class cls  = [Sub class];
Class meta = object_getClass(cls);                  // 元类
Method m1 = class_getClassMethod(cls, @selector(factory));
Method m2 = class_getClassMethod(cls, @selector(hook_factory));
class_addMethod(meta, @selector(factory), method_getImplementation(m2), ...);
```

把 `meta` 写成 `cls` 会怎样：

```text
错误写法：class_addMethod 打在 Sub 上
  返回 YES

[Sub  factory] = Base.factory
[Base factory] = Base.factory
```

返回 `YES`，没有报错，hook 完全没生效。因为 `class_addMethod(cls, @selector(factory), ...)` 给 `Sub` 加的是一个实例方法 `-factory`。`Sub` 确实没有同名的实例方法，所以第一节那条规则如实返回了 `YES`。类方法 `+factory` 一动没动，方法表里倒是多了两条永远不会被调到的实例方法。

正确版本对照：

```text
正确写法：class_addMethod 打在 Sub 上
  返回 YES

[Sub  factory] = H(Base.factory)
[Base factory] = Base.factory
```

两段输出里"打在 Sub 上"这句话一模一样，这正是它难查的原因：

```text
Sub  = 0x104f881d0   object_getClass(Sub) = 0x104f881a8 (Sub)
class_isMetaClass(Sub)=0  class_isMetaClass(meta)=1
```

`class_getName` 两边都返回 `Sub`，地址差 40 个字节。日志里打类名根本区分不出你操作的是类还是元类，只有 `class_isMetaClass` 能。类与元类的结构见 [[Runtime/对象与类的本质]]。

而 `class_addMethod` 返回 `YES` 在这里是个假信号。它回答的永远是"这个类自己有没有这个 selector"，第一节就说过；它不会替你检查"这个 selector 是不是该在这一层"。

---

## 九、Swift 下什么能 swizzle

已有两篇完全没有覆盖这块，而现在的项目基本都是混编。

```swift
class Demo: NSObject {
    @objc dynamic func dyn() -> String { return "原始 dyn" }
    @objc          func objcOnly() -> String { return "原始 objcOnly" }
                   func plain() -> String { return "原始 plain" }
}

class PureSwift {                    // 不继承 NSObject
    func hello() -> String { return "原始 hello" }
}
```

用 `imp_implementationWithBlock` + `method_setImplementation` 逐个尝试替换，然后分别从 Swift 侧和 ObjC 侧调用：

```text
① 三种方法在 ObjC 方法列表里的存在情况
   dyn       在 Demo 的方法列表里? true
   objcOnly  在 Demo 的方法列表里? true
   plain     在 Demo 的方法列表里? false
   PureSwift.hello 在方法列表里? false

② 逐个尝试 hook
   plain: class_getInstanceMethod 返回 nil，Runtime 里根本没有这个方法
   hello: class_getInstanceMethod 返回 nil，Runtime 里根本没有这个方法

③ hook 之后，从 Swift 侧直接调用
   d.dyn()       -> !! 被 hook 了 (dyn) !!
   d.objcOnly()  -> 原始 objcOnly
   d.plain()     -> 原始 plain
   PureSwift().hello() -> 原始 hello

④ hook 之后，从 ObjC 侧（perform / objc_msgSend）调用
   perform(dyn) -> !! 被 hook 了 (dyn) !!
   perform(objcOnly) -> !! 被 hook 了 (objcOnly) !!
```

中间那一行是全场最危险的：**`@objc` 而非 `@objc dynamic` 的方法，替换会"成功"，从 ObjC 侧调用也确实走 hook，但 Swift 侧的调用点看不见它。**

`@objc` 只负责把方法暴露给 ObjC 运行时，让它在方法列表里有一份 `method_t`。它不改变 Swift 编译器在 Swift 调用点上的派发策略，那里仍然是 vtable 直派。`dynamic` 才是那个"所有调用都必须过 `objc_msgSend`"的声明。

这个组合最坑的地方在于它一半有效。`class_getInstanceMethod` 不返回 `nil`，`method_setImplementation` 不报错，你写单元测试从 ObjC 侧验证还是通过的。只有真实的 Swift 调用路径静悄悄地绕过去了。如果你在 hook 一个混编项目里的埋点方法，会看到"部分页面有数据、部分页面没有"这种最难查的现象。

`plain` 和 `PureSwift.hello` 反而是好情况：`class_getInstanceMethod` 直接返回 `nil`，一眼就知道不行。

还有一条硬限制在 objc4 里写死了。Swift 类的 category 不允许有 `+load`：

```cpp
        if (cls->isSwiftStable()) {
            _objc_fatal("Category %s on Swift class %s has +load method. Swift "
                        "class extensions and categories on Swift "
                        "classes are not allowed to have +load methods.",
                        cat->name, cls->nameForLogging());
        }
```

这是 `_objc_fatal`，直接终止进程。所以混编项目里"在 `+load` 里 swizzle Swift 类"这条路整个是封死的，只能挪到 `+initialize` 或者显式的启动函数里，然后自己承担第五节讲的那些时机问题。

我的实际做法：需要被 hook 的 Swift 方法，一律显式写 `@objc dynamic`，并且在注释里写清楚"这个方法被 XX 模块 hook，不要去掉 dynamic"。指望别人从 `@objc` 猜出 hook 意图是不现实的。

---

## 十、不交换 selector 的那条路

前面所有坑都来自同一个动作：往方法表里塞一个额外的 selector，再靠"它现在指向原实现"这个约定把球传回去。命名冲突、`_cmd` 被改、`dispatch_once` 抵消，全是这个动作的副产品。

还有一条路不塞 selector。入口是一个很容易被忽略的事实，拿一个类不响应的 selector 分别问两个 API：

```text
   s_work    class_getInstanceMethod        = 0x0          <未知>
             class_getMethodImplementation  = 0x18006b860  _objc_msgForward
```

`class_getInstanceMethod` 返回 `NULL`，`class_getMethodImplementation` 返回的却是 `_objc_msgForward`。这就是"找不到方法就走转发"在方法表这一层的样子。Aspects 那类库把它反过来用：主动把一个存在的方法的 IMP 设成 `_objc_msgForward`，让本来能走通的调用掉进完整转发。

[[Runtime/应用篇]] 第 6.2 节给了伪代码，这里补一份能跑的最小版：

```objc
+ (void)install {
    Method m = class_getInstanceMethod(self, @selector(doJob:));
    // 把原 IMP 备份到一个别名 selector 上
    class_addMethod(self, @selector(aspect_original_doJob:),
                    method_getImplementation(m), method_getTypeEncoding(m));
    // 换掉 forwardInvocation:
    Method myFwd = class_getInstanceMethod(self, @selector(aspect_forwardInvocation:));
    class_addMethod(self, @selector(forwardInvocation:),
                    method_getImplementation(myFwd), method_getTypeEncoding(myFwd));
    // 关键一步
    method_setImplementation(m, _objc_msgForward);
}

- (void)aspect_forwardInvocation:(NSInvocation *)inv {
    printf("  进入 forwardInvocation:  selector = %s\n", sel_getName(inv.selector));
    printf("  before hook\n");
    inv.selector = @selector(aspect_original_doJob:);
    [inv invoke];
    printf("  after hook\n");
}
```

```text
hook 前: 完成 打包

  安装完毕。doJob: 现在的 IMP = 0x18006b860，_objc_msgForward = 0x18006b860，相等: 是

hook 后调用:
  进入 forwardInvocation:  selector = doJob:
  before hook
  after hook
  返回值 = 完成 打包

respondsToSelector: 还是 YES
```

`0x18006b860` 和上面那行输出是同一个地址，两个实验对上了。

比起交换 selector，它换来两样东西。方法列表里不会凭空多出一个假 selector，`respondsToSelector:` 保持 `YES`，内省不说谎；进入 `forwardInvocation:` 时拿到的 `inv.selector` 是真名 `doJob:`，切面层知道自己在 hook 谁。这一点 selector 交换永远做不到，那种写法一进 hook，原始 selector 的身份信息就已经没了。

代价是得自己处理 `NSInvocation` 的参数和返回值，而这件事历史上很麻烦。老一辈 hook 库都要为"返回结构体的方法"单独写一条 `_objc_msgForward_stret` 分支。这条分支今天是死代码，当前 SDK 的 `objc/message.h` 里写着：

```c
OBJC_EXPORT void
_objc_msgForward_stret(void /* id receiver, SEL sel, ... */ ) 
    OBJC_AVAILABLE(10.6, 3.0, 9.0, 1.0, 2.0)
    OBJC_ARM64_UNAVAILABLE;
```

`OBJC_ARM64_UNAVAILABLE` 在 `objc-api.h` 里展开成 `OBJC_UNAVAILABLE("not available in arm64")`。`objc_msgSend_stret` 和 `objc_msgSendSuper_stret` 同样标了它。iOS 从 iPhone 5s 起全是 arm64，所以老文章里那一整段关于 stret 的讨论，对今天的 iOS 已经没有对象了。

最后诚实一点：上面这个最小版仍然有 `_cmd` 问题。`inv.selector` 换成别名再 invoke，原实现看到的 `_cmd` 是别名。这和 [[Runtime/应用篇]] 里分析的方式 A 是同一个毛病，转发式并不自动解决它，只是把它挪到了一个你能看见的地方。

## 十一、几个我不同意的说法

- "用了 `class_addMethod` 模板就安全了。" 第二节整节都在反驳这句。目标方法是继承来的时候，模板把父类当时的 IMP 冻成了裸指针，父类后 swizzle 就断链。
- "category 的 `+load` 顺序由编译顺序决定。" 现在还得加一句前置条件：类和 category 在同一个二进制里时，链接器默认会把 category 合并进类，那个 `+load` 就变成类的 `+load`，走的是父类优先规则。`-Wl,-no_objc_category_merging` 可以关掉。
- "`+initialize` 不适合 swizzle，因为可能永远不被调用。" 说反了重点。它更常见的问题是被调用多次：每个没有实现 `+initialize` 的子类都会把父类那份再触发一遍，两次交换正好抵消。这里也纠正我自己在 [[Runtime/Swizzling]] 里写的"可能被子类覆盖导致多次执行"：真正触发多次的原因恰恰是子类没有覆盖。
- "`+load` 里必须包 `dispatch_once`。" 运行时本来就只调一次。`dispatch_once` 只能拦住手写的 `[super load]` / `[Cls load]`，拦不住"同一份代码被链进两个二进制各带一个 token"这种真实场景。
- "`+load` 里不能访问其他类。" 类是可以正常发消息的，`_read_images` 阶段就 realize 好了。不能依赖的是别的类的 `+load` 副作用，也不能在里面同步等任何跟镜像加载沾边的东西。
- "Swift 里加了 `@objc` 就能 swizzle。" 方法会出现在方法列表里，替换也会成功，但 Swift 侧的调用点仍然是直派，看不到 hook。要 `dynamic`。
- "`method_exchangeImplementations` 不是原子操作，所以要加锁。" 这个函数本身在 `runtimeLock` 保护下完成两次 `setImp`，它是原子的。不原子的是 `class_getInstanceMethod` → `class_addMethod` → `class_replaceMethod` 这个三步序列，每一步各自加锁再释放，中间有窗口，而且那个窗口里调用目标方法会无限递归（第二节有输出）。RSSwizzle 加 `@synchronized` 加的是这个序列，不是单个 API。
- "`+load` 在 `main()` 之前、在主线程调用。" 只对启动期那批镜像成立。运行期 `dlopen` 会在任意线程触发 `+load`，第四节实测 `pthread_main_np()` 返回 0。
- "交换类方法就是把 `class_getInstanceMethod` 换成 `class_getClassMethod`。" 不够。`class_addMethod` 的第一个参数也得从类换成 `object_getClass(cls)`。忘了换不会报错，`class_addMethod` 照样返回 `YES`，只是给类加了两个没人调用的实例方法，hook 静默失效。

---

## 总结

`class_addMethod` 先行解决的是"别改到父类的方法表"，它没解决"父类之后也来 swizzle"。子类先跑就会把父类当时的 IMP 冻结成裸指针，父类的 hook 被静默跳过，而谁先跑取决于工具链。这套模板还附赠一个递归窗口：`class_addMethod` 已经生效、`class_replaceMethod` 还没执行的那一瞬间，调用目标方法就是无限递归。RSSwizzle 用一个调用时才求值的 block 把这两件事一起修掉了，Aspects 干脆禁止同一继承链上 hook 同一个方法两次。

现在的链接器默认把同一二进制内的 category 合并进类，`__objc_catlist` 会整个消失，category 的 `+load` 变成类的 `+load`。旧链接器（`-Wl,-ld_classic`）不这么干。这件事会掩盖上面那个 bug，也意味着"线上跑了很久没出事"不能当作模板正确的证据。

`+load` 是函数指针直调，`+initialize` 是 `objc_msgSend`。父类优先、类先于 category、每个 category 各调一次、手动调只有一个赢家、`+initialize` 会被没实现它的子类反复触发，全部是这一条的推论。

`+load` 里其他类是可用的，但它们的 `+load` 可能还没跑，`dlopen` 进来的镜像的 `+load` 一定还没跑，而任何跨线程等待都可能撞上 `loadMethodLock` 死锁。

Swift 侧只有 `@objc dynamic` 是完整可 swizzle 的。`@objc` 单独用会造成"ObjC 侧生效、Swift 侧不生效"的半失效状态，这是混编项目里最难查的一类问题。

下一篇 [[iOS GCD：队列不是线程，以及死锁的准确边界]]。

## 参考资料

### 一手源码

- [apple-oss-distributions/objc4](https://github.com/apple-oss-distributions/objc4)：`objc-loadmethod.mm` 是 `+load` 调度的全部逻辑，三百多行读得完；`objc-runtime-new.mm` 里 `addMethod` / `class_addMethod` / `class_replaceMethod` / `method_exchangeImplementations` / `schedule_class_load` / `load_images`；`objc-initialize.mm` 里 `callInitialize` 和那句带 radar 号的注释
- `man ld`：`-no_objc_category_merging` 一节，category 合并的官方描述
- 当前 SDK 的 `usr/include/objc/message.h` 与 `objc-api.h`：`_objc_msgForward_stret` / `objc_msgSend_stret` 上的 `OBJC_ARM64_UNAVAILABLE` 标注
- [rabovik/RSSwizzle](https://github.com/rabovik/RSSwizzle)：`RSSwizzle.m` 里的 `originalImpProvider` 是本文第二节那个 bug 的标准解法
- [steipete/Aspects](https://github.com/steipete/Aspects)：`aspect_isSelectorAllowedAndTrack` 里的继承链检查与 `AspectTracker`

### 本地

- [[Runtime/Swizzling]]：四种写法的完整代码，包括 AFNetworking 那套类簇探测
- [[Runtime/应用篇]]：`_cmd` 被篡改的后果、五类应用场景、`NSProxy` 与 Aspects 两种 AOP
- [[Runtime/消息发送与转发]]：`objc_msgSend` 的查找流程，本文所有"改的是 SEL→IMP 映射"的说法都建立在这上面
- [[Runtime/Category：加载、覆盖与关联对象]]：category 附加时机与方法覆盖规则
- [[Runtime/对象与类的本质]]：类与元类的结构，第八节那个"打在类上还是元类上"的背景
- [[dyld]]：`_objc_init` 注册回调到 `notifyObjCInit` 再到 `load_images` 的完整链路
- [[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]]：isa-swizzling 是另一条路，改对象的类而不是类的方法表

---

实验环境：Xcode 26.6（Apple clang 21，`ld-1267`，Swift 6.3.3），iOS 模拟器 iPhone 16 / iOS 18.3（arm64，Apple Silicon Mac），`clang -fobjc-arc -target arm64-apple-ios17.0-simulator`，`xcrun simctl spawn booted` 真跑。

链接器 category 合并那组实验依赖当前 ld 的默认行为，属于工具链实现细节，换 Xcode 版本需要重新验证。验证方法很轻：`otool -l <binary> | grep sectname | grep -i cat`，看 `__objc_catlist` 在不在。

> 待真机补测：`+load` 与 `dlopen` 那组时序、以及 `loadMethodLock` 死锁在真机 iOS 26 上是否一致。代码原样拿到真机跑即可，唯一要改的是 dylib 的绝对路径。
>
> 待真机补测：Swift 那组结论来自 `swiftc -target arm64-apple-ios17.0-simulator` 的 Debug 构建。`@objc` 非 `dynamic` 在 Release + 全模块优化下会不会被直接内联掉（那样连"半失效"都算不上，是彻底看不见），需要在真机 Release 包上复现一次。
