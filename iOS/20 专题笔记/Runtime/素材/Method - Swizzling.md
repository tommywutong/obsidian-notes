# Method Swizzling

Method Swizzling 是利用 Objective-C Runtime 交换两个方法 IMP（实现指针）的技术，常用于 Hook 系统方法、埋点、AOP 等场景。

**核心原理**：ObjC 方法调用本质是消息发送 `objc_msgSend(receiver, selector)`，Runtime 通过 selector 在类的方法列表里查找对应的 IMP（函数指针）再执行。Swizzling 的本质就是修改这张"selector → IMP"的映射表，让某个 selector 指向另一个函数实现。

---

## 方式一：简单交换（直接 method_exchangeImplementations）

### 代码

```objc
#import "ViewController.h"
#import <objc/runtime.h>

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self SwizzlingMethod];
    [self originalFunction];   // 输出：swizzledFunction
    [self swizzledFunction];   // 输出：originalFunction
}

- (void)SwizzlingMethod {
    Class class = [self class];
    SEL originalSelector = @selector(originalFunction);
    SEL swizzledSelector = @selector(swizzledFunction);
    Method originalMethod = class_getInstanceMethod(class, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
    method_exchangeImplementations(originalMethod, swizzledMethod);
}

- (void)originalFunction {
    NSLog(@"originalFunction");
}

- (void)swizzledFunction {
    NSLog(@"swizzledFunction");
}

@end
```

### IMP 状态变化

```
交换前：
  originalFunction selector → IMP_A（打印 "originalFunction"）
  swizzledFunction selector → IMP_B（打印 "swizzledFunction"）

交换后：
  originalFunction selector → IMP_B（打印 "swizzledFunction"）
  swizzledFunction selector → IMP_A（打印 "originalFunction"）
```

`method_exchangeImplementations` 底层等价于：

```objc
IMP impA = method_getImplementation(originalMethod);
IMP impB = method_getImplementation(swizzledMethod);
method_setImplementation(originalMethod, impB);
method_setImplementation(swizzledMethod, impA);
```

适合在自己定义的类内部做演示，两个方法都在当前类中，不涉及继承问题。

**缺陷**：
- 没有 `dispatch_once` 保护，重复调用会把 IMP 换回去，效果消失
- 没有处理"方法来自父类"的情况，直接 swizzle 会修改父类方法表，影响所有子类
- 没有在 `+load` 里执行，时机不可控

---

## 方式二：安全交换（class_addMethod + method_exchangeImplementations）

### 代码

```objc
#import "UIViewController+Swizzling.h"
#import <objc/runtime.h>

@implementation UIViewController (Swizzling)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        SEL originalSelector = @selector(originalFunction);
        SEL swizzledSelector = @selector(swizzledFunction);
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        BOOL didAddMethod = class_addMethod(class,
                                            originalSelector,
                                            method_getImplementation(swizzledMethod),
                                            method_getTypeEncoding(swizzledMethod));
        if (didAddMethod) {
            // 当前类没有自己的 originalFunction（继承自父类）
            // class_addMethod 已把 originalSelector 注册到当前类，IMP 为 swizzledMethod 的 IMP
            // 现在把 swizzledSelector 的 IMP 改成原始方法的 IMP，完成交换
            class_replaceMethod(class,
                                swizzledSelector,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        } else {
            // 当前类自己有 originalFunction，直接交换
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

- (void)originalFunction {
    NSLog(@"originalFunction");
}

- (void)swizzledFunction {
    NSLog(@"swizzledFunction");
}

@end
```

### 为什么需要 class_addMethod 这一步

假设 `UIViewController` 的父类 `UIResponder` 实现了 `originalFunction`，而 `UIViewController` 自己没有：

- 直接 `method_exchangeImplementations` → 会修改 `UIResponder` 的方法表 → 所有继承自 `UIResponder` 的类都被影响，这不是我们想要的
- 先 `class_addMethod` → 在 `UIViewController` 自己的方法表里新增一条记录 → 再交换 → 只影响 `UIViewController` 及其子类，父类不受影响

### +load 与 +initialize 的区别

- `+load`：镜像加载时调用，早于 `main()`，每个类只调用一次，不会被子类覆盖，适合做 swizzling
- `+initialize`：第一次向类发送消息时调用，可能被子类覆盖导致多次执行，不适合做 swizzling

