---
title: 【iOS】KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发
published: 2026-07-27
description: KVC 的 setter 搜索链上有一步 setIs<Key>:，Apple 的归档文档和头文件注释都没写，但它真的会被调用。KVO 那条"忘了移除观察者就野指针崩溃"的标准答案，在 iOS 11 之后已经不成立。
tags:
  - iOS
  - Objective-C
  - Runtime
  - KVC
  - KVO
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 11
draft: true
---
# KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发

面试里问到 KVO，标准答案有一段几乎是固定的：运行时生成一个 `NSKVONotifying_` 前缀的中间类，改掉对象的 isa，覆写被观察属性的 setter，在里面插 `willChangeValueForKey:` 和 `didChangeValueForKey:`；观察者一定要记得移除，否则对象死了还在发通知，会野指针崩溃。

前半段今天仍然成立，我用 `dladdr` 把中间类里每个 setter 的真实符号名打出来了，第五节有。后半段不成立。我按教科书的写法构造了三种"应该崩"的场景，在 iOS 18.3 上一次都没崩：观察者先释放再改值五千次，没事；被观察对象带着未移除的观察者去 dealloc，没事；把 `automaticallyNotifiesObserversForKey:` 关掉再来一遍，还是没事。Apple 在 macOS 10.13 / iOS 11 的 Foundation Release Notes 里明确写了这件事，第八节有原文。

还有一个我没料到的：**KVC 的 setter 搜索链上有 `setIs<Key>:` 这一步，而 Apple 的归档文档和 `NSKeyValueCoding.h` 的注释里都没有它。** 我写了个只实现 `setIsName:`、连一个 ivar 都没有的类，`setValue:@"V" forKey:@"name"` 精确地打中了它。

这篇把 KVC 的两条搜索链、KVO 的中间类结构、以及"什么时候根本不生成中间类"这三件事全部用实验走一遍。前面 [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]] 留了一句"这一点在第三周的 KVO 笔记里还会再用到"，说的就是 `[obj class]` 和 `object_getClass` 在这里会给出两个不同的答案。

实验环境写在文末，所有输出都是真跑出来的。

---

## 一、KVC 不绕过 setter

流传最广的一句是"KVC 能绕开 setter 直接写内存"。这句话把因果说反了。

`NSKeyValueCoding.h` 的方法注释是这么写的：

> Searches the class of the receiver for an accessor method whose name matches the pattern `-set<Key>:`. […] if the type of the method's parameter is an object pointer type the method is simply invoked with the value as the argument.

只要 `set<Key>:` 存在，KVC 一定调它，包括你手写的、带副作用的、会发通知的那个 setter。写 ivar 是找不到 setter 之后的兜底，不是首选。这个先后关系搞反了，后面 KVO 那半篇会连着理解错。

那到底找哪几个 setter？我定义了一组只实现单个方法的类，用运行时把方法组合逐个装上去测：

```objc
static void trySetters(const char *label, NSArray<NSString *> *sels) {
    Class c = objc_allocateClassPair([Skel class], cn.UTF8String, 0);
    for (NSString *s in sels)
        class_addMethod(c, NSSelectorFromString(s), (IMP)rec, "v@:@");
    objc_registerClassPair(c);
    id o = [[c alloc] init];
    [o setValue:@"V" forKey:@"name"];       // key 固定是 "name"
    ...
}
```

`Skel` 基类身上带着 `_name` / `_isName` / `name` / `isName` 四个 ivar，所以只要方法层全部落空，就能从"哪个 ivar 被写了"看出来。

```text
setName: 单独                -> 方法命中: setName:       ivar 命中: (none)
_setName: 单独               -> 方法命中: _setName:      ivar 命中: (none)
setIsName: 单独              -> 方法命中: setIsName:     ivar 命中: (none)
_setIsName: 单独             -> 方法命中: (无)           ivar 命中: _name
setName: + _setName:         -> 方法命中: setName:       ivar 命中: (none)
setName: + setIsName:        -> 方法命中: setName:       ivar 命中: (none)
_setName: + setIsName:       -> 方法命中: _setName:      ivar 命中: (none)
setIsName: + _setIsName:     -> 方法命中: setIsName:     ivar 命中: (none)
全部四个                      -> 方法命中: setName:       ivar 命中: (none)
一个都没有                    -> 方法命中: (无)           ivar 命中: _name
```

顺序是 `set<Key>:` → `_set<Key>:` → `setIs<Key>:`。`_setIs<Key>:` 不在链上。

### `setIs<Key>:` 这一步，两份官方材料都没写

Apple 归档的 Accessor Search Patterns 只给了两个：

> Look for the first accessor named `set<Key>:` or `_set<Key>`, in that order.

`NSKeyValueCoding.h` 的注释更详细，但 setter 那一段同样只有 `-set<Key>:`，`-_set<Key>:` 被放进 Compatibility notes 里，注明是为了兼容早已废弃的 `takeValue:forKey:`：

> For backward binary compatibility with `-takeValue:forKey:`'s behavior, a method whose name matches the pattern `-_set<Key>:` is also recognized in step 1. KVC accessor methods whose names start with underscores were deprecated as of Mac OS 10.3 though.

两份材料里搜不到 `setIs`。我一开始以为是自己的实验被 ivar 污染了，于是写了一个干净到不能再干净的对照：

```objc
@interface Pure : NSObject @end
@implementation Pure
- (void)setIsName:(id)v { printf("      >>> setIsName: 被调用，参数 = %s\n", ...); }
- (void)setValue:(id)v forUndefinedKey:(NSString *)k { printf("      >>> forUndefinedKey:%s\n", ...); }
@end
```

没有属性，没有 ivar，没有别的方法，还把 `setValue:forUndefinedKey:` 接管了当哨兵。

```text
Pure 类只实现 setIsName:，setValue:@"V" forKey:@"name"：
      >>> setIsName: 被调用，参数 = V
Ctrl 类只实现 setIsNickname:，setValue:@"V" forKey:@"nickname"：
      >>> setIsNickname: 被调用
```

哨兵一次都没响。

我自己去年那篇 KVC 总结里写的就是"`set<Key>: → _set<Key> → setIs<Key>`"三步，当时是从中文资料里抄的，写完还犯嘀咕，因为查 Apple 文档只有两步。现在可以确认：那三步是对的，Apple 的文档漏了一步。中文圈那批文章在这一点上比官方材料更接近真相，虽然它们多半也是互相抄的，未必自己验过。

我的判断是：这一步真实存在，但因为不在任何规格文档里，工程上不该依赖它。它更像 `is<Key>` 那套布尔命名约定在 setter 侧留下的对称遗迹。

---

## 二、落到 ivar 那一步，语义变了

四个 setter 全落空、`accessInstanceVariablesDirectly` 又返回 YES 时，KVC 去找 ivar。顺序官方写得很清楚。我也验了一遍，用 `class_addIvar` 动态造类，每次只保留一部分：

