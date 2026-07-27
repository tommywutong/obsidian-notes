---
title: 【iOS】JSONModel 源码：Runtime 驱动的属性映射
published: 2026-07-27
description: NSArray<Item *> 里那个泛型参数，在 property_getAttributes 的返回值里一个字节都不剩。所以 JSONModel 只能拿尖括号里的协议名当类名的走私通道，而那个"协议"在运行时根本不存在。顺便把它比手写慢的 6.6 倍拆成五笔账。
tags:
  - iOS
  - Objective-C
  - Runtime
  - JSON
  - JSONModel
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 32
draft: true
---
# JSONModel 源码：Runtime 驱动的属性映射

先看两行东西。同一个类里挨着的两个属性，写法只差一点：

```objc
@property (nonatomic) NSArray <Kid> *protoOnly;            // 尖括号里是协议
@property (nonatomic) NSArray<Kid *> <Optional> *genOnly;  // 尖括号里是 ObjC 泛型
```

`property_getAttributes` 给出的字符串是：

```text
protoOnly  T@"NSArray<Kid>",&,N,V_protoOnly
genOnly    T@"NSArray<Optional>",&,N,V_genOnly
```

**第二行里的 `Kid *` 没了。ObjC 泛型是纯编译期检查，类型编码里一个字节都不留，运行时问不到。** 而第一行那个 `<Kid>` 留下来了，因为协议名是类型编码的一部分。

JSONModel 用协议名传类名，就是从这儿来的。它要在运行时知道"这个数组里装的是 Kid"，能拿到这条信息的通道只有一个。更离谱的是这个"协议"你只需要前向声明 `@protocol Kid;`，不用定义。`NSProtocolFromString(@"Kid")` 返回的是 NULL。`Kid` 类也并不遵守 `<Kid>`。它从头到尾只是一个借着编译器语法混进类型编码里的字符串。

这篇把 JSONModel 的属性自省、缓存、KVC 调用点全部走一遍，重点在两件事：它慢在哪，以及它哪几种失败是不出声的。开头先把三个数摆出来，都是自己跑的。解析一个中等复杂度 JSON，它比手写 `initWithDictionary:` 慢 6.6 倍。这 6.6 倍里有 42% 能用五个小补丁删掉。一个类第一次被解析，比之后每一次贵 124 倍。

性能对比留给 [[iOS YYModel 源码：为什么比 JSONModel 快]]，`NSJSONSerialization` 和归解档那一层留给 [[iOS 序列化：NSCoding、NSSecureCoding 与 JSON 三条路]]。这篇只讲 JSONModel 自己。

先说选型信息。这个库的最后一次提交是 2018-09-19，同一天打了 v1.8.0 的 tag。CHANGELOG 里那一行写的是 "support for Swift 3"。按年数提交：

```text
 166  2012
 268  2013
  91  2014
  87  2015
 107  2016
   8  2017
   5  2018
```

2016 年之后就停了。

---

## 一、先把它编起来

不用 Xcode 工程，不用模拟器。核心就五个 `.m`，直接编成 macOS 命令行程序：

```shell
clang -fobjc-arc -framework Foundation \
  -IJSONModel/JSONModel -IJSONModel/JSONModelTransformations \
  JSONModel/JSONModel/*.m JSONModel/JSONModelTransformations/*.m \
  lab/models.m lab/prog.m -o out && ./out
```

`JSONModelNetworking/` 那三个文件要链 SystemConfiguration，跟属性映射没关系，不编。

编译本身就有信息量。当前 clang 下五个文件一共七条告警：

```text
JSONModel.m:434  warning: incompatible operand types ('NSString *' and 'Class')
JSONModel.m:992  warning: incompatible operand types ('Class' and 'NSString *')
JSONModel.m:1354 warning: 'unarchiveObjectWithData:' is deprecated: first deprecated in macOS 10.14
JSONModel.m:1355 warning: 'archivedDataWithRootObject:' is deprecated: first deprecated in macOS 10.14
JSONValueTransformer.m:27  warning: duplicate key in dictionary literal  (三处)
```

第 434 行那个类型不匹配就在拼转换器 selector 的地方，第四节会碰到。1354/1355 是 `copyWithZone:` 的实现。它把对象归档再解档一遍来做拷贝，用的是 iOS 12 就废弃的非安全接口。

`JSONValueTransformer.m:23` 的重复 key 值得单独看一眼，因为它是个真 bug，只是恰好没炸：

```objc
_primitivesNames = @{@"f":@"float", @"i":@"int", @"d":@"double", @"l":@"long", @"B":@"BOOL", @"s":@"short",
                     @"I":@"unsigned int", @"L":@"usigned long", @"q":@"long long", @"Q":@"unsigned long long", ...
                     @"I":@"NSInteger", @"Q":@"NSUInteger", @"B":@"BOOL",
                     @"@?":@"Block"};
```

`@"I"` 和 `@"Q"` 各出现两次，后写的赢。于是编码 `I`（`unsigned int`）被映射成字符串 `"NSInteger"`。编码 `Q` 映射成 `"NSUInteger"`。这两个字符串都在 `allowedPrimitiveTypes` 白名单里，属性照样能通过校验。还有个更好玩的：`@"L":@"usigned long"` 是拼错的。而白名单里那一项拼错得一模一样。两个错误正好对上了。

---

## 二、属性自省：一个字段一个字段拆

入口是 `JSONModel.m:530` 的 `__inspectProperties`。它沿着继承链往上走，一直走到 `JSONModel` 本身为止：

```objc
    // inspect inherited properties up to the JSONModel class
    while (class != [JSONModel class]) {
        unsigned int propertyCount;
        objc_property_t *properties = class_copyPropertyList(class, &propertyCount);
```

