---
title: 【iOS】序列化：NSCoding、NSSecureCoding 与 JSON 三条路
published: 2026-07-27
description: decodeObjectOfClass: 在非 secure 的 unarchiver 上完全不检查类型，头文件写了，Apple 自己的安全编码指南没写。另外 NSJSONSerialization 解析 -1e400 会成功，返回一个它自己拒绝写回去的对象。
tags:
  - iOS
  - Objective-C
  - Foundation
  - 序列化
  - 归档
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 31
draft: true
---
# 序列化：NSCoding、NSSecureCoding 与 JSON 三条路

先摆两个我这次实测出来、和流行说法对不上的结果。

第一个。下面这行是 Apple《Secure Coding Guide》原话推荐的写法：

```objc
_text = [c decodeObjectOfClass:NSString.class forKey:@"text"];
```

我构造了一份归档，让这一行返回了一个 `Fake` 实例，稳稳赋进声明为 `NSString *` 的属性里。没有警告，没有 error，没有异常。原因在 `NSCoder.h` 的注释里写着，而那份指南没写。第三节有完整过程。

第二个。`NSJSONSerialization` 解析 `[-1e400]` 会成功，返回一个 `-inf` 的 `NSNumber`。而同一个类的头文件白纸黑字写着 "NSNumbers are not NaN or infinity"。把这个对象原样写回去，当场抛 `NSInvalidArgumentException`。换成 `[1e400]`，解析被拒绝。只有负号那一侧漏了。第六节。

这篇只讲编解码这一层：一个对象怎么变成字节，字节怎么变回对象。存放在哪、怎么选存储介质是另一篇 [[iOS 持久化选型：沙盒、plist、NSUserDefaults 与 Keychain]] 的事。JSONModel 和 YYModel 怎么把映射自动化，是后面两篇的事，这里只把"为什么需要它们"讲透。

---

## 一、归档产物是一张扁平表，环靠 UID 表达

这一节是全文我最想写的。`NSKeyedArchiver` 的输出常被当成黑盒，其实 `plutil -p` 一敲就全看见了。

先归档一个带嵌套对象的类：`Person` 有 `name`、`age`、`addr`（一个 `Addr` 对象）、`tags`（字符串数组）。

```objc
NSData *d = [NSKeyedArchiver archivedDataWithRootObject:p requiringSecureCoding:YES error:&err];
[d writeToFile:@"/tmp/ser/nocycle.plist" atomically:YES];
```

444 字节。头 8 个字节是 `bplist00`，它就是一个二进制 property list。`plutil -p` 打开：

```text
{
  "$archiver" => "NSKeyedArchiver"
  "$objects" => [
    0 => "$null"
    1 => {
      "$class" => <CFKeyedArchiverUID>{value = 10}
      "addr"   => <CFKeyedArchiverUID>{value = 3}
      "age"    => 30
      "friend" => <CFKeyedArchiverUID>{value = 0}
      "name"   => <CFKeyedArchiverUID>{value = 2}
      "tags"   => <CFKeyedArchiverUID>{value = 6}
    }
    2 => "Tommy"
    3 => { "$class" => {value = 5}, "city" => {value = 4}, "zip" => 310000 }
    4 => "Hangzhou"
    5 => { "$classes" => ["Addr", "NSObject"], "$classname" => "Addr" }
    6 => { "$class" => {value = 9}, "NS.objects" => [{value = 7}, {value = 8}] }
    7 => "ios"
    8 => "objc"
    9 => { "$classes" => ["NSArray", "NSObject"], "$classname" => "NSArray" }
    10 => { "$classes" => ["Person", "NSObject"], "$classname" => "Person" }
  ]
  "$top" => { "root" => <CFKeyedArchiverUID>{value = 1} }
  "$version" => 100000
}
```

四个顶层键。`$archiver` 是归档器类名，`$version` 恒为 100000，`$top` 指向根对象，`$objects` 装所有东西。

关键在 `$objects` 的形状。它是一个扁平数组，没有嵌套的对象树。对象之间的引用一律写成 `CFKeyedArchiverUID`，值就是数组下标。`Person` 在槽位 1，它的 `addr` 存的是 `{value = 3}`，槽位 3 才是那个 `Addr` 字典。类信息也是对象。槽位 5、9、10 分别是 `Addr`、`NSArray`、`Person` 的类描述，每个都带一条 `$classes` 继承链。

槽位 0 永远是字符串 `"$null"`。`friend` 我没赋值，于是指向 UID 0。nil 在归档里有一个固定住址。

标量不走这套。`age => 30`、`zip => 310000` 直接是 plist 里的整数，没有 UID 包装。因为我用的是 `encodeInt:` / `encodeInteger:`。

### 环怎么办

Apple 的归档文档把问题说得很清楚：

> An object graph is not necessarily a simple tree structure. Two objects can contain references to each other, for example, creating a cycle. If a coder follows every link and blindly encodes each object it encounters, this circular reference will generate an infinite loop in the coder.

给出的解法也在原文里：

> If the coder is asked to encode an object more than once, the coder encodes a reference to the first encoding instead of encoding the object again.

看输出比看这段话直接。让 `p.friend = q`，`q.friend = p`，跑一遍：

```text
cycle archive bytes = 492
```

492 字节，没有无限递归。`$objects` 长这样（只留相关行）：

```text
    1  => { "$class" => {value=12}, "addr" => {value=3}, "age" => 30,
            "friend" => {value=10}, "name" => {value=2}, "tags" => {value=6} }
    10 => { "$class" => {value=12}, "addr" => {value=3}, "age" => 28,
            "friend" => {value=1},  "name" => {value=11}, "tags" => {value=0} }
```

槽位 1 的 `friend` 指 10，槽位 10 的 `friend` 指回 1。环在文件里就是两个互指的下标，一个静态的、有限的东西。图论上的环，落到存储上是一张邻接表。

