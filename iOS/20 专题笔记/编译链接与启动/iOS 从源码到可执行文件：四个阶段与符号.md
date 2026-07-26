---
title: 【iOS】从源码到可执行文件：四个阶段与符号
published: 2026-07-27
description: 200 个没人引用的 C 函数被链接器删得一个不剩，200 个没人引用的 ObjC 类一个都删不掉，二进制多出 103 KB。这不是 bug，是编译器往 section 上打的一个标志位。
tags:
  - iOS
  - Objective-C
  - Compile
  - Link
  - Mach-O
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 24
draft: true
---
# 从源码到可执行文件：四个阶段与符号

先看一组我跑出来的数字。

一个空的 `.m` 文件链出来的可执行文件是 51144 字节。往里加 200 个谁也没引用过的 C 函数，开 `-dead_strip` 链接，产物还是 51144 字节，一个字节不差。换成 200 个谁也没引用过的 Objective-C 类，同样开 `-dead_strip`，产物变成 154560 字节。多出来 103416 字节，`nm` 里 200 个类、200 个方法一个没少。

包体积优化里这是个常见困惑：明明开了死代码剥离，为什么没用的类还在。答案不在链接器的算法里，在编译器生成的一行汇编指令上，第七节会拆开看。

要看懂那一行，得先知道从 `.m` 到可执行文件中间到底发生了什么。这篇把预处理、编译、汇编、链接四个阶段各自单独跑一遍，把每一步的产物摊开，再顺着符号这条线走到 ObjC 元数据是怎么进二进制的。

---

## 一、四个阶段，两个进程

拿这个文件当靶子：

```objc
#import <Foundation/Foundation.h>

#define GREETING "hello"

@interface Greeter : NSObject
- (void)greet;
@end

@implementation Greeter
- (void)greet {
    NSLog(@"%s from %@", GREETING, NSStringFromClass([self class]));
}
@end

int counter = 0;

static int bump(int n) { return n + 1; }

int main(int argc, char *argv[]) {
    @autoreleasepool {
        Greeter *g = [Greeter new];
        [g greet];
        counter = bump(counter);
    }
    return 0;
}
```

四个阶段分别是四条命令：

```shell
clang -E prog.m -o prog.i          # 预处理
clang -S prog.m -o prog.s          # 编译（到汇编）
clang -c prog.m -o prog.o          # 汇编（到目标文件）
clang prog.o -framework Foundation -o prog   # 链接
```

产物的样子：

| 阶段 | 产物 | 字节 | 行数 |
| --- | --- | --- | --- |
| 源文件 | `prog.m` | 457 | 26 |
| 预处理 | `prog.i` | 4,839,643 | 90,384 |
| 编译（IR） | `prog.ll` | 7,320 | 112 |
| 编译（汇编） | `prog.s` | 5,667 | 189 |
| 汇编 | `prog.o` | 3,544 | 二进制 |
| 链接 | `prog` | 51,048 | 二进制 |

26 行源码先膨胀到 9 万行，再收缩到 189 行汇编。中间那一次膨胀全部来自 `#import`，下一节专门算这笔账。

`clang -ccc-print-phases` 会把驱动器自己的阶段划分打出来，比"四个阶段"这个说法更细：

```text
               +- 0: input, "prog.m", objective-c
            +- 1: preprocessor, {0}, objective-c-cpp-output
         +- 2: compiler, {1}, ir
      +- 3: backend, {2}, assembler
   +- 4: assembler, {3}, object
   |- 5: input, "Foundation", object
+- 6: linker, {4, 5}, image
7: bind-arch, "arm64", {6}, image
```

编译被拆成 compiler（前端，出 LLVM IR）和 backend（后端，出汇编文本）两步，这正是 `-S -emit-llvm` 和 `-S` 的分界线。

但阶段划分不等于进程划分。`clang -###` 打出真正 fork 出去的命令：

```text
"…/usr/bin/clang" …
"…/usr/bin/ld" …
```

只有两个。预处理、前端、后端、汇编器全在一个 `clang` 进程里完成，中间产物根本不落盘。汇编器是集成的，加 `-fno-integrated-as` 才会真的调外部的 `as`。所以"四个阶段是四个程序依次执行"这句话，在今天的工具链上只对了一半。

### 每一步在做什么

预处理只干三件事：展开宏、把 `#include` / `#import` 的文件内容原样贴进来、处理条件编译。它不认识 C 语法。我这个文件里 `GREETING` 被换成了 `"hello"`，`@interface` 之类原样留着：

```objc
@implementation Greeter
- (void)greet {
    NSLog(@"%s from %@", "hello", NSStringFromClass([self class]));
}
@end
```

编译是词法、语法、语义分析加代码生成，产物是汇编文本。ObjC 的特殊之处在这一步就体现出来了，`prog.s` 里除了 `__TEXT,__text` 还有一串 `__objc_` 开头的 section：