拿到每个属性之后，第一件事是取属性字符串：

```objc
            const char *attrs = property_getAttributes(property);
            NSString* propertyAttributes = @(attrs);
            NSArray* attributeItems = [propertyAttributes componentsSeparatedByString:@","];

            //ignore read-only properties
            if ([attributeItems containsObject:@"R"]) {
                continue; //to next property
            }
```

### 属性字符串长什么样

这是我实际打出来的一个 model 类的全部属性：

```text
orderId      Tq,N,V_orderId
country      T@"NSString",&,N,V_country
isInEurope   TB,N,V_isInEurope
totalPrice   Td,N,V_totalPrice
note         T@"NSString<Optional>",&,N,V_note
homepage     T@"NSURL<Optional>",&,N,V_homepage
mainItem     T@"Item",&,N,V_mainItem
items        T@"NSArray<Item>",&,N,V_items
raw          T@"NSMutableArray<Optional>",&,N,V_raw
copied       T@"NSString",C,N,V_copied
weakling     T@"<Optional>",W,N,V_weakling
on           TB,N,GisOn,V_on
```

逗号分段，每段一个字母开头：

- `T` 后面是类型编码。`q` = `long long`，`NSInteger` 在 64 位上就是它。`B` = `BOOL`，`d` = `double`。`@"NSString"` 是对象类型，带类名。
- `&` = retain，`C` = copy，`W` = weak。三者都没有就是 assign。`copied` 那行的 `C` 和 `weakling` 那行的 `W` 是我特意加进去对照的。
- `N` = nonatomic。
- `G<name>` = 自定义 getter，`on` 那行的 `GisOn` 来自 `getter=isOn`。
- `R` = readonly。
- `V_<name>` = 后备 ivar 名。

JSONModel 只认其中三样：`R`（直接跳过）、`T` 后面的类型、类名后面尖括号里的协议列表。`C` / `&` / `W` 它一概不看。它写属性走 KVC，所有权语义交给合成出来的 setter。

### 类型是怎么判出来的

一台 `NSScanner`，三个分支。先跳到 `T`：

```objc
            scanner = [NSScanner scannerWithString: propertyAttributes];
            [scanner scanUpToString:@"T" intoString: nil];
            [scanner scanString:@"T" intoString:nil];

            //check if the property is an instance of a class
            if ([scanner scanString:@"@\"" intoString: &propertyType]) {

                [scanner scanUpToCharactersFromSet:[NSCharacterSet characterSetWithCharactersInString:@"\"<"]
                                        intoString:&propertyType];

                p.type = NSClassFromString(propertyType);
                p.isMutable = ([propertyType rangeOfString:@"Mutable"].location != NSNotFound);
                p.isStandardJSONType = [allowedJSONTypes containsObject:p.type];
```

对象类型：扫到 `@"` 就一路读到 `"` 或者 `<` 为止，中间那段就是类名。`isMutable` 的判定是在类名里搜 `"Mutable"` 这个子串，字符串匹配，没有别的依据。

结构体走 `{` 分支，只记 `structName`，`isStandardJSONType` 直接置 NO。

剩下的当基本类型，查 `primitivesNames` 表把 `q` 之类的编码翻成名字，不在白名单里就当场抛异常：

```objc
                if (![allowedPrimitiveTypes containsObject:propertyType]) {
                    @throw [NSException exceptionWithName:@"JSONModelProperty type not allowed"
                                                   reason:[NSString stringWithFormat:@"Property type of %@.%@ is not supported by JSONModel.", self.class, p.name]
                                                 userInfo:nil];
                }
```

这是个 `@throw`，不是 `NSError`。我拿一个 `long double` 属性试了：

```text
  long double 属性 -> 抛 JSONModelProperty type not allowed: Property type of Weird.ld is not supported by JSONModel.
```

`long double` 的编码是 `D`，映射表里没有，`propertyType` 变成 nil，白名单查不到，直接炸。炸的时机是第一次解析这个类，跟 JSON 里有没有这个字段无关。[[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]] 那篇里，`long double` 也是唯一一个没有 `_NSSet<Type>ValueAndNotify` 的类型。两处都是"落在类型表之外"。

### Optional 和泛型走的是同一个槽

类名扫完之后，紧跟着一个 while 循环读协议列表：

```objc
                //read through the property protocols
                while ([scanner scanString:@"<" intoString:NULL]) {

                    NSString* protocolName = nil;

                    [scanner scanUpToString:@">" intoString: &protocolName];

                    if ([protocolName isEqualToString:@"Optional"]) {
                        p.isOptional = YES;
                    } else if([protocolName isEqualToString:@"Index"]) {
                        p.isIndex = YES;
                        ...
                    } else if([protocolName isEqualToString:@"Ignore"]) {
                        p = nil;
                    } else {
                        p.protocol = protocolName;
                    }

                    [scanner scanString:@">" intoString:NULL];
                }
```

三个保留名字，其余的全存进 `p.protocol` 当类名用。所以 `<Optional>`、`<Ignore>` 和 `<Kid>` 挤在同一个语法位置上，靠字符串比对区分。写 `NSArray <Kid, Optional> *` 时属性字符串是 `T@"NSArray<Kid><Optional>"`。循环两轮，两件事都办了。

`Optional` / `Ignore` / `Index` 在 `JSONModel.h:22-48` 是真用 `@protocol` 声明过的空协议。还配了一句：

```objc
@interface NSObject (JSONModelPropertyCompatibility) <Optional, Ignore>
@end
```

让所有对象都"遵守"这两个协议，纯粹为了让编译器闭嘴。你自己的 model 协议没这待遇，所以要写一句 `@protocol Kid;` 前向声明。

### 前向声明的协议根本不存在