**结论**：swizzling 必须在 `+load` 里做，配合 `dispatch_once` 是工程标准写法。

---

## 方式三：函数指针式（class_replaceMethod + C 函数指针）

### 代码

```objc
#import "UIViewController+PointerSwizzling.h"
#import <objc/runtime.h>

typedef IMP *IMPPointer;

// 全局函数指针，存储原始方法的 IMP
static void (*MethodOriginal)(id self, SEL _cmd, id arg1);

// 替换方法：C 函数，签名与 ObjC 方法一致（id self, SEL _cmd, 参数...）
static void MethodSwizzle(id self, SEL _cmd, id arg1) {
    NSLog(@"swizzledFunc");
    MethodOriginal(self, _cmd, arg1);  // 直接调用原始 IMP，不走消息发送
}

BOOL class_swizzleMethodAndStore(Class class, SEL original, IMP replacement, IMPPointer store) {
    IMP imp = NULL;
    Method method = class_getInstanceMethod(class, original);
    if (method) {
        const char *type = method_getTypeEncoding(method);
        // class_replaceMethod：替换 original 的 IMP 为 replacement，返回旧 IMP
        imp = class_replaceMethod(class, original, replacement, type);
        if (!imp) {
            // 返回 NULL 说明当前类没有自己的实现（继承来的），从 method 结构体里直接取父类 IMP
            imp = method_getImplementation(method);
        }
    }
    if (imp && store) { *store = imp; }  // 把原始 IMP 写入函数指针变量
    return (imp != NULL);
}

@implementation UIViewController (PointerSwizzling)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        [self swizzle:@selector(originalFunc) with:(IMP)MethodSwizzle store:(IMP *)&MethodOriginal];
    });
}

+ (BOOL)swizzle:(SEL)original with:(IMP)replacement store:(IMPPointer)store {
    return class_swizzleMethodAndStore(self, original, replacement, store);
}

- (void)originalFunc {
    NSLog(@"originalFunc");
}

@end
```

### 与方式二的核心区别

方式二调用原始实现的方式：

```objc
[self swizzledFunction];
// 实质：objc_msgSend(self, @selector(swizzledFunction))
// 靠 SEL 指向的 IMP 已经换成原始实现这一"潜规则"来工作
// 阅读代码时容易误解为递归
```

方式三调用原始实现的方式：

```objc
MethodOriginal(self, _cmd, arg1);
// 直接调用函数指针，绕过 objc_msgSend，意图清晰
// 性能略高（省去消息查找开销）
```

### 注意事项

`MethodOriginal` 是全局静态变量。如果同一个 SEL 在多处被 swizzle，后一次会覆盖前一次存入的 IMP，导致调用链断裂。每个需要 swizzle 的方法必须对应一个独立的全局指针变量。这也是 RSSwizzle 用 block 捕获 IMP 而非全局变量的原因。

---

## 方式四：AFNetworking 实战写法（遍历继承链 + 动态探测）

### 代码