```asm
	.section	__TEXT,__objc_classname,cstring_literals
	.section	__DATA,__objc_const
	.section	__DATA,__objc_data
	.section	__TEXT,__objc_methname,cstring_literals
	.section	__TEXT,__objc_methtype,cstring_literals
	.section	__DATA,__objc_classrefs,regular,no_dead_strip
	.section	__DATA,__objc_classlist,regular,no_dead_strip
	.section	__DATA,__objc_imageinfo,regular,no_dead_strip
```

类的结构体、方法列表、方法名字符串，全是编译器在这一步铺进静态数据区的。运行时不"创建"类，它只是把这些现成的数据登记一遍。第六节展开。

汇编把汇编文本翻译成机器码，产出一个 Mach-O 目标文件（`file` 说 `Mach-O 64-bit object arm64`）。到这一步地址还都是相对的，外部符号还悬着。

链接把若干 `.o` 和库合并成一个映像，解析符号引用，做重定位，写出 Mach-O 可执行文件。

---

## 二、一行 `#import` 的账单

单独把这两行编译一次：

```objc
#import <Foundation/Foundation.h>
int main(void){return 0;}
```

`clang -E` 之后是 90,360 行、4.84 MB。按 `# line` 指令统计，展开了 647 个不同的头文件。

贡献行数最多的前几名，没有一个是 Foundation 自己的：

```text
  2998  <SDK>/…/CarbonCore.framework/Headers/MacErrors.h
  2724  <SDK>/usr/include/mach/task.h
  2072  <SDK>/…/Security.framework/Headers/cssmapi.h
  2030  <SDK>/…/Security.framework/Headers/cssmtype.h
  1798  <SDK>/usr/include/mach/mach_port.h
  1766  <SDK>/…/CarbonCore.framework/Headers/Gestalt.h
```

文本包含是传递闭包，你导入 Foundation，就顺着依赖把 Security、CoreServices、mach 的一大片头文件全拖了进来。做个横向对照：`#include <stdio.h>` 是 568 行，Foundation 是 90,360 行，Foundation 加 AppKit 是 156,362 行。

代价是可以量出来的。同一段函数体，加不加那行 `#import`，`clang -c` 的耗时（各取 5 次最小值）：

```text
带 Foundation:  -E 0.161s   -c 0.224s
无任何 import:  -E 0.023s   -c 0.026s
```

差 8.6 倍，绝对值 0.2 秒。一个文件 0.2 秒不算什么，一千个文件就是 200 秒，而且每次全量编译都要重来一遍——同样的 9 万行，被反复解析一千次。

### PCH 和 modules 各自解决了哪一半

预编译头（PCH）的思路是把这 9 万行解析一次、把 AST 序列化下来，之后每个文件直接反序列化。modules 的思路是让每个框架自己编成一个二进制模块（`.pcm`），谁要用谁加载。

我用 20 个各自 `#import <Foundation/Foundation.h>` 的小文件测了四轮：

| 方式 | 20 个文件的编译耗时 |
| --- | --- |
| 文本包含 | 4.38 ~ 4.60 s |
| modules（冷缓存，含建模块） | 1.79 ~ 1.99 s |
| modules（热缓存） | 0.89 ~ 0.98 s |
| PCH（另计 0.71 s 生成 16 MB 的 .pch） | 0.52 s |

热缓存下 modules 比文本包含快 4.7 倍。冷缓存也快 2.3 倍，因为建一次模块的钱能摊到 20 个文件上。模块缓存目录是 26 MB、31 个 `.pcm`，Foundation 一个 import 带出来的：CoreFoundation、CoreGraphics、Security、Dispatch、IOKit、XPC……和上面那张行数榜是同一批。

modules 开着的时候，`clang -E` 的输出直接变成 9 行：

```text
# 1 "big.m"
…
#pragma clang module import Foundation /* clang -E: implicit import for #import <Foundation/Foundation.h> */
int main(void) { @autoreleasepool { NSLog(@"%@", [NSString stringWithFormat:@"%d", 1]); } return 0; }
```

90,360 行文本被一条 pragma 顶掉了。这就是 `@import Foundation;` 在干的事，它和 `#import <Foundation/Foundation.h>` 在 `-fmodules` 下等价，区别只是前者不开 modules 会直接报错（`use of '@import' when modules are disabled`）。

顺便验一件我以前只是"听说"的事：模块是独立编译的，所以调用方的宏进不去头文件。构造一个恶意的：

```objc
#define NSString Boom
#import <Foundation/Foundation.h>
int main(void){ @autoreleasepool { NSLog(@"%@", [[NSString alloc] initWithUTF8String:"x"]); } return 0; }
```

文本包含下，报错点全在 SDK 头文件里：

```text
…/Foundation.framework/Headers/NSObjCRuntime.h:633:53: error: format argument not a string type
…/Foundation.framework/Headers/NSString.h:195:63: error: format argument not a string type
…/Foundation.framework/Headers/NSString.h:376:81: error: function does not return string type
```

`-fmodules` 下只有一条，指着我自己的第 3 行：

```text
poison2.m:3:51: error: use of undeclared identifier 'Boom'
1 error generated.
```