这一步我原本以为只是省事。测完才发现它是必需的：

```text
  NSClassFromString(@"Kid") = 0x1041c1390   NSProtocolFromString(@"Kid") = 0x0
  Kid 类是否真的遵守 <Kid> 协议? 否
```

`@protocol Kid;` 只是一句前向声明，没有 `@end`，运行时里压根没有这个协议对象。`NSProtocolFromString` 返回 NULL。JSONModel 也从来不去查它，`__transform:` 第一行是：

```objc
    Class protocolClass = NSClassFromString(property.protocol);
```

拿协议名去查类。整套机制里唯一被用到的是那个字符串本身。

再看一遍开头那个对照的实际后果。同一份 JSON、四个写法不同的属性：

```text
  protoOnly  T@"NSArray<Kid>"           -> protoOnly[0] 的类 = Kid
  genOnly    T@"NSArray<Optional>"      -> genOnly[0]   的类 = __NSSingleEntryDictionaryI
  both       T@"NSArray<Kid>"           -> both[0]      的类 = Kid
  protoOpt   T@"NSArray<Kid><Optional>" -> protoOpt[0]  的类 = Kid
```

只写 ObjC 泛型 `NSArray<Kid *> *` 的那个，数组里躺着的还是原始字典，而且 `error` 是 nil。没有告警，没有报错，编译器还觉得你类型写得很规范。这是我在这个库里见过最容易踩的一个坑，因为 Xcode 会主动提示你给容器补泛型参数。

`NSArray<Kid *> <Kid> *` 两个都写是官方推荐姿势，属性字符串里泛型那部分依然被擦掉，留下的还是 `<Kid>`。

### 自省的最后一步：拼一堆 selector

属性信息填完之后，`JSONModel.m:671-702` 还会给每个属性预先查一遍自定义存取器：

```objc
                SEL getter = NSSelectorFromString([NSString stringWithFormat:@"JSONObjectFor%@", name]);
                if ([self respondsToSelector:getter])
                    p.customGetter = getter;
                ...
                for (Class type in allowedJSONTypes)
                {
                    NSString *class = NSStringFromClass([JSONValueTransformer classByResolvingClusterClasses:type]);
                    if (p.customSetters[class]) continue;
                    SEL setter = NSSelectorFromString([NSString stringWithFormat:@"set%@With%@:", name, class]);
                    if ([self respondsToSelector:setter])
                        p.customSetters[class] = [NSValue valueWithBytes:&setter objCType:@encode(SEL)];
                }
```

`allowedJSONTypes` 有九个类，去重后剩五个。再加一个 generic setter 和一个 getter。于是每个属性要 `NSSelectorFromString` + `respondsToSelector:` 七次。十个属性就是七十次。`NSSelectorFromString` 会把没见过的名字注册进全局选择器表，那是要加锁的。这笔钱下一节会付。

---

## 三、缓存：`__inspectProperties` 到底跑几次

自省结果存在哪，`JSONModel.m:713` 写得很直白：

```objc
    objc_setAssociatedObject(
                             self.class,
                             &kClassPropertiesKey,
                             [propertyIndex copy],
                             OBJC_ASSOCIATION_RETAIN // This is atomic
                             );
```

关联对象，挂在 Class 对象上，不是实例上。key 是四个文件级静态变量的地址：

```objc
static const char * kMapperObjectKey;
static const char * kClassPropertiesKey;
static const char * kClassRequiredPropertyNamesKey;
static const char * kIndexPropertyNameKey;
```

取的时候在 `__setup__` 里判一次有没有：

```objc
-(void)__setup__
{
    //if first instance of this model, generate the property list
    if (!objc_getAssociatedObject(self.class, &kClassPropertiesKey)) {
        [self __inspectProperties];
    }
```

`__setup__` 是 `-init` 调的，所以每 new 一个实例就查一次关联对象，命中就跳过自省。

### 设计一个实验证明它真的生效

光读代码不算数。我用 [[iOS Method Swizzling：正确姿势、+load 时机与那些坑]] 里的办法，在 `JSONModel` 上挂个 category 把 `__inspectProperties` 换掉计数：

```objc
@implementation JSONModel (Spy)
- (void)spy__inspectProperties { inspectCalls++; [self spy__inspectProperties]; }
+ (void)load {
    Method a = class_getInstanceMethod(self, @selector(__inspectProperties));
    Method b = class_getInstanceMethod(self, @selector(spy__inspectProperties));
    method_exchangeImplementations(a, b);
}
@end
```

两组对照。A 组：同一个类解析 1000 次。B 组：1000 个结构完全相同的类各解析一次。B 组的类用 `objc_allocateClassPair` + `class_addIvar` + `class_addProperty` 现造，创建成本不计入计时。各 10 个属性，跑七轮：

```text
轮 1  同一类 1000 次:   10.27 ms (__inspectProperties 1 次)   1000 个类各 1 次:  282.63 ms (__inspectProperties 1000 次)
轮 2  同一类 1000 次:    6.05 ms (__inspectProperties 1 次)   1000 个类各 1 次:  252.37 ms (__inspectProperties 1000 次)
轮 3  同一类 1000 次:    5.75 ms (__inspectProperties 1 次)   1000 个类各 1 次:  242.68 ms (__inspectProperties 1000 次)
轮 4  同一类 1000 次:   15.27 ms (__inspectProperties 1 次)   1000 个类各 1 次:  269.62 ms (__inspectProperties 1000 次)
轮 5  同一类 1000 次:    6.04 ms (__inspectProperties 1 次)   1000 个类各 1 次:  257.52 ms (__inspectProperties 1000 次)
轮 6  同一类 1000 次:    6.13 ms (__inspectProperties 1 次)   1000 个类各 1 次:  283.72 ms (__inspectProperties 1000 次)
轮 7  同一类 1000 次:    6.48 ms (__inspectProperties 1 次)   1000 个类各 1 次:  256.32 ms (__inspectProperties 1000 次)

中位数  同一类: 6.13 ms    不同类: 257.52 ms    倍数: 41.99x
```