```text
_name / _isName / name / isName 全有   -> 写进了 _name
去掉 _name                            -> 写进了 _isName
去掉 _name, _isName                   -> 写进了 name
只有 isName                           -> 写进了 isName
只有 getName（不该命中）               -> <NSUnknownKeyException>
```

`_<key>` → `_is<Key>` → `<key>` → `is<Key>`。头文件里还留了一条历史注记：Mac OS 10.2 时代顺序是反的，`<key>` 在前。

真正会咬人的是这一步的内存语义。头文件写的是：

> If such an instance variable is found and its type is an object pointer type the value is retained and the result is set in the instance variable, after the instance variable's old value is first released.

retain，不是 copy。所以一个声明成 `copy` 但没有 setter 的属性，走 KVC 会丢掉 copy 语义：

```objc
@interface CopyLost : NSObject { @public NSString *_text; } @end

CopyLost *c = [CopyLost new];
NSMutableString *ms = [NSMutableString stringWithString:@"原值"];
[c setValue:ms forKey:@"text"];
```

```text
写进去的对象 = 0x600000c04690，ms = 0x600000c04690，同一个? 是（只 retain 没 copy）
改完 ms 之后 _text = 原值被外部改了
_text 的类 = __NSCFString
```

外部握着的可变字符串改一下，属性跟着变。[[iOS 属性关键字：从所有权推导，而不是从类型名猜]] 那篇讲过 `copy` 修饰 `NSString` 是为了防这个，这里等于把防线绕过去了。前提是这个属性没有合成 setter。`@property (copy)` 正常声明是有 setter 的，只有你自己在 extension 里写 `readonly` 又没补 setter 时才会命中。

### readonly 属性能被 KVC 改

顺着同一条链就能推出来。`readonly` 属性编译器不生成 `set<Key>:`，但仍然合成 `_<key>` ivar：

```text
类上有 setToken: 吗? 没有
setValue:forKey: 之后 b.token = secret
ivars: _token
```

这不是 Apple 承诺的能力，是搜索规则的副作用。想堵住它，覆写 `+accessInstanceVariablesDirectly` 返回 NO：

```text
抛出 NSUnknownKeyException
  reason: [<IvarBlocked 0x600000010040> setValue:forUndefinedKey:]: this class is not key value coding-compliant for the key name.
```

### 一个能省半小时的细节

上面那行异常名是 `NSUnknownKeyException`，但你在代码里 `@catch` 时写的常量叫 `NSUndefinedKeyException`。这两个不是两个异常。头文件里写着：

> The actual value of this constant string is "NSUnknownKeyException," to match the exceptions that are thrown by KVC methods that were deprecated in Mac OS 10.3.

常量名和字符串值不一致，纯历史遗留。第一次照着崩溃日志去搜 `NSUnknownKeyException` 却在头文件里搜不到的时候，很容易怀疑人生。

给标量属性传 nil 是另一个异常：

```text
setValue:nil forKey:@"age" -> NSInvalidArgumentException
  [<Scalars 0x600000c0c270> setNilValueForKey]: could not set nil as the value for the key age.
```

覆写 `setNilValueForKey:` 可以接管。字典转模型时后端返回 `null` 就走这条路，是线上崩溃的常客。

---

## 三、取值端：四个 getter 和三个集合代理

`valueForKey:` 的方法层顺序，同样用运行时装方法测：

```text
getName                      -> getName
name                         -> name
isName                       -> isName
_name                        -> _name
_isName                      -> <NSUnknownKeyException
getName+name                 -> getName
name+isName                  -> name
isName+_name                 -> isName
_name+getName                -> getName
```

`get<Key>` → `<key>` → `is<Key>` → `_<key>`。注意 `_is<Key>` 作为方法不在链上，它只作为 ivar 名存在。取值端和设值端在这里不对称。

找不到简单 getter 时，KVC 会尝试拼一个集合代理出来。这部分归档文档只写了两种，头文件写了三种，我把三种都测了：

```objc
@interface OnlyCount : NSObject @end
@implementation OnlyCount
- (NSUInteger)countOfName { return 3; }
- (id)objectInNameAtIndex:(NSUInteger)i { return @(i * 10); }
@end
```

```text
只有 countOf/objectIn              -> ( 0, 10, 20 )
  代理对象的类 = NSKeyValueArray，isKindOfClass:[NSArray class] = YES
countOf+enumeratorOf+memberOf      -> 类 = NSKeyValueSet, isKindOfClass:[NSSet class] = YES
countOf+indexInNameOfObject+objectIn -> 类 = NSKeyValueOrderedSet
```

三个代理类的真名：`NSKeyValueArray`、`NSKeyValueSet`、`NSKeyValueOrderedSet`。它们都不是真容器，只是把 `count`、`objectAtIndex:` 这些消息翻译成你实现的那几个原语。

### 两份官方材料在这里打架，实验说了算

归档文档把 `_<key>` 放在 step 1 的第四位。头文件说它是兼容路径，位置是"between steps 1 and 3"。而头文件里 step 2 恰好是 NSOrderedSet 代理。两种读法会给出不同的优先级。

造一个两边都有的类就能分开：

```objc
@interface Decide : NSObject @end
@implementation Decide
- (id)_name { return @"_name 方法赢了"; }
- (NSUInteger)countOfName { return 2; }
- (NSUInteger)indexInNameOfObject:(id)o { return 0; }
- (id)objectInNameAtIndex:(NSUInteger)i { return @(i); }
@end
```

```text
只有 orderedset 三件套           -> NSKeyValueOrderedSet
_name 方法 + orderedset 三件套    -> __NSCFConstantString  "_name 方法赢了"
isName + orderedset 三件套        -> __NSCFConstantString  "isName"
ivar _name + orderedset 三件套    -> NSKeyValueOrderedSet
```

`_name` 方法排在集合代理之前，归档文档那种读法对得上。最后一行还顺便确认了 ivar 永远是最后一步。

顺着这条链再往下就是 `valueForUndefinedKey:`，默认抛 `NSUnknownKeyException`，reason 里那句 "this class is not key value coding-compliant for the key xxx" 是所有 KVC 崩溃日志的共同特征。

---

## 四、集合运算符里藏着一个和文档相反的实现

Apple 对 `@sum` 和 `@avg` 的描述是这样的：

> reads the property specified by the right key path for each element of the collection, converts it to a `double` (substituting 0 for nil values), and computes the arithmetic average / the sum of these.

我第一次跑的时候是想验"nil 当 0"这半句，结果先撞上了另一半：

```text
@[@0.1, @0.2] @sum.self = 0.3
纯 double 相加          = 0.30000000000000004441
@[@1,@2,@2] @avg.self   = 1.66666666666666666666666666666666666666
1.6666… 的 double       = 1.66666666666666674068
@sum 返回对象的类 = NSDecimalNumber
```