我的判断：这一条比编译速度更值钱。头文件顺序、宏污染引发的诡异报错，是 ObjC 工程里最难查的一类构建问题，modules 把它从根上消掉了。

要提醒的是 `-fmodules` 在我这台机器上不是默认打开的（`clang -###` 里找不到相关参数），Xcode 工程靠 `CLANG_ENABLE_MODULES = YES` 打开。

---

## 三、`nm` 到底在说什么

链接器眼里没有函数和类，只有符号。`nm prog.o`：

```text
0000000000000000 t -[Greeter greet]
                 U _NSLog
                 U _NSStringFromClass
0000000000000228 s _OBJC_CLASSLIST_REFERENCES_$_
00000000000001f0 S _OBJC_CLASS_$_Greeter
                 U _OBJC_CLASS_$_NSObject
00000000000001c8 S _OBJC_METACLASS_$_Greeter
                 U _OBJC_METACLASS_$_NSObject
0000000000000160 s __OBJC_$_INSTANCE_METHODS_Greeter
0000000000000180 s __OBJC_CLASS_RO_$_Greeter
                 U ___CFConstantStringClassReference
                 U __objc_empty_cache
00000000000000c4 t _bump
00000000000002a0 S _counter
000000000000004c T _main
                 U _objc_autoreleasePoolPop
                 U _objc_autoreleasePoolPush
                 U _objc_msgSend$greet
                 U _objc_opt_class
                 U _objc_opt_new
```

字母表来自 `man nm`，一共就这么几个：

| 字母 | 含义 |
| --- | --- |
| `U` | undefined，本文件用到但没定义 |
| `T` | `__TEXT,__text` 里的符号，代码 |
| `D` | `__DATA,__data` 里的符号，有初值的数据 |
| `B` | `__DATA,__bss` 里的符号，零初始化数据 |
| `C` | common，暂定符号 |
| `S` | 上面几种以外的 section 里的符号 |
| `A` | 绝对符号 |
| `I` | 间接符号 |
| `-` | 调试符号表项（配 `-a`） |

小写表示 local（non-external），链接器看不见，只在本文件内有效。

**macOS 的 `nm` 里没有 `W`。** 我特意翻了 `man nm` 逐字确认，上面那张表就是全部。中文教程里常见的"`W` 表示弱符号"是 GNU nm 在 ELF 上的行为，Mach-O 这边要看 `nm -m`：

```text
0000000000000034 (__TEXT,__text) weak external _weak_fn
00000000000000e8 (__DATA,__data)  weak external _weak_var
0000000000000038 (__TEXT,__text) private external _hid
```

同样这三个符号，不加 `-m` 的话打出来是 `T`、`D`、`T`，弱和 hidden 的信息全丢了。我自己现在的习惯是查符号一律带 `-m`，`nm` 裸跑只用来 grep 名字。

### 几个具体的坑

`int g_zero;`（全局、无初值）在 C 里是 `C`，common 符号。common 的特殊之处是允许多个 `.o` 各定义一份，链接器合并成一个，不报重复定义。`static int s_zero;` 则是 `b`，进 `__bss`，作用域锁死在本文件。

`-[Greeter greet]` 是 `t`，小写。ObjC 的方法根本不是链接期符号，它只是 `__objc_const` 里那张方法列表中的一个 IMP 指针。这解释了两件事：方法名冲突永远不会报重复符号；调一个没实现的方法，编译和链接全都过得去，只在运行时炸。

```objc
@interface Half : NSObject
- (void)declared;
@end
@implementation Half
@end          // 没写 declared
```

编译只给一条 warning（`method definition for 'declared' not found`），链接一声不吭，跑起来：

```text
*** Terminating app due to uncaught exception 'NSInvalidArgumentException',
    reason: '-[Half declared]: unrecognized selector sent to instance 0x102861eb0'
```

类倒是真符号，叫 `_OBJC_CLASS_$_<类名>`，元类叫 `_OBJC_METACLASS_$_<类名>`。所以"只声明了 `@interface` 没写 `@implementation`"是链接错误：

```text
Undefined symbols for architecture arm64:
  "_OBJC_CLASS_$_Ghost", referenced from:
       in e2.o
```

### `_objc_msgSend$greet` 是什么

上面那份符号表里没有 `_objc_msgSend`，只有 `_objc_msgSend$greet`，而且整个 `prog.o` 里连 `__objc_selrefs` 这个 section 都没有。第一次看到我以为环境坏了。

这是 arm64 上的 msgSend selector stub。编译器不再自己生成"取 selref 到寄存器、call objc_msgSend"那几条指令，而是对每个用到的选择器发一个未定义符号 `_objc_msgSend$<sel>`，由链接器去合成桩函数。加 `-fno-objc-msgsend-selector-stubs` 就退回老写法，`.o` 里立刻出现 `U _objc_msgSend` 和 `__objc_selrefs` section。

链接之后这个符号的身份变了：

