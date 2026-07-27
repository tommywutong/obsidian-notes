---
title: 【iOS】YYModel 源码：为什么比 JSONModel 快
published: 2026-07-27
description: 实测 14 倍。把这 14 倍拆开之后发现，被写烂了的「objc_msgSend 强转函数指针」只值 1.6 倍，而几乎没人提的键映射时机占了将近一半。顺便测出一个从 2015 年留到现在的 int 属性赋值 bug。
tags:
  - iOS
  - Objective-C
  - Runtime
  - YYModel
  - 性能
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 33
draft: true
---
# YYModel 源码：为什么比 JSONModel 快

搜「YYModel 为什么快」，能翻到几十篇文章，结论高度一致：做了缓存，用了 `objc_msgSend` 强转函数指针，用了 CoreFoundation 遍历。这些都对。但它们是并列摆着的，没人说哪一条值多少。

我把它们一条条关掉测了一遍，结果和排序直觉正好相反。**把强转函数指针换回 `setValue:forKey:`，YYModel 只会慢 1.6 倍；而键映射这一件事，就占掉了 15 倍里的将近一半。** 那个被反复引用的技巧，是四条里贡献最小的。

先摆数字。同一份 GitHub user JSON（28 个 key），同一个 30 属性的 model，同一个进程里各解析 10000 次，21 轮取中位数。YYModel 26.4 ms，JSONModel 384 ms，14.5 倍。换成一条完整的微博 JSON（580 行，嵌套、容器、日期都有），1000 次：YYModel 124 ms，JSONModel 724 ms，5.8 倍。同一对库，同一台机器，倍数差了 2.5 倍。workload 一换，「快一个数量级」这个说法就站不住了。

JSONModel 自己的实现细节归 [[iOS JSONModel 源码：Runtime 驱动的属性映射]]，这篇只借它当参照物。

---

## 一、先说我第一次测错的地方

我最早的写法是这样：

```objc
begin = now_ms();
@autoreleasepool {
    for (int i = 0; i < 10000; i++) { [[JSGHUser alloc] initWithDictionary:json error:nil]; }
}
end = now_ms();
```

跑三次，JSONModel 的中位数分别是 441 ms、791 ms、553 ms，最大值 1315 ms。倍数在 15.5x 和 22.5x 之间乱跳。YYModel 那边倒是稳的。

问题出在 `@autoreleasepool` 的位置。池套在循环外面，10000 个 model 全挂在同一个池里，直到循环结束才排空。JSONModel 每次 init 还要额外造一堆临时的 `NSSet`／`NSArray`／`NSString`，也一样挂在那里。YYModel 每次只留一个对象，JSONModel 每次留几十个，于是内存压力和 malloc 行为完全不同。我测的有一部分是「谁更能撑爆自动释放池」。

把池挪进循环：

```objc
for (int i = 0; i < N; i++) @autoreleasepool {
    [[JSGHUser alloc] initWithDictionary:json error:nil];
}
```

空池循环本身的开销先量一遍：10000 次 push/pop 是 0.07 ms，相对两边的量级可以忽略。改完之后三次跑出 14.10x、14.55x、15.80x，min 和 max 也收拢到了 10% 以内。后面所有数字都是这个写法。

这个坑值得单独说，因为它偏袒的方向刚好和结论一致：分配越多的库越吃亏，而 JSONModel 恰好分配得多。仪器和结论同向的时候最危险。

> 实验环境：macOS 26.5.2，Apple Silicon（arm64），Apple clang 21.0.0，`clang -fobjc-arc -O2 -framework Foundation`，YYModel 与 JSONModel 的 `.m` 直接编进同一个命令行程序。没开模拟器。

---

## 二、把 14 倍拆成四笔账

### 2.1 键映射：一次，还是每个属性每次两遍

这是最大的一笔，也是文章里最少被提的一笔。

要把它单独量出来，得让两边除了「有没有键映射」之外完全一样。我的做法是准备两份 JSON。一份是原始的 snake_case，配一个声明了自定义映射的 model。另一份把 key 全部改名成属性名，配一个不声明任何映射的 model。属性数一样是 30，key 数一样是 28，值完全相同。

```text
=== same 30 properties, same 28 keys, 10000 iterations, median of 15 (ms) ===
  YYModel   with custom mapper :   26.73      without :   21.82   (mapping costs  +4.91)
  JSONModel with keyMapper     :  401.81      without :  166.99   (mapping costs +234.82)
  ratio with mapping 15.0x   ratio without mapping 7.7x
```