顺便看两处共享。两个 `Person` 的 `addr` 都是 `{value=3}`，同一个 `Addr` 只写了一份；两个的 `$class` 都是 `{value=12}`，类描述也只写一份。

自引用更干净。`r.friend = r`，归档 265 字节：

```text
    1 => { "$class" => {value=3}, "friend" => {value=1}, "name" => {value=2}, ... }
```

`friend` 指着自己所在的槽位。

### 解档侧：你会在 initWithCoder: 里拿到半成品

归档不会死循环，我原本以为解档同理，没细想。实际跑一遍，发现里面有个坑值得单说。

两个节点 A、B 互指，在 `initWithCoder:` 进出口各打一行日志：

```text
  initWithCoder ENTER  self=0x10485eb90 name=A
  initWithCoder ENTER  self=0x10485ec30 name=B
  initWithCoder EXIT   self=0x10485ec30 name=B  next=0x10485eb90 next.name=A next.next=0x0
  initWithCoder EXIT   self=0x10485eb90 name=A  next=0x10485ec30 next.name=B next.next=0x10485eb90
结果 a2=0x10485eb90 a2.next=0x10485ec30 a2.next.next=0x10485eb90  环恢复? 1
```

环恢复了。代价看第三行：B 在自己的 `initWithCoder:` 里拿到了 A 的指针，而此刻 A 的 `initWithCoder:` 还没返回，`A.next` 还是 `0x0`。

也就是说，unarchiver 在调 `initWithCoder:` 之前就把分配好的实例登记进了 UID 表，环才断得掉。副作用是，`initWithCoder:` 里解出来的对象可能是个半成品。读它的字段做校验、算 hash、往集合里塞，都可能读到空值。我自己的规矩是：`initWithCoder:` 只做赋值。任何依赖别的对象状态的初始化逻辑，一律挪到解档结束之后手动跑一趟。

### encodeConditionalObject: 是给 weak 用的

上面那个环，两个方向都是强引用，所以两份都得写。父子关系里的 `weak parent` 不该把父亲拖进归档。文档给了专门的入口：

> A conditional object is an object that should be encoded only if it is being encoded unconditionally elsewhere in the object graph... Typically, conditional objects are used to encode weak references to objects.

实测两种情形：

```text
A 从 parent 归档: child.weakParent = parent          (258 字节)
B 只归档 child : weakParent = (nil)                  (235 字节)
```

从 parent 归档时，parent 在别处被无条件编码过，条件引用正常连上。只归档 child 时没人无条件编码 parent，那个字段解出来是 nil。`$objects` 里能看到它被写成什么：

```text
    1 => { "child" => {value=0}, "name" => {value=2}, "parent" => {value=3} }
    3 => "$null"
```

`child` 本来就是 nil，指向公用的 UID 0。`parent` 是"条件编码但没命中"，归档器给它单开了槽位 3，内容同样是 `"$null"`。两种 nil 走了两条路，落在同一个语义上。文档没提这个，是从原始输出里看出来的。

---

## 二、NSSecureCoding 防的到底是什么

Apple 的《Secure Coding Guide》把攻击面讲得很具体：

> First, an object archive expands into an object graph that can contain arbitrary instances of arbitrary classes. If an attacker substitutes an instance of a different class than you were expecting, you could get unexpected behavior.

翻成人话：归档里的类名是数据。你解档时，是这份数据在告诉运行时该实例化哪个类。

我把这个攻击真跑了一遍。两个类，字段布局完全一样，只有 `initWithCoder:` 不同：

```objc
@implementation Exploit
- (instancetype)initWithCoder:(NSCoder *)c {
    if (self = [super init]) {
        _text = [c decodeObjectOfClass:NSString.class forKey:@"text"];
        printf("  !!! Exploit -initWithCoder: 执行了。\n");
    }
    return self;
}
@end
```

归档一个 `Payload`，把 bplist 转成 XML。两处 `<string>Payload</string>` 改成 `<string>Exploit</string>`，再喂回去：

```text
原始归档 222 字节
  篡改：替换了 2 处类名 Payload -> Exploit

[1] 非 secure 路径 unarchiveObjectWithData:（已 deprecated）
  !!! Exploit -initWithCoder: 执行了。
  解出来的类 = Exploit，text=hello
  它是 Payload 吗? 0

[2] secure 路径 unarchivedObjectOfClass:Payload fromData:error:
  返回 nil
  error = The data couldn't be read because it isn't in the correct format.
  domain=NSCocoaErrorDomain code=4864
  debugDescription = value for key 'root' was of unexpected class 'Exploit' (0x104808378) [/tmp/ser].
  Allowed classes are:
   {( "'Payload' (0x104808328) [/tmp/ser]" )}
```

完整的类型混淆复现。老 API 老老实实构造了 `Exploit`，执行了它的 `initWithCoder:`，返回给调用方。调用方接的变量类型写着 `Payload *`。编译器一句话都不会说。

secure 路径给出 `NSCocoaErrorDomain` 4864，`debugDescription` 里把实际类、镜像路径、白名单全列了出来。

有一个细节值得看：情形 2 里那行 `!!!` 没有打印。类检查发生在实例化之前，不是先构造出来再回头判断。这条防线是预防性的，不是事后审计。

`initWithCoder:` 是攻击者免费拿到的一次代码执行机会。可以是一次文件删除。也可以是指南里说的那种：

> If your `initWithCoder:` method does not carefully validate all the data it decodes to make sure it is well formed and does not exceed the memory space reserved for it, then by carefully crafting a corrupted archive, an attacker could potentially cause a buffer overflow or trigger another vulnerability and possibly seize control of the system.

---

## 三、防线在 unarchiver 上，不在 decodeObjectOfClass: 上

上面那份指南给的处方是这句：

> you should use `decodeObjectOfClass:forKey:` instead, and you should limit the contents of your file format to classes that conform to the `NSSecureCoding` protocol.