```text
0000000100000aa0 s _objc_msgSend$greet     ← 落在 __TEXT,__objc_stubs
                 U _objc_msgSend           ← 桩里再去调真正的 objc_msgSend
```

链接器同时把 `__objc_selrefs` 建了出来。所以 `.o` 里没有 selrefs 不代表这个机制没了，只是搬到了链接期。显式写 `@selector(xxx)` 的地方仍然会在 `.o` 里生成 selrefs，因为那是取 SEL 值而不是发消息。

---

## 四、重复符号、weak 与可见性

两个 `.o` 各定义一个 `hello`：

```text
duplicate symbol '_hello' in:
    …/a.o
    …/b.o
ld: 1 duplicate symbols
```

给其中一个加 `__attribute__((weak))` 之后，行为分两种情况，我把四种组合都跑了：

```text
strong(a.o) + weak(bw.o)     → A
weak(bw.o) + strong(a.o)     → A        （调换顺序，结果不变）
weak(aw.o) + weak(bw.o)      → A(weak)
weak(bw.o) + weak(aw.o)      → B(weak)  （调换顺序，结果变了）
```

规则很干净：强符号永远赢，和顺序无关；两个都弱的时候，命令行上先出现的赢。C++ 的 inline 函数和模板实例化天然就是 weak，`nm -m` 里能看到 `weak external`，多个 `.o` 各自生成一份也不冲突，靠的就是这条。

可见性是另一条正交的线。`-fvisibility=hidden` 把 external 降级成 private external：

```text
默认：           (__TEXT,__text) external _api_a
-fvisibility=hidden：(__TEXT,__text) private external _api_a
```

编成 dylib 之后，导出表（`nm -gU`）里默认版有 `_api_a` `_api_b` `_api_c` 三个，hidden 版只剩显式标了 `visibility("default")` 的 `_api_c`。文件也小了一点（33416 → 33400），因为没被引用又不导出的符号可以被剥掉。

这个开关对 ObjC 类符号同样生效，`_OBJC_CLASS_$_Exported` 会变成 private external。但下面这件事我原本猜错了：

```objc
// 编 dylib 时加了 -fvisibility=hidden
@interface SecretCls : NSObject
- (NSString *)say;
@end
```

```text
$ nm -gU libhid.dylib
00000000000008ac T _kickoff        ← 只有它

$ ./probe
lib loaded
NSClassFromString = SecretCls
调用结果 = secret says hi
```

链接期完全不可见（直接引用 `SecretCls` 会得到 `Undefined symbols: _OBJC_CLASS_$_SecretCls`），运行期照样能被 `NSClassFromString` 找出来、照样能发消息。因为类是否注册进运行时，取决于它在不在 `__objc_classlist` 里，和符号导出表是两套东西。**链接期可见性管不住 ObjC 运行时。** 想真正藏一个类，`-fvisibility=hidden` 只是把门锁上了，窗户还开着。

---

## 五、链接顺序，以及一个我证伪的说法

先说结论：**"静态库必须放在引用它的 `.o` 后面，否则 undefined symbol"这条规则在 Apple 的链接器上不成立。** 我一开始也是照着这条去设计实验的，结果怎么调顺序都不报错，差点以为是自己的库打错了。

`man ld` 里写得清清楚楚：

> A static library (aka static archive) is a collection of .o files with a table of contents that lists the global symbols in the .o files. ld will only pull .o files out of a static library if needed to resolve some symbol reference. Unlike traditional linkers, ld will continually search a static library while linking. There is no need to specify a static library multiple times on the command line.

实测：

```shell
clang libA2.a m2.o -o r1        # 库放最前面 → 正常链接，正常运行
clang mp.o libQ.a libP.a -o t2  # P 依赖 Q，却把 Q 放前面 → 正常
clang -Wl,-ld_classic mp.o libQ.a libP.a -o t3   # 换旧链接器 → 还是正常
```

那条规则来自 GNU ld 的单遍扫描，抄到 iOS 语境里就失效了。GNU ld 上要靠 `-Wl,--start-group` 解决的循环依赖，在这边不存在。

链接顺序真正决定的是仲裁：两个归档里有同名符号时，谁被选中。

```shell
clang m2.o libA.a libB.a -o p1   # → from libA
clang m2.o libB.a libA.a -o p2   # → from libB
```

而且粒度是归档成员（`.o`），不是整个归档。我第一版实验就栽在这里：把 `shared` 和 `onlyA` 塞在同一个 `.c` 里，结果 main 一引用 `onlyA` 就把整个成员拖了进来，两边都被拉，直接变成 duplicate symbol 错误。把 `shared` 拆成独立的 `.o` 再打包，才复现出"静默选一个"的经典行为。

这个静默在 ObjC 上更危险。两个静态库都有个叫 `Shared` 的类，代码只用到这个类：

```text
clang u2.o libA.a libB.a …   → who = A
clang u2.o libB.a libA.a …   → who = B
```

不报错、不警告，换个 Pod 顺序行为就变了。改成两个 dylib 的话，运行时至少还会喊一声：