`0.1 + 0.2` 在 double 下必然是 `0.30000000000000004`，KVC 给出的是精确的 `0.3`；`@avg` 输出 38 位有效数字，double 最多给 17 位。返回类型是 `NSDecimalNumber`。**`@sum` 和 `@avg` 走的是十进制定点运算，不是文档说的 double。** 集合里塞个 `NSNull` 时报的错也是同一个方向的旁证：

```text
含 NSNull @sum.self -> NSInvalidArgumentException: -[NSNull decimalValue]: unrecognized selector
```

它对元素发的是 `decimalValue`。

这条对做金额统计的人有实际价值。用 `@sum.amount` 汇总一列价格，精度比自己写循环累加 double 更好。反过来，如果你的单元测试拿 double 相加的结果去对 `@sum`，末位对不上是正常的。

### nil 和 NSNull 是两件事

文档说的"nil 用 0 替代"指的是取右键路径取不到值：

```text
键缺失 @sum.v = 4        // @[@{@"v":@1}, @{}, @{@"v":@3}]
键缺失 @avg.v = 1.33333333333333333333333333333333333333
键缺失 @max.v = 3
```

`@sum` 得到 4（1+3），`@avg` 得到 1.333 而不是 2——那个取不到值的元素按 0 计入了分子，也计入了分母。`@max` 则直接忽略它，和文档里 "The search ignores nil valued collection entries" 对得上。

而集合里真的放了 `NSNull` 对象，走的是上面那条 `unrecognized selector`。这两种情况在很多文章里被混为一谈。

`@count` 的三条特性一次全验：

```text
@count      = 5
@count.self = 5      // 右键路径写了也被忽略
空数组 @count = 0 / @sum.self = 0 / @avg.self = (null) / @max.self = (null)
```

空集合上 `@count` 和 `@sum` 给 0，`@avg` 和 `@max` 给 nil。

还有一个只存在于头文件的运算符：`NSUnionOfSetsKeyValueOperator`（也就是 `@unionOfSets`）在 `NSKeyValueCoding.h` 里从 10.4 起就导出了，但归档文档的运算符列表里没有它，NSHipster 甚至明确写过"因为 set 天然去重所以只有 distinct 版本"。这条我没找到能把它和 `@distinctUnionOfSets` 区分开的场景，先记在这里。

---

## 五、KVO 的中间类，覆写了哪几个方法

现在换到 KVO。观察前后各打一次：

```text
注册观察者之前：
  object_getClass(obj) = Person                     0x100fb8710
  [obj class]          = Person                     0x100fb8710
  真实类的 superclass   = NSObject

注册观察者之后：
  object_getClass(obj) = NSKVONotifying_Person      0x6000026000c0
  [obj class]          = Person                     0x100fb8710
  真实类的 superclass   = Person
  isKindOfClass:[Person class]   = YES
  isMemberOfClass:[Person class] = YES
  中间类自己实现的方法（4 个）: setName: class dealloc _isKVOA
  真实类自己的 ivar：0 个
```

四个方法：被观察属性的 setter、`class`、`dealloc`、`_isKVOA`。没有新增 ivar，superclass 指向原类。

`isMemberOfClass:` 返回 YES 这件事需要说一句。[[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]] 里抄过 objc4 的四份实现，实例版是这样的：

```objc
- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}
```

起点就是 `[self class]`，而 `-class` 恰好是中间类覆写的四个方法之一。所以整套类型判断从入口起就看不见中间类。编译器会把这类调用优化成 `objc_opt_isMemberOfClass`，那条快速路径确实直接读 isa，但它的开关 `hasCustomCore()` 会被 `class` 这个 core selector 的覆写关掉，于是退回 `objc_msgSend`，又绕回上面这份实现。两条路殊途同归。

所以 KVO 下想知道真实类型只有一条路：

```objc
object_getClass(obj)     // NSKVONotifying_Person
[obj class]              // Person，撒谎
```

### 这次伪装是要付费的

上面提到快速路径被 `hasCustomCore()` 关掉，这句话可以直接测。同一个类，一个对象没被观察，一个被观察，各跑三百万次 `isKindOfClass:`：

```text
轮次  未观察     被观察     倍数
1     11.8 ms    57.5 ms    4.89x
2     7.5  ms    63.0 ms    8.37x
3     7.1  ms    54.0 ms    7.64x
4     10.5 ms    57.5 ms    5.49x
5     7.0  ms    57.7 ms    8.23x
```

五轮的倍数在 4.9 到 8.4 之间跳，噪声不小，方向从没反过。注意被测的是 `isKindOfClass:`，不是被观察的那个属性。一个对象只要被 KVO 观察过，它身上所有的类型判断都会变慢几倍，因为覆写 `-class` 这一个动作就把整个类的核心选择器快速路径关掉了。在一个每帧对 cell 做类型判断的列表里，这笔账记在 KVO 头上，但没人会往那儿想。

setter 本身的账更重：

```text
每组一百万次赋值，跑 5 轮
轮次  无观察者   1 个观察者  3 个观察者  注册过又全部移除
1     2.3 ms     322.4 ms   535.6 ms    1.7 ms
2     1.6 ms     321.2 ms   531.8 ms    1.6 ms
3     1.6 ms     321.2 ms   528.6 ms    1.6 ms
4     1.7 ms     330.4 ms   616.4 ms    1.9 ms
5     1.7 ms     359.4 ms   576.5 ms    1.6 ms
```

一个观察者就是约两百倍。最后一列是"注册过又全部移除"的对象，成本回到基线，这也从侧面确认了 isa 确实还原干净了。

> 这两组耗时来自 iOS 模拟器（arm64），只能用来说明"快速路径被关掉了""通知链路确实走了一大圈"这两个行为，不能当作真机上的性能结论。
> 待真机补测：同样的循环在 iPhone 15 / iOS 26.5 上的绝对耗时与倍数。

### setter 被换成了什么，dladdr 能直接告诉你

Foundation 不开源，但符号表在。把中间类每个方法的 IMP 拿去 `dladdr` 反查：

```objc
static void showIMP(Class dyn, Class orig, SEL sel) {
    IMP b = class_getMethodImplementation(dyn, sel);
    Dl_info info; const char *sym = "?";
    if (dladdr((void *)b, &info) && info.dli_sname) sym = info.dli_sname;
    ...
}
```

```text
setI:         原类 0x102960b24  实际 0x1810bb4a8  (已替换) 符号 = _NSSetIntValueAndNotify
setB:         原类 0x102960b68  实际 0x1810bac9c  (已替换) 符号 = _NSSetBoolValueAndNotify
setD:         原类 0x102960bac  实际 0x1810bb170  (已替换) 符号 = _NSSetDoubleValueAndNotify
setLd:        原类 0x102960bec  实际 0x102960bec  (未替换) 符号 = -[T setLd:]
setR:         原类 0x102960c48  实际 0x1810bc4c8  (已替换) 符号 = _NSSetRectValueAndNotify
setS:         原类 0x102960c9c  实际 0x1810bab04  (已替换) 符号 = _NSSetObjectValueAndNotify
setUntouched: 原类 0x102960cf0  实际 0x102960cf0  (未替换) 符号 = -[T setUntouched:]
class         原类 0x18005efdc  实际 0x1810b97a4  (已替换) 符号 = NSKVOClass
dealloc       原类 0x18005f354  实际 0x1810b9158  (已替换) 符号 = NSKVODeallocate
```