计数器直接给出答案。1000 次解析，自省一次。缓存粒度是类，不是实例。

倍数是 42。反过来算，一个 10 属性的类第一次自省摊到 `(257.52 - 6.13) / 1000 ≈ 0.25 ms`。

### 真实类的冷热比

动态造的类可能有偏差，所以我拿编译期定义的类又测了一遍。一个 12 属性的 `Order`，内含一个嵌套 model 和两个 model 数组。第一次解析要连带自省 `Order` / `Item` / `Tag` 三个类。开 15 个进程，每个进程先量第一次，再量之后 2000 次的中位数：

```text
冷 中位数 2.483 ms  区间 [1.649, 7.748]
热 中位数 0.0200 ms 区间 [0.0200, 0.0210]
冷/热 = 124x
```

热的那一列稳到小数点后两位，冷的那一列抖得厉害，这符合"一次性成本里混着选择器注册和内存分配"的预期。

这个 124 倍有个很实际的推论。启动后第一屏数据往往就是第一次解析，模型类越多、属性越多，这笔钱越集中地砸在首屏。想摊掉它很容易。启动后找个空闲时机，把主要 model 各 `[[X alloc] init]` 一下，`-init` 里就会触发自省。

### 属性的处理顺序不是声明顺序

`__properties__` 的返回值是：

```objc
    NSDictionary* classProperties = objc_getAssociatedObject(self.class, &kClassPropertiesKey);
    if (classProperties) return [classProperties allValues];
```

`NSDictionary` 的 `allValues`。同一个类跑四遍看看：

```text
extra note raw tags homepage isInEurope dialCode orderId items totalPrice country mainItem
extra note raw tags homepage isInEurope dialCode orderId items totalPrice country mainItem
extra note raw tags homepage isInEurope dialCode orderId items totalPrice country mainItem
extra note raw tags homepage isInEurope dialCode orderId items totalPrice country mainItem
```

稳定，但和声明顺序毫无关系，而且没有任何契约保证它稳定。你要是写了自定义 setter，指望另一个属性已经赋好值，这个假设随时会翻车。

---

## 四、它在哪里用 KVC，在哪里不用

`__importDictionary:withKeyMapper:validation:error:` 是导入的主循环。从 `JSONModel.m:272` 到 477，两百行，一个大 for 套一串 if。写属性一共五个出口，全是 KVC：

```objc
                if (jsonValue != [self valueForKey:property.name]) {
                    [self setValue:jsonValue forKey: property.name];      // 0) 基本类型
                }
...
                if (![value isEqual:[self valueForKey:property.name]]) {
                    [self setValue:value forKey: property.name];          // 1) 嵌套 model
                }
...
                    if (![jsonValue isEqual:[self valueForKey:property.name]]) {
                        [self setValue:jsonValue forKey: property.name];  // 3.1) 标准 JSON 类型
                    }
```

每个出口都是同一个形状：先用 `valueForKey:` 把当前值读出来比一下，不相等才写。我用 swizzle 数了一次解析的调用次数。测试用的 JSON 有 12 个属性、1 个嵌套 model、3 个 Item、2 个 Tag：

```text
  valueForKey:      25 次
  setValue:forKey:  25 次
```

25 和 25。我照着源码手工数了一遍分支，也是 25，两边对上了，说明计数器没漏。（同一段代码里还有 `[dict valueForKeyPath:]`，我第一版把它一起算了，结果只数到 1 次。查下去发现是仪器问题。`NSDictionary` 自己覆写了 `valueForKeyPath:`，我 swizzle 的是 `NSObject` 上那份，根本没走到。）

那个"先读再写"的比较是有代价的。KVC 读一次属性和直接调 getter 的差距：

```text
200000 次取值，中位数：
  [order valueForKey:@"country"]     27.20 ms
  order.country                       1.09 ms   (24.9x)
```

写一侧同理：

```text
200000 次: setValue:forKey: 18.96 ms   直接 setter 1.27 ms   15x
```

25 次 KVC 读、25 次 KVC 写，这就是 JSONModel 每解析一个对象的固定开销。

### 唯独转换器不走 KVC

有意思的是它对转换器的调用方式完全不同。`JSONModel.m:452`：

```objc
                    if (foundCustomTransformer) {
                        IMP imp = [valueTransformer methodForSelector:selector];
                        id (*func)(id, SEL, id) = (void *)imp;
                        jsonValue = func(valueTransformer, selector, jsonValue);
```

拿 IMP 转成函数指针直接调，绕开 `objc_msgSend`。`__customSetValue:forProperty:` 也一样。所以同一份代码里，写属性用最慢的 KVC，调转换器用最快的直接跳转。这个不对称我猜是历史原因：转换器返回值类型不确定，走 `performSelector:` 会有 ARC 告警，作者索性用了函数指针。

### selector 是拼出来的

转换规则的核心在 `JSONModel.m:428-449`。原文照抄：

```objc
                    Class sourceClass = [JSONValueTransformer classByResolvingClusterClasses:[jsonValue class]];

                    //build a method selector for the property and json object classes
                    NSString* selectorName = [NSString stringWithFormat:@"%@From%@:",
                                              (property.structName? property.structName : property.type), //target name
                                              sourceClass]; //source name
                    SEL selector = NSSelectorFromString(selectorName);

                    //check for custom transformer
                    BOOL foundCustomTransformer = NO;
                    if ([valueTransformer respondsToSelector:selector]) {
                        foundCustomTransformer = YES;
                    } else {
                        //try for hidden custom transformer
                        selectorName = [NSString stringWithFormat:@"__%@",selectorName];
                        selector = NSSelectorFromString(selectorName);
                        if ([valueTransformer respondsToSelector:selector]) {
                            foundCustomTransformer = YES;
                        }
                    }
```