我见过不少代码只落实了前半句：`initWithCoder:` 里整整齐齐全是 `decodeObjectOfClass:`，读起来很安全。

`NSCoder.h` 里那句注释说的是另一回事：

> Specify what the expected class of the allocated object is. If the coder responds YES to `-requiresSecureCoding`, then an exception will be thrown if the class to be decoded does not implement `NSSecureCoding` or is not `isKindOfClass:` of the argument. **If the coder responds NO to `-requiresSecureCoding`, then the class argument is ignored and no check of the class of the decoded object is performed, exactly as if `decodeObjectForKey:` had been called.**

class 参数被忽略。整行退化成 `decodeObjectForKey:`。

实测。`Payload.text` 声明是 `NSString *`，`initWithCoder:` 里规规矩矩写 `decodeObjectOfClass:NSString.class`。我用 `object_setIvar` 往 `_text` 里塞一个 `Fake` 实例再归档，然后两条路各解一次：

```text
[A] 非 secure 的 unarchiver（老 API）
    !!! Fake 被实例化了
    requiresSecureCoding=0  text 解出来的类 = Fake
  返回 Payload

[B] secure 的 unarchiver
    requiresSecureCoding=1  text 解出来的类 = nil
  返回 nil err=The data couldn't be read because it isn't in the correct format.
  debug: value for key 'text' was of unexpected class 'Fake' (0x102a742f8) [/tmp/ser].
  Allowed classes are: {( "'NSString' (0x1fa36a100) [.../Foundation.framework]" )}
```

A 里 `_text` 现在指着一个 `Fake`。一个声明为 `NSString *` 的属性，装着连 `length` 都没有的对象。下次谁碰它谁崩。崩溃栈指向的是那个使用点，离真正的问题十万八千里。

所以决定安不安全的是解档器怎么建的，写法只是配合。`+unarchivedObjectOfClass:fromData:error:` 和 `-initForReadingFromData:error:` 都默认打开 `requiresSecureCoding`。头文件的原话是 "Enables `requiresSecureCoding` by default"。`unarchiveObjectWithData:` 不开。指南只说了写法那一半，是我认为它今天最容易误导人的地方。

### 白名单不递归，容器要把每一层都列出来

`Cart` 里有一个 `NSArray<Item *>`。解 `items` 时给三种白名单：

```text
[mode 0] 完整白名单 {NSArray, Item}
    items = 非 nil  (2 个)  coder.error=nil
  顶层返回 非 nil  error=nil

[mode 1] 漏掉 Item，只写 {NSArray}
    items = nil  coder.error=The data couldn't be read...
    debug: value for key 'NS.objects' was of unexpected class 'Item'.
           Allowed classes are: {( "'NSArray' ... )}
  顶层返回 nil

[mode 2] 漏掉 NSArray，只写 {Item}
    items = nil  coder.error=...
    debug: value for key 'items' was of unexpected class 'NSArray'.
           Allowed classes are: {( "'Item' ... )}
  顶层返回 nil
```

容器和元素都要在白名单里。mode 1 的报错定位很准，直接点名 `NS.objects` 这个键。第一节里 `NSArray` 就是用它存元素的。归档格式在这里漏了出来。

嵌套字典套数组套自定义对象的话，四个类一个都不能少。这是手写 `NSSecureCoding` 最容易出错的地方。而且错了不一定当场发现，顶层可能只是安静地返回 nil。

失败还会传染。mode 1 里 `items` 挂掉之后，同一个 `initWithCoder:` 里后面那句解 `meta` 也拿不到东西了。`NSCoder.h` 解释了这个设计：

> While `.error` is non-nil, all attempts to decode data from this coder will return a nil/zero-equivalent value.

一个字段解错，整次解档就废了。这是好事，省得你拿着半个对象往下走。

### 子类是放行的，NSObject 是后门

头文件说得明确：

> The class of the object may be any class in the provided NSSet, **or a subclass of any class in the set**.

数据里放 `Derived`，白名单写四种：

```text
[白名单写父类 Base]      payload=Derived  顶层 OK
[白名单写 NSObject]      payload=Derived  顶层 OK
[白名单写 Derived 本身]  payload=Derived  顶层 OK
[白名单写无关类 NSString] payload=nil     顶层 nil
[payload 本来就是 nil]    payload=nil     顶层 OK  ← 无 error
```

第二行是重点。把 `NSObject` 放进 `allowedClasses`，整套机制当场作废，所有类都合法，而代码看上去还是在用 secure API。图省事写 `NSObject.class` 等于把门拆了。

最后一行也值得记一下：字段本来就是 nil 时不算违规，安静通过。

### +supportsSecureCoding 返回 NO 会怎样

不实现这个方法（协议都不采纳）的类，归档阶段就被拦：

```text
  Legacy conformsToProtocol(NSSecureCoding) = 0
  归档 requiringSecureCoding:YES -> data=nil err=The data couldn't be written because it isn't in the correct format.
  改用 requiringSecureCoding:NO 归档成功 214 字节
  再用 secure 解档 -> nil err=The data couldn't be read because it isn't in the correct format.
  debug: This decoder will only decode classes that adopt NSSecureCoding. Class 'Legacy' does not adopt it.
```

写和读两端都拦，报错文案还挺清楚。

还有一条藏在 `NSObject.h` 的协议注释里，改造老代码时容易漏：

> Subclasses of classes that adopt NSSecureCoding and override `initWithCoder:` must also override this method and return YES.

父类实现了不算数。子类只要覆写了 `initWithCoder:`，`+supportsSecureCoding` 就得自己再写一遍。

---

## 四、API 换代：老的还能用，但已经 deprecated 八年

`NSKeyedArchiver.h` 里的原文：