不是运行时给你现编一个 setter，而是按参数类型套一族预先写好的 `_NSSet<Type>ValueAndNotify` 函数。`class` 换成 `NSKVOClass`，`dealloc` 换成 `NSKVODeallocate`。没被观察的 `setUntouched:` 原样不动。

`setLd:` 那行是意外收获。`long double` 属性的 setter 完全没被替换，设值不发通知，而且这个属性根本就不是 KVC 兼容的：

```text
valueForKey:@"ld"  -> NSUnknownKeyException: this class is not key value coding-compliant for the key ld.
手动 will/did      -> NSUnknownKeyException: 同上
对照 valueForKey:@"d" = 1.5 (__NSCFNumber)
```

`double` 正常，`long double` 直接不在 KVC 的类型表里。这个类型在 iOS 上很少见，但它说明 `_NSSet<Type>ValueAndNotify` 是一张有限的类型表，不在表里的属性 KVO 拦不住，而且失败是静默的。

这张表有多大，`nm` 一遍 Foundation 就知道：

```text
_NSSetBoolValueAndNotify            _NSSetShortValueAndNotify
_NSSetCharValueAndNotify            _NSSetSizeValueAndNotify
_NSSetDoubleValueAndNotify          _NSSetUnsignedCharValueAndNotify
_NSSetFloatValueAndNotify           _NSSetUnsignedIntValueAndNotify
_NSSetIntValueAndNotify             _NSSetUnsignedLongLongValueAndNotify
_NSSetLongLongValueAndNotify        _NSSetUnsignedLongValueAndNotify
_NSSetLongValueAndNotify            _NSSetUnsignedShortValueAndNotify
_NSSetObjectValueAndNotify          _NSSetValueAndNotify
_NSSetPointValueAndNotify           _NSSetRangeValueAndNotify
_NSSetRectValueAndNotify
```

十九个。十八个具体类型加一个兜底的 `_NSSetValueAndNotify`。`NSInteger` 落在 `LongLong` 上，`CGPoint` / `CGSize` / `CGRect` / `NSRange` 各有一条专线，`long double` 一条都没有，和上面那个静默失败对得上。

### 同一个函数怎么知道自己在给哪个 key 发通知

这族函数还有一个性质，比"它们不是现编的"更值得说。给两个毫不相干的类各自的 `name` 属性挂上观察者：

```text
  A 的中间类 setName: IMP = 0x180ead574
  B 的中间类 setName: IMP = 0x180ead574
  两个不同的类，同一个函数地址？ 是
```

同一个地址。也就是说 `_NSSetObjectValueAndNotify` 是全进程共用的一份代码，它被装进多少个中间类的多少个槽位都无所谓。可它总得知道自己该给谁发通知。

函数体里能区分调用现场的东西只剩两个：`self` 和 `_cmd`。`self` 给出观察记录，`_cmd` 给出 key。它拿到 `setName:` 这个选择器，掐掉 `set` 和冒号、首字母转小写，还原出 `name`。

这解释了一件我以前只当成惯例的事：KVO 为什么对 setter 的命名这么死板。因为通知路径上的 key 是从选择器名字里反推出来的，setter 一旦不是 `set<Key>:` 的形状，这条反推就断了。第一节讲 KVC 时那套命名规则，到这里成了运行时的硬依赖。

同一份符号表里还挂着一整套集合版本：`_NSKVOInsertObjectAtIndexAndNotify`、`_NSKVORemoveObjectAtIndexAndNotify`、`_NSKVOReplaceObjectsAtIndexesAndNotify`、`_NSKVOUnionSetAndNotify`、`_NSKVOMinusSetAndNotify`、`_NSKVOIntersectSetAndNotify`。第七节讲 `mutableArrayValueForKey:` 时会碰到它们。

还有一个 `NSKVODeallocateBreak`。名字的形状和 objc4 里那些 `BREAKPOINT_FUNCTION` 一模一样，Foundation 二进制里那句提示直接说明了用途：

> Set a breakpoint on NSKVODeallocateBreak to stop here in the debugger.

所以"对象带着观察者死了"这类问题可以直接 `b NSKVODeallocateBreak` 断在现场，不用去猜。

### 一个类只有一个中间类

```text
一个观察者后 isa = NSKVONotifying_Person (0x6000026000c0)
两个观察者后 isa = NSKVONotifying_Person (0x6000026000c0)  同一个类? 是
此刻中间类的方法（5 个）: setAge: setName: class dealloc _isKVOA
```

观察第二个属性不会再套一层，而是往同一个类里再加一个 setter 覆写。这带来一个"连坐"效应：`a` 只观察 `name`、`b` 只观察 `age`，两者共用同一个中间类，于是 `a` 设置 `age` 时也要走一遍通知流程。

别的实例不受影响：

```text
q 的 object_getClass = Person
q 和 p 同类吗? 否
```

### 移除之后 isa 会还原

这条和流传的说法相反，所以我测得比较细：

```text
移除 name（还剩 age）后 isa = NSKVONotifying_Person
全部移除后 isa = Person
中间类还在运行时里吗? objc_getClass("NSKVONotifying_Person") = 0x6000026000c0
再设一次 p.name（不应有通知）        // 无输出
重新注册后 isa = NSKVONotifying_Person (0x6000026000c0)，和第一次同一个类? 是
```

最后一个观察者被移除时，实例的 isa 指回原类；而中间类这个 Class 对象本身留在运行时里，下次再观察同一个类直接复用。"移除后 isa 不还原"和"中间类会被销毁"这两个说法各错一半，合起来正好把真相说反了。

多段 keyPath 会逐层 swizzle：

```text
观察 root.leaf.v
  root  isa = NSKVONotifying_Root
  leaf  isa = NSKVONotifying_Leaf
```

---

## 六、关掉自动通知，中间类根本不生成

这是我这次最意外的一个结果。教科书讲 KVO 一律说"注册观察者就会 isa-swizzle"，但 `automaticallyNotifiesObserversForKey:` 返回 NO 的那些 key 不会。

用一个两个 key 待遇不同的类来测：

```objc
@interface Mixed : NSObject
@property (nonatomic, assign) int manualKey;
@property (nonatomic, assign) int autoKey;
@end
@implementation Mixed
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    if ([key isEqualToString:@"manualKey"]) return NO;
    return [super automaticallyNotifiesObserversForKey:key];
}
@end
```

