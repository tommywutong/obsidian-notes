---
title: "【iOS】Effective Objective-C 熟悉Oc"
slug: "effective-objc-basics"
published: 2026-02-12
description: "熟悉Objective C 了解Objective C语言的起源 1. 使用消息结构的语言，其运行时所执行的代码由运行环境决定，使用函数调用的语言，则由编译器决定 详细说说： 在C语言中，编译器在编译阶段或链接阶段就已经知道了函数在内存中的地址（或相对偏移量），他生成的汇编指令是"
tags: ["Objective-C", "Effective Objective-C"]
category: "iOS"
draft: false
---

## 熟悉Objective-C


### 了解Objective-C语言的起源


#### 1. 使用消息结构的语言，其运行时所执行的代码由运行环境决定，使用函数调用的语言，则由编译器决定


详细说说：


在C语言中，编译器在编译阶段或链接阶段就已经知道了函数在内存中的地址（或相对偏移量），他生成的汇编指令是硬编码的。**运行时的行为：** CPU 执行到这一行指令时，毫不犹豫地跳到那个地址执行。运行时环境（Runtime），也没有查找过程。如果不小心那个地址是错的，程序直接崩溃，没有任何商量的余地。


在 Objective-C 中，**运行环境 (Runtime)** 是真正的“决策者”。编译器只是一个“发令员”，它不决定具体执行哪段代码，它只负责发送请求。他不知道具体的代码在哪里，甚至不知道能不能响应这个消息。在程序运行到这一行时，objc_msgSend接管控制权（拿到obj指针；查找所属Class；在Class方法列表/缓存中通过方法名查找对应的函数指针；如果找到了，Runtime才跳转去执行函数指针指向的代码，如果找不到，Runtime会去父类找，或者启动消息转发机制）。


这个区别带来了什么？ 正是因为由运行环境决定，所以在Oc中我们可以重写原有方法，对动态类型（ID）发送任何消息，抑或实现消息转发。


#### 2. Objective - C 的对象强制只能在堆（Heap）上分配内存，而不能在栈（Stack）上分配。我们在栈上拥有的，永远只是指向那个堆内存的指针。


C++ 允许在栈上分配对象，但同时也带来了“对象切割”的风险。Objective-C 为了支持纯粹的动态多态性，必须避免这个问题。


- **栈的特性：** 栈上的变量在编译期必须确定**确切的大小**（Size）。
- **多态的矛盾：** 在面向对象编程中，子类通常比父类大（因为子类有更多的属性）。


如果ObjC允许栈分配：


```
// 假设这是合法的（实际上是非法的）
NSObject obj; // 编译器在栈上预留了 NSObject 大小的空间（很小，只有一个 isa 指针）
​
// 如果我们尝试把一个巨大的子类赋值给它
MyHugeObject *huge = [[MyHugeObject alloc] init];
obj = *huge; // jj了
```


因为 `obj` 在栈上只预留了很小的空间，`MyHugeObject` 中多出来的属性根本塞不进去，会被强行“切掉”（Sliced off）。这会导致数据丢失，对象变成一个残废的 `NSObject`，多态性完全失效。


因此在ObjC中，强制使用指针。无论对象本身多大，指针的大小是固定的，栈上只储存指针，而真正的内容存在堆中，完美支持多态。


Objective-C 是一门**引用计数（Reference Counting）**语言。对象经常需要在不同的函数、不同的控制器之间传递。为了让对象能够“活”得比创建它的函数更长，ObjC 选择将对象全部分配在堆上，通过 `retain/release` 来灵活控制它的生死。


Objective-C 的核心是 Runtime。所有的对象都必须以 `isa` 指针开头。


- **isa 指izzling：** KVO（键值观察）等技术会在运行时动态修改对象的 `isa` 指针，甚至动态添加类。
- **内存布局：** 虽然理论上栈对象也可以有 `isa`，但 ObjC 的消息发送机制（`objc_msgSend`）以及弱引用系统（Weak Table）等基础设施，都是围绕着“对象是堆上的稳定内存块”这一假设构建的。如果对象在栈上，随着函数调用栈的推入弹出，内存地址不稳定，会让 Runtime 的很多特性难以实现。


Objective-C 禁止栈上分配对象，是用**微小的性能损耗**（堆分配比栈慢，且有指针间接访问的开销），换取了：


- **无限的多态能力**（彻底解决对象切割）。
- **灵活的生命周期管理**（引用计数）。
- **强大的动态运行时特性**。


### 在类的头文件中尽量少引入其他头文件