```objc
+ (NSData *)archivedDataWithRootObject:(id)rootObject
    API_DEPRECATED("Use +archivedDataWithRootObject:requiringSecureCoding:error: instead",
                   macosx(10.2,10.14), ios(2.0,12.0), watchos(2.0,5.0), tvos(9.0,12.0));

+ (nullable id)unarchiveObjectWithData:(NSData *)data
    API_DEPRECATED("Use +unarchivedObjectOfClass:fromData:error: instead",
                   macosx(10.2,10.14), ios(2.0,12.0), watchos(2.0,5.0), tvos(9.0,12.0));

+ (BOOL)archiveRootObject:(id)rootObject toFile:(NSString *)path
    API_DEPRECATED("Use +archivedDataWithRootObject:requiringSecureCoding:error: and -writeToURL:options:error: instead", ...);
```

iOS 12 / macOS 10.14 起废弃。今天还能不能编？能：

```text
e4.m:4:34: warning: 'archivedDataWithRootObject:' is deprecated:
    first deprecated in macOS 10.14 - Use +archivedDataWithRootObject:requiringSecureCoding:error: instead
e4.m:5:34: warning: 'unarchiveObjectWithData:' is deprecated:
    first deprecated in macOS 10.14 - Use +unarchivedObjectOfClass:fromData:error: instead
2 warnings generated.
exit=0
```

两条警告，正常链接，正常运行。我把部署目标压到 `-mmacosx-version-min=10.13` 再编，警告一条都没有。`API_DEPRECATED` 是按部署目标门控的。老工程升 SDK 不升部署目标，这些调用可以一直安静地待着。

新旧的实质差别只有一条：新版把 `requiringSecureCoding` 提成了必填参数。你得当场决定开不开，不能默认糊过去。老 API 的默认值是不开。

`NSKeyedArchiveRootObjectKey` 这个常量今天还在，值就是 `"root"`，第一节的 `$top` 里能看到。自己 `initRequiringSecureCoding:` 手动编码根对象时用它，产物结构和 `+archivedDataWithRootObject:` 一致。头文件的说法是 "To produce archives whose structure matches those previously encoded using `+archivedDataWithRootObject`"。

---

## 五、NSJSONSerialization 拒绝什么

头文件把契约列成了四条，写在类注释和 `isValidJSONObject:` 上，一字不差重复了两遍：

```text
 An object that may be converted to JSON must have the following properties:
  - Top level object is an NSArray or NSDictionary
  - All objects are NSString, NSNumber, NSArray, NSDictionary, or NSNull
  - All dictionary keys are NSStrings
  - NSNumbers are not NaN or infinity
```

后面还跟一句免责：

> Other rules may apply. Calling this method or attempting a conversion are the definitive ways to tell if a given object can be converted to JSON data.

逐条实测。左边是 `isValidJSONObject:` 的返回，右边是 `dataWithJSONObject:` 的结果：

```text
顶层 NSDictionary            valid=1  -> {"a":1}
顶层 NSArray                 valid=1  -> [1]
顶层 NSString                valid=0  EXCEPTION: Invalid top-level type in JSON write
顶层 NSString + Fragments    valid=0  -> "hello"
顶层 NSNumber  + Fragments   valid=0  -> 42
顶层 NSNull    + Fragments   valid=0  -> null
NaN                          valid=0  EXCEPTION: Invalid number value (NaN) in JSON write
Infinity                     valid=0  EXCEPTION: Invalid number value (infinite) in JSON write
-0.0                         valid=1  -> {"x":-0}
非字符串 key (NSNumber)      valid=0  EXCEPTION: Invalid (non-string) key in JSON dictionary
NSNull 作为值                valid=1  -> {"x":null}
NSDate 作为值                valid=0  EXCEPTION: Invalid type in JSON write (__NSTaggedDate)
NSData 作为值                valid=0  EXCEPTION: Invalid type in JSON write (_NSZeroData)
自定义对象作为值             valid=0  EXCEPTION: Invalid type in JSON write (NSObject)
```

几点值得单独记。

顶层标量今天可以写，条件是加 `NSJSONWritingFragmentsAllowed`。但 `isValidJSONObject:` 对它照样返回 0，因为这个方法不接受 options 参数，只按最严的规则判。**`isValidJSONObject:` 返回 NO 不等于写不出去，返回 YES 也只是"多半可以"**，头文件那句免责就是为此写的。

失败方式有两种，混在一起用会出事。类型不合法走的是 `NSInvalidArgumentException`，`@try` 能接；只有部分错误才落到 `error` 参数上。写 JSON 时只写 `if (!data) { ... }` 是接不住的。

`NSNull` 和 nil 是两回事。`NSNull` 是合法值，写出去是 `null`；nil 在 `NSDictionary` 里根本塞不进去。这个区别在解析端同样成立：`{"a":null}` 解出来 `a` 的值是 `NSNull` 实例，`dict[@"a"]` 非空，`dict[@"a"] == nil` 为假。判空得写 `[v isKindOfClass:NSNull.class]`。

`NSDate` 和 `NSData` 不合法。它俩在 plist 和归档里都是一等公民，到 JSON 就没有对应类型了，得自己定时间戳和 base64 的约定。

### 环：崩，而且崩得接不住

```text
isValidJSONObject(自引用数组) = 0
isValidJSONObject(互指字典)   = 0
isValidJSONObject(DAG 共享子树) = 1
DAG 序列化 -> [[9],[9]]
现在真的调 dataWithJSONObject: 环…
exit=139   signal = 11  SEGV
```

`isValidJSONObject:` 认得出环，返回 NO。直接调 `dataWithJSONObject:` 就是一路递归到爆栈。SIGSEGV。不是异常，`@try` 拦不住，也没有 `error` 可读。

这条和第一节形成了对照。同样一份带环的对象图，`NSKeyedArchiver` 靠 UID 表安然写完，`NSJSONSerialization` 直接把进程干掉。差别在数据模型：JSON 里没有对象身份，只有值。