```text
注册前 isa = Mixed
观察 autoKey 后 isa = NSKVONotifying_Mixed
  >> 回调 autoKey
移除后 isa = Mixed
观察 manualKey 后 isa = Mixed          <- 没有中间类
两个都观察时 isa = NSKVONotifying_Mixed
真实类的方法（4）: setAutoKey: class dealloc _isKVOA
```

**只观察一个关掉了自动通知的 key，isa 一动不动，中间类根本不会被创建。** 两个 key 都观察时中间类出现了，但它只覆写 `setAutoKey:`，`setManualKey:` 不在里面。

这个行为其实很合理。既然你声明了"这个 key 我自己发通知"，KVO 就没有拦截 setter 的必要，也就没有生成子类的必要。反过来它也解释了第七节那些没有 setter 的 key 为什么照样能观察。`willChangeValueForKey:` / `didChangeValueForKey:` 是 `NSObject` 上的普通方法，它们从对象的 `observationInfo` 里取观察记录再逐个回调，整条路和 isa 无关。

`observationInfo` 存在哪，Apple 的 API 文档里有一句现成的：

> The default implementation of this method retrieves the information from a global dictionary of observed objects keyed by memory addresses.

一张以对象地址为 key 的全局字典，不是关联对象。中间类新增的 ivar 数是 0，这条也对得上。

### 手动通知的配对

```text
m.score = 5（合成 setter，自动通知已关）：      // 无输出
手动 will/didChangeValueForKey:
  [通知] key=score kind=1 old=5 new=99
只调 didChange 不调 willChange：                // 无输出
只调 willChange 不调 didChange：                // 无输出
（然后再来一对完整的）
  [通知] key=score kind=1 old=123 new=7
```

单独调 `didChangeValueForKey:` 不会发通知，`willChange` 是取旧值的地方，没有它就没有 change 字典。多个 key 一起变时官方要求嵌套，`did` 的顺序和 `will` 相反。

Apple 还有一个知名度很低的 per-key 版本：`+automaticallyNotifiesObserversOf<Key>`，从 OS X 10.5 起 `automaticallyNotifiesObserversForKey:` 的默认实现会先去找它。比在一个方法里写一串 `isEqualToString:` 干净。

---

## 七、观察一个没有 setter 的 key

`@property (readonly)` 只有 getter，KVO 能观察吗？我原以为会抛异常。

```text
注册成功，没抛异常。isa = NSKVONotifying_ReadOnly
中间类自己实现的方法（3 个）: class dealloc _isKVOA
直接改 _counter = 7：                        // 无通知
手动 will/did：
  [通知] key=counter kind=1 old=- new=8
KVC setValue:@9 forKey:@"counter"：
  [通知] key=counter kind=1 old=- new=9
```

注册成功，中间类照样生成，但里面只有三个方法，没有 setter 可覆写。之后手动 `will/did` 能发通知，`setValue:forKey:` 也能。

再退一步，连属性都不声明，只留一个 `_tag` ivar：

```objc
@interface BareIvar : NSObject { @public NSString *_tag; } @end
@implementation BareIvar @end
```

```text
注册成功，isa = NSKVONotifying_BareIvar
中间类自己实现的方法（3 个）: class dealloc _isKVOA
KVC setValue:@"T" forKey:@"tag"：
  [通知] key=tag kind=1 old=<null> new=T
_tag = T
```

`setValue:forKey:` 走的是第二节那条 ivar 兜底路径，最后直接写 `_tag`，通知照样发出来了。

这就回到了第一节的那个先后关系。KVO 有两条独立的触发路径：一条是被 swizzle 的 setter（依赖 isa），另一条在 `setValue:forKey:` 内部（不依赖 isa）。所以"KVO 只观察属性不观察成员变量"这句话得改一下：直接写 `_tag = @"x"` 确实不发通知，但用 KVC 写同一个 ivar 会发。

区别只在有没有经过消息派发。`_tag = @"x"` 编译成一条内存写指令，没有任何拦截点。

### 依赖 key

派生属性用 `keyPathsForValuesAffecting<Key>`：

```objc
+ (NSSet<NSString *> *)keyPathsForValuesAffectingProgress {
    return [NSSet setWithObjects:@"total", @"current", nil];
}
```

```text
isa = NSKVONotifying_Download
中间类自己实现的方法（5 个）: setCurrent: setTotal: class dealloc _isKVOA
d.total = 100：
  [通知] key=progress kind=1 old=0 new=0
d.current = 30：
  [通知] key=progress kind=1 old=0 new=0.3
```

你观察的是 `progress`，被 swizzle 的却是 `setTotal:` 和 `setCurrent:`。机制在方法列表里一目了然。

官方对它有一条硬限制，Guide 里写得很直白：

> The `keyPathsForValuesAffectingValueForKey:` method does not support key-paths that include a to-many relationship.

想让 `Department.totalSalary` 跟着 `employees.salary` 变，这条路走不通，只能在增删 employee 时自己维护观察关系。

### 集合与 Prior

```text
[pp.friends addObject:@"x"]：                              // 无通知
[[pp mutableArrayValueForKey:@"friends"] addObject:@"y"]：
  [通知] key=friends kind=2 new=( y )
代理对象的类 = NSKeyValueNotifyingMutableArray
代理 == 真数组? 否   真数组内容 = ( x, y )
整体换一个数组 pp.friends = ...：
  [通知] key=friends kind=1 old=( x, y ) new=( z )
```

`kind=2` 是 `NSKeyValueChangeInsertion`。代理对象是 `NSKeyValueNotifyingMutableArray`，写操作会透传到真数组。

这个代理落到哪一层，取决于你给它准备了什么。三个类分别只实现一种，往代理里加一个元素：

```text
有 insertObject:inItemsAtIndex: / removeObjectFromItemsAtIndex:
      [走 insertObject:inItemsAtIndex:]
    >> items { kind = 2; indexes = (0); new = (A) }

只有 getter + setter
      [走 setItems:  新数组 (A)]
    >> items { kind = 2; indexes = (0); new = (A) }
      [走 setItems:  新数组 (A, B)]
    >> items { kind = 2; indexes = (1); new = (B) }

只有 ivar
    >> items { kind = 2; indexes = (0); new = (A) }     （直接写 ivar）
```

中间那档是个性能陷阱。没有插入/删除访问器时，代理每加一个元素都要把整个数组重建一遍再调一次 `setItems:`。往里塞一千个元素就是一千次全量拷贝，而外部看到的通知形状和第一档一模一样，看通知根本发现不了。

还有个细节：没有任何观察者的时候，`mutableArrayValueForKey:` 返回的是 `NSKeyValueIvarMutableArray`，有观察者才换成 `NSKeyValueNotifyingMutableArray`。代理类是按需挑的，通知那一层不白挂。

`Prior` 和 `Initial` 各带一个通知：