`<目标类型>From<源类型>:`。源类型先经过 `classByResolvingClusterClasses:` 归一化。`__NSCFString`、`NSTaggedPointerString` 这些全收敛成 `NSString`。`__NSCFNumber` 收敛成 `NSNumber`。JSON 里是数字、属性是 `NSString`，就去找 `NSStringFromNSNumber:`。属性是 `NSURL`，就去找 `NSURLFromNSString:`。`JSONValueTransformer.m` 里那二十来个方法就是这么被调到的。

找不到就在前面加两个下划线再找一遍，那是"隐藏转换器"，日期和空字典兼容用的。两遍都找不到，报 `errorInvalidDataWithTypeMismatch`。

第 434 行那句 `property.structName ? property.structName : property.type` 就是编译器抱怨的地方。一个是 `NSString *`，一个是 `Class`，三目两边类型不同。能跑是因为 `%@` 对两者都成立。

想加自定义转换，办法就是给 `JSONValueTransformer` 开个 category 补一个名字对得上的方法。整个"扩展点"就是选择器命名约定，没有注册表，没有协议。

### 所以类型不匹配为什么大多不崩

KVC 之前有两道防线。第一道是 `isNull()`：

```objc
extern BOOL isNull(id value)
{
    if (!value) return YES;
    if ([value isKindOfClass:[NSNull class]]) return YES;
    return NO;
}
```

`nil` 和 `NSNull` 一视同仁，命中就跳过这个属性（可选）或者报错（必填）。所以 `NSNull` 永远不会被塞进一个属性里。第二道是 `allowedJSONTypes` 白名单，把不可能出现在 JSON 里的类型挡在外面。

我拿一个 `int count` 属性把四种输入都试了一遍：

```text
  Prim   {"count":7}              -> count = 7
  Prim   {"count":"7"}            -> count = 7
  Prim   {"count":[1,2]}          -> * 抛异常 NSInvalidArgumentException: -[__NSArrayI intValue]: unrecognized selector sent to instance
  Prim   {"count":{"a":1}}        -> * 抛异常 NSInvalidArgumentException: -[__NSSingleEntryDictionaryI intValue]: unrecognized selector
```

字符串塞进 `int` 属性不但没崩，还转对了。这不是 JSONModel 干的，是 KVC 本身的行为。单独验证一遍，一个普通 `NSObject` 子类：

```text
  setValue:@"7" forKey:@"count"   -> count = 7
  setValue:@"abc" forKey:@"count" -> count = 0
  setValue:@[@1] forKey:@"count"  -> 抛异常 NSInvalidArgumentException
  setValue:nil forKey:@"count"    -> 抛异常 NSInvalidArgumentException: could not set nil as the value for the key count.
```

KVC 给标量属性赋值时会向对象要 `intValue` 这类原语。`NSString` 恰好有，所以能过。`"abc"` 的 `intValue` 是 0，于是静默变成 0。数组和字典没有 `intValue`，直接 unrecognized selector。

**标量属性 + JSON 里给了数组或对象，是 JSONModel 唯一一条会抛 `NSException` 的数据路径，而且它绕过 `error:` 参数。** 你写了 `initWithString:error:` 并认真判了 error，仍然会崩在这里。后端把一个 `count` 字段从数字改成对象，App 就整片挂掉。

对象类型的属性没这个问题，因为它走的是拼 selector 那条路，找不到转换器就老老实实返回 `NSError`：

```text
  Str    {"s":[1,2]}     -> nil, Invalid JSON data. The JSON type mismatches the expected type.
  Str    {"s":{"a":1}}   -> nil, Invalid JSON data. The JSON type mismatches the expected type.
```

### 一个 KVC 的坑在这里恰好不成立

[[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]] 里测过，`setValue:forKey:` 找不到 setter 落到 ivar 时是 retain 而不是 copy，声明成 `copy` 的属性会丢掉 copy 语义。JSONModel 这里不会中招。model 类的属性都是正常 `@property`，编译器合成了 `set<Key>:`。KVC 命中搜索链第一步就返回，根本走不到 ivar 那一层。`copy` 的属性拿到的确实是副本。

反过来，`readonly` 属性在 `__inspectProperties` 就被 `containsObject:@"R"` 过滤掉了。JSONModel 压根不认识它。所以那篇里"readonly 属性能被 KVC 改"的路径这里也用不上。实测确认：

```text
Box {"kids":[],"ro":"X","skipped":"Y"} -> ro 和 skipped 都没被赋值
Box 被 JSONModel 记住的属性： extra str kids
```

---

## 五、6.6 倍花在哪

这是全文我最想讲清楚的一节。

先给基准。JSON 有 12 个字段，含 1 个嵌套 model、3 个 Item、2 个 Tag、1 个字典、1 个数组。解析 5000 次，跑 9 轮取中位数。对照组是我手写的 `initWithDictionary:`，做完全一样的事，包括数字转字符串、字符串转 `NSURL`、递归子 model：

```text
中位数（5000 次）: JSONModel 126.25 ms   手写 19.14 ms   NSJSONSerialization 24.95 ms
单次: JSONModel 25.2 us   手写 3.8 us   倍数 6.6x
```

九轮的倍数是 6.5 / 6.6 / 6.5 / 6.7 / 6.6 / 6.6 / 6.5 / 6.6 / 6.8，几乎不动。