```text
objc[78361]: Class Shared is implemented in both …/libDB.dylib and …/libDA.dylib.
This may cause spurious casting failures and mysterious crashes.
One of the duplicates must be removed or renamed.
```

我自己的做法是：怀疑符号打架的时候直接加 `-ObjC` 重链一次。它会把所有含 ObjC 代码的成员全部拉进来，静默冲突当场变成 duplicate symbol 错误：

```text
duplicate symbol '_OBJC_CLASS_$_Shared' in:
    …/libA.a[2](libA.o)
    …/libB.a[2](libB.o)
```

### `-ObjC` / `-all_load` / `-force_load`

`man ld` 的原文分得很清楚：

> -all_load Loads all members of static archive libraries.
>
> -ObjC Loads all members of static archive libraries that implement an Objective-C class, category or a Swift struct, class or an extension.
>
> -force_load path_to_archive — Loads all members of the specified static archive library.

三者的区别就是作用范围：`-force_load` 针对指定的一个归档，`-ObjC` 针对所有归档但只挑含 ObjC 内容的成员，`-all_load` 全都要。

用一个含纯 C 成员的库量一下：

| 选项 | `unused_c_helper` 进二进制了吗 | 产物大小 |
| --- | --- | --- |
| 不加 | 否 | 50864 |
| `-ObjC` | 否 | 50912 |
| `-all_load` | 是 | 50976 |

`-ObjC` 存在的理由是 category。category 编译出来不会产生任何 external 符号，如果一个 `.o` 里只有 category、没有别的东西被引用，这个归档成员永远不会被拉进来：

```text
$ ./app1
name = Base
*** Terminating app … reason: '-[Base extra]: unrecognized selector sent to instance 0x10257d7f0'
```

加 `-ObjC`（或 `-all_load` 或 `-force_load`）之后就正常了。这是所有"静态库里的 category 方法找不到"问题的标准解释，也是 CocoaPods 默认往 `OTHER_LDFLAGS` 里塞 `-ObjC` 的原因。

代价是包体积和重复符号风险。我的阈值：能定位到具体是哪个库出问题就用 `-force_load` 点名，全局 `-ObjC` 只在排查阶段临时开。

---

## 六、ObjC 元数据是怎么进二进制的

写一个带协议、ivar、属性、`+load`、category、`super` 调用和显式 `@selector` 的类，编出来的 `.o` 里 objc 相关 section 是这些：

```text
__DATA   __objc_classlist     所有类的指针数组，运行时靠它遍历注册
__DATA   __objc_nlclslist     non-lazy class，实现了 +load 的类单独列一份
__DATA   __objc_catlist       所有 category 的指针数组
__DATA   __objc_protolist     协议
__DATA   __objc_classrefs     "我引用了哪些类"，每个引用点一个槽
__DATA   __objc_superrefs     [super xxx] 用到的类
__DATA   __objc_selrefs       "我用了哪些选择器"，运行时会把它们唯一化
__DATA   __objc_ivar          ivar 偏移量变量
__DATA   __objc_data          class_t / metaclass_t 结构体本体
__DATA   __objc_const         class_ro_t、方法列表、协议列表这些只读部分
__TEXT   __objc_classname     类名字符串
__TEXT   __objc_methname      方法名（选择器）字符串
__TEXT   __objc_methtype      方法类型编码字符串
__DATA   __objc_imageinfo     这个镜像的 ObjC ABI 版本和标志
```

`__objc_classlist` 是运行时的入口。dyld 加载完镜像，libobjc 的 `map_images` 遍历这张表，把每个类 realize 进类表。`__objc_nlclslist` 是需要在加载期立刻处理的那一批（实现了 `+load` 的类），这条线在 [[iOS Method Swizzling：正确姿势、+load 时机与那些坑]] 里展开过。

`__objc_selrefs` 和 `__objc_classrefs` 是两张"引用表"。ObjC 里 `@selector(foo)` 的值必须全进程唯一，编译期各个 `.o` 各写各的字符串，运行时统一把 `__objc_selrefs` 里的每个槽改写成唯一化之后的 SEL 指针。类引用同理，`[Greeter new]` 编译出来是先从 `__objc_classrefs` 的槽里读类指针，再调 `objc_opt_new`。

`otool -o` 能把这些结构解出来：

```text
Contents of (__DATA_CONST,__objc_classlist) section
0000000100004058 0x80c0
    isa        0x10000000008098
    superclass 0x801000000000000a
    data       0x8048
        name           0x10000000000ae5
        baseMethods    0x50000000000ac0
            entsize 12 (relative)
            count   1
            name    0x75c8 (0x100008090)
            types   0x21 (0x100000aed)
            imp     0xfffffea8 (0x100000978)
```

`entsize 12 (relative)` 是相对方法列表：每项三个 32 位偏移量而不是三个 64 位指针，省一半空间，而且不需要 rebase。方法项里的 `name` 指向 `0x100008090`，回头查一下这个地址正是 `__objc_selrefs` 的起始地址。方法列表里存的不是选择器字符串，是选择器引用槽的地址，运行时唯一化一次就同时修好了方法表和所有调用点。

