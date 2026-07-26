---
title: "【iOS】Objective - C初探之面向对象"
slug: "ios-objc-oop-intro"
published: 2025-08-31
description: "面向对象与面向过程设计思想 面向过程 是以过程（步骤）为中心的编程方式。 把一件要做的事，从头到尾， 按照步骤拆成一系列操作 ，一步步完成。 重点是 怎么做，顺序是什么 面向过程的特点 注重 流程控制 （先做什么、后做什么） 编程思维是： “先干这个，再干那个” 主要用 函数 组"
tags: ["Objective-C", "面向对象"]
category: "iOS"
draft: false
---

## 面向对象与面向过程设计思想


**面向过程**是以过程（步骤）为中心的编程方式。


把一件要做的事，从头到尾，**按照步骤拆成一系列操作**，一步步完成。


重点是**怎么做，顺序是什么**


**面向过程的特点**


注重**流程控制**（先做什么、后做什么）


编程思维是：**“先干这个，再干那个” **


主要用**函数**组织代码


代码容易从上到下直接执行


小程序简单直接，速度快


面向对象以对象（数据 + 行为）为中心的编程方式


把现实世界的实体抽象为程序中的对象，每个对象既包含数据（属性），又有能做的事（方法），强调封装 继承 多态


**面向对象的特点**


- 注重**数据和行为的封装**
- 编程思维是：**“谁负责做这件事”**
- 代码模块化，便于维护和扩展
- 适合**大型、复杂系统**


## 类与对象的概念


类是对同一类食物高度的抽象，类中定义了这一类对象所应具有的静态属性（属性）和动态属性（方法）


对象是类的一个实例，是一个具体事物


类与对象是抽象与具体的关系


类其实就是一种数据类型，他的变量就是 对象


### 类与类之间的关系


1. 继承（is-a） 一个类是另一个类的特殊版 “学生是人” 2. 组合 / 聚合（has-a） 一个类拥有另一个类作为成员 “学校有学生” 3. 依赖（use-a） 一个类短暂使用另一个类 “老师使用教案” 4. 关联（link-a） 类之间长期联系 “医生关联病人”


## OC与面向对象


对象是oc程序的核心


类是用来创造同一类型的对象的模板，在一个类中定义了该类对象所具有的成员变量和方法


类可以看成是静态属性（实例变量）和动态属性（方法）的结合体 

 


## OC类的声明和实现


```objective-c
@interface NewClassName: ParentClassName
{
    实例变量;
    ...
}
方法的声明;
    ...
@end

@implementation NewClassName
方法的实现
{
    //code
}
@end
```


```objective-c
@interface Person : NSObject
{
    //实例变量声明
    int identify;
    int age;
}
//方法声明
- (id) initWithAge:(int) _age identify: (int) _identify;
- (int) getIdentify;
- (int) getAge;
- (void) setAge:(int) _age;
@end
```


类的声明放在 “类名.h”文件


### 实例变量


实例变量可以使用oc语言任何一种数据类型（包括基本类型和指针类型）


在声明实例变量的时候不能为其初始化，系统默认会初始化


实例变量的默认作用域是整个类


### OC的方法声明


oc中方法和其他语言一样，是一段用来完成特定功能的代码片段：声明格式：


![图片](./csdn-147576942-oc-objective-c-images/01-1cdff7c30b1249b6a07f55f65fefee6b.png)


```objective-c
- (void) setAge:(int) _age;
- (int) getAge;
- (id) initWithAge:(int) _age identify:(int) _identify
+ (Person *) shareRerson;
### OC中方法的调用


oc语言中采用特定的语言调用类或者实例（对象）的方法称为发送消息或方法调用


oc中方法的调用有两种：


【类名或对象名     方法名】


【对象名.方法名】 


```objective-c
[ClassOrInstance method];

[ClassOrInstace method:arg1];

[ClassOrInstance method1:arg1 method2:arg2];

[[ClassOrInstance method:arg1] otherMethod];
### 类的实现


```objective-c
@implementation Person

-(id) initWithAge:(int)_age identify:(int)_identify:(int) _identify
{
    if(self = [super init])
    {
        age = _age;
        identify = _identify;
    }
    return self;
}
- (int) getIdentify
{
    return identify;
}
-(int) getAge
{
    return age;
}
-(void) setAge:(int)_age
{
    age = _age;
}

@end
```