```text
注册（带 Initial + Prior）：
  [通知] key=age kind=1 old=- new=0
pp.age = 3：
  [通知] key=age kind=1 old=0 new=-  (PRIOR)
  [通知] key=age kind=1 old=0 new=3
```

Initial 那条只有 new，Prior 那条只有 old。官方对 Prior 的用途给了一个很具体的场景：你自己的某个属性依赖被观察属性，需要在它变之前调自己的 `willChange`，等收到变更后的通知就来不及了。

### 通知和值是两件事

前面几节的结论合起来其实是一句话：KVO 认的是通知，值变没变它一无所知。这句话可以直接演示。

```objc
@implementation Ignore
- (void)setName:(NSString *)n { /* 故意什么都不做 */ }
- (void)setOther:(NSString *)o { _other = [o copy]; _name = [o copy]; }  // 这里也改了 name
@end
```

观察 `name`，然后各走一次：

```text
1. setter 收到值但什么都不做：
    >> name  { kind = 1; new = "<null>"; old = "<null>"; }
   实际读回来 name = (nil)

2. 另一个 setter 偷偷改了 name 的 ivar：
   实际读回来 name = 偷偷改（值确实变了）
   （无 name 的通知）
```

第一次值一点没动，通知发了。第二次值真的变了，一声不吭。

同一条逻辑还有个更常见的表现：设成和当前完全一样的值，通知照发。

```text
  setValue:@"B"（当前就是 B）：
    >> name  { kind = 1; new = B; old = B; }
```

KVO 不做去重。观察端如果对重复回调敏感（比如回调里要发网络请求或者刷 UI），得自己比 old 和 new。

回调的线程也钉一下，这个我第一次测的时候被自己的仪器骗了。我用 `dispatch_sync` 派到全局队列里改值，打印出来是主线程，差点写成"KVO 回调总在主线程"。实际上 `dispatch_sync` 本来就可能在调用线程上直接执行。换成 `dispatch_async` 加信号量：

```text
主线程 = 0x102d41b10
在主线程改值：     回调发生在：主线程  线程 0x102d41b10
在子线程改值：     回调发生在：子线程  线程 0x102d47720
```

回调是同步的，谁改值谁回调。所以在后台线程改一个被 UI 观察的属性，`observeValueForKeyPath:` 就在后台线程执行，里面碰 UIKit 直接出事。这条比"忘了 removeObserver"常见得多。

---

## 八、忘了 removeObserver 到底会怎样

这一节是这篇文章要填的坑。我自己去年写 KVO 那篇的时候，照抄的是这段标准表述：

> 释放后，观察者不会自动将其自身移除。被观察对象继续发送通知，而忽略了观察者的状态。但是，与发送到已释放对象的任何其他消息一样，更改通知会触发内存访问异常。

Apple 的 KVO Programming Guide 至今还这么写。但它在 iOS 11 之后已经不准了。

### 官方改过一次，有 release note

Foundation Release Notes for macOS 10.13 and iOS 11 里有一节叫 Relaxed Key-Value Observing Unregistration Requirements：

> Prior to 10.13, KVO would throw an exception if any observers were still registered after an autonotifying object's -dealloc finished running. Additionally, if all observers were removed, but some were removed from another thread during dealloc, the exception would incorrectly still be thrown. This requirement has been relaxed in 10.13, subject to two conditions:
>
> • The object must be using KVO autonotifying, rather than manually calling -will and -didChangeValueForKey: (i.e. it should not return NO from +automaticallyNotifiesObserversForKey:)
> • The object must not override the (private) accessors for internal KVO state
>
> If all of these are true, any remaining observers after -dealloc returns will be cleaned up by KVO; this is also somewhat more efficient than repeatedly calling -removeObserver methods.

被观察对象带着未移除的观察者去 dealloc，不再抛异常，剩下的观察记录由 KVO 自己清掉。

### 实测比 release note 承诺的还宽

我把三种场景都跑了：

```text
场景 0：被观察对象带着未移除的观察者 dealloc（观察者仍然活着）
  注册后 observationInfo: <NSKeyValueObservationInfo 0x600000202040> ( <NSKeyValueObservance ...> )
    >> 回调 name
  [Subject dealloc]
  == 正常退出 ==
```

没有异常。换成 `automaticallyNotifiesObserversForKey:` 返回 NO 的类再跑一遍。按 release note 的第一个条件，这种情况本该还抛：

```text
场景 1：手动通知的对象带着观察者去死
  注册完，isa = ManualNotify（没有中间类）
    >> 收到 name
  对象已释放，进程还在。
  == 正常退出 ==
```

也没抛。这一条超出了 Apple 白纸黑字承诺的范围，我不建议依赖。

再看更经典的那个方向：观察者先死，被观察对象继续改值。

```text
场景 1：观察者先 dealloc，被观察对象继续设值
    >> 回调 name
  [Ob 0x600000004060 dealloc]
  观察者已死: <NSKeyValueObservance 0x600000c00900: Observer: 0x0, Key path: name, ...>
  现在设值 —— 教科书说这里会崩：
  设了 5000 次，还活着。isa = NSKVONotifying_Subject
```

**观察记录还在，但 `Observer` 字段变成了 `0x0`。** 观察者被释放的那一刻，KVO 把指向它的那个指针清零了，之后通知静默跳过。这是 weak 引用的行为特征，不是野指针。我又加了一步：让观察者死掉之后疯狂分配两万个同类对象去抢那块内存，抢到了也照样不崩。

KVO 对两边都不持有，这是佐证：

```text
注册前 ob 引用计数 = 1
注册后 ob 引用计数 = 1
移除后 ob 引用计数 = 1
注册前 r  引用计数 = 1
注册后 r  引用计数 = 1
```

这条同样没有官方文档，Apple 的 Guide 里那段"会触发内存访问异常"还在原地。

有一件事我做不到，得说清楚：这台机器上只有 iOS 18.3 / 18.4 / 26.5 三个模拟器运行时，全都在 iOS 11 之后。所以"观察者是 zeroing weak"这条我只能证明它现在成立，没法定位它是哪个版本引入的，也没法验证 release note 里那两个条件在今天是否仍然是必要条件（上面手动通知那一组就已经超出了承诺范围）。

> 待补：找一个 iOS 10 或更早的运行时/设备，把第八节这三组程序原样跑一遍，确认行为分界点。在那之前，"iOS 11 起被观察对象侧会自动清理"这句只有 release note 背书，"观察者侧是弱引用"这句只有当前系统的实测背书。

我给自己划的线是：**行为变了，纪律不变。** 理由不是怕崩溃，是别的东西。

### 还在崩的那些

配对纪律该守还是得守，因为下面这几条一条都没放松。

移除一个没注册的观察者：

```text
从未注册就移除 -> NSRangeException
  Cannot remove an observer <Watcher 0x60000000c030> for the key path "name" from <Person 0x600000212860> because it is not registered as an observer.
用错 context 移除 -> NSRangeException（同一条 reason）
再按正确 context 移除：没抛异常
```