第三列那个数我一开始没打算测，测完发现挺有说服力。`NSJSONSerialization` 把整段 JSON 文本解析成字典只要 5.0 us。JSONModel 把这个现成的字典搬进对象，要 25.2 us。**解析 JSON 本身，比"字典搬进 model"这一步便宜五倍。**

### 归因方法

光知道 6.6 倍没用。得知道钱花在哪。我的做法是给 `JSONModel.m` 打补丁，一次只改一处，各自和原版比。

这里有个坑我先撞了一次。不同进程之间有 3% 到 8% 的漂移（热节流、其他负载），有几个补丁的效果就在这个量级里，直接比绝对毫秒数会得出相反的结论。解决办法是同一个进程里同时测手写版本，用比值做归一化，再跨 5 个进程取中位数。

五个补丁：

1. 属性没有自定义 setter 时，跳过 `__customSetValue:forProperty:`。原版每个属性都调它。而它第一句就是这个：`NSStringFromClass([JSONValueTransformer classByResolvingClusterClasses:[value class]])`。一次类继承链遍历，加一次字符串构造。
2. `jsonKeyPath` 里没有点时直接用下标取值，不走 `valueForKeyPath:`。
3. 去掉五处"写之前先 `valueForKey:` 读一次做比较"。
4. 跳过 `__doesDictionary:matchModelWithKeyMapper:` 里每次解析都重建的必填 key 校验。
5. `__properties__` 里 `[classProperties allValues]` 每次现建数组，改成按类缓存。

```text
每个变体跑 5 个进程，每进程 9 轮 × 5000 次取中位数；倍数 = JSONModel / 同进程内的手写

变体     倍数(中位)          区间     相对 base   说明
base        6.68x  [6.57,7.35]       +0.0%   原样
v1          5.72x  [5.56,5.76]      -14.3%   1. 无 custom setter 时跳过 __customSetValue:
v2b         6.44x  [6.40,6.58]       -3.5%   2. 直接走下标，不用 valueForKeyPath:
v3          6.13x  [5.95,6.22]       -8.2%   3. 去掉「写前先 KVC 读一次」
v4          5.72x  [5.60,6.09]      -14.3%   4. 跳过每次都重建的必填 key 校验
v5          6.42x  [6.37,7.15]       -3.9%   5. __properties__ 数组按类缓存
all2        3.85x  [3.72,3.98]      -42.3%   全部叠加
```

五个单项加起来是 44.2%，一起打是 42.3%，基本可加。这种一致性本身就是结论可靠的旁证。打完补丁解析结果我核对过，和原版一致。

**6.6 倍里有 42% 是纯粹的实现浪费，跟"用 runtime 做映射"这件事没关系。** 剩下的 3.85 倍才是 KVC 和动态派发的真实代价。

### 几点值得单独说的

`__customSetValue:` 那 14.3% 最出人意料。 这个方法是为"用户给某个属性写了 `setXxxWithNSString:`"准备的，绝大多数 model 一个都没有。但主循环无条件调它。每个属性都要走一遍 `classByResolvingClusterClasses:` 加 `NSStringFromClass`。自省阶段其实已经把 `p.customSetters` 填好了。判一下 `count` 是不是 0 就能全跳过。

必填校验那 14.3% 也是白花的。 `__doesDictionary:` 每次解析都要做三件事：`[dict allKeys]`、`[NSSet setWithArray:]`、`[requiredProperties mutableCopy]`。构造两个集合，做一次 `isSubsetOfSet:`。用完就扔。这些完全可以在自省阶段算好。

`valueForKeyPath:` 只值 3.5%，比我预期的低得多。 我最初的补丁写的是"先 `rangeOfString:@"."` 判断有没有点，没有点才走下标"，结果测出来比原版慢 12.4%。`NSString` 的 `rangeOfString:` 本身比省下来的那点还贵。改成无条件走下标才拿到 3.5%。这条给我提了个醒：优化补丁自己也要计时，别默认它是免费的。

单看这三个 API 的成本：

```text
200000 次，中位数：
  [dict valueForKeyPath:@"country"]   6.10 ms      // 没有点
  [dict valueForKeyPath:@"a.b"]      51.57 ms      // 有一个点
  [dict objectForKey:@"country"]      2.78 ms
```

没有点的时候 `valueForKeyPath:` 只比下标贵 2.2 倍，一旦真的有点就是 18.5 倍。这个差距下一节还会再出现一次。

---

## 六、keyMapper 的账

属性名和 JSON key 对不上时用 `+keyMapper`。三种写法我都验了：

```text
--- 字典映射 + 嵌套 keyPath ---
Mapped {"id":104,"orderDetails":{"name":"P1","price":{"usd":12.95}},"userName":"tom"}
       -> orderId=104  productName=P1  price=12.95  userName=tom

--- snake_case ---
Snake  {"user_name":"tom","item_count":3}  -> userName=tom  itemCount=3
Snake  {"userName":"tom","itemCount":3}    -> nil, Required JSON keys are missing from the input.
mapperForSnakeCase 对 userName 的转换结果 = user_name
```

第二行要注意：装了 `mapperForSnakeCase` 之后驼峰 key 就认不出来了。映射是全量替换，不是"先按原名找、找不到再转换"。字典式映射同理。没列进字典的属性名会原样透传，就是 `initWithModelToJSONDictionary:` 里那句 `?: keyName`。

代价不小。三个结构完全相同的 model，各解析 20000 次，跑 6 个进程取中位数比值：

```text
  无 keyMapper             2.4 us/次   1.00x
  平铺 keyMapper           3.8 us/次   1.58x
  带嵌套 keyPath 的 mapper  6.0 us/次   2.45x
```

