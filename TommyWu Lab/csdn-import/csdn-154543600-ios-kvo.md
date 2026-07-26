---
title: "【iOS】KVO"
slug: "ios-kvo"
published: 2025-11-09
description: "In order to understand key value observing, you must first understand key value coding. KVC是键值编码 ，在对象创建完成后，可以 动态的给对象属性赋值 ，而 KVO是键值观察 ，提供了一种监"
tags: ["iOS", "Objective-C", "KVO"]
category: "iOS"
draft: false
---

In order to understand key-value observing, you must first understand key-value coding.

`KVC是键值编码`，在对象创建完成后，可以`动态的给对象属性赋值`，而`KVO是键值观察`，提供了一种监听机制，当指定的对象的属性被修改后，则对象会收到通知，所以可以看出`KVO是基于KVC的基础上对属性动态变化的监听`


在iOS日常开发中，经常`使用KVO来监听对象属性的变化`，并及时做出响应，即当指定的被观察的对象的属性被修改后，`KVO会自动通知相应的观察者`，那么`KVO`与`NSNotificatioCenter`有什么区别呢？


- 相同点
- 1、两者的实现原理都是观察者模式，都是用于监听


- 2、都能实现一对多的操作


- 不同点
- 1、KVO只能用于监听对象属性的变化，并且属性名都是通过NSString来查找，编译器不会帮你检测对错和补全,纯手敲会比较容易出错


- 2、NSNotification的发送监听（post）的操作我们可以控制，kvo由系统控制。


- 3、KVO可以记录新旧值变化


## KVO使用


### 1、 基本使用


- 注册观察者


```objective-c
[person addObserver:person forKeyPath:@"age" options:NSKeyValueObservingOptionNew context:nil];
```


- 设置KVO回调


```objective-c
-(void) observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
  if ([keyPath isEqualToString:@"age"]) {
    NSLog(@"KVO发送变化：old = %@, new = %@", change[NSKeyValueChangeOldKey], change[NSKeyValueChangeNewKey]);
  }
}
```


- 移除观察者


```objective-c
[person removeObserver:person forKeyPath:@"age"];
### 2、context使用


在官方文档中，针对`context`有以下说明

![请添加图片描述](./csdn-154543600-ios-kvo-images/01-0174aa4d14744f619b8041463cbb3784.png)


大致含义就是：`addObserver：forKeyPath：options：context：`方法中的上下文context指针包含任意数据，这些数据将在相应的更改通知中传递回观察者。可以通过`指定context为NULL`，从而依靠keyPath即键路径字符串传来确定更改通知的来源，但是这种方法可能会导致对象的父类由于不同的原因也观察到相同的键路径而导致问题。所以可以为每个观察到的keyPath创建一个不同的context，从而`完全不需要进行字符串比较`，从而可以更有效地进行通知解析


通俗的讲，context上下文主要是用于区分**不同对象的同名属性**，从而在KVO回调方法中可以直接使用context进行区分，可以`大大提升性能，以及代码的可读性`


context用来处理有多个观察者的情况：


- 同名键路径冲突：当多个对象或同一对象的不同属性使用相同的 keyPath 时，用 context 精准定位通知来源。
- 性能优化：通过指针地址直接匹配 context，避免字符串比较（keyPath 判断）的性能损耗。
- 安全性：防止父类与子类观察同一 keyPath 时的逻辑混淆。


#### context使用总结


- 不使用context，使用keyPath区分通知来源


```objective-c
//context的类型是 nullable void *，应该是NULL，而不是nil
[self.person addObserver:self forKeyPath:@"nick" options:NSKeyValueObservingOptionNew context:NULL];
```


- 使用context区分通知来源


```objective-c
//定义context
static void *PersonNickContext = &PersonNickContext;
static void *PersonNameContext = &PersonNameContext;

//注册观察者
[self.person addObserver:self forKeyPath:@"nick" options:NSKeyValueObservingOptionNew context:PersonNickContext];
[self.person addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew context:PersonNameContext];


//KVO回调
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context{
    if (context == PersonNickContext) {
        NSLog(@"%@",change);
    }else if (context == PersonNameContext){
        NSLog(@"%@",change);
    }
}
### 3、移除KVO通知的必要性


![请添加图片描述](./csdn-154543600-ios-kvo-images/02-330bbf68b44a493186f233a510229573.png)


删除观察者时，请记住以下几点：