```objc
static inline void af_swizzleSelector(Class theClass, SEL originalSelector, SEL swizzledSelector) {
    Method originalMethod = class_getInstanceMethod(theClass, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(theClass, swizzledSelector);
    method_exchangeImplementations(originalMethod, swizzledMethod);
}

static inline BOOL af_addMethod(Class theClass, SEL selector, Method method) {
    return class_addMethod(theClass, selector,
                           method_getImplementation(method),
                           method_getTypeEncoding(method));
}

@implementation _AFURLSessionTaskSwizzling

+ (void)load {
    if (NSClassFromString(@"NSURLSessionTask")) {
        NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
        NSURLSession *session = [NSURLSession sessionWithConfiguration:configuration];
        // 创建真实 task，通过它拿到系统私有子类的 Class
        NSURLSessionDataTask *localDataTask = [session dataTaskWithURL:nil];
        IMP originalAFResumeIMP = method_getImplementation(class_getInstanceMethod([self class], @selector(af_resume)));
        Class currentClass = [localDataTask class];

        while (class_getInstanceMethod(currentClass, @selector(resume))) {
            Class superClass = [currentClass superclass];
            IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));
            IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));
            // 条件一：当前类自己重写了 resume（不是纯继承父类）
            // 条件二：当前类的 resume 还没被 AF swizzle 过
            if (classResumeIMP != superclassResumeIMP &&
                originalAFResumeIMP != classResumeIMP) {
                [self swizzleResumeAndSuspendMethodForClass:currentClass];
            }
            currentClass = [currentClass superclass];
        }

        [localDataTask cancel];
        [session finishTasksAndInvalidate];
    }
}

+ (void)swizzleResumeAndSuspendMethodForClass:(Class)theClass {
    Method afResumeMethod = class_getInstanceMethod(self, @selector(af_resume));
    Method afSuspendMethod = class_getInstanceMethod(self, @selector(af_suspend));

    if (af_addMethod(theClass, @selector(af_resume), afResumeMethod)) {
        af_swizzleSelector(theClass, @selector(resume), @selector(af_resume));
    }
    if (af_addMethod(theClass, @selector(af_suspend), afSuspendMethod)) {
        af_swizzleSelector(theClass, @selector(suspend), @selector(af_suspend));
    }
}

- (void)af_resume {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    [self af_resume]; // 交换后，af_resume SEL → 原始 resume IMP，不递归
    if (state != NSURLSessionTaskStateRunning) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidResumeNotification object:self];
    }
}

- (void)af_suspend {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    [self af_suspend];
    if (state != NSURLSessionTaskStateSuspended) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidSuspendNotification object:self];
    }
}

@end
```

### 为什么这里要遍历继承链

`NSURLSessionTask` 是类簇，内部继承结构大致如下（随 iOS 版本变化）：

```
NSObject
  └── NSURLSessionTask
        └── __NSCFURLSessionTask
              └── __NSCFLocalDataTask   ← 你实际拿到的对象
```

每一层可能都有自己的 `resume` 实现。如果只 swizzle 最底层的私有类，而上层某个类的 `resume` 实现里没有调 `[super resume]`，那 hook 就会失效。所以 AF 对继承链上每个"自己有 resume 实现"的类都独立做了 swizzle，保证全覆盖。

### 两个 IMP 判断条件的深层含义

```objc
classResumeIMP != superclassResumeIMP
```
过滤掉"只继承父类实现、没有自己重写"的中间层。这些层做 swizzle 没有意义，而且可能引发问题。

```objc
originalAFResumeIMP != classResumeIMP
```
防止重入。如果某次循环里这个类已经被 swizzle 过了（比如继承链上有两个类共享了同一个 IMP），不重复操作。

---

## 三方框架对比

| 框架 | 核心方法 | 线程安全 | 继承处理 | 多次 swizzle 同一方法 |
|------|----------|----------|----------|----------------------|
| JRSwizzle | `method_exchangeImplementations` | 否 | class_addMethod 前置处理 | 全局 SEL，有冲突风险 |
| RSSwizzle | `class_replaceMethod` | 是（加锁） | 更精确 | block 捕获 IMP，各自独立 |

---

**Swizzling 的本质只有一件事**：改变 selector 到 IMP 的映射。所有写法的差异，都是在解决"怎么更安全地改"这个问题。

**四种写法的演进逻辑**：

方式一是最裸露的实现，展示原理。方式二在此基础上加了三道防护：`+load` 保证时机、`dispatch_once` 保证幂等、`class_addMethod` 保证不污染父类，这三点缺一不可，是工程实践的基本要求。方式三用函数指针替换 ObjC SEL 间接调用，意图更明确，适合对性能或可读性有更高要求的场景。方式四是在前两种思路之上，针对"目标类未知"这个现实问题加了运行时探测层，是工程复杂度最高但也最完备的写法。

**使用 Swizzling 时需要注意的风险**：

- 命名冲突：swizzled 方法名如果和其他 Category 重名，会静默覆盖，难以排查。前缀（如 `af_`）是必要习惯

    1. 应该只在 `+load` 中执行 Method Swizzling。
	程序在启动的时候，会先加载所有的类，这时会调用每个类的 `+load` 方法。而且在整个程序运行周期只会调用一次（不包括外部显示调用)。所以在 `+load` 方法进行 Method Swizzling 再好不过了。
	
	而为什么不用 `+initialize` 方法呢。
	
	因为 `+initialize` 方法的调用时机是在 第一次向该类发送第一个消息的时候才会被调用。如果该类只是引用，没有调用，则不会执行 `+initialize` 方法。  
	Method Swizzling 影响的是全局状态，`+load` 方法能保证在加载类的时候就进行交换，保证交换结果。而使用 `+initialize` 方法则不能保证这一点，有可能在使用的时候起不到交换方法的作用。