`context` 对不上等同于没注册过。注意 KVO 不会因为你少移除而报警，只会因为你多移除而崩。这个方向性很关键：真正会炸的是重复移除。

重复注册会实打实收到多份通知：

```text
注册两次，设值一次：
  [通知] key=name new=twice
  [通知] key=name new=twice
移除一次后再设值：
  [通知] key=name new=again
移除两次后再设值：           // 干净了
```

一个界面 push 两次、`viewWillAppear` 里注册而 `dealloc` 里移除，就会走到这里。回调跑两遍带来的数据错乱，比崩溃难查得多。

还有一个几乎所有人都写错过的：

```text
给一个没实现回调的 NSObject 发通知：
-> NSInternalInconsistencyException
reason: <NSObject: 0x60000000c040>: An -observeValueForKeyPath:ofObject:change:context: message was received but not handled. | Key path: leaf | Observed object: <Root: 0x60000000c000> | ...
```

`NSObject` 的默认实现是抛异常。而官方又要求"遇到不认识的 context 要调 super"：

> the observer should always call the superclass's implementation of `observeValueForKeyPath:ofObject:change:context:` when it does not recognize the context, because this means a superclass has registered for notifications as well.

这两条合起来的意思是：如果你的观察者直接继承 `NSObject`，无条件调 super 就是在给自己埋一颗雷。只有当你确定父类也在观察东西的时候才该往上传。

### context 为什么必须用

Mike Ash 2008 年那篇讲得最透，我摘一段：

> The context pointer is useless. […] you can't tell if your superclass will be interested in the notification by examining the key path or the object […] You must create a unique pointer that your superclass can't possibly be using.

论证链是这样的：所有回调挤在 `observeValueForKeyPath:` 一个入口；父类完全可能注册了一模一样的 object + keyPath（`NSObject` 实现 bindings 时自己就在观察东西）；于是 `(object, keyPath)` 这个二元组没法唯一标识"这条是给我的"；只剩 context 能当身份令牌。而它一旦被征用成身份令牌，就再也不能承载它名字暗示的那种"上下文数据"了。

惯用写法是 NSHipster 推广的：

```objc
static void *PersonNameContext = &PersonNameContext;
```

用变量自己的地址当值，全局唯一，父类不可能撞上。

---

## 九、Swift 侧：`@objc dynamic` 到底在解决什么

Swift 的 `observe(\.keyPath)` 看着是另一套 API，底下还是同一套东西。

```text
观察前 object_getClass = swift_kvo.Model
观察后 object_getClass = ..NSKVONotifying_swift_kvo.Model
观察后 type(of:)       = Model
```

还是 `NSKVONotifying_` 前缀（前面那两个点是 Swift 的命名修饰）。`type(of:)` 走 `-class`，同样看不到中间类。

`@objc dynamic` 里两个修饰符各管一件事，可以分开测。给一个只有 `@objc`、没有 `dynamic` 的属性注册字符串 KVO：

```text
给 @objc（无 dynamic）属性设值：
  ↑ 上面有没有回调？        // 没有
用 KVC 给同一个属性设值：
  [字符串 KVO 回调] notDynamic change=[kind: 1, new: KVC 改的]
```

同一个属性，Swift 里直接赋值没通知，`setValue:forKey:` 有通知。看方法列表就明白了：

```text
当前类自己的方法：["setNotDynamic:", "setName:", "class", "dealloc", "_isKVOA"]
```

`setNotDynamic:` 被覆写了，但 Swift 的直接赋值不走 `objc_msgSend`，编译器直接调静态派发的那个实现，绕过了中间类。`dynamic` 的作用就是强制这次派发走消息发送。

`@objc` 管的是另一头。Swift overlay 的第一步是把 KeyPath 降级成字符串：

```swift
guard let keyPathString = keyPath._kvcKeyPathString else {
    fatalError("Could not extract a String from KeyPath \(keyPath)")
}
```

编译器只在整条 KeyPath 每一段都对 ObjC 可见时才填 `_kvcKeyPathString`。所以纯 Swift 属性不是"KVO 不支持"，是压根走不到 KVO。

`NSKeyValueObservation` 解决的是配对问题：

```text
token 出作用域后设值：
  ↑ 没有 [临时 token 回调] 就说明 NSKeyValueObservation 析构时自动注销了
t2.invalidate() 之后：不再回调
```

deinit 自动 `invalidate()`，`invalidate()` 幂等。它的实现里还有个细节：`context` 传的是 nil，靠"每个观察配一个独立 Helper 对象"来消歧。这正是 Mike Ash 2008 年在 MAKVONotificationCenter 里的方案。抱怨提出九年之后，Apple 用了同一个思路。

`observe` 没有标 `@discardableResult`，所以不接返回值会有编译警告。这是 API 在提醒你必须持有 token。

---

## 十、几个已经不准的说法

- "KVC 绕过 setter 直接写内存。" 只要 `set<Key>:` 存在就一定调它，ivar 是兜底。
- "setter 搜索链是 `set<Key>:` 和 `_set<Key>:` 两步。" 实测还有第三步 `setIs<Key>:`，两份 Apple 官方材料都没写。反过来，`_set<Key>:` 是 `takeValue:forKey:` 的兼容遗留，10.3 起已废弃。
- "`@sum` / `@avg` 把值转成 double。" 文档这么写，实测返回 `NSDecimalNumber`，走十进制定点。元素上收到的选择器是 `decimalValue`。
- "中间类里给被观察属性生成了一个调 super 的 setter。" 一行代码都没生成。槽位里填的是 Foundation 里现成的 `_NSSet<Type>ValueAndNotify`，两个毫不相干的类拿到的是同一个函数地址，key 靠 `_cmd` 从选择器名字反推。
- "KVO 只是给 setter 加了一层壳，别的不受影响。" 覆写 `-class` 会让 `hasCustomCore()` 置位，`isKindOfClass:` 的快速路径整个关掉，实测慢 5 到 8 倍，而被测的根本不是那个属性。
- "注册 KVO 一定会 isa-swizzle。" `automaticallyNotifiesObserversForKey:` 对该 key 返回 NO 时，中间类根本不生成。
- "移除观察者后 isa 不会还原 / 中间类会被销毁。" 实例的 isa 会还原成原类；Class 对象常驻并在下次观察时复用。两个说法各错一半。
- "忘了移除观察者会野指针崩溃。" iOS 11 起被观察对象侧由 KVO 自动清理（有官方 release note），观察者侧实测是 zeroing weak，`Observer` 字段被置 0。真正还会崩的是重复移除（`NSRangeException`）和重复注册导致的重复回调。
- "KVO 只观察属性，不观察成员变量。" 直接写 ivar 不触发；用 `setValue:forKey:` 写同一个 ivar 会触发。触发点在消息派发，不在"是不是属性"。
- "`observationInfo` 存在关联对象里。" 官方文档写的是一张以对象地址为 key 的全局字典。
- "回调里要无条件调 `[super observeValueForKeyPath:...]`。" `NSObject` 的默认实现直接抛 `NSInternalInconsistencyException`。只在确定父类也注册了观察时才往上传。
- "`NSUndefinedKeyException` 和 `NSUnknownKeyException` 是两个异常。" 是一个。常量名是前者，字符串值是后者。