- 要求被移除为观察者（如果尚未注册为观察者）会导致`NSRangeException`。您可以对`removeObserver：forKeyPath：context：`进行一次调用，以对应对`addObserver：forKeyPath：options：context：`的调用，或者，如果在您的应用中不可行，则将removeObserver：forKeyPath：context：调用在`try / catch块`内处理潜在的异常。
- `释放后，观察者不会自动将其自身移除。`被观察对象继续发送通知，而忽略了观察者的状态。但是，与发送到已释放对象的任何其他消息一样，更改通知会触发内存访问异常。因此，您可以`确保观察者在从内存中消失之前将自己删除。`
- 该协议无法询问对象是观察者还是被观察者。构造代码以避免发布相关的错误。一种典型的模式是在观察者初始化期间（例如，`在init或viewDidLoad中）注册为观察者`，并在释放过程中（通常`在dealloc中）注销`，以`确保成对和有序地添加和删除消息，并确保观察者在注册之前被取消注册，从内存中释放出来。`

所以，总的来说，KVO`注册观察者 和移除观察者是需要成对出现的`，如果只注册，不移除，会出现类似`野指针的崩溃，`


崩溃的原因是，由于第一次注册`KVO观察者后没有移除`，再次进入界面，会导致第二次注册KVO观察者，导致`KVO观察的重复注册`，而且第一次的通知对象还在内存中，没有进行释放，此时接收到属性值变化的通知，会出现`找不到原有的通知对象，只能找到现有的通知对象，`即第二次KVO注册的观察者，所以导致了类似野指针的崩溃，即一直保持着一个野通知，且一直在监听


注：这里的崩溃案例是通过`单例对象`实现（崩溃有很大的几率，不是每次必现），因为单例对象在内存是常驻的，针对一般的类对象，貌似不移除也是可以的，但是为了防止线上意外，建议还是移除比较好


### 4、KVO的自动触发和手动出发


- 自动开关，返回NO，就监听不到，返回YES，表示监听


```objective-c
// 自动开关
+ (BOOL) automaticallyNotifiesObserversForKey:(NSString *)key{
    return YES;
}
```


- 自动开关关闭的时候，可以通过`手动关闭监听`


```objective-c
- (void)setName:(NSString *)name{
    //手动开关
    [self willChangeValueForKey:@"name"];
    _name = name;
    [self didChangeValueForKey:@"name"];
}
```


使用手动开关的好处就是你想监听就监听，不想监听关闭即可，比自动触发更方便灵活


### 5、KVO观察：一对多


KVO观察中的`一对多` ，意思是通过`注册一个KVO观察者，可以监听多个属性的变化`


以下载进度为例，比如目前有一个需求，需要根据`总的下载量totalData`和`当前下载量currentData` 来计算当前的`下载进度currentProcess`，实现有两种方式


- 分别观察 总的下载量totalData 和当前下载量currentData 两个属性，当其中一个发生变化计算 当前下载进度currentProcess
- 实现`keyPathsForValuesAffectingValueForKey`方法，将两个观察合为一个观察，即`观察当前下载进度currentProcess`


```objective-c
//1、合二为一的观察方法
+ (NSSet<NSString *> *)keyPathsForValuesAffectingValueForKey:(NSString *)key{

    NSSet *keyPaths = [super keyPathsForValuesAffectingValueForKey:key];
    if ([key isEqualToString:@"currentProcess"]) {
        NSArray *affectingKeys = @[@"totalData", @"currentData"];
        keyPaths = [keyPaths setByAddingObjectsFromArray:affectingKeys];
    }
    return keyPaths;
}

//2、注册KVO观察
[self.person addObserver:self forKeyPath:@"currentProcess" options:(NSKeyValueObservingOptionNew) context:NULL];

//3、触发属性值变化
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    self.person.currentData += 10;
    self.person.totalData  += 1;
}

//4、移除观察者
- (void)dealloc{
    [self.person removeObserver:self forKeyPath:@"currentProcess"];
}
```


> **解释一下上面这段代码：** 我们注册的监听是currentProcess，但在触摸事件里改动的却是 currentData和totalData， 也就是说，你没有直接改 currentProcess， 但是你希望当这两个值（currentData 和 totalData）变化时， currentProcess 的观察回调也会自动触发。而这就是 keyPathsForValuesAffectingValueForKey: 的作用！


其实还有另一种方法，借助ai的回答如下：


![请添加图片描述](./csdn-154543600-ios-kvo-images/03-3591ab4748eb485aacaf02c7c5ec678c.png)


### 6、KVO观察可变数组


KVO是基于KVC基础之上的，所以可变数组如果直接添加数据，是不会调用`setter方法`的，所有`对可变数组的KVO观察下面这种方式不生效`的,即直接通过`[self.person.dateArray addObject:@"1"];`向数组添加元素，是不会触发kvo通知回调的