倒数第三行是同一件事的温和版本。`shared` 这个数组在 `dag` 里出现了两次。这不是环，序列化成功，写出来是 `[[9],[9]]`。**内容复制了两份，解回来是两个独立数组。** 归档会保持它俩是同一个对象，JSON 不会。往返一次，共享结构就变成了副本，改一个不影响另一个。做本地缓存时，如果原来的对象图有共享节点，这一点会咬人。

### 两个和 RFC 8259 对不上的地方

**尾逗号被接受了。** RFC 8259 第 4 节的语法里没有它的位置：

```text
      object = begin-object [ member *( value-separator member ) ] end-object
      member = string name-separator value
```

`value-separator` 只出现在两个 member 之间。实测：

```text
  {"a":1,}         -> {a = 1;}
  [1,2,]           -> (1, 2)
  [1,]             -> (1)
  {"a":1,"b":2,}   -> {a = 1; b = 2;}
  [,]              -> REJECT
  {,}              -> REJECT
  {"a":1,,}        -> REJECT
  [1,,2]           -> REJECT
```

恰好一个尾逗号放行，多余的逗号还是拒。Python 的 `json.loads` 对同样三个输入全部报 `Illegal trailing comma`。

我一开始以为这是 `NSJSONReadingJSON5Allowed` 忘了关，那个选项 iOS 15 加进来，专门放宽这类语法。跑一遍对照发现不是：

```text
  {"a":1,}       默认=接受  JSON5=接受
  [1,2,]         默认=接受  JSON5=接受
  {a:1}          默认=拒绝  JSON5=接受
  {'a':1}        默认=拒绝  JSON5=接受
  {"a":1 /*c*/}  默认=拒绝  JSON5=接受
  [+1]           默认=拒绝  JSON5=接受
  [.5]           默认=拒绝  JSON5=接受
  [0x10]         默认=拒绝  JSON5=接受
  [Infinity]     默认=拒绝  JSON5=接受
  [NaN]          默认=拒绝  JSON5=接受
  [01]           默认=拒绝  JSON5=拒绝
```

无引号 key、单引号、注释、`+1`、`.5`、十六进制、`Infinity`、`NaN`，全都老老实实关在 JSON5 后面。只有尾逗号一项漏进了默认路径。

**重复 key 取第一个。** RFC 8259 第 4 节：

> When the names within an object are not unique, the behavior of software that receives such an object is unpredictable. Many implementations report the last name/value pair only.

规范承认这里没有统一行为，但点名了主流做法。Foundation 反着来：

```text
  {"a":1,"a":2}                -> {a = 1;}
  {"a":1,"a":2,"a":3}          -> {a = 1;}
  {"a":"first","a":"second"}   -> {a = first;}
```

Python 同一串给 `{'a': 2}`，JavaScript 也是后者赢。同一份带重复 key 的响应，服务端用 Node 自测通过，iOS 上取到另一个值。这种 bug 我不想在线上遇到第二次。

其余边界和 RFC 一致：`[01]` 前导零拒、`[1.]` 拒、`[.1]` 拒、`[+1]` 拒、孤立代理 `["\uD800"]` 拒。顶层裸标量默认拒，加 `NSJSONReadingFragmentsAllowed` 就接受。`42` 解成 `__NSCFNumber`，`null` 解成 `NSNull`，`true` 解成 `__NSCFBoolean`。

### 那个只在负号一侧存在的溢出

RFC 第 6 节自己举了 `1E400` 当反面例子。原话是它 "suggests that the software that created it expects receiving software to have greater capabilities for numeric magnitude and precision than is widely available"。Foundation 拒绝它。合理。

问题是加个负号就不拒了。这条我按规范第七条的要求，多扫了几个轴才敢写：

```text
轴1: 容器形态
  数组    [1e400]        -> REJECT
  数组    [-1e400]       -> -inf
  对象    {"x":1e400}    -> REJECT
  对象    {"x":-1e400}   -> -inf
  裸标量  1e400          -> REJECT
  裸标量  -1e400         -> -inf

轴2: 指数写法
  e   [1e400] REJECT / [-1e400] -inf
  E   [1E400] REJECT / [-1E400] -inf
  e+  [1e+400] REJECT / [-1e+400] -inf

轴3: 不用指数，直接写 311 位数字
  正长串 -> REJECT
  负长串 -> REJECT

轴4: 临界点
  [1e308] -> 1e+308   [-1e308] -> -1e+308
  [1e309] -> REJECT   [-1e309] -> -inf
  [1e310] -> REJECT   [-1e310] -> -inf
```

容器形态换了三种、指数写法换了三种，正号一侧全拒，负号一侧全给 `-inf`。临界点是 `1e309`，也就是刚越过 binary64 的上限。轴 3 说明这条只在指数记法的代码路径上，把同样大小的数原原本本写成一长串数字，两边都拒。

后果在写回的时候暴露：

```text
  isValidJSONObject(解析结果) = 0
  写回抛 NSInvalidArgumentException: Invalid number value (infinite) in JSON write
```

解析器交给你一个对象。同一个类的头文件说这种对象非法，同一个类的写入端拒绝接收它。服务端返回一个 `-1e400`，你解析成功，转手序列化去写磁盘缓存，进程就没了。而且这个崩溃发生在写缓存那一行，跟解析那一行隔着十万八千里。

我没有在 iOS 上复核这一条。`_NSJSONWriter`、`_NSJSONReader` 都在 Foundation 里，不走平台专属分支。所以我认为两个平台一致。但这是推断，不是实测。

---

## 六、JSON 数字的类型到底丢了没有

流传的说法是"JSON 解出来的数字全是 `NSNumber`，类型信息丢光了"。实测下来这句话对一半。

把各种字面量解回来，逐个打 `objCType` 和 `CFNumberGetType`：