---

## 总结

KVC 是一套字符串到访问器的查表规则，两条链各四五步，记住"方法优先于 ivar、下划线优先于裸名"就能推出大半。真正会咬人的是链末端那一步：找不到 setter 时它直接 retain 进 ivar，`copy` 语义丢了、`readonly` 拦不住、`weak` 行为文档没定义。

KVO 是在这套规则上插了两个拦截点。一个是被 swizzle 的 setter，靠 isa 指向中间类实现，实测里那族 `_NSSet<Type>ValueAndNotify` 就是它的真身；另一个在 `setValue:forKey:` 内部，不依赖 isa。理解了有两条路，"readonly 属性也能观察""KVC 改 ivar 也发通知""关掉自动通知就不生成子类"这三件看着矛盾的事就串起来了。

中间类这层壳收两笔费。第一笔是 setter 本身，一个观察者约两百倍。第二笔藏得深：覆写 `-class` 把核心选择器的快速路径关了，这个对象身上所有 `isKindOfClass:` 跟着慢 5 到 8 倍，和被观察的那个属性毫无关系。两笔都是模拟器上的量级，真机待补，但方向不会反。

至于移除观察者，我的结论是行为变了但纪律不变。Apple 确实在 iOS 11 兜住了被观察对象那一侧，观察者那一侧也实测不再崩，但重复移除照样 `NSRangeException`、重复注册照样收两份回调，而后者引起的数据错乱比崩溃难查得多。

最后说一句方法论。这篇里三条和文档相反的结论（`setIs<Key>:` 存在、`@sum` 走 decimal、关掉自动通知就不 swizzle），没有一条是读文章读出来的，都是先写一个尽可能干净的类，再看运行时到底做了什么。Foundation 不开源，但 `class_copyMethodList`、`dladdr`、`observationInfo` 这三样加起来，已经能把它剖开一大半。

下一篇 [[iOS 对象通信：delegate、通知、target-action 与 block 回调]]。

## 参考资料

### 官方

- [Key-Value Coding Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/index.html)：[Accessor Search Patterns](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/SearchImplementation.html) 是两条搜索链的出处，[Collection Operators](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/CollectionOperators.html) 里"converts it to a double"那句和实测对不上
- `Foundation/NSKeyValueCoding.h`：比归档文档精确得多，Compatibility notes 里记着 `_set<Key>:` 的废弃状态和 10.2/10.3 的 ivar 顺序变更。只有它写了 `NSUndefinedKeyException` 的字符串值是 `NSUnknownKeyException`
- [KVO Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html)：`keyPathsForValuesAffectingValueForKey:` 不支持 to-many 的限制、context 的官方建议、"调 super"那段都在这里。关于观察者释放会崩的那段已经过时
- [KVO Implementation Details](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/Articles/KVOImplementation.html)：全文只有四段，只说了 isa 指向 "an intermediate class"。`NSKVONotifying_` 前缀、`_isKVOA`、`class` 被覆写这些全是社区逆向的结论，不是官方说法
- [Foundation Release Notes for macOS 10.13 and iOS 11](https://developer.apple.com/library/archive/releasenotes/Foundation/RN-Foundation/index.html)：Relaxed Key-Value Observing Unregistration Requirements 一节，第八节引的原文
- [Using Key-Value Observing in Swift](https://developer.apple.com/documentation/swift/using-key-value-observing-in-swift)：只说要写 `@objc dynamic`，没解释为什么

### 经典

- [Mike Ash — Key-Value Observing Done Right](https://www.mikeash.com/pyblog/key-value-observing-done-right.html)：context 为什么必须用的原始论证，2008 年写的，逻辑至今没被推翻
- [Mike Ash — KVO Done Right: Take 2](https://www.mikeash.com/pyblog/friday-qa-2012-03-02-key-value-observing-done-right-take-2.html)：作者其实是 Gwynne Raskind。她盘点了四年里 KVO 修了什么，答案是三宗罪只解决了一条
- [NSHipster — Key-Value Observing](https://nshipster.com/key-value-observing/)：`static void *Context = &Context;` 这个惯用法的推广者
- [swift/stdlib/public/Darwin/Foundation/NSObject.swift](https://github.com/swiftlang/swift/blob/main/stdlib/public/Darwin/Foundation/NSObject.swift)：`NSKeyValueObservation` 的完整实现，三百来行，能看到 context 传 nil、Helper 挂关联对象、`invalidate()` 幂等这些细节

### 本地

- [[20 专题笔记/Runtime/Part 1 - 对象与类的本质]]：isa 与元类
- [[20 专题笔记/Runtime/Part 4 - Runtime 应用篇]]：KVO 动态子类的用法层面，本文只补机制和实测，不重复
- [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]]：`[self class]` 与 `object_getClass` 的分岔、`hasCustomCore` 与核心选择器，第五节那笔性能账接的是这里
- [[iOS 属性关键字：从所有权推导，而不是从类型名猜]]：`copy` 语义为什么会被 KVC 的 ivar 兜底路径绕过
- 早年自写：[【iOS】KVC总结](https://blog.csdn.net/2402_86720949/article/details/152734765)（搜索规则那一节的三步链是对的，Apple 文档反而漏了一步）· [【iOS】KVO](https://blog.csdn.net/2402_86720949/article/details/154543600)（第八节填的就是那篇照抄的"不移除就野指针崩溃"）

---

实验环境：Xcode 26.6（Apple clang 21），iPhoneSimulator 26.5 SDK，运行在 iOS 18.3 模拟器（arm64，Apple Silicon Mac），`clang -fobjc-arc -target arm64-apple-ios17.0-simulator`。Swift 侧用 `swiftc` 同 target 构建。符号表和那两句警告文案来自 iOS 18.3 运行时里的 Foundation 二进制，用 `nm -a` 和 `strings -a` 读出来，路径在 `/Library/Developer/CoreSimulator/Volumes/` 下面。

第八节那组 dealloc 实验我另外用 `-target arm64-apple-ios9.0-simulator` 和 `ios11.0` 各编了一份对照，结果完全一致，说明这个行为不是按 deployment target 门控的，是 Foundation 的运行时行为。

> 待真机补测：`_NSSet<Type>ValueAndNotify` 那组符号名和"观察者被释放后 `Observer` 字段置 0"这两条在真机上复现。把第五节和第八节的程序原样拿到设备上跑即可，`dladdr` 在真机同样可用。