加一层键映射，YYModel 多花 4.9 ms，JSONModel 多花 235 ms。**15 倍里有 1.95 倍纯粹来自这一件事。**

原因在源码里一眼可见。YYModel 把映射关系在建 meta 的时候就烧进了 `_mapper` 这个字典，key 直接就是 JSON 里的 key（`NSObject+YYModel.m:551-610`）：

```objc
[customMapper enumerateKeysAndObjectsUsingBlock:^(NSString *propertyName, NSString *mappedToKey, BOOL *stop) {
    _YYModelPropertyMeta *propertyMeta = allPropertyMetas[propertyName];
    if (!propertyMeta) return;
    [allPropertyMetas removeObjectForKey:propertyName];

    if ([mappedToKey isKindOfClass:[NSString class]]) {
        if (mappedToKey.length == 0) return;

        propertyMeta->_mappedToKey = mappedToKey;
        NSArray *keyPath = [mappedToKey componentsSeparatedByString:@"."];
```

`componentsSeparatedByString:` 这种字符串操作只在这里出现一次，此后每次解析都是拿 JSON 的 key 去查一次哈希表。JSONModel 走的是另一条路：属性名先经过 `__mapString:withKeyMapper:` 转成 JSON key，再拿去查字典，而这个转换每个属性每次 init 要做两遍。一遍在校验（`JSONModel.m:215`），一遍在导入（`JSONModel.m:278`），中间没有任何记忆。

YYModel 那 +4.9 ms 我没能定位到具体来源。它稳定为正，量级在 20%~35%，我猜和 snake_case key 的哈希与比较成本有关，但没有证据，所以只说到这里。

### 2.2 元数据缓存：值 49 倍，但只值一次

`_YYModelMeta` 按 Class 缓存在一个全局 `CFMutableDictionaryRef` 里。互斥用的是 `dispatch_semaphore`，不是 `@synchronized`，也不是 GCD 队列（`NSObject+YYModel.m:628-649`）：

```objc
+ (instancetype)metaWithClass:(Class)cls {
    if (!cls) return nil;
    static CFMutableDictionaryRef cache;
    static dispatch_once_t onceToken;
    static dispatch_semaphore_t lock;
    dispatch_once(&onceToken, ^{
        cache = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
        lock = dispatch_semaphore_create(1);
    });
    dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
    _YYModelMeta *meta = CFDictionaryGetValue(cache, (__bridge const void *)(cls));
    dispatch_semaphore_signal(lock);
    if (!meta || meta->_classInfo.needUpdate) {
        meta = [[_YYModelMeta alloc] initWithClass:cls];
```

`YYClassInfo.m:329-355` 里 `+classInfoWithClass:` 是同一个模式，两张表（class 和 metaclass）共用一把信号量。

要测「第一次 vs 第二次」，只测一个类是不够的，第二次的数字会被各种一次性初始化污染。我用宏生成了 40 个结构完全相同、名字不同的类，先各解析一次，再各解析一次：

```text
=== metadata cache: cold vs warm (40 distinct classes, 30 properties each) ===
  1st parse of a class : median   0.1644 ms  (min 0.1578 max 1.2930)
  2nd parse of a class : median   0.0034 ms  (min 0.0032 max 0.0040)
  cold/warm            : 48.7x   absolute cost of building meta ~= 0.1610 ms
```

首次 0.16 ms，稳态 3.4 µs，差了 49 倍。按 1% 的阈值算，同一个类解析大约 4800 次之后，这笔建表开销才摊薄到可以忽略。

一个 App 冷启动时要解析的 model 类可能有几十上百个，每个 0.16 ms，加起来是十几毫秒的一次性成本。这笔钱花得值不值，取决于你解析多少次，而不取决于缓存本身有多快。

这里要纠正一个我原本打算分开写的点。**「类型编码预解析」不是独立的第四条优化，它就在这 0.16 ms 里面。** `YYEncodingGetType()` 把 `T@"NSString"` 这样的字符串变成一个枚举（`YYClassInfo.m:60-91`）：

```c
    switch (*type) {
        case 'v': return YYEncodingTypeVoid | qualifier;
        case 'B': return YYEncodingTypeBool | qualifier;
        case 'c': return YYEncodingTypeInt8 | qualifier;
```