```text
t        cls=__NSCFBoolean     objCType=c  CFType=Char      isBool=1  desc=1
f        cls=__NSCFBoolean     objCType=c  CFType=Char      isBool=1  desc=0
one      cls=__NSCFNumber      objCType=q  CFType=SInt64    isBool=0  desc=1
onef     cls=__NSCFNumber      objCType=d  CFType=Float64   isBool=0  desc=1
neg      cls=__NSCFNumber      objCType=q  CFType=SInt64    isBool=0  desc=-1
i32      cls=__NSCFNumber      objCType=q  CFType=SInt64    isBool=0  desc=2147483647
i64      cls=__NSCFNumber      objCType=q  CFType=SInt64    isBool=0  desc=9223372036854775807
u64      cls=__NSCFNumber      objCType=Q  CFType=SInt64    isBool=0  desc=18446744073709551615
big      cls=NSDecimalNumber   objCType=d  CFType=Double    isBool=0  desc=123456789012345678901234567890
pi       cls=__NSCFNumber      objCType=d  CFType=Float64   isBool=0  desc=3.141592653589793
exp      cls=__NSCFNumber      objCType=d  CFType=Float64   isBool=0  desc=1000
zero     cls=__NSCFNumber      objCType=q  CFType=SInt64    isBool=0  desc=0
negzero  cls=__NSCFNumber      objCType=d  CFType=Float64   isBool=0  desc=-0
```

丢的是位宽：`1`、`2147483647`、`9223372036854775807` 全成了 `SInt64`，源文里是几字节看不出来了。

没丢的是类别。整数 `q`、浮点 `d`、布尔 `c`，三者分得清清楚楚。`{"onef":1.0}` 解回来 `objCType` 是 `d`，尽管它的 `description` 打出来是 `1`。第三行和第四行的 `desc` 都是 `1`，`objCType` 一个 `q` 一个 `d`。想知道服务端写的是整数还是小数，看 `objCType`，别看打印值。

`u64` 那行有个小分歧：`objCType` 说 `Q`（unsigned long long），`CFNumberGetType` 说 `SInt64`。CFNumber 的类型枚举里压根没有无符号类型，只能报个最接近的。两个 API 问同一个对象，答案不一样。

### @YES 和 @1 分得清吗

分得清，但不能用你第一反应的那个方法：

```text
t.boolValue=1 one.boolValue=1
[t isEqual:one] = 1      [t isEqualToNumber:one] = 1
[t isEqual:@YES] = 1     [one isEqual:@1] = 1
strcmp(t.objCType,"c")==0 ? 1
t == (id)kCFBooleanTrue ? 1
[t isKindOfClass:NSClassFromString(@"__NSCFBoolean")] = 1
```

`[@YES isEqual:@1]` 是 YES。`NSNumber` 的相等按数值算，`true` 和 `1` 数值相同。所以 `isEqual:` 这条路走不通。

能用的有三条：`objCType` 是不是 `'c'`、指针是不是等于 `kCFBooleanTrue` / `kCFBooleanFalse`、类是不是 `__NSCFBoolean`。第二条最精确。第一条有假阳性：

```text
  numberWithChar:1  objCType=c  cls=__NSCFNumber  ==kCFBooleanTrue? 0  isEqual:@YES? 1
```

一个真正的 `int8_t` 也是 `'c'`，但它不是那个布尔单例。JSON 里没有 char 这种东西。所以对"从 JSON 解出来的数字"用 `objCType == 'c'` 是安全的，对任意来源的 `NSNumber` 就不安全。这个区别是 YYModel 那类库要处理的边界之一。

写出去的时候类型是保住的：

```text
{"a": @YES, "b": @1, "c": @1.0, "d": @(1.0f)}  ->  {"a":true,"b":1,"c":1,"d":1}
```

`@YES` 写成 `true`，`@1` 写成 `1`。但 `@1.0` 和 `@(1.0f)` 都写成了 `1`，浮点的"点零"在文本里没了。

### 大整数会不会退化成 double

通说是会，阈值 2^53。实测在 Foundation 上不成立。

```text
2^53+1     cls=__NSCFNumber  objCType=q  desc=9007199254740993
  longLongValue = 9007199254740993
  往返写回 = {"id":9007199254740993}
超 int64   cls=NSDecimalNumber objCType=d desc=99999999999999999999
  往返写回 = {"id":99999999999999999999}
```

`9007199254740993` 精确保留。超过 int64 之后也没变成 double，而是升到了 `NSDecimalNumber`，往返依然精确。

完整的阶梯扫出来是三级：

| 整数位数 | 落到的类 | objCType | 是否精确 |
|---|---|---|---|
| ≤ 19 位 | `__NSCFNumber` | `q` / `Q` | 精确 |
| 20 ~ 39 位 | `NSDecimalNumber` | `d` | 精确 |
| ≥ 40 位 | `NSDecimalNumber` | `d` | 尾部补零 |

40 位那一档具体是这样：

```text
  40 位 NSDecimalNumber exact=0  2345678912345678912345678912345678912340
  45 位 NSDecimalNumber exact=0  234567891234567891234567891234567891234000000
```

保住 39 位有效数字，后面全变 0。所以"JSON 大整数丢精度"这个说法在 Foundation 上的真实阈值是 39 位有效数字，不是 2^53。2^53 那个数字来自 JavaScript，`JSON.parse` 把所有数字都塞进 binary64。这条经验从 Web 抄到 iOS 会抄错。

RFC 里那句可以拿来对照：

> numbers that are integers and are in the range [-(2**53)+1, (2**53)-1] are interoperable in the sense that implementations will agree exactly on their numeric values.

规范只承诺 2^53 以内跨实现一致。Foundation 做得比承诺多，但你不能反过来依赖它。服务端换个语言，或者中间过一次 JS 网关，超过 2^53 的 ID 就变形了。**长整型 ID 用字符串传，这条规矩和 Foundation 强不强没关系。**

小数一侧也有个切换点：

```text
  小数位 16 -> __NSCFNumber     exact=1
  小数位 17 -> __NSCFNumber     exact=0
  小数位 18 -> NSDecimalNumber  exact=1
```