一个只做名字替换、不改任何逻辑的 mapper，让解析贵了 58%。

钱花在 `__doesDictionary:` 里。没有 mapper 时它只做一次集合比较。有 mapper 时它要把所有属性名逐个转换，再逐个去字典里查一遍。然后 `__importDictionary:` 再查第二遍。每个字段被查两次。 把这段校验跳掉再测：

```text
  原样                        平铺 1.58x  嵌套 2.45x   （各次: 1.66, 1.58, 1.56, 1.45, 1.67, 1.57）
  跳过 __doesDictionary 校验    平铺 1.29x  嵌套 1.90x   （各次: 1.30, 1.25, 1.29, 1.28, 1.31, 1.30）
```

区间不重叠，方向明确。

我还怀疑过另一个地方。`__setup__` 里这一句：

```objc
    id mapper = [[self class] keyMapper];
    if ( mapper && !objc_getAssociatedObject(self.class, &kMapperObjectKey) ) {
```

先无条件调 `+keyMapper`，再判断要不要存。而 `+keyMapper` 的标准写法长这样：`return [[JSONKeyMapper alloc] initWithModelToJSONDictionary:@{...}]`。每次调用都新建一个 mapper 对象、一个 block、一个字典。数了一下：

```text
再解析 1000 次，+keyMapper 又被调用了 1000 次
```

1000 次解析，`+keyMapper` 调 1000 次，1000 个对象建完就扔。是实打实的浪费。可我把判断顺序调过来重测，效果被噪声淹了。原版 1.63x，区间 1.51–1.68。补丁版 1.556x，区间 1.37–1.66。大面积重叠。所以这条我只能说"确实在做无用功，但在这个体量的 JSON 上量不出收益"，不下方向性结论。model 属性更多、mapper 字典更大的时候应该会显出来，我没验。

---

## 七、坑的实测清单

前面散着提了一些，这里把跑过的结果集中列一遍。全部是真跑出来的输出。

必填与 null。 属性不带 `<Optional>` 就是必填。四种情况：

```text
Req {"must":"a","may":"b"}  -> must=a  may=b
Req {"must":"a"}            -> must=a  may=<nil>
Req {"may":"b"}             -> nil, Required JSON keys are missing from the input.
Req {"must":null,"may":"b"} -> nil, Invalid JSON data: Value of required model key must is null
Req {"must":"a","may":null} -> must=a  may=<nil>
```

key 缺失和值为 `null` 走的是两条不同的错误路径，报的 message 不一样。可选属性拿到 `null` 就是 nil，干净。

这一点比裸字典强很多。后端返回 `null`，你从字典里取出来是 `NSNull`，一 `length` 就崩。过一遍 JSONModel，属性里要么是对象要么是 nil。

但只限于属性本身。`NSDictionary` 或 `NSArray` 类型的属性是整块存进去的，里面的 `NSNull` 原样保留：

```text
Box {"kids":[],"extra":{"k":null}}  -> extra = { k = "<null>"; }
```

类型自动转换。 转换规则就是有没有对得上的 `<目标>From<源>:` 方法：

```text
{"num":"3.5"}                     -> num = 3.5          NSNumberFromNSString:
{"str":49}                        -> str = "49"         NSStringFromNSNumber:
{"url":"https://a.b/c"}           -> url = https://a.b/c NSURLFromNSString:
{"set":[1,2,2]}                   -> set = {1, 2}       NSSetFromNSArray:（附带去了重）
{"date":"2026-07-27T10:00:00+0800"} -> 2026-07-27 02:00:00 +0000
{"date":"2026-07-27"}             -> date = nil         格式对不上，静默给 nil
{"num":"abc"}                     -> num = 0            doubleValue 的行为
{"num":[1,2]}                     -> nil, type mismatch
{"val":"x"}                       -> nil, type mismatch （NSValue 没有转换器）
```

两条静默失败要留神。日期只认 `yyyy-MM-dd'T'HHmmssZZZ` 一种格式，对不上不报错，给 nil。`"abc"` 转数字给 0，也不报错。

model 数组里混进非字典元素，静默丢弃。

```text
{"kids":[{"label":"a"},{"label":"b"}]} -> 2 个 Kid
{"kids":["a","b"]}                     -> 空数组，error 是 nil
{"kids":[{"label":"a"},"b"]}           -> 1 个 Kid，error 是 nil
{"kids":{"label":"a"}}                 -> nil, type mismatch
```

`arrayOfModelsFromDictionaries:` 的循环里，非字典非数组的分支就一句注释：

```objc
        } else
        {
            // This is very bad
        }
```

作者自己也知道这里不对劲。整个数组是对象就报错，数组里的元素不是对象就当没看见。

输入本身不对。

```text
Req [1,2,3]    -> nil, ... the dictionary parameter was not an 'NSDictionary'.
Req not json   -> nil, Malformed JSON. Check the JSONModel data input.
```

这两条都规规矩矩返回 `NSError`。

会抛 `NSException` 的只有两处，前面都说过，这里并列一下。一处是属性类型不在白名单，比如 `long double`，第一次自省时抛。另一处是标量属性收到数组或字典，每次解析都可能抛。两处都绕过 `error:`。

---

## 八、这个库今天还该不该用

新项目我不会用了。理由不是慢。

6.6 倍听着吓人，落到绝对值是每个对象 25 微秒。一屏 20 条数据就是 0.5 毫秒，一帧的预算是 16.7 毫秒。这个开销在绝大多数业务里根本量不出来。真要优化，我给自己的阈值是：单次列表刷新解析超过 500 个对象，或者有明确的卡顿数据指到解析上，才值得动。到那个时候换掉的收益也远不止 6.6 倍这一档，因为瓶颈通常已经在别的地方了。