整个建表过程里最贵的是带 `NSScanner` 的那段（`YYClassInfo.m:172-193`），它负责从 `@"NSArray<Foo>"` 里抠出类名和协议名。这些工作全部发生在 meta 构造期，运行时只剩 `switch (meta->_type & YYEncodingTypeMask)`。所以它省下来的时间，已经算在缓存那 49 倍里了，不能再算第二遍。JSONModel 在这一点上做得同样正确，它的 `NSScanner` 解析也只在类扫描时跑一次。

### 2.3 赋值机制：最有名的一招，最小的一笔

这就是被抄了几十遍的那段代码（`NSObject+YYModel.m:717-773`）：

```objc
static force_inline void ModelSetNumberToProperty(__unsafe_unretained id model,
                                                  __unsafe_unretained NSNumber *num,
                                                  __unsafe_unretained _YYModelPropertyMeta *meta) {
    switch (meta->_type & YYEncodingTypeMask) {
        case YYEncodingTypeBool: {
            ((void (*)(id, SEL, bool))(void *) objc_msgSend)((id)model, meta->_setter, num.boolValue);
        } break;
        case YYEncodingTypeInt8: {
            ((void (*)(id, SEL, int8_t))(void *) objc_msgSend)((id)model, meta->_setter, (int8_t)num.charValue);
        } break;
```

对象属性走的是同一套（`NSObject+YYModel.m:800`）：

```objc
                            ((void (*)(id, SEL, id))(void *) objc_msgSend)((id)model, meta->_setter, value);
```

单独测这一步。同一个对象、同一个值，一边 `setValue:forKey:`，一边强转函数指针调 setter，10000 次，11 轮中位数：

```text
=== assignment: -setValue:forKey: vs objc_msgSend cast to the setter signature ===
  object property  KVC :   0.512 ms / 10000   |  msgSend cast :   0.103 ms   -> 4.9x
  uint32 property  KVC :   0.655 ms / 10000   |  msgSend cast :   0.050 ms   -> 13.1x
```

单看倍数很吓人：对象属性 4.9 倍，数值属性 13 倍。但换算成绝对时间就凉了。一次 KVC 赋值 51 ns，一次强转调用 10 ns，一个属性省 41 ns。30 个属性、10000 次，一共省 13 ms。

而 YYModel 无映射版本总共花了 21.8 ms。也就是说，如果 ibireme 当年老老实实用 KVC，这个 case 会变成大约 35 ms，仍然比 JSONModel 的 167 ms 快 4.8 倍。这一招值 1.6 倍。

数值属性那 13 倍看着最唬人，实际最不值钱：一个典型 model 里数值属性只有几个，剩下全是对象属性。

### 2.4 遍历谁：属性表和 JSON，哪边短走哪边

`yy_modelSetWithDictionary:` 的分叉只有一行（`NSObject+YYModel.m:1496`）：

```objc
    if (modelMeta->_keyMappedCount >= CFDictionaryGetCount((CFDictionaryRef)dic)) {
        CFDictionaryApplyFunction((CFDictionaryRef)dic, ModelSetWithDictionaryFunction, &context);
```

属性数 ≥ key 数就遍历字典，否则遍历属性表。回调函数是纯 C 的（`NSObject+YYModel.m:1114-1125`）：

```objc
static void ModelSetWithDictionaryFunction(const void *_key, const void *_value, void *_context) {
    ModelSetContext *context = _context;
    __unsafe_unretained _YYModelMeta *meta = (__bridge _YYModelMeta *)(context->modelMeta);
    __unsafe_unretained _YYModelPropertyMeta *propertyMeta = [meta->_mapper objectForKey:(__bridge id)(_key)];
```

这里的 `__unsafe_unretained` 不是风格问题。同样一段读字典的循环，把结果存进 strong 局部变量比存进 `__unsafe_unretained` 慢 1.3 ms/10 万次。ARC 在每次赋值上插的那对 retain/release 是能被量出来的。

分叉的效果，用一个 12 属性的 model 配两份 JSON 来量：一份正好 12 个 key，一份是这 12 个 key 加 100 个用不上的 key。

```text
=== 12-property model, 10000 iterations, median of 15 ===
  json with  12 keys : YYModel    9.37 ms   JSONModel   83.75 ms
  json with 112 keys : YYModel   11.05 ms   JSONModel  101.58 ms
  100 extra junk keys cost YYModel +1.68 ms, JSONModel +17.83 ms
```

多出来的 100 个 key，YYModel 每次解析多花 168 ns，JSONModel 多花 1.78 µs。这个场景在真实项目里非常常见：服务端返回一个大对象，客户端只关心其中十来个字段。