> 2. Method Swizzling 在 `+load` 中执行时，不要调用 `[super load];`。

上边我们说了，程序在启动的时候，会先加载所有的类。如果在 `+ (void)load`方法中调用 `[super load]` 方法，就会导致父类的 `Method Swizzling` 被重复执行两次，而方法交换也被执行了两次，相当于互换了一次方法之后，第二次又换回去了，从而使得父类的 `Method Swizzling` 失效。

> 3. Method Swizzling 应该总是在 `dispatch_once` 中执行。

Method Swizzling 不是原子操作，`dispatch_once` 可以保证即使在不同的线程中也能确保代码只执行一次。所以，我们应该总是在 `dispatch_once` 中执行 Method Swizzling 操作，保证方法替换只被执行一次。

> 4. 使用 Method Swizzling 后要记得调用原生方法的实现。

在交换方法实现后记得要调用原生方法的实现（除非你非常确定可以不用调用原生方法的实现）：APIs 提供了输入输出的规则，而在输入输出中间的方法实现就是一个看不见的黑盒。交换了方法实现并且一些回调方法不会调用原生方法的实现这可能会造成底层实现的崩溃。

> 5. 避免命名冲突和参数 `_cmd` 被篡改。

1. 避免命名冲突一个比较好的做法是为替换的方法加个前缀以区别原生方法。一定要确保调用了原生方法的所有地方不会因为自己交换了方法的实现而出现意料不到的结果。  
    在使用 Method Swizzling 交换方法后记得要在交换方法中调用原生方法的实现。在交换了方法后并且不调用原生方法的实现可能会造成底层实现的崩溃。
    
2. 避免方法命名冲突另一个更好的做法是使用函数指针，也就是上边提到的 **方案 B**，这种方案能有效避免方法命名冲突和参数 `_cmd` 被篡改。
    

> 6. 谨慎对待 Method Swizzling。

使用 Method Swizzling，会改变非自己拥有的代码。我们使用 Method Swizzling 通常会更改一些系统框架的对象方法，或是类方法。我们改变的不只是一个对象实例，而是改变了项目中所有的该类的对象实例，以及所有子类的对象实例。所以，在使用 Method Swizzling 的时候，应该保持足够的谨慎。

例如，你在一个类中重写一个方法，并且不调用 super 方法，则可能会出现问题。在大多数情况下，super 方法是期望被调用的（除非有特殊说明）。如果你是用同样的思想来进行 Method Swizzling ，可能就会引起很多问题。如果你不调用原始的方法实现，那么你 Method Swizzling 改变的越多代码就越不安全。

> 7. 对于 Method Swizzling 来说，**调用顺序** 很重要。

`+ load` 方法的调用规则为：

1. 先调用主类，按照编译顺序，顺序地根据继承关系由父类向子类调用；
2. 再调用分类，按照编译顺序，依次调用；
3. `+ load` 方法除非主动调用，否则只会调用一次。

这样的调用规则导致了 `+ load` 方法调用顺序并不一定确定。一个顺序可能是：`父类 -> 子类 -> 父类类别 -> 子类类别`，也可能是 `父类 -> 子类 -> 子类类别 -> 父类类别`。所以 Method Swizzling 的顺序不能保证，那么就不能保证 Method Swizzling 后方法的调用顺序是正确的。

所以被用于 Method Swizzling 的方法必须是当前类自身的方法，如果把继承父类来的 IMP 复制到自身上面可能会存在问题。如果 `+ load` 方法调用顺序为：`父类 -> 子类 -> 父类类别 -> 子类类别`，那么造成的影响就是调用子类的替换方法并不能正确调起父类分类的替换方法。原因解释可以参考这篇文章：[南栀倾寒：iOS界的毒瘤-MethodSwizzling](https://www.jianshu.com/p/19c5736c5d9a)

关于调用顺序更细致的研究可以参考这篇博文：[玉令天下的博客：Objective-C Method Swizzling](http://yulingtianxia.com/blog/2017/04/17/Objective-C-Method-Swizzling/)