#### 1. 除非确有必要，否则不要引入头文件。一般来说，应在某个类的头文件中使用向前声明来提及别的类，并在实现文件中应入那些类的头文件，这样做可以尽量降低类之间的耦合


- 解决了循环依赖
- 减少了编译时间


当编译器不需要知道具体的内存布局或者方法细节时，可以使用Class


```
@class NLSong; // .h 文件
@property (nonatomic, strong) NLSong *currentSong;
​
- (void)playSong:(NLSong *)song;
```


当编译器必须知道具体细节时，必须#import


- 继承：父类的所有属性和方法子类都要有，编译器必须知道父类
- 遵循协议：虽然也可以用 `@protocol` 向前声明，但通常直接 import 更常见，除非是为了解耦


#### 2，有时无法使用向前声明，比如声明某个类遵循某一协议。这种情况下，尽量把“该类遵循某协议”的这条声明单独移至"class-continuation分类中"。如果不行的话，就把协议单独放在一个头文件中，然后引入


我们详细来说说：当你只是**使用**某个类或协议作为类型时，编译器不需要知道它的内部结构，只需要知道名字。但是，当你声明一个**类遵循某个协议**时，情况就变了：编译器在编译类的定义时，必须验证类是否有能力实现协议里规定的方法，或者至少需要知道协议里到底有什么方法（继承关系、属性等）。**编译器必须看到协议的完整定义**。因此，你被迫在 `.h` 文件中写 `#import`


如果协议定义在一个很大的头文件里，那么引入就会连带引入一堆无关的东西，导致编译变慢，甚至引发循环引用。


##### 方案1： 移至“Class-Extension” (类扩展)


适用场景于 这个协议只是你**内部实现细节**，外部使用者不需要知道你遵循了这个协议。


我们在h中不引入任何协议头文件，自在m中引入，不影响外部编译


```
// NLPlayerManager.m
#import "NLPlayerManager.h"
#import <SDWebImage/SDWebImageManager.h> // ✅ 引用只发生在这里，不扩散
​
//  这里就是 Class-Extension
// 编译器在编译 .m 时会看到这个声明，知道你要实现这些方法
@interface NLPlayerManager () <SDWebImageManagerDelegate>
// 你甚至可以在这里定义私有属性
@property (nonatomic, strong) SDWebImageManager *imageManager;
@end
​
@implementation NLPlayerManager
​
- (void)imageManager:(SDWebImageManager *)imageManager didFinishWithImage:(UIImage *)image {
    // 实现协议方法
}
​
@end
```


**封装性 (Encapsulation)**：遵循什么协议是类的“内部实现细节”。外部不需要知道你用 SDWebImage 还是 YYImage。


**编译隔离**：如果 `SDWebImage` 升级了或改名了，所有引用 `NLPlayerManager.h` 的其他业务模块（比如 `HomeVC`）完全不需要重新编译，因为它们根本不知道 `SDWebImage` 的存在。


##### 方案2： 协议单独放在一个头文件


**适用场景**： 这个协议必须是**公开**的，外部使用者需要知道你遵循了这个协议，或者外部需要使用这个协议类型。


把协议从类文件中剥离出来，放到一个只有协议的 `.h` 文件中。


```
// NLPlayable.h
// 这个文件非常轻量，没有复杂的依赖
​
@class NLSong; // 配合向前声明使用
​
@protocol NLPlayable <NSObject>
- (void)playSong:(NLSong *)song;
@end
```