反过来的场景一样有效。一个只有 3 个属性的 model 去解析这份 28 key 的 JSON，YYModel 是 3.7 ms/10000，比 30 属性版本的 26.4 ms 快了七倍。它只做了 3 次字典查找。

### 四笔账合起来

| | 值多少 | 什么时候值钱 |
|---|---|---|
| 键映射在建表期解析完 | 1.95x | 有自定义映射时；映射越多越值钱 |
| 赋值走函数指针不走 KVC | 1.6x | 属性越多越值钱，数值属性收益最大 |
| 遍历短的那一边 | 视 key/属性数之比而定，实测多余 key 的代价差 10 倍 | JSON 字段远多于 model 属性时 |
| 元数据缓存 | 首次 49x，之后 0 | 同一个类解析 4800 次以后基本白送 |

前两条相乘是 3.1 倍。实测总倍数 15 倍。剩下的 4.8 倍不在 YYModel 这边，在 JSONModel 每次 init 都要重做的那些事上：两遍完整属性遍历、每个属性一个 `@try/@catch`、三四个临时集合、每个属性一次 `NSStringFromClass`。那些归 [[iOS JSONModel 源码：Runtime 驱动的属性映射]]。

和那一篇对个账。它测的是 JSONModel 比手写 `initWithDictionary:` 慢 6.6 倍，用的是没有 keyMapper 的 model。我这边「不带键映射」那一栏是 7.7 倍，参照物换成了 YYModel。两个数落在同一档，说明 YYModel 的稳态成本确实接近手写代码，也说明那 6.6 倍和这 7.7 倍量的是同一件事。

---

## 三、`objc_msgSend` 强转函数指针，到底为什么必须精确

上一节说这一招只值 1.6 倍。但它是全篇技术含量最高的地方，而且网上讲它的文章基本停在「这样能省一次消息发送」——这句话本身就是错的，它一次消息发送都没省。

### 它为什么合法

`objc_msgSend` 不是普通 C 函数。它是一段汇编 trampoline：查方法缓存，找到 IMP，然后原样跳过去，全程不碰 x2 及以后的参数寄存器。所以调用方必须按目标方法的签名把寄存器摆好，而让 C 编译器摆对的唯一办法，就是用目标方法的精确签名去声明这次调用。

这不是黑魔法，是 Apple 写在头文件里的用法。`<objc/message.h>` 第 52-53 行：

> These functions must be cast to an appropriate function pointer type before being called.

而且在 arm64 上，SDK 直接把变参原型关死了。`<objc/objc-api.h>` 第 100-105 行：

```c
/* The arm64 ABI requires proper casting to ensure arguments are passed
 *  * correctly.  */
#if defined(__arm64__) && !__swift__
#   undef OBJC_OLD_DISPATCH_PROTOTYPES
#   define OBJC_OLD_DISPATCH_PROTOTYPES 0
#endif
```

`OBJC_OLD_DISPATCH_PROTOTYPES` 为 0 时，`objc_msgSend` 的声明是 `void objc_msgSend(void)`。连参数都没有。你就算想不强转也不行。

### 它为什么必须精确

写个最小例子。一个 `double` 属性，三种调法：

```objc
@interface Box : NSObject
@property (nonatomic, assign) double d;
@property (nonatomic, assign) float f;
@property (nonatomic, assign) int32_t i;
@end

// (1) 精确签名
((void (*)(id, SEL, double))(void *)objc_msgSend)(b, setD, 3.5);
// (2) 声明成变参
((void (*)(id, SEL, ...))(void *)objc_msgSend)(b, setD, 3.5);
// (3) 用错误的返回类型去读 double getter
double  good = ((double  (*)(id, SEL))(void *)objc_msgSend)(b, getD);
void   *bad  = ((void * (*)(id, SEL))(void *)objc_msgSend)(b, getD);
```

arm64 上跑出来：

```text
exact cast      : d=3.5 f=2.25 i=-7
variadic proto  : not available -- the SDK declares `void objc_msgSend(void)`
variadic cast   : d=0   (setter expects it in d0)
double return   : correct cast = 3.5   |  read from x0 = 0x10156a7d0
```

第三行是关键。`3.5` 一个字节都没进 setter。Apple 的 arm64 调用约定里，变参函数的参数走栈，而 `setD:` 是个普通方法，它从 `d0` 读。两边对不上，setter 拿到的是 `d0` 里的残留值。最后一行同理：`double` 返回值在 `d0`，按 `id` 去读拿到的是 `x0` 里的垃圾指针。