```objective-c
//1、注册可变数组KVO观察者
self.person.dateArray = [NSMutableArray arrayWithCapacity:1];
    [self.person addObserver:self forKeyPath:@"dateArray" options:(NSKeyValueObservingOptionNew) context:NULL];

//2、KVO回调
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context{
    NSLog(@"%@",change);
}

//3、移除观察者
- (void)dealloc{
 [self.person removeObserver:self forKeyPath:@"dateArray"];
}

//4、触发数组添加数据
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    [self.person.dateArray addObject:@"1"];
}
```


在KVC官方文档中，针对`可变数组的集合类型，`有如下说明，即访问集合对象需要需要通过`mutableArrayValueForKey`方法，这样才能将元素添加到可变数组中


![请添加图片描述](./csdn-154543600-ios-kvo-images/04-8b655f8a30814a498995b8a8f39d15d7.png)


修改代码如下：


```objective-c
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    // KVC 集合 array
    [[self.person mutableArrayValueForKey:@"dateArray"] addObject:@"1"];
}
```


就可以实现了。

打印出来的kind是一个枚举类型：


```objective-c
typedef NS_ENUM(NSUInteger, NSKeyValueChange) {
    NSKeyValueChangeSetting = 1,//设值
    NSKeyValueChangeInsertion = 2,//插入
    NSKeyValueChangeRemoval = 3,//移除
    NSKeyValueChangeReplacement = 4,//替换
};
```


mutableArrayValueForKey


**步骤 1：** 优先查找数组操作方法


方法名规则：

在对象的类中搜索以下方法（优先级顺序）：


- insertObject:inAtIndex: 和 removeObjectFromAtIndex:（对应 NSMutableArray 的基础增删方法）
- insert:atIndexes: 和 removeAtIndexes:
- replaceObjectInAtIndex:withObject: 或 replaceAtIndexes:with:（高性能替换方法）

**触发条件：**

若类实现了至少 一个插入方法 和 一个删除方法，则所有 NSMutableArray 操作（如 addObject:、removeLastObject）都会被 自动映射到这些自定义方法，确保数据同步和 KVO 通知。


**步骤 2：** 退而使用 Setter 方法


- 方法名规则： 若未找到数组操作方法，则查找 set: 方法。
- 触发条件： 每次通过代理对象修改数组时，会 生成新数组 并调用 set: 方法 更新原属性。 性能问题：频繁生成新数组会导致性能损耗（需优先实现步骤 1 的方法优化）。

**步骤 3：** 直接访问实例变量


- 变量名规则： 若 accessInstanceVariablesDirectly 返回 YES，则按顺序查找实例变量 _ 或 。
- 触发条件： 代理对象直接操作实例变量（必须是 `NSMutableArray 或其子类实例`），修改会 直接影响原数据 并触发 KVO 通知。

**步骤 4：** 兜底异常处理


- 触发条件： 若上述方法均未找到，返回一个代理对象，但其操作会调用 setValue:forUndefinedKey:。
- 默认行为： 抛出 NSUndefinedKeyException 异常。可通过重写 setValue:forUndefinedKey: 自定义处理逻辑。

用一句话理解，mutableArrayValueForKey: 就是给数组加了一个 KVO 代理壳，让你对数组做增删改操作时，系统能够自动触发观察者通知，并且按优先级选择最优的触发方式。


## KVO底层的一些结论


KVO对`成员变量`不观察，只对`属性`观察，属性和成员变量的区别在于`属性多一个 setter 方法，而KVO恰好观察的是setter 方法`


在注册观察者后，实例对象的`isa指针`指向由`LGPerson类`变为了`NSKVONotifying_LGPerson中间类`，即实例对象的isa指针指向发生了变化


关于`中间类`，有如下说明：


- 实例对象isa的指向在注册KVO观察者之后，由`原有类更改为指向中间类`


- 中间类重写了观察属性的`setter方法、class、dealloc、_isKVOA`方法


- dealloc方法中，移除KVO观察者之后，实例对象`isa指向由中间类更改为原有类`


- 中间类从创建后，就一直存在内存中，不会被销毁


自定义KVO大致分为以下几步


注册观察者 & 响应

1、验证是否存在setter方法


2、保存信息


3、动态生成子类，需要重写class、setter方法


4、在子类的setter方法中向父类发消息，即自定义消息发送


5、让观察者响应


移除观察者

1、更改isa指向为原有类


2、重写子类的dealloc方法

---

原文发布于 CSDN：[【iOS】KVO](https://blog.csdn.net/2402_86720949/article/details/154543600)