### 两个 section 在链接之后消失了

对照 `.o` 和可执行文件的 section 列表，有两个不见了。

`__objc_catlist` 是被合并掉的。当前链接器默认把同一镜像内的 category 折进类本体，加 `-Wl,-no_objc_category_merging` 它就回来。这件事以及它对 `+load` 顺序的影响，那篇 swizzling 笔记里有完整的实验，这里不重复。

`__objc_classrefs` 的消失我一开始以为是自己命令敲错了。默认链接的可执行文件里查不到这个 section，换 `-Wl,-ld_classic` 就出现。但换旧链接器只是绕开了问题，真正的开关是部署目标版本。同一份源码，只改 `-target` 的版本号：

```text
ios14.0   有 __objc_classrefs        macos12.0   有
ios15.0   有                          macos13.0   有
ios16.0   有                          macos14.0   有
ios17.0   有                          macos15.0   没有
ios18.0   没有                        macos26.0   没有
ios26.0   没有
```

门槛是 iOS 18.0 / macOS 15.0，两条线正好是同一年的系统。跨过门槛之后类引用折进 `__got`，同时 `__objc_superrefs` 从可写的 `__DATA` 挪进了只读的 `__DATA_CONST`。两处变化指向同一件事：新运行时不再需要在加载期回写这两张表。

这直接废掉一个流传很广的配方：网上讲"未使用类检测 = `__objc_classlist` 里有、`__objc_classrefs` 里没有"，部署目标只要到 iOS 18，可执行文件里根本没有 `__objc_classrefs` 这个 section 给你减。想继续用这套，得把部署目标临时降到 17 链一版，或者去解析 `__got` 和 chained fixups。验一下自己的工程只要一条命令：`otool -l <binary> | grep objc_classrefs`。

---

## 七、回到开头：dead strip 为什么剥不掉 ObjC 类

现在可以解释第一段那 103 KB 了。

`clang -S` 生成的汇编里，那一行长这样：

```asm
	.section	__DATA,__objc_classlist,regular,no_dead_strip
	.p2align	3, 0x0
l_OBJC_LABEL_CLASS_$:
	.quad	_OBJC_CLASS_$_Greeter
```

`no_dead_strip` 是段属性。它对应 `<mach-o/loader.h>` 里的：

```c
#define S_ATTR_NO_DEAD_STRIP	 0x10000000	/* no dead stripping */
```

我 `otool -l` 查了链出来的二进制，`__objc_classlist` 的 `flags` 正是 `0x10000000`。

链接器的死代码剥离是可达性分析：从入口和一批"根"出发，能到的留下，到不了的删掉。带 `no_dead_strip` 的 section 无条件当根。而 `__objc_classlist` 里躺着每一个类的指针，于是每个类都可达，类可达则它的 `class_ro_t`、方法列表、方法实现、类名字符串全都可达。整条链一个都删不掉。

LLVM IR 里还有第二道保险：

```llvm
@"OBJC_LABEL_CLASS_$" = private global [1 x ptr] [ptr @"OBJC_CLASS_$_Greeter"],
    section "__DATA,__objc_classlist,regular,no_dead_strip", align 8
@llvm.compiler.used = appending global [6 x ptr] [ptr @OBJC_CLASS_NAME_, …,
    ptr @"OBJC_LABEL_CLASS_$"], section "llvm.metadata"
```

编译器自己先用 `llvm.compiler.used` 保住它，再给 section 打上 `no_dead_strip` 让链接器也别动。

这是必须的。ObjC 的类可以完全靠 `NSClassFromString` 拿到，没有任何静态引用。链接器无法证明一个类没人用，所以只能全留。第四节那个 hidden 类的实验就是活证据：符号表里查不到，`NSClassFromString` 照样找得到。

量化一下代价。同一份 `main.o`，分别链上 200 个死类和 200 个死函数：

```text
空文件          bin = 51,144
200 个死 C 函数  bin = 51,144      ← 和空文件一字节不差，全被剥掉
200 个死 ObjC 类 bin = 154,560     ← 多 103,416 字节，约 517 字节/类
```

```text
死类符号数: 200
死方法数:   200
死函数数:   0
```

`strip -x` 也救不了。它把符号表从 1224 项砍到 415 项、文件从 154,560 砍到 120,504，但 `__objc_classlist` 的 size 一字不动（`0x650` = 202 个条目），200 个类名字符串一个不少地留在 `__TEXT` 里。因为这些是运行时要读的数据，不是给调试器看的符号表。

同理，一个活着的类里没人调的方法也删不掉，它挂在类的方法列表上，跟着类一起可达。

`-Wl,-map` 生成的 link map 在这里会骗人，我差点被它带偏。`# Dead Stripped Symbols` 一节里赫然列着 `__OBJC_$_INSTANCE_METHODS_NeverUsed`：