那为什么这么多年大家在 x86 上乱写也没出事？我把同一份代码用 `-target x86_64-apple-macos13 -DOBJC_OLD_DISPATCH_PROTOTYPES=1` 编了一遍：

```text
exact cast      : d=3.5 f=2.25 i=-7
variadic proto  : d=3.5 f=0 i=-7
double return   : correct cast = 3.5   |  read from x0 = 0x34
```

`double` 侥幸对了，`float` 当场就是 0。x86_64 的变参约定和普通约定共用寄存器，所以 `double` 走了狗屎运。`float` 就没这个运气。它在变参里会被提升成 `double`，setter 从 `xmm0` 低 32 位读 float，读到的是 double 尾数的低半段。所以「x86 上一直没事」这个说法只对了一半。

### selector stubs 时代它还成不成立

Xcode 14 起，arm64 上编译 `[obj foo]` 默认不再直接调 `objc_msgSend`，而是走一个每个 selector 一份的 stub。有人因此担心 YYModel 这种写法会不会失效。把两种写法的汇编摆一起看就清楚了：

```objc
void viaCast(id o, SEL s, double v) {
    ((void (*)(id, SEL, double))(void *)objc_msgSend)(o, s, v);
}
void viaSyntax(Box *o, double v) {
    [o setD:v];
}
```

```text
_viaCast:
	b	_objc_msgSend
_viaSyntax:
	b	"_objc_msgSend$setD:"
```

加上 `-fno-objc-msgsend-selector-stubs` 重编。`_viaCast` 一个字节没变。`_viaSyntax` 变成先 `adrp/ldr` 加载 selector，再 `b _objc_msgSend`。

selector stub 是给「selector 在编译期已知」的消息发送用的优化，它的作用是把加载 selector 那两条指令从每个调用点挪进共享 stub，省代码体积。YYModel 的 selector 是运行时从 meta 里取出来的变量，压根不具备生成 stub 的条件，所以这套机制从头到尾和它无关。

顺便澄清那句「省一次消息发送」。强转版本调的还是 `objc_msgSend`，还是要查方法缓存，还是要跳 IMP。它省掉的是 `setValue:forKey:` 那一整套。解析 key，按 `set<Key>:`、`_set<Key>:` 的顺序找 setter，按方法签名把 `NSNumber` 拆箱成对应的标量，找不到 setter 时再去按四种命名找 ivar。KVC 的搜索链我在 [[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]] 里逐条实测过，那才是 41 ns 的去处。

---

## 四、快是有代价的

### 一个从 2015 年留到今天的 fall-through

看 `ModelSetNumberToProperty` 里 int32 这一段（`NSObject+YYModel.m:736-741`）：

```objc
        case YYEncodingTypeInt32: {
            ((void (*)(id, SEL, int32_t))(void *) objc_msgSend)((id)model, meta->_setter, (int32_t)num.intValue);
        }
        case YYEncodingTypeUInt32: {
            ((void (*)(id, SEL, uint32_t))(void *) objc_msgSend)((id)model, meta->_setter, (uint32_t)num.unsignedIntValue);
        } break;
```

`YYEncodingTypeInt32` 后面没有 `break`。

这不像是故意的。整个文件里有五处故意的 fall-through，每一处都带着同一句注释：`// break; commented for code coverage in next line`。行号是 770、1001、1093、1595、1620。第 738 行什么都没有。

要证明它真的漏了，写个带计数的 setter 就行：

```objc
@interface P : NSObject
@property (nonatomic, assign) int i;
@property (nonatomic, assign) long long q;
@end
@implementation P
- (void)setI:(int)v { calls++; _i = v; }
@end
```

```text
json -2.7     -> int i=0            (setter called 2)   long long q=-2
json -0.5     -> int i=0            (setter called 2)   long long q=0
json 2.7      -> int i=2            (setter called 2)   long long q=2
json -3.9     -> int i=-3           (setter called 2)   long long q=-3
json -10000000000 -> int i=0        (setter called 2)   long long q=-10000000000
json -100     -> int i=-100         (setter called 2)   long long q=-100
```

**每一个 `int` 属性的 setter 都被调用了两次，第二次的值覆盖第一次。** 同样的输入，`long long` 属性一次就对，`int` 属性在负小数上直接变成 0。`-[NSNumber unsignedIntValue]` 对负的浮点数返回什么本来就不该被依赖，而这段代码依赖上了。