```
// NLMusicPlayer.h
#import "NLPlayable.h" // 引入这个轻量级文件
​
// 因为协议文件很小，即使 import 也没什么负担
@interface NLMusicPlayer : NSObject <NLPlayable>
@end
### 多用字面量语法（语法糖）


**类型** **字面量语法 (推荐)** **等价的旧式写法 (Verbose)** **备注** **NSNumber**(整数) `@100` `[NSNumber numberWithInt:100]` 还有 `numberWithInteger:` **NSNumber**(浮点) `@3.14` `[NSNumber numberWithDouble:3.14]` **NSNumber**(布尔) `@YES` `[NSNumber numberWithBool:YES]` **NSNumber**(字符) `@'A'` `[NSNumber numberWithChar:'A']` **NSString** `@"Hello"` `[NSString stringWithUTF8String:"Hello"]` (仅作示意，NSString 本身就是特殊的) **NSArray** `@[a, b, c]` `[NSArray arrayWithObjects:a, b, c, nil]` **注意结尾的 nil** **NSDictionary** `@{key : value}` `[NSDictionary dictionaryWithObjectsAndKeys:value, key, nil]` **注意 Key-Value 顺序**


- 字面量创建的对象全是**“不可变”**的
- **小心 `nil`**：在使用字面量插入数据前，如果数据可能为空，务必做判空处理，否则 App 会闪退。
- 通过取下标的操作来访问数组下表或字典中的键对应的元素。


### 多用类型常量，少用#define预处理指令


- #### 不要用预处理指令定义常量。这样定义的常量不含类型信息，编译器知识会在编译前据此执行查找和替换操作。即使有人重新定义常量值，也不会产生警告


const活在预处理阶段，把代码里所有的宏名直接替换成后面的字符串。而const则是在编译阶段，编译器知道这是一个变量，有具体的类型（`NSTimeInterval`），并且被标记为只读。它会进入符号表（Symbol Table）。


- #### 在实现文件中还是要static const 来定义“只在编译单元内可见的常量”。由于此类常量不在全局符号表中，所以无需为其名称加前缀


在我们不加static的时候，默认是extern，编译器会创建一个全局符号，如果你在另一个文件也写了相同的一句定义，那么编译器在编译时都没有问题，但在链接阶段，链接器会报错：重复定义。而加上static就不会出现这种情况


- #### 在头文件使用extern来声明全局变量，并在相关实现文件中定义其值。这种常量要出现在全局符号表中，所以其名称应加以区隔，通常用与之相关的类名做前缀


在头文件中，我extern声明这个变量，但是现在不分配内存 先进行编译，等链接阶段再去找到他。


在实现文件中，对他进行赋值，为其分配内存，写入具体的值。


**什么是全局符号表？** 链接器（Linker）在把所有的 `.o` 文件（编译产物）合并成一个可执行文件时，会把所有全局符号放在一张大表里进行匹配。如果不加前缀，如果你的文件中定义来一个变量，同时，你引入的第三方库中也定义了一个全局变量，链接器在合并时就会因符号重复定义而崩溃。


因此，需要加上类前缀或模块前缀来模拟命名空间，保证全局唯一性。


- #### 对于对象类型，const位置也很重要 正确：`NSString * const` 是指针常量。
- `NLNotification` 这个**指针**一旦指向了那个字符串对象的地址，就不能变了。这才是我们要的“常量”。


	错误：`const NSString *`


- 是指向常量的指针。
- 这意味着你不能通过指针修改字符串内容（但在 ObjC 中 NSString 本来就不可变），但你可以让 `NLNotification` 指向别的字符串。这就不安全了。


### 用枚举表示状态、选项、状态码


在Objective-C中，我们应该摒弃原始的 C 语言 `enum` 写法，转而使用 Apple 提供的两个宏：**`NS_ENUM`** 和 **`NS_OPTIONS`**。


这两个宏不仅让代码更规范，还能确保你的代码在 C++ 模式下编译通过，并且能完美地桥接到 Swift。


- #### 用枚举表示状态机的状态、传递给方法的选项以及状态码等值，给这些值起一个易懂的名字
- #### 如果把传递给某个方法的选项表示为枚举类型，而多个选项又可以同时使用，那么就讲各选项值定义为2的幂，以便通过按位或操作将其组合起来
- #### NS_ENUM与NS_OPTIONS定义枚举类型，并指明底层数据类型，可以确保枚举使用开发者所选的底层数据类型实现的，而不是编译器所选类型


在古老的 C 语言标准中，编译器在处理 `enum` 时有很大的自由度。编译器只需要保证底层类型能装得下你定义的最大值即可。


```
enum NLConnectionState {
    NLConnectionStateDisconnected = 0,
    NLConnectionStateConnected = 1,
};
```


**编译器 A** 可能会看这两个值很小（0 和 1），为了省内存，决定用 `char` (1 字节) 来存储它。


**编译器 B** 可能会为了性能对齐，决定用 `int` (4 字节) 来存储它.


如果你编写的是一个静态库（.a 或 .framework），你的库是用编译器 A 编译的（占 1 字节）。而使用你库的 App 是用编译器 B 编译的（以为占 4 字节）。当 App 试图读取你的枚举值时，内存读写就会错位，导致崩溃或数据异常。


而apple提供的宏强制制定底层数据类型会在与swift桥接和向前声明中更有优势。


- #### 处理枚举类型的switch不要实现default分之。 不写 `default` 分支是为了利用编译器的**全面性检查 (Exhaustiveness Check)**，这是防止“漏改”逻辑的最后一道防线。

---

原文发布于 CSDN：[Effective Objective-C 熟悉Oc](https://blog.csdn.net/2402_86720949/article/details/157810765)