真正让我放弃的是那几种不出声的失败：

- ObjC 泛型写法完全无效，数组里留着原始字典，无告警无报错。而 Xcode 一直在提示你给容器补泛型。
- model 数组里的非字典元素被丢掉，`error` 是 nil。后端某条数据脏了，你只是少几行，查不到。
- 日期格式对不上给 nil，字符串转数字失败给 0，都不报错。
- 反过来，唯一会硬崩的那条路（标量属性收到数组或对象）恰恰绕过了 `error:`，你判了 error 也拦不住。

这几条合起来的性质是一致的：**该报错的地方它沉默，该沉默的地方它抛异常。** 对一个专门处理不可信输入的库来说，这个方向反了。

再加上停更八年这个事实。最后一版的 CHANGELOG 写着 "support for Swift 3"。`copyWithZone:` 还在用 iOS 12 就废弃的 `unarchiveObjectWithData:`。而它同时声明了 `supportsSecureCoding` 返回 YES。这些不会自己坏掉，但也不会有人来修。

有个说法我不同意：「JSONModel 慢是因为它用了 runtime 和 KVC」。第五节的归因表摆在那里，五个补丁删掉 42%，没有一个动到 runtime 自省或者 KVC 本身。慢的是每个属性都白跑一遍 `__customSetValue:`、每次解析都重建一遍必填集合、每次写之前都多读一次。这些是普通的实现问题，跟机制无关。把"用了 runtime"当成慢的原因，会让人以为省不掉，其实一大半是能省的。

那已经在用的项目呢？我的建议是别为了性能重写，先做几件小事。给所有 model 数组补上协议写法，然后检查一遍是不是真的解出了 model 对象。把可能变类型的字段从标量属性改成 `NSNumber <Optional> *`，这一步直接把唯一的崩溃路径堵死。启动后找个空闲时机，把主要 model 类各 `init` 一次，那个 124 倍的冷启动成本就挪出首屏了。这几件事的收益比换库高，成本低两个数量级。

---

## 总结

- 尖括号里的协议名是 JSONModel 唯一能在运行时拿到"数组元素类型"的通道，因为 ObjC 泛型在 `property_getAttributes` 里被完全擦掉。那个协议只需要前向声明，`NSProtocolFromString` 拿到的是 NULL，全程只用到字符串本身。
- 属性自省按类缓存在 Class 的关联对象上，实测 1000 次解析只自省 1 次。代价集中在第一次：真实 model 冷热比 124 倍，一个 10 属性的类摊 0.25 毫秒。
- 比手写慢 6.6 倍，其中 42% 是可以删掉的实现浪费（每属性白跑 `__customSetValue:`、每次重建必填集合、写前多读一次），剩下 3.85 倍才是 KVC 和动态派发的真实成本。
- 它对不可信输入的处理方向是反的：泛型写错、数组元素类型不对、日期格式对不上都静默；唯一会抛 `NSException` 的标量属性收到容器，反而绕过 `error:`。
- 最后一次提交 2018-09-19。不推荐新项目使用，但已有项目不必为性能重写。

下一篇 [[iOS YYModel 源码：为什么比 JSONModel 快]]。

## 参考资料

### 一手

- [jsonmodel/jsonmodel](https://github.com/jsonmodel/jsonmodel)：本文所有源码引用来自 `master`，HEAD 是 `78f8da0`（2018-09-19）。`JSONModel.m` 1387 行是全部核心逻辑
- [Objective-C Runtime Programming Guide — Declared Properties](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html)：属性字符串里每个字母的含义，`T` / `R` / `C` / `&` / `W` / `G` / `V` 都在这
- [Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)：`q` / `B` / `d` / `D` 这些编码的出处
- `objc/runtime.h`：`class_copyPropertyList`、`property_getAttributes`、`class_addProperty` 的签名

### 本地

- [[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]]：`setValue:forKey:` 的完整 setter 搜索链、落到 ivar 时是 retain 不是 copy、标量属性收到 nil 的 `NSInvalidArgumentException`，这三条本文直接引用不重测。第四节"这个坑在这里恰好不成立"接的是那篇
- [[iOS Method Swizzling：正确姿势、+load 时机与那些坑]]：第三节数 `__inspectProperties` 调用次数用的就是那篇的写法
- [[iOS 序列化：NSCoding、NSSecureCoding 与 JSON 三条路]]：`NSJSONSerialization`、`NSSecureCoding` 本身，以及 `unarchiveObjectWithData:` 为什么被废弃
- [[iOS YYModel 源码：为什么比 JSONModel 快]]：性能对比和另一套实现思路

---

实验环境：macOS 26（Darwin arm64，Apple Silicon），Apple clang。`clang -fobjc-arc -framework Foundation` 编成原生命令行程序直接跑，全程没有开模拟器。计时用 `clock_gettime(CLOCK_MONOTONIC)`。性能实验统一 9 轮取中位数。跨变体对比额外跑 5 到 6 个进程，并用同进程内的手写版本做归一化，因为进程之间有 3% 到 8% 的漂移。JSONModel 源码为 `78f8da0`，用 `-O2` 构建；补丁变体只改 `JSONModel.m`，其余文件复用同一份。

> 本文所有耗时都是 macOS 原生二进制的数字，只用来说明相对关系和归因比例，不能当作 iOS 真机的性能结论。绝对值在真机上会显著不同，但五个补丁的归因方向不依赖平台。
> 待补：一是 GitHub 上这个仓库的 issue / PR 现状和是否已 archive，我没联网核实，只从 `git log` 判断活跃度；二是超大输入（几 MB、上万个 model 对象）下的表现，本文只测了单个中等复杂度对象；三是在 Swift 里通过桥接使用时的行为，完全没测。