多调一次 setter 还有别的后果。`willChangeValueForKey:`／`didChangeValueForKey:` 会发两遍。自己写的 setter 里的副作用也执行两遍。

加一个 `-Wimplicit-fallthrough` 重编，编译器当场就报出来了：

```text
NSObject+YYModel.m:739:9: warning: unannotated fall-through between switch labels [-Wimplicit-fallthrough]
  739 |         case YYEncodingTypeUInt32: {
```

整个文件里这个开关一共报四处，另外三处落在 `default: break;` 上，没有实际后果。739 是唯一一处落在真 `case` 上的。这个开关不在 `-Wall` 里，所以默认构建看不见它。

我没找到这个 bug 被修过的痕迹。仓库 HEAD 停在 2017-08-07，这段代码从 1.0 一路带到 1.0.4。

### NSDate 的格式白名单是硬编码的

`YYNSDateFromString` 用字符串长度当索引查一张 block 表（`NSObject+YYModel.m:135-243`）：

```objc
    typedef NSDate* (^YYNSDateParseBlock)(NSString *string);
    #define kParserNum 34
    static YYNSDateParseBlock blocks[kParserNum + 1] = {0};
```

```objc
    if (!string) return nil;
    if (string.length > kParserNum) return nil;
    YYNSDateParseBlock parser = blocks[string.length];
    if (!parser) return nil;
    return parser(string);
```

表里被填过的下标只有 10、19、20、23、24、25、28、29、30、34 这十个。长度不在里面，直接返回 nil，一句日志都没有。实测：

```text
  len=10  2014-01-20                      YY:2014-01-20 00:00:00 +0000  JS:nil
  len= 8  2014-1-2                        YY:nil                        JS:nil
  len=19  2014-01-20 12:24:48             YY:2014-01-20 12:24:48 +0000  JS:nil
  len=20  2014-01-20T12:24:48Z            YY:2014-01-20 12:24:48 +0000  JS:2014-01-20 12:24:48 +0000
  len=24  2014-01-20T12:24:48.000Z        YY:2014-01-20 12:24:48 +0000  JS:nil
  len=27  2014-01-20T12:24:48.000000Z     YY:nil                        JS:nil
  len=30  Fri Sep 04 00:12:21 +0800 2015  YY:2015-09-03 16:12:21 +0000  JS:nil
  len=23  2014-01-20T12:24:48.00Z         YY:nil                        JS:nil
  unix timestamp 1420000000               YY:nil                        JS:2014-12-31 04:26:40 +0000
```

六位微秒（长度 27）不认。两位毫秒加 Z（长度 23）也不认。23 这个槽位放的是不带时区的 `yyyy-MM-dd'T'HH:mm:ss.SSS`。而 Unix 时间戳，YYModel 完全不支持，`case YYEncodingTypeNSDate:` 只处理 `NSDate` 和 `NSString` 两种输入。

有意思的是最后一列。JSONModel 支持时间戳，却不认 `2014-01-20` 和带毫秒的 ISO8601。两个库的日期能力是交叉的，谁也不是谁的超集。真要接不确定格式的接口，两边都得自己写转换。

用长度查表这个设计本身是聪明的，`O(1)` 定位到唯一一个候选格式，比挨个 `NSDateFormatter` 试快得多。代价就是格式集合被焊死在源码里。

### 有些类型它根本不接

```text
  readonly NSString *ro   -> nil (no setter, skipped)
  char *cstr (from str)   -> NULL
  struct Pt (from dict)   -> {0, 0}
  long double ld          -> 1.5
  SEL sel (from string)   -> length
  struct Pt (from NSValue with matching objCType) -> {3, 4}
```

- `readonly` 属性没有 setter，`_YYModelMeta initWithClass:` 里 `if (!meta->_getter || !meta->_setter) continue;` 直接跳过（`NSObject+YYModel.m:536`）。
- `char *`、`void *` 只接受 `objCType` 是 `^v` 的 `NSValue`，从 JSON 字符串是拿不到的。
- struct 只接受 `objCType` 逐字节相等的 `NSValue`，JSON 里的字典喂不进去。

struct 那条还有个额外的细节。这是全文件里唯一一处 YYModel 自己回退到 KVC 的地方（`NSObject+YYModel.m:1071-1081`）：