17 位小数是唯一一档既没升级到 `NSDecimalNumber`、又已经装不下的。RFC 第 6 节举的另一个例子 `3.141592653589793238462643383279`（30 位）实测解成 `NSDecimalNumber`，精确保留。

`NSDecimalNumber` 这一档有个陷阱：它的 `objCType` 报 `d`，`CFNumberGetType` 报 `Double`，两个都在说"我是双精度浮点"。真拿 `doubleValue` 去读就废了：

```text
big  doubleValue = 1.23457e+29   stringValue = 123456789012345678901234567890
```

要精确值就读 `stringValue` 或者走 `NSDecimal` 运算。按 `objCType` 分派类型的映射代码在这里会直接掉进去。

### 0.1 往返一次就不是 0.1 了

```text
原文 {"x":0.1,"y":1.0,"z":3.0e2}
往返 {"x":0.10000000000000001,"y":1,"z":300}
```

`0.1` 解析成 Float64，写回去变成 17 位有效数字的 `0.10000000000000001`。可同一个对象的 `description` 打出来还是 `0.1`。`NSNumber` 的 stringValue 用最短往返算法，`NSJSONSerialization` 的写入端不用。同一个数，两个 API 给两种文本。

`1.0` 写成 `1`，`3.0e2` 写成 `300`。想让 JSON 文本稳定往返，中间那一步只能用 `NSDecimalNumber`：

```text
直接写 @(0.1) double:            {"x":0.10000000000000001}
直接写 NSDecimalNumber 0.1:      {"x":0.1}
直接写 @(0.1f) float:            {"x":0.10000000149011612}
```

签名、HMAC、幂等键这类要求字节级一致的场景，"解析出来再序列化回去"这条路走不通。留原始 `NSData`。

---

## 七、手写 NSCoding 的账单

前面六节都在讲机制，这节讲为什么会有 JSONModel 和 YYModel。

一个十个属性的 `User`，手写 `NSSecureCoding` 的完整代价：

```objc
- (void)encodeWithCoder:(NSCoder *)c {
    [c encodeObject:self.userId    forKey:@"userId"];
    [c encodeObject:self.nickname  forKey:@"nickname"];
    [c encodeObject:self.avatarURL forKey:@"avatarURL"];
    [c encodeInteger:self.age      forKey:@"age"];
    [c encodeBool:self.verified    forKey:@"verified"];
    [c encodeDouble:self.balance   forKey:@"balance"];
    [c encodeObject:self.email     forKey:@"email"];
    [c encodeObject:self.phone     forKey:@"phone"];
    [c encodeObject:self.createdAt forKey:@"createdAt"];
    [c encodeObject:self.tags      forKey:@"tags"];
}
```

`initWithCoder:` 是对称的另一坨，每个属性还要挑对 `decodeObjectOfClass:` 还是 `decodeIntegerForKey:`。数下来：encode 12 行，init 14 行，加 `+supportsSecureCoding` 一行，27 行。

加一个属性要动三处：encode 一行、decode 一行，如果引入了新类型还得改 `allowedClasses` 那个集合。

漏一处会怎样？我在 `encodeWithCoder:` 里故意不写 `tags`：

```text
归档前  userId=u-001 nickname=Tommy age=30 verified=1 balance=123.45 tags=(ios, objc)
归档 err = nil   (500 字节)
解档 err = nil
解档后  userId=u-001 nickname=Tommy age=30 verified=1 balance=123.45 tags=(null)
```

两端 `err` 都是 nil。归档成功，解档成功，`tags` 静默变 nil。

没有任何机制能发现这件事。编译器不知道你少写了一行。运行时不知道那个键本该存在。`decodeObjectOfClass:` 拿不到值就返回 nil，这是合法行为。第三节刚验过，字段本来是 nil 时 secure coding 也不报错。只有测试覆盖到那个字段才能发现。属性一多，这就是必然要踩的坑。

### 十几行就能自动化

`class_copyPropertyList` 把属性名要过来，剩下的交给 KVC。这两件事在 [[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]] 那篇里都拆过。`setValue:forKey:` 优先走 setter，标量的装箱拆箱由 KVC 负责：

```objc
@implementation AutoCoding
+ (BOOL)supportsSecureCoding { return YES; }

+ (NSArray<NSString *> *)ac_propertyNames {
    NSMutableArray *names = [NSMutableArray array];
    for (Class c = self; c && c != AutoCoding.class; c = class_getSuperclass(c)) {
        unsigned n = 0;
        objc_property_t *ps = class_copyPropertyList(c, &n);
        for (unsigned i = 0; i < n; i++) [names addObject:@(property_getName(ps[i]))];
        free(ps);
    }
    return names;
}
- (void)encodeWithCoder:(NSCoder *)coder {
    for (NSString *k in [self.class ac_propertyNames])
        [coder encodeObject:[self valueForKey:k] forKey:k];
}
- (instancetype)initWithCoder:(NSCoder *)coder {
    if (!(self = [super init])) return nil;
    for (NSString *k in [self.class ac_propertyNames])
        [self setValue:[coder decodeObjectOfClasses:coder.allowedClasses forKey:k] forKey:k];
    return self;
}
@end
```

22 行，一次写完。`AutoUser` 继承它，一行 `NSCoding` 代码都不用写：

```text
扫到属性 10 个: userId, nickname, avatarURL, age, verified, balance, email, phone, createdAt, tags
归档前  userId=u-001 nickname=Tommy age=30 verified=1 balance=123.45 tags=(ios, objc)
归档 605 字节
解档后  userId=u-001 nickname=Tommy age=30 verified=1 balance=123.45 tags=(ios, objc)
```

27 行换 0 行，而且漏字段这件事从此不可能发生——属性列表是从类元数据里现拿的，加属性自动生效。