```text
# Dead Stripped Symbols:
<<dead>>	0x0000001C	[  1] _never_called_c
<<dead>>	0x00000020	[  1] __OBJC_$_INSTANCE_METHODS_NeverUsed
```

但同一份 map 的存活区里它也在，地址 `0x100000A20`，大小从 `0x20` 变成了 `0x14`，来源文件编号从 `[1]`（我的 `.o`）变成了 `[0]`（链接器自己）。被剥掉的是 `.o` 里那份 32 字节的绝对地址方法列表，链接器另建了一份 20 字节的相对方法列表顶上。类和方法一个都没少。看 link map 判断有没有被剥掉，得同时查存活区，只看 dead 那一节会得出相反的结论。

最后是我原以为会不一样的一点：`-dead_strip` 在我这台机器上默认没开。`clang` 驱动器不会自动传，Xcode 26.6 的 `Ld.xcspec` 里 `DEAD_CODE_STRIPPING` 的 `DefaultValue = NO`，我扫了 Templates 目录下所有 plist，没有一个工程模板把它打开。

所以清理无用类只能靠工具在编译产物上做静态分析（扫 `__objc_classlist` 的类名，再全局搜引用），链接器帮不上忙。这也是为什么各家包体积治理方案里，"无用类扫描"永远是单独的一个环节。

---

## 八、四类错误，四个阶段

排查构建问题时，先把错误归到阶段上，能省一大半时间。

第一类在编译期。语法、类型、未声明的标识符。`clang -c` 就报，报错点带行号列号。

```text
e1.m:2:17: error: call to undeclared function 'undeclared_thing'
```

第二类在链接期。符号找不到或者重复。`-c` 一路绿灯，链接才炸，报错里给的是符号名和引用它的 `.o`。

```text
Undefined symbols for architecture arm64:
  "_OBJC_CLASS_$_Ghost", referenced from:
       in e2.o
  "_declared_but_never_defined", referenced from:
      _main in e2.o
```

第三类在加载期。链接时 dylib 里有这个符号，运行时那个 dylib 换了版本、符号没了。dyld 在进程起来之前就终止：

```text
dyld[66588]: Symbol not found: _extra
  Referenced from: … /user
  Expected in:     … /libdl.dylib
```

我这个实验是把 dylib 换成缺符号的版本重跑，`main` 里第一行 `printf` 都没执行到。这个二进制的 load command 里有 `LC_DYLD_CHAINED_FIXUPS`、没有 `__la_symbol_ptr`，绑定在启动时一次做完，没有"用到才绑"的懒绑定兜底。

第四类在运行期的消息派发上。类在、方法不在。这一类编译链接加载全都过，只在真正发消息的那一刻抛异常：

```text
'-[Base extra]: unrecognized selector sent to instance 0x10257d7f0'
```

第三节说过方法不是链接期符号，这四类错误的分界线基本就是从那里来的：C 层面的东西链接器管，ObjC 方法层面的东西链接器不管。

---

## 九、几个已经不准的说法

- "`nm` 里 `W` 表示弱符号。" macOS 的 `nm` 没有这个字母，`man nm` 里的完整字母表是 `U A T D B C S I -` 加小写变体和一个 `u`。Mach-O 上要 `nm -m` 才看得到 `weak external`。这条是从 GNU nm / ELF 抄过来的。
- "静态库必须放在引用它的目标文件后面。" GNU ld 的规则。`man ld` 明确写了 Apple 的链接器会持续搜索静态库，也不需要重复指定同一个库。实测把库放最前面、把被依赖的库放前面，都能正常链接，旧链接器 `-ld_classic` 也一样。链接顺序真正影响的是重复符号选谁。
- "四个阶段是四个程序依次执行。" `clang -###` 显示实际只 fork 了 `clang` 和 `ld` 两个进程，预处理、前端、后端、汇编器全在一个进程内完成，中间产物不落盘。
- "开了 dead code stripping，没用的类就会被删掉。" 编译器给 `__objc_classlist` 打了 `no_dead_strip`（`S_ATTR_NO_DEAD_STRIP = 0x10000000`），这个 section 是死代码分析的根，所有类因此都可达。200 个死类实测多占 103 KB，`strip -x` 也拿不掉。
- "未使用类 = `__objc_classlist` 减 `__objc_classrefs`。" 部署目标到 iOS 18.0 / macOS 15.0 之后，类引用折进了 `__got`，可执行文件里根本没有 `__objc_classrefs` 这个 section。实测门槛卡得很干净：ios17.0 有，ios18.0 没有；macos14.0 有，macos15.0 没有。
- "`-all_load` 和 `-ObjC` 差不多。" `-ObjC` 只拉含 ObjC 类/category/Swift 类型的归档成员，纯 C 的成员不拉；`-all_load` 全拉。实测同一个库，纯 C 函数只在 `-all_load` 下进了二进制。
- "`-fvisibility=hidden` 能把类藏起来。" 藏的是链接期符号。类照样在 `__objc_classlist` 里注册，`NSClassFromString` 照样拿得到，方法照样发得出去。
- "`.o` 里没有 `__objc_selrefs` 说明没用消息发送。" arm64 上编译器默认走 msgSend selector stub，发一个 `_objc_msgSend$<sel>` 未定义符号让链接器合成桩，selrefs 到链接期才建。`-fno-objc-msgsend-selector-stubs` 可以看到老形态。