```objc
            case YYEncodingTypeStruct:
            case YYEncodingTypeUnion:
            case YYEncodingTypeCArray: {
                if ([value isKindOfClass:[NSValue class]]) {
                    const char *valueType = ((NSValue *)value).objCType;
                    const char *metaType = meta->_info.typeEncoding.UTF8String;
                    if (valueType && metaType && strcmp(valueType, metaType) == 0) {
                        [model setValue:value forKey:meta->_name];
                    }
                }
            } break;
```

原因很实际：struct 的大小和布局在编译期不确定，没法写出一个固定的函数指针签名。所以这里只能交给 KVC。

### 类型不匹配时的自动转换

```text
  int  <- "42"                 -> n=42
  int  <- "42.9"               -> n=42
  int  <- "abc"                -> n=0
  int  <- NSNull               -> n=0
  BOOL <- "true"               -> flag=1
  BOOL <- "YES"                -> flag=1
  BOOL <- "no"                 -> flag=0
  BOOL <- "maybe"              -> flag=0
  NSString <- 123              -> s=123
  NSURL <- "  https://a.b  "   -> u=https://a.b
  NSArray <- "notanarray"      -> arr=nil
```

字符串转数字的实现是 `YYNSNumberCreateFromID`。它先查一张 24 项的字典，里面是 `TRUE`/`True`/`true`/`YES`/`Yes`/`yes`、对应的假值、以及各种 null 写法。没命中就看有没有小数点。有小数点走 `atof`，没有走 `atoll`。`atof("abc")` 返回 0，所以非法字符串会被静默写成 0，属性原来的值被覆盖掉了。

`"maybe"` 也是 0。想区分「服务端明确给了 false」和「服务端给了垃圾」，用 `BOOL` 属性做不到，得声明成 `NSNumber *`。

最后一条行为要记住：

```text
  keep=before  n=99   (keys absent from the json are left alone)
  after {"keep": null} -> keep=nil
```

JSON 里没有的 key，属性保持原值；JSON 里显式给 `null` 的，属性被清空。用 `yy_modelSetWithDictionary:` 做增量更新时这个区别是有用的。

---

## 五、今天还该不该用

先看维护状态。`git log` 很干脆：

```text
HEAD 1230e60 2017-08-07 01:25:51 +0800
73 commits in 2015
46 commits in 2016
17 commits in 2017
```

最后一个 tag 是 1.0.4。核心文件 `NSObject+YYModel.m` 最后一次有实质改动是 2017-04-21。九年没动了。

好消息是它还能编。我用 Xcode 26 的 clang 21、`-O2`、ARC 直接把源码编进命令行程序，`-Wall` 下零 warning，行为全对。这个库的依赖面小到只有 Foundation 和 `<objc/runtime.h>`，Apple 这些年没动过它用到的任何 API。

坏消息是那个 int32 的 fall-through 不会有人修了。

### 和 Codable 比

关键的比较不能停在 dict → model 这一步。真实业务拿到的是 `NSData`，中间那次 `NSJSONSerialization` 是跑不掉的。加上它之后：

```text
=== NSData -> model, 10000 iterations, median of 15 (ms) ===
  NSJSONSerialization alone (data -> NSDictionary) :   111.29
  YYModel   yy_modelWithJSON:                      :   136.23
  JSONModel initWithData:error:                    :   520.24
```

同一份 JSON，同一组字段，Swift 这边：

```text
JSONDecoder  data -> struct : 117.94 ms / 10000
JSONSerialization data->dict: 113.79 ms / 10000
```

`JSONDecoder` 从 `Data` 直接到 struct，118 ms；YYModel 从 `Data` 到 model，136 ms。**端到端，Codable 比 YYModel 快。**

而且它快得有道理。`JSONDecoder` 的 118 ms 里，只有 4 ms 是映射层。剩下 114 ms 是 JSON 解析本身。现代 Foundation 的 JSON 解析器直接把字节流填进 struct，中间根本不产生 `NSDictionary`。YYModel 那 136 ms 里有 111 ms 花在把 JSON 变成一棵字典树上，它的 26 ms 再快，也改不了这棵树必须先被建出来的事实。

YYModel 对 JSONModel 那 14.5 倍，一放到端到端就塌成 3.8 倍。这不是 YYModel 变慢了，是分母变了。

有个前提要说清楚：`JSONDecoder` 这个数字依赖 Foundation 的 Swift 重写版（macOS 15 / iOS 18 起）。更早的系统上 `JSONDecoder` 是架在 `JSONSerialization` 上的，成绩会差很多。这组数据是 macOS arm64 原生跑的，未在 iOS 上复核。复现方法就是把 `codable.swift` 和 `bench8.m` 原样搬到目标平台跑。