代价也在数字里：605 字节 vs 手写版的 500 字节。`valueForKey:` 把 `age`、`verified`、`balance` 三个标量装箱成了 `NSNumber` 对象，各自在 `$objects` 里占一个槽位。手写版用 `encodeInteger:`，直接写成 plist 整数。自动化省下的行数，是拿体积和装箱开销换的。

这段代码离能用还差得远。`readonly` 属性没处理，`setValue:forKey:` 到那里会抛异常。结构体属性没处理。属性名和归档键不一致时的迁移没处理。还有一条最要命：一点缓存都没有，每次归档都重新 `class_copyPropertyList` 一遍。JSONModel 和 YYModel 解决的就是这堆问题，YYModel 的很大一部分性能优势正来自最后那条。下一篇从 `initWithDictionary:error:` 开始追。

---

## 总结

`NSKeyedArchiver` 的产物是一个二进制 plist，核心是 `$objects` 这张扁平表。对象之间靠 `CF$UID` 下标互指，所以环、共享节点、类描述都只写一份。解档时 unarchiver 在调 `initWithCoder:` 之前就把实例登记进表，环才能还原。代价是你在 `initWithCoder:` 里可能拿到一个还没初始化完的对象。

`NSSecureCoding` 防的是类型混淆：归档里的类名是数据，攻击者改掉它就能让你的进程构造任意类并执行其 `initWithCoder:`。我把这个攻击跑通了。但防线在解档器上。`decodeObjectOfClass:` 遇到 `requiresSecureCoding == NO` 的 coder，会忽略 class 参数，退化成 `decodeObjectForKey:`。白名单不递归，容器每一层都要列；写 `NSObject.class` 等于没写。

`NSJSONSerialization` 有两处和 RFC 8259 不一致，都是我实测出来的。默认路径接受一个尾逗号，而 JSON5 的其它放宽都关着。重复 key 取第一个，主流实现取最后一个。另外 `-1e400` 会解析成功并给出 `-inf`，这个对象 `isValidJSONObject:` 判 NO、写入端抛异常，而 `1e400` 是直接拒绝的。环序列化是 SIGSEGV，`@try` 接不住。

JSON 数字的类型没有全丢：整数 `q`、浮点 `d`、布尔 `c`，`objCType` 分得清。丢的是位宽。大整数不会退化成 double，19 位以内走 `SInt64`，之后升 `NSDecimalNumber`，39 位有效数字以内精确。2^53 那个阈值是 JavaScript 的，不是 Foundation 的。

十个属性手写 `NSSecureCoding` 是 27 行，漏一个属性会静默变 nil 且两端都不报错。用 `class_copyPropertyList` 加 KVC 自动化，22 行的基类可以覆盖所有子类，换来的是标量装箱带来的约 20% 体积膨胀。

下一篇 [[iOS JSONModel 源码：Runtime 驱动的属性映射]]。

## 参考资料

### 官方

- [Archives and Serializations Programming Guide — Archives](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Archiving/Articles/archives.html)：根对象、循环引用、条件对象三个概念的权威出处
- [Secure Coding Guide — Validating Input and Interprocess Communication](https://developer.apple.com/library/archive/documentation/Security/Conceptual/SecureCodingGuide/Articles/ValidatingInput.html)："Modifications to Archived Data" 一节讲清了替换攻击。它给的处方只覆盖了写法那一半，见第三节
- [RFC 8259 — The JavaScript Object Notation (JSON) Data Interchange Format](https://www.rfc-editor.org/rfc/rfc8259)：第 4 节重复 key、第 5 节数组语法、第 6 节数字语法与 2^53 互操作性
- [JSONSerialization](https://developer.apple.com/documentation/foundation/jsonserialization)：官方 API 页；文档站是 SPA 抓不到正文，本文引的四条契约与免责声明抄自 SDK 里的 `NSJSONSerialization.h`
- SDK 头文件（本文所有 API 契约的一手来源，`$(xcrun --sdk macosx --show-sdk-path)/System/Library/Frameworks/Foundation.framework/Headers/`）：`NSCoder.h` 的 `decodeObjectOfClass:` 注释、`decodingFailurePolicy` 与 `.error` 语义；`NSKeyedArchiver.h` 的 `API_DEPRECATED` 原文；`NSObject.h` 的 `NSSecureCoding` 协议注释；`NSJSONSerialization.h` 的四条契约

### 本地

- [[iOS 持久化选型：沙盒、plist、NSUserDefaults 与 Keychain]]
- [[iOS KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发]]
- [[iOS 对象模型：类型判断、内存对齐与 Tagged Pointer]]
- [[iOS JSONModel 源码：Runtime 驱动的属性映射]]

---

实验环境：macOS 26.5.2（Build 25F84），Apple Silicon，arm64 原生二进制，Apple clang 21.0.0，SDK MacOSX26.5。全部实验用 `clang -fobjc-arc -framework Foundation -o out prog.m && ./out` 编译运行，没有开模拟器。归档产物用 `plutil -p` 查看。

关于 iOS 上是否一致。本文的结论全部落在 `NSKeyedArchiver`、`NSJSONSerialization`、`NSCoder` 上。这三者在 iOS 与 macOS 是同一份 Foundation 实现，不涉及 UIKit，不涉及沙盒，也不涉及平台条件编译分支；引用到的那些 API，`API_AVAILABLE` 在四个平台上版本一致。我认为结论可以直接迁移。有一条例外要标注：

> 待真机补测：第五节两条解析器行为没在 iOS 上复核，也没做系统版本回扫，我只能保证 macOS 26.5 上如此。一是 `-1e400` 解析成 `-inf` 而 `1e400` 被拒的正负不对称，二是默认路径接受一个尾逗号。复现方法：对 `{"x":-1e400}`、`{"x":1e400}`、`{"a":1,}` 三个输入各调一次 `JSONObjectWithData:options:0 error:` 打印结果，编成 iOS target 后 `xcrun simctl spawn booted ./out` 跑一遍，和上文四个轴的表格逐行对照。