```objective-c
#import "FKPerson.h"

@implementation FKPerson
{
    int _testAttr;
}

-(void) setName:(NSString*) n andAge: (int) a
{
    _name = n;
    _age = a;
}

-(void) say:(NSString *) content
{
    NSLog(@"%@" , content);
}

-(NSString*) info
{
    [self test];
    return [NSString stringWithFormat:
            @"我是一个人，名字为：%@，年龄为%d。" , _name, _age];
}

-(void) test
{
    NSLog(@"FKPersonw类的类方法，通过类名调用");
}

+ (void) foo
{
    NSLog(@"FKPerson的类的类方法，通过类名调用");
}
@end
```


```objective-c
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface FKPerson : NSObject
{
    //下面定义两个成员变量
    NSString* _name;
    int _age;
}
//下面定义了一个setName:andAge:方法
-(void) setName:(NSString*) name andAge: (int) age;
//下面定义了一个say：方法，但不提供实现
-(void) say: (NSString *) content;
//下面定义了一个不带形参的info方法
-(NSString*) info;
//定义了一个类方法
+ (void) foo;
@end

NS_ASSUME_NONNULL_END
```


## OC面向对象的基本概念——指针


oc语言中除基本数据类型意外的变量类型都称为指针类型


oc中的对象是通过指针对其操作的  


```objective-c
// 声明了一个NSString类型的指针变量，但他并没有指向任何一个对象

NSString *s

//使用alloc的方法创建了一个NSString类型的对象并用s 指向他，以后可以通过s完成对其的操作

s = [[NSString alloc] initWithString:@"Hello iPhine4S"];
```


类是静态的概念


对象是alloc出来的，存放在堆区，类的每个实例变量在不同的对象中都有不同的值（静态变量除外）


方法也只是在被调用的时候，程序运行的是才占用内存


### 对象的创建和使用


oc中对象通过指针来声明。如： ClassA *object;


oc中对象的创建，使用alloc来创建一个对象。编译器会对object对象分配一块可用的内存地址。然后需要对对象进行初始化即调用init方法，这样这个对象才可以使用。如：


Persom* person = [Person alloc];


person = [person init];


ClassA* object = [[ClassA alloc] init];


### 对象的初始化


对象必须先创建，然后初始化，才能使用。


NSObject *object = [[NSObject alloc] init];  


### self关键字


self关键字总是指向该方法的调用者（对象或类），但self出现在实例方法中时，self代表调用该方法的对象；self出现在类方法时，self代表调用该方法的类


![图片](./csdn-147576942-oc-objective-c-images/02-e2c26ea2dcc6424ba8c8cb9a898ca0dd.png)


这样的做法当然不够好， 按照语法的话，我们需要创建一个对象，调用这个对象的方法，才可实现我们的run方法。但是在实际设计中，我们发现我们其实做了一件毫无意义的事情，我们实际上无需创建一个新的对象去执行这一个操作，我们可以通过**self**关键字可以获得调用该方法的对象。


self总是代表当前类的对象，当出现在方法体时，它所代表的对象是不确定的，但是类型是确定的，它所代表的对象只能是当前类的实例。当我们在主函数中调用方法时，他的对象就确定了。谁调用这个方法，self就代表谁。

但是self却不能出现在类方法中，因为self所代表的是一个实例，而不是一个类。


```objective-c
@interface Myperson : NSObject {
    NSString* _name;
    int _age;
}
- (void) say: (NSString*) content andAge : (int) age;
- (void) whatSay;
+ (void) foo;
- (void) rap;
- (void) dance;
@end
@implementation Myperson {

}
- (void) say: (NSString*) content andAge : (int) age {
    _name = content;
    _age = age;
}
- (void) whatSay {
    NSLog(@"我叫%@，今年%d", _name, _age);
}
- (void) rap {
    NSLog(@"man");
}
- (void) dance {
    [self rap];
    NSLog(@"what can i say");
}
+ (void) foo {
    NSLog(@"我是老大");
}
@end
```


当self 作为对象或类本身的默认引用使用时，程序可以像访问普通指针变量一样访问这个self引用，甚至可以把self当成普通方法的返回值


![图片](./csdn-147576942-oc-objective-c-images/03-df58115c560441a8a9b6a5e272f79b21.png)


### id类型


Oc 提供了一个 id 类型，这个id类型可以代表所有对象类型。任意类的对象都能赋值给id类型变量。

 


id类型的变量来调用方法时，oc会执行动态绑定，跟踪判断该对象所属的类，并在运行时确定需要动态调用的方法。

![图片](./csdn-147576942-oc-objective-c-images/04-4ba6d5f81ac04011a8999d4d5050eacb.png)

---

原文发布于 CSDN：[【OC】Objective - C初探之面向对象](https://blog.csdn.net/2402_86720949/article/details/147576942)