### 我的判断

新写的 Swift 代码，用 Codable。这件事没有讨论余地：性能不输，类型安全好一个数量级，解析失败会抛出带 keyPath 的错误而不是静默写个 0 进去。

老的 ObjC 工程，YYModel 继续用没问题。但要做三件事。把版本锁死。把所有 `int`／`int32_t` 属性改成 `NSInteger` 或 `long long`，绕开第四节那个 fall-through。日期字段自己写 `modelCustomTransformFromDictionary:`。

有一个流传很广的说法我不同意：「YYModel 快是因为绕开了 KVC」。这句话在 2015 年的语境下是对的一半，今天连一半都不剩。绕开 KVC 值 1.6 倍，键映射的时机值 1.95 倍，而这两条加起来在端到端里只影响 15% 的总耗时。真正让 JSONModel 慢下来的，是它每次 init 都要把属性表完整走两遍这件事——那是架构选择，和 KVC 没关系。

还有一条更值得记的：ibireme 在那篇评测的附录里列了八条优化 tip，第八条是「减少遍历的循环次数」，排在最后。按我测出来的账，如果按贡献排序，它应该排第一。写优化文章的人排序时用的是「这一条有多难想到」，读的人以为那是「这一条有多值钱」。

---

## 总结

同一份 JSON、同一个 model，YYModel 比 JSONModel 快 14.5 倍；换成嵌套复杂的微博数据，只剩 5.8 倍；算上 `NSJSONSerialization`，只剩 3.8 倍。倍数是 workload 的函数，不是库的属性。

这 15 倍拆开来：键映射在建表期解析完值 1.95 倍，赋值走函数指针值 1.6 倍，剩下的 4.8 倍是 JSONModel 每次 init 的固定开销。元数据缓存不在稳态账里，它把首次解析的 0.16 ms 摊到后面几千次上。类型编码预解析不是独立的一笔，它就在那 0.16 ms 里。

`objc_msgSend` 强转函数指针是 Apple 头文件明确要求的用法，arm64 上 SDK 直接把变参原型关死了，因为变参走栈而普通方法走寄存器。它和 selector stub 无关，因为 selector 是运行时变量。

YYModel 的 `int32_t` 属性 setter 会被调用两次，负小数会被写成 0。这个 fall-through 从 2015 年留到现在。

今天写 Swift 就用 Codable，端到端它比 YYModel 快。

## 参考资料

### 一手源码

- [ibireme/YYModel](https://github.com/ibireme/YYModel)：本文所有行号来自 HEAD `1230e60`（2017-08-07）
- [jsonmodel/JSONModel](https://github.com/jsonmodel/jsonmodel)：对照组，HEAD `78f8da0`（2018-09-19）
- `<objc/message.h>`（第 52-53 行）、`<objc/objc-api.h>`（第 100-105 行），MacOSX26.5.sdk：强转要求与 arm64 上的 `OBJC_OLD_DISPATCH_PROTOTYPES` 强制关闭

### 二手

- [iOS JSON 模型转换库评测 — ibireme](https://blog.ibireme.com/2015/10/23/ios_model_framework_benchmark/)（2015-10-23）：附录《YYModel 性能优化的几个 Tip》是理解这份代码设计意图的最好材料。注意正文里的两张柱状图没有数据标签。网上流传的那些「YYModel 比 JSONModel 快 X 倍」的精确数字，在原文里找不到出处

### 本地

- [[iOS JSONModel 源码：Runtime 驱动的属性映射]]
- [[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]]
- [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]]

---

实验环境：macOS 26.5.2，Apple Silicon（arm64），Apple clang 21.0.0，`clang -fobjc-arc -O2 -framework Foundation`，Swift 侧 `swiftc -O`。YYModel 与 JSONModel 的源码直接参与编译，没有走 CocoaPods 或预编译产物。未开模拟器。

所有耗时都是同一进程内多轮的中位数，`@autoreleasepool` 套在循环体内。第一节说明了为什么这个细节会影响结论方向。

> 待真机补测：`JSONDecoder` 与 YYModel 的端到端对比在 iPhone / iOS 26 上复现。macOS 15 与 iOS 18 起 Foundation 换成了 Swift 重写版，更早的系统上 `JSONDecoder` 的成绩会显著不同，这一条结论的适用范围取决于部署目标。