---

## 总结

四个阶段的产物是可以逐个拿到手里看的：`-E` 出 9 万行文本，`-S` 出 189 行汇编，`-c` 出一个符号还悬着的 Mach-O 目标文件，链接把它们缝成映像。膨胀和收缩都发生在前两步，一行 `#import <Foundation/Foundation.h>` 就是 90,360 行、647 个头文件、每个文件 0.2 秒。PCH 和 modules 解决的是同一笔重复开销，modules 额外把宏污染问题从根上消掉了。

符号这条线上有三个容易踩的地方：Mach-O 的 `nm` 没有 `W`，弱符号要 `-m` 才看得见；ObjC 方法不是链接期符号，所以方法冲突永远不报错、缺实现只在运行时炸；链接顺序不影响能不能找到符号，只影响重复符号选谁。

ObjC 的元数据在编译期就以 section 的形式铺进了静态数据区，运行时只负责登记。`__objc_classlist` 上那个 `no_dead_strip` 标志是本文最实用的一条：它让死代码剥离对 ObjC 类彻底失效，实测 200 个没人引用的类多占 103 KB，而同样数量的 C 函数被剥得一字不剩。

方法论上还是老一套。这篇里推翻的三条流行说法（`nm` 的 `W`、静态库链接顺序、dead strip 对类有效），都不是靠读更多文章发现的，是跑一遍加 `man` 里翻原文找到的。**编译工具链的问题，`man` 加一次实验比十篇博客可靠。**

下一篇 [[iOS Mach-O：结构、符号绑定与 chained fixups]] 接着往下走：这个可执行文件的 header、load commands、段和节到底怎么组织，以及本文第八节那条 `LC_DYLD_CHAINED_FIXUPS` 换掉了什么。更早的碎片笔记在 [[Mach-O]]，装载过程在 [[dyld]]。

## 参考资料

### 官方

- `man ld`：静态库搜索策略、`-all_load` / `-ObjC` / `-force_load` / `-no_objc_category_merging` 的权威措辞，本文几处引用都抄自这里
- `man nm`：符号类型字母表的完整定义，是"`W` 不存在"这条的依据
- `<mach-o/loader.h>`（SDK 内）：`S_ATTR_NO_DEAD_STRIP` 等 section 属性常量
- [Clang Modules 文档](https://clang.llvm.org/docs/Modules.html)：modules 的动机、模块映射与语义
- [WWDC 2018 - Behind the Scenes of the Xcode Build Process](https://developer.apple.com/videos/play/wwdc2018/415/)：编译器、构建系统、链接器三者的分工
- [WWDC 2022 - Link fast: Improve build and launch times](https://developer.apple.com/videos/play/wwdc2022/110362/)：新链接器的行为变化，含 category 合并与静态库处理

### 本地

- [[iOS Method Swizzling：正确姿势、+load 时机与那些坑]]：链接器 category 合并与 `+load` 顺序的完整实验
- [[iOS Mach-O：结构、符号绑定与 chained fixups]]：本文的直接后续，符号绑定从惰性绑定到 chained fixups
- [[Mach-O]]：header / load commands / segment 与 section 的结构
- [[dyld]]：镜像装载、fixups 与初始化器

---

实验环境：macOS 26.5.2（arm64，Apple Silicon），Xcode 26.6，Apple clang 21.0.0，`ld-1267`。

**全程没有开过任何模拟器。** 这篇的实验全部是编译器和链接器层面的，`clang` / `nm` / `otool` / `ar` / `strip` / `ld` 在 macOS 上原生跑就够，不需要 iOS 运行环境。所有产物都是 `clang -framework Foundation` 编出来的 macOS arm64 二进制。

iOS 目标下的复核也做了，同样没有启动模拟器：交叉编译只用到 iPhoneSimulator SDK 的头文件和 tbd，`clang -isysroot $(xcrun --sdk iphonesimulator --show-sdk-path) -target arm64-apple-ios18.0-simulator …` 编完直接 `otool` / `nm` 静态检查即可。结果是第七节那组对照在 iOS 18 目标下完整复现：空文件 51152 字节，加 200 个死 C 函数还是 51152 字节（一字节不差），加 200 个死 ObjC 类变成 154560 字节，200 个类符号一个没少。绝对值和 macOS 差 8 个字节，结论一模一样。category 合并、msgSend selector stub 的行为也一致。唯一有平台差异的是第六节那条 `__objc_classrefs`，差异来自部署目标版本而不是平台，门槛数据已经写进正文。

要看 iOS 上运行期的行为（`+load` 时机、`NSClassFromString` 之类）才需要模拟器，这篇没有这类实验。
