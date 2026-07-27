---
title: 【iOS】App 启动：三代 dyld、pre-main 与可测量的优化项
published: 2026-07-27
description: DYLD_PRINT_STATISTICS 不是坏了，是在 dyld-940（macOS 12）随 dyld4 重写被删干净的，Apple 没公告过。200 个空 +load 交错测 61 轮差值 0 微秒，而同样 100 个函数摊成 100 个 dylib 要多花 9.3 ms。
tags:
  - iOS
  - dyld
  - 启动优化
  - Mach-O
  - 性能
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 27
draft: true
---
# App 启动：三代 dyld、pre-main 与可测量的优化项

我按照那篇被抄了十年的教程，在 scheme 里设上 `DYLD_PRINT_STATISTICS=1`，跑起来，控制台干干净净。

```shell
$ DYLD_PRINT_STATISTICS=1 ./hello
2026-07-27 03:15:43.952 hello[17786:15642356] hi 1
$ DYLD_PRINT_STATISTICS_DETAILS=1 ./hello
2026-07-27 03:15:43.959 hello[17907:15642572] hi 1
```

一行都没有。

这不是本文最重要的发现。但它是个合适的开场。接下来要验的每一条经典建议，出身都和它一样：2016 年前后的一批文章，写的是当时的 dyld2。十年里被反复转述，很少有人回去重跑。

本文的规矩是：每一条建议后面必须跟一个"怎么证明它有用"。测不出来的，就写测不出来。

chained fixups、符号绑定、`__DATA_CONST` 的 `mprotect`，[[iOS Mach-O：结构、符号绑定与 chained fixups]] 讲完了。这些 section 怎么被编译器和链接器造出来，[[iOS 从源码到可执行文件：四个阶段与符号]] 讲完了。本篇不重复。本篇只讲这些字节被装载之后发生了什么。

用户自己那篇 [[dyld]] 抄了 dyld4 的源码主干：`start` → `prepare` → `runAllInitializersForMain` → `notifyObjCInit`。本篇是它的实测对照面。源码说的顺序，我用时间戳排一遍，看是不是那样。

---

## 一、今天还剩下哪些 dyld 环境变量

先把开场那件事查实。

我第一反应是 `strings` 一下 dyld，看变量名还在不在：

```shell
$ strings -a /usr/lib/dyld | grep -ci "statistic"
0
```

零。看起来石锤了。

但我停了一下，先做了个对照。这类"符号不在就是被删了"的推断，我以前吃过亏：

```shell
$ strings -a /usr/lib/dyld | grep -c "DYLD_LIBRARY_PATH"
0
```

`DYLD_LIBRARY_PATH` 也搜不到，可它明明是活的：

```shell
$ cd /tmp && /tmp/dyldlab/libtest/mq
dyld[53433]: Library not loaded: libq.dylib
$ cd /tmp && DYLD_LIBRARY_PATH=/tmp/dyldlab/libtest /tmp/dyldlab/libtest/mq
（正常退出）
```

`DYLD_PRINT_SEARCHING` 还能看到它生效的那一行：

```text
dyld[53452]:   possible path(DYLD_FRAMEWORK/LIBRARY_PATH): "/tmp/dyldlab/libtest/libq.dylib"
dyld[53452]:   found: dylib-from-disk: "/tmp/dyldlab/libtest/libq.dylib"
```

dyld 把 `DYLD_FRAMEWORK_PATH` 和 `DYLD_LIBRARY_PATH` 合在一处处理，日志里也是合着打的。独立字符串自然就没有。

所以 `strings` 搜不到只能当旁证。这是本文第一个"先怀疑仪器"的地方，后面还有两处。

真正能下结论的是三条证据，互相独立。

功能测试：设了没输出，而且 `DYLD_PRINT_ENV=1` 证明 dyld 确实看见了这个变量，只是不理它。

```shell
$ DYLD_PRINT_STATISTICS=1 DYLD_PRINT_ENV=1 ./hello 2>&1 | grep DYLD_
dyld[53762]: DYLD_PRINT_STATISTICS=1
dyld[53762]: DYLD_PRINT_ENV=1
```

官方文档：SDK 里带的 `dyld(1)` man page 是苹果自己维护的。我手上有两代 SDK。它们列出的变量一字不差，`DYLD_PRINT_STATISTICS` 两代都不在。

```shell
$ grep -ci statistic .../MacOSX15.4.sdk/usr/share/man/man1/dyld.1
0
$ grep -ci statistic .../MacOSX26.5.sdk/usr/share/man/man1/dyld.1
0
```

跨平台跨版本：模拟器 runtime 里的 `dyld_sim` 是一份独立的 iOS 版 dyld。它就躺在磁盘上，`strings` 一下不用开模拟器。我机器上装了三个：

| runtime | dyld 版本 | `DYLD_PRINT_STATISTICS` 字符串 | `move loaded to delayed` 字符串 |
| --- | --- | --- | --- |
| iOS 18.3 | dyld-1245.1 | 无 | 有 |
| iOS 18.4 | dyld-1284.10 | 无 | 有 |
| iOS 26.5 | dyld-1378 | 无 | 有 |

本机 `/usr/lib/dyld` 是 dyld-1378（`LC_SOURCE_VERSION` 1378.0，`strings` 里 `@(#)PROGRAM:dyld  PROJECT:dyld-1378`），macOS 26.5.2。

按上一条的教训，"strings 里没有"对 iOS 侧同样只是旁证。我没有真机，iOS 上的功能测试做不了。

至于它是哪一版没的，答案在苹果自己的开源 man page 里。dyld 项目的 `doc/man/man1/dyld.1` 每个 tag 都能直接拉下来数：

```shell
$ for t in dyld-832.7.1 dyld-940 dyld-1378; do
    curl -s https://raw.githubusercontent.com/apple-oss-distributions/dyld/$t/doc/man/man1/dyld.1 \
      | grep -ci DYLD_PRINT_STATISTICS
  done
3
0
0
```

dyld-832.7.1 里还有三处，被删掉的那段原文是：

```text
.B DYLD_PRINT_STATISTICS
Right before the process's main() is called, dyld prints out information about how
dyld spent its time. Useful for analyzing launch performance.
```

到 dyld-940 变成 0，之后一直是 0。下一节会讲 dyld-940 是什么版本：它是第一个 dyld4。**`DYLD_PRINT_STATISTICS` 不是坏了，它是随 dyld4 那次重写一起被删掉的。**苹果没有为此发过任何 release note，也没有标 deprecated。

### 今天还有输出的

我把能想到的全跑了一遍。以下是本机实测还有输出的：

| 变量 | 输出内容 | 一个 ObjC hello world 的输出量 |
| --- | --- | --- |
| `DYLD_PRINT_LIBRARIES` | 每个 image 的 UUID + 路径，以及被移入 delayed 的库 | 853 行 |
| `DYLD_PRINT_INITIALIZERS` | 每个 initializer 的地址和所属 image | 404 行 |
| `DYLD_PRINT_SEGMENTS` | 每个 Segment 的映射区间和权限 | 4381 行 |
| `DYLD_PRINT_BINDINGS` | 每个绑定的符号和目标地址 | 17 行 |
| `DYLD_PRINT_SEARCHING` | 每个库的候选路径和命中方式 | 1229 行 |
| `DYLD_PRINT_APIS` | 每次 dyld API 调用及返回值 | 714 行 |
| `DYLD_PRINT_ENV` | 全部环境变量 | 76 行 |

还有一个 dyld4 新增的，正好是 `DYLD_PRINT_STATISTICS` 的精神继任者：

```shell
$ DYLD_PRINT_LOADERS=1 ./hello
dyld[20560]: using JustInTimeLoader 0x1fa087670 for /private/tmp/dyldlab/hello
dyld[20560]: using PrebuiltLoader 0x2e91d2460 for /System/Library/Frameworks/Foundation.framework/...
dyld[20560]: using PrebuiltLoader 0x2e91ce464 for /usr/lib/libobjc.A.dylib
dyld[20560]: using PrebuiltLoader 0x2e91d0d50 for /System/Library/Frameworks/CoreFoundation.framework/...
```

601 行，每个镜像一行，直接说它走的是哪条路。我自己编的那个二进制是 `JustInTimeLoader`，系统库全是 `PrebuiltLoader`。这两个类名是 dyld4 的核心，下一节会说它们为什么长这样。

`DYLD_IN_CACHE=0` 我也试了。程序正常跑完，没有额外输出。它也没有把系统库改成从磁盘加载，因为磁盘上根本没有那些文件，见下一节。

再更正我自己一个疏漏。我第一次用 `^DYLD_[A-Z_]+$` 去 `strings` 里捞变量名，捞到 33 个。换成宽松匹配变成 47 个。多出来的里面有一个 `DYLD_PRINT_PROTETED_MEMORY_STATUS`，苹果把 PROTECTED 拼错了，字符串里就是这么写的。正则写死行首行尾，漏掉的不止拼错的那一个。

`DYLD_PRINT_BINDINGS` 只有 17 行。这本身就是 chained fixups 的证据：一个 hello world 只需要绑 8 个符号，剩下的交给链走。其中一行要单独抄出来：

```text
dyld[17971]: Setting up kernel page-in linking for /private/tmp/dyldlab/hello
dyld[17971]: __DATA_CONST (rw.) 0x000100BE4000->0x000100BE8000 (fileOffset=0x4000, size=16KB)
```

page-in linking 是真的开着的。

[[iOS Mach-O：结构、符号绑定与 chained fixups]] 结尾留了一句"本机没有直接的观察手段"。那条待核实可以撤掉了。`DYLD_PRINT_BINDINGS` 会明说它在给哪个 Segment 设置内核 page-in linking。

最后，SIP 会屏蔽对系统二进制的注入，这点没变：

```shell
$ csrutil status
System Integrity Protection status: enabled.
$ DYLD_PRINT_LIBRARIES=1 /bin/echo hi
hi
```

对自己编的二进制全部生效，上面那张表就是这么测出来的。

---

## 二、三代 dyld：分界线能查到，而且不是社区说的那个

我原本准备在这一节写"我没找到一手出处"。写之前又找了一轮，找到了。

苹果有个仓库叫 `apple-oss-distributions/distribution-macOS`，按 OS 版本打 tag。每个 tag 下有一份 `release.json`，列出这个系统用的每个开源项目的版本。一条 curl 就能查：

```shell
$ curl -s https://raw.githubusercontent.com/apple-oss-distributions/distribution-macOS/macos-265/release.json \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print([p['tag'] for p in d['projects'] if p['project']=='dyld'])"
['dyld-1378']
```

`dyld-1378` 和本机 `/usr/lib/dyld` 的 `LC_SOURCE_VERSION` 一模一样。这条链是通的，那就可以往回查：

| macOS | dyld tag |
| --- | --- |
| 10.15 | dyld-732.8 |
| 11.0.1 | dyld-832.7.1 |
| 12.0.1 | dyld-940 |
| 13.0 | dyld-1042.1 |
| 14.0 | dyld-1122.1 |
| 15.0 | dyld-1231.3 |
| 26.5 | dyld-1378 |

接下来只需要定位哪个 tag 第一次出现 dyld4。三条互相独立的证据都指向 dyld-940。

源码目录结构换了。dyld-852.2 的根目录还是 dyld3 那套：`dyld3/`、`src/`、`interlinked-dylibs/`。dyld-940 变成了 `dyld/`、`libdyld/`、`common/`、`cache-builder/`。

设计文档第一次出现。`doc/dyld4.md` 在 dyld-852.2 下是 404，在 dyld-940 下是 200。

namespace 第一次出现。`dyld-940/dyld/Loader.h` 第 47 行就是 `namespace dyld4 {`。

**所以 dyld4 落地于 macOS 12，不是流传得更广的 macOS 13。**上一节那个 `DYLD_PRINT_STATISTICS` 从 man page 消失的 tag，正好也是 dyld-940。同一次重写，一进一出。

### iOS 侧没有这张表

`distribution-macOS` 只覆盖 macOS。iOS 没有对应的仓库。iOS 版本到 dyld 版本的映射我只能从模拟器 runtime 里反推，也就是上一节那三个 `dyld_sim`。这个不对称绕不开。

按 macOS 12 和 iOS 15 同期发布推，dyld4 在 iOS 侧大概率是 iOS 15。但这一步是推断，不是证据。

### dyld4 是来推翻 dyld3 的

`doc/dyld4.md` 是苹果自己写的设计文档，它对 dyld3 的评价比任何一篇 WWDC 都直白：

```text
The current dyld3 model is problematic for two reasons: roots and versions.

We designed dyld3 to be optimal for the common customer case where the OS is not
changing and the apps are not changing... But, we discovered over time, that inside
Apple we are rarely in the common case.

In the dyld3 roll out, we were able to side step (postpone) issues in dyld3 by falling
back to dyld2. But that meant we had two implementations of everything in dyld and
libdyld. It also made it confusing for our clients to know which dyld mode a process
was running in.

In dyld4 there is only one code base. But, on an image by image basis, we instantiate
either a PrebuiltLoader or a JustInTimeLoader.
```

dyld3 的核心赌注是"系统和 App 都不怎么变，所以提前算好一份 closure 反复用"。苹果自己承认这个前提在内部几乎从不成立。dyld4 的解法是取消模式，把粒度降到单个镜像：每个 image 各自决定用 `PrebuiltLoader` 还是 `JustInTimeLoader`。

前面 `DYLD_PRINT_LOADERS` 那几行就是这段文字的运行时形态。我自己编的二进制走 `JustInTimeLoader`，shared cache 里的库全走 `PrebuiltLoader`。文档里有一句约束能解释这个分布：

```text
One important constraint is that a PrebuiltLoader can only have other PrebuiltLoader
as dependents.
```

`PrebuiltLoader` 的依赖只能是 `PrebuiltLoader`。你的 App 主二进制是新编出来的，必然是 `JustInTimeLoader`。它依赖的系统库在 cache 里有现成的 `PrebuiltLoader` 图，可以整片接上。

dyld4 把 dyld3 的另一个方向也掉了头。dyld3 是 "moving most code into libdyld.dylib"。dyld4 的第一句设计说明是 "libdyld.dylib is thin and dyld contains all the runtime code"。代码搬出去，又搬了回来。

### 一件小事：Apple 从没公开说过 "dyld4"

这个词只存在于开源代码里。WWDC22 那场讲链接器的 session、SDK 的 `usr/include/`、Xcode 自带的全部 man page、出厂 `/usr/lib/dyld` 的字符串表。一处都没有。

```shell
$ strings -a /usr/lib/dyld | grep -c dyld4
0
```

所以"dyld4"是社区从源码 namespace 里捡来的叫法，不是官方术语。苹果面向开发者的文档里，这套东西至今没有名字。更麻烦的是苹果自己的文档还停在上一代。`Reducing your app's launch time` 至今在用 launch closure 这个 dyld3 时期的说法。


---

## 三、启动各阶段的真实顺序

这一节是我自己最想写的。

网上讲启动顺序的图画法基本一致：dylib 加载 → rebase/bind → ObjC setup → initializer → `main()`。图不算错。但它把 initializer 那一坨揉成了一个方块，而实际用得上的信息全在方块内部。

我的做法是造一个记录器，把所有能挂钩的时机全挂上，统一用 `mach_absolute_time` 打点。

记录器单独做成一个 dylib。这样主二进制和别的 dylib 都能调它，而且它自己最先被初始化，不会污染时间轴：

```c
static const char *gNames[MAXN];
static uint64_t gTs[MAXN];
static int gN = 0;
void rec(const char *name){ if(gN<MAXN){ gNames[gN]=name; gTs[gN]=mach_absolute_time(); gN++; } }
```

只写静态数组。不做 I/O，不碰 ObjC。打印留到 `main()` 里。

依赖链是 `orderprog` → `liblow.dylib` → `librec.dylib`。`liblow` 和主二进制里各放一份完整的挂钩：`__attribute__((constructor))`、类的 `+load`、category 的 `+load`、C++ 全局对象的构造函数。主二进制再多放一个 `constructor(101)` 和一个 `+initialize`。

跑出来是这样。

```text
#    事件                                         相对 t0 (us)
0    lowlib: [LowClass load]                                 0.0
1    lowlib: [LowClass(Cat) load]                            0.1
2    lowlib: __attribute__((constructor))                    1.1
3    lowlib: C++ 全局对象构造                                1.2
4    main 二进制: [MainClass load]                           2.3
5    main 二进制: [MainClass(Cat) load]                      2.3
6    main 二进制: constructor(101) 高优先级                  3.6
7    main 二进制: __attribute__((constructor))               3.7
8    main 二进制: C++ 全局对象构造                           3.7
9    main() 第一行                                         404.2
10   main 二进制: [MainClass initialize]                    404.8
11   首次向 MainClass 发消息之后                            405.1
12   main() 最后一行                                        412.6
```

四条规则直接能读出来。

同一个镜像内，`+load` 跑在 `constructor` 和 C++ 全局构造之前。这对应 [[dyld]] 里抄的 `runInitializersBottomUp`：先 `state.notifyObjCInit(this)` 回调 libobjc 的 `load_images`，再 `this->runInitializers(state)` 跑初始化器表。源码怎么写的，时间戳就怎么排。

依赖库整体跑在使用方之前。`liblow` 的四个事件（0–3）全部早于主二进制的四个（4–8）。这是自底向上的深度优先，不是"谁先链接谁先跑"。

`constructor` 和 C++ 全局对象构造函数住在同一张表里。`constructor(101)` 靠优先级排到了前面。默认优先级的那两个之间，就是声明顺序。它们对 dyld 是同一种东西，没有"C 的先跑还是 C++ 的先跑"这回事。

`+initialize` 和启动流程无关。它在 404.8 μs 才出现，因为我在 `main()` 里第一次给这个类发消息。它由 `objc_msgSend` 触发，不由 dyld 触发。

`+load` 与 `+initialize` 更细的区别、`+load` 内部父类和分类的顺序规则，[[iOS Method Swizzling：正确姿势、+load 时机与那些坑]] 有完整实验。这里只做跨阶段的相对位置。

### 那个 400 微秒的洞

上面 8 号到 9 号之间空了 400 μs。主二进制的最后一个 constructor 已经跑完，`main()` 还没进。这中间是什么？

我一开始以为是 dyld 收尾的开销（`state.decWritable()` 要把一批页 `mprotect` 回只读）。用 `DYLD_PRINT_INITIALIZERS` 配合 stderr 标记一对照，答案完全不是：

```text
    init: /private/tmp/dyldlab/order/liblow.dylib          × 2
>>> MARK main-exe +load
    init: /private/tmp/dyldlab/order/orderprog2
>>> MARK main-exe constructor
    init: /System/Library/Frameworks/Network.framework     × 365
    init: .../DebugSymbols.framework
    init: .../Heimdal.framework
    init: /System/Library/Frameworks/CoreDisplay.framework × 8
    init: .../SkyLight.framework                           × 13
    init: /System/Library/Frameworks/LDAP.framework
>>> MARK main() 第一行
```

主二进制的 constructor 跑完之后，dyld 又跑了 389 个 initializer，光 Network.framework 一个就 365 个。

原因在 upward link。`dyld_info` 一查就清楚。

```shell
$ dyld_info -dependents /System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation
    -linked_dylibs:
        attributes     load path
        re-export      /usr/lib/libobjc.A.dylib
        upward         /System/Library/PrivateFrameworks/CoreServicesInternal.framework/...
        upward         /System/Library/Frameworks/Foundation.framework/Versions/C/Foundation
```

CoreFoundation 向上依赖 Foundation 和 CoreServicesInternal，构成环。`runInitializersBottomUp` 遇到 upward link 不敢递归进去。它把这个节点丢进 `danglingUpwards`，留给第二轮。

第二轮由 `runInitializersBottomUpPlusUpwardLinks` 在第一轮全部结束之后统一跑。而"第一轮全部结束"包含了主二进制自己。

**所以主二进制的 `constructor` 不是 `main()` 之前的最后一段代码。** 环形依赖那一批排在它后面，而且量可以很大。想在"绝对最后"插一段代码的 hook 方案，这里有个坑。

把这个洞放进整个 pre-main 里看比例：

```text
exec -> 本镜像 +load                     3276 us
exec -> constructor(101)                 3292 us
exec -> 最后一个 constructor             3299 us
exec -> main() 第一行                    3701 us
```

一个 ObjC hello world，从内核 exec 到自己第一行代码，3.3 ms；自己那几行代码，23 μs；之后到 `main()`，400 μs。89% 的 pre-main 花在你的代码跑起来之前。

这个数字是怎么测的：`sysctl(KERN_PROC)` 拿本进程的 `p_starttime`，也就是内核在 exec 时记的时刻，和当前 `gettimeofday` 相减。本文所有 pre-main 数字都是这个口径。

```c
static inline double premain_us(void){
    struct kinfo_proc kp; size_t len = sizeof(kp);
    int mib[4] = { CTL_KERN, KERN_PROC, KERN_PROC_PID, (int)getpid() };
    struct timeval now; gettimeofday(&now, NULL);
    if (sysctl(mib, 4, &kp, &len, NULL, 0) != 0) return -1;
    struct timeval st = kp.kp_proc.p_un.__p_starttime;
    return (double)(now.tv_sec - st.tv_sec)*1e6 + (double)(now.tv_usec - st.tv_usec);
}
```

> 这个口径在 iOS 上不能直接用。iOS 的冷启动还包含 `UIApplicationMain`、scene 连接、首帧渲染，`main()` 只是中间的一个点。本文之后所有毫秒数都只用来回答"有没有差异"，不能当 iOS 上的收益。
>
> 待真机补测：用 Instruments 的 App Launch 模板录一次冷启动，它会把 pre-main 拆成 dyld 的各个子阶段，并给出到首帧的完整时间轴。同一台设备、同一个构建配置、重启后第一次启动、至少 10 次取中位数，改动前后各录一遍。

---

## 四、逐条量化经典优化建议

这一节每个实验都用同一套方法：造两个只差一个变量的二进制，交错运行 A、B、A、B，各 21 到 61 轮，取中位数。

交错是为了抵消机器状态漂移。我一开始先跑完 A 再跑完 B，两次测同一个程序能差 40%。

先说一个会毁掉所有测量的东西。

### 首次启动比之后慢 150 倍

```text
新链接 t1  首跑: 522687 us   二跑: 1931 us
cp 到 t2   首跑: 265090 us   二跑: 1966 us
touch t1   首跑: 163310 us   二跑: 2397 us
重签名 t1  首跑: 514792 us   二跑: 1975 us
```

一个刚链接出来的二进制，第一次跑要半秒。第二次 2 毫秒。`touch` 一下（内容不变、inode 不变、只改 mtime）也会重新付一次。这是 macOS 对新文件的一次性检查加上 dyld 首次构建 PrebuiltLoaderSet 的合计开销，具体各占多少我没有拆开。

它的意义在于：任何"改了代码，跑一次，感觉变快了"的结论都是噪声。我在所有 A/B 里先空跑 5 轮丢弃，就是为了绕开它。

### 减少动态库数量：这一条是真的

造 N 个只含一个函数的 dylib，主程序全部链接并调用：

| 自造 dylib 数 | pre-main 中位数 | p10 | p90 |
| --- | --- | --- | --- |
| 0 | 3147 μs | 2504 | 4127 |
| 1 | 3038 μs | 2488 | 4155 |
| 10 | 3744 μs | 3406 | 4456 |
| 50 | 7141 μs | 6642 | 7805 |
| 100 | 11556 μs | 11033 | 12166 |

复跑一遍，1 / 10 / 50 / 100 分别是 2244 / 2932 / 6678 / 11584 μs。绝对值有漂移，斜率稳定在每个 dylib 90 μs 上下。

但这个实验有个自变量没锁死：100 个 dylib 里的代码总量也是 100 份。我把它锁上重测了一次，同样 100 个函数、同样的符号，一次装进 1 个 dylib，一次摊成 100 个：

```text
100 个函数装进 1 个 dylib : 中位数   2255 us
100 个函数装进 100 个 dylib: 中位数  11523 us
1 个 dylib(1 个函数)       : 中位数   2105 us
```

**代码量完全相同，摊成 100 个镜像要多花 9.3 ms，每多一个镜像约 93 μs。** 成本是按镜像数收的，跟里面有多少代码、多少符号无关。

怎么验证它对你的工程有没有用。`otool -L` 数主二进制的 `LC_LOAD_DYLIB` 条数，`DYLD_PRINT_LIBRARIES=1` 数运行时实际加载了多少个非系统镜像。把两个动态库合成一个，前后各录一次启动，差值应该接近"减少的镜像数 × 单镜像成本"。

单镜像成本在 iOS 真机上必然大于 macOS。签名校验、文件系统、内存带宽都不一样，基准得自己测一次。

> 待真机补测：在真机上用 Instruments App Launch 模板录 pre-main，合并前后各 10 次取中位数，反推本机的单镜像成本。macOS 上的 93 μs 只用来说明"成本按镜像数收"，不是 iOS 的数字。

### 减少 `+load`：测不出来

200 个类带空 `+load` vs 200 个类不带，交错 61 轮：

```text
./k200_load            中位数   4318 us
./k200_noload          中位数   4318 us
差值 0 us
```

零。不是"很小"。中位数一个微秒都不差。

先确认两个二进制真的不一样，别是我编错了。

```shell
$ otool -l k200_load | grep -A3 "sectname __objc_nlclslist"
  sectname __objc_nlclslist
   segname __DATA_CONST
      size 0x0000000000000640      ← 1600 字节 = 200 × 8
$ otool -l k200_noload | grep "sectname __objc_nlclslist"
（无输出）
```

`__objc_nlclslist` 确实只在一边有。200 个 non-lazy class 一个不少。

那把量级拉大：2000 个空 `+load` vs 2000 个不带的，差值 509 μs，也就是一次空 `+load` 调用约 0.25 μs。200 个就是 50 μs，本来就在噪声底下。

再验一次仪器有没有失灵。往 `+load` 里塞一件真实工程常干的事：建一个 `NSDateFormatter` 并格式化一次。

```objc
+ (void)load {
    NSDateFormatter *f = [[NSDateFormatter alloc] init];
    f.dateFormat = @"yyyy-MM-dd";
    (void)[f stringFromDate:[NSDate date]];
}
```

```text
./k200_work            中位数  20030 us
./k200_noload          中位数   6301 us
差值 13729 us
```

同样 200 个 `+load`，加了这几行，多花 13.7 ms，每个 68 μs。仪器好得很。

结论要改写："减少 `+load` 的数量"这条建议基本没有意义，`+load` 的调用本身便宜到测不出来。有意义的是"`+load` 里别干活"，而这跟它有几个无关，跟里面写了什么有关。同样的话对 `__attribute__((constructor))` 也成立。

怎么验证：`dyld_info -inits <你的二进制>` 列出本镜像所有初始化器，包括 `+load`：

```shell
$ dyld_info -inits liblow.dylib
    -inits:
        0x000009A8  __ZL7lowCtorv
        0x00000A9C  __GLOBAL__sub_I_lowlib.mm
        0x000009C4  +[LowClass load]
        0x000009F0  +[LowClass(Cat) load]
```

数量不重要，把这张表拿去逐个看里面写了什么才重要。真机上用 Instruments 的 App Launch 模板，`+load` 的耗时会挂在对应 image 的初始化区间里。

### 减少 ObjC 类数量：有影响，但是次线性

| 类数 | 相对 0 类的差值 | 摊到每个类 |
| --- | --- | --- |
| 500 | +447 μs | 0.89 μs |
| 2000 | +1089 μs | 0.54 μs |

2000 个类，每个反而比 500 个便宜。这符合 `_read_images` 的做法。它一次性遍历本镜像的 `__objc_classlist`，批量分配、批量注册，固定开销被摊薄。数量翻四倍，成本只涨 1.4 倍。

一个真实 App 的类数量常在一两万，按这个斜率也就几毫秒，而且你没法真的删掉业务类。这条建议的实际操作空间在"删掉没人用的模块"，而那属于包体积话题，启动只是搭一程便车。

### 减少 C++ 静态初始化对象：几乎测不出来

2000 个全局对象，构造函数定义在别的编译单元里（防止常量折叠）：

```text
./cx_dyn_2000          中位数   2771 us
./cx_dyn_0             中位数   2675 us
差值 96 us
```

2000 个对象一共 96 μs。看 section 就懂了。

```shell
$ otool -l cx_dyn_2000 | grep -A3 "sectname __init_offsets"
  sectname __init_offsets
   segname __TEXT
      size 0x0000000000000004        ← 4 字节，一个条目
```

clang 把 2000 个对象的构造合并成了一个函数。dyld 只调一次。

这里还有个和老文章对不上的地方：section 叫 `__TEXT,__init_offsets`，不叫 `__DATA,__mod_init_func`。4 字节一条的相对偏移，不是 8 字节的绝对指针，因此不需要 rebase，也搬进了 `__TEXT`。老文章讲的 `__mod_init_func` 在我这台机器默认参数下压根不生成。

那如果对象分散在 200 个独立编译单元里呢？这才是真实工程的样子。这一步是我怕自己只扫了一个自变量才补的。

```shell
$ otool -l tu/prog | grep -A3 "sectname __init_offsets"
      size 0x0000000000000320        ← 800 字节 = 200 个条目
```

200 个独立的初始化函数，dyld 老老实实调 200 次。

```text
./tu/prog              中位数   2280 us
./cx_dyn_0             中位数   2327 us
差值 -47 us
```

负数。还是测不出来。dyld 调 200 次函数指针，这件事本身不值钱。

还有一件事值得验：把构造函数改成 `constexpr`，`__init_offsets` 这个 section 直接消失，2000 个对象全部变成 `__data` 里的静态字节。这才是"消除静态初始化"的正确形态——不是把对象删掉，是让它的初值在编译期就能算出来。

怎么验证：`otool -l <binary> | grep -A3 __init_offsets`，条目数 = size / 4。数量本身不用管，和 `+load` 一样，要看的是这些函数里写了什么。

### 四条建议的实测汇总

| 经典建议 | 本机实测 | 我的判断 |
| --- | --- | --- |
| 减少动态库数量 | 每镜像约 93 μs，代码量锁死后依然成立 | 有效，且是四条里唯一能线性外推的 |
| 减少 `+load` 数量 | 200 个空 `+load` 差值 0 μs；一次调用 0.25 μs | 该改写成"`+load` 里别干活" |
| 减少 ObjC 类数量 | 500 类 +447 μs，2000 类 +1089 μs，次线性 | 影响真实但很小，实际操作空间在删模块 |
| 减少 C++ 静态初始化对象 | 2000 个对象 96 μs，200 个独立 TU 测不出 | 该改写成"让初值在编译期算出来" |

四条里三条要改写。它们成文的时候是 dyld2。惰性绑定还在，`__mod_init_func` 还在，shared cache 还没吃下 3645 个库。今天这三条的开销都被摊平到了噪声底下。

只有"镜像数"是 dyld 绕不过去的：每个镜像都要单独查路径、单独映射、单独走一遍 fixup 链。

---

## 五、dyld 自己已经做掉的两件事

### dylib 粒度的延迟初始化

`DYLD_PRINT_LIBRARIES` 的输出末尾有一批我没见过的行：

```text
dyld[18747]: move loaded to delayed: MobileSystemServices
dyld[18747]: move loaded to delayed: libxslt.1.dylib
dyld[18747]: move loaded to delayed: libcurl.4.dylib
dyld[18747]: move loaded to delayed: LDAP
```

dyld 把已经解析出来的库又"移回"延迟状态。数一下。

| 程序 | 解析出的 image | 移入 delayed | 差 |
| --- | --- | --- | --- |
| 纯 C `printf` | 600 | 555 | 45 |
| ObjC + Foundation | 600 | 253 | 347 |

这个数我不敢直接信，换一把尺子对一遍。在 `main()` 里读 `_dyld_image_count()`：

```text
纯 C  : _dyld_image_count() = 45
ObjC  : _dyld_image_count() = 347
```

600 − 555 = 45，600 − 253 = 347。两边分毫不差。

dyld 把依赖图完整解析出 600 个镜像，再判断哪些在本次启动里用不着，把它们摘出去。`main()` 拿到的只有实际需要的那些。

这不是我瞎猜的机制。`loader.h` 里有对应的 load command 扩展。

```c
struct dylib_use_command {
    uint32_t    cmd;
    uint32_t    cmdsize;
    uint32_t    nameoff;
    uint32_t    marker;                  /* == DYLIB_USE_MARKER */
    uint32_t    current_version;
    uint32_t    compat_version;
    uint32_t    flags;                   /* DYLIB_USE_... flags */
};
#define DYLIB_USE_WEAK_LINK	0x01
#define DYLIB_USE_REEXPORT	0x02
#define DYLIB_USE_UPWARD	0x04
#define DYLIB_USE_DELAYED_INIT	0x08
#define DYLIB_USE_MARKER	0x1a741800
```

dyld 二进制里还有两条说明什么情况下延迟不了的错误串：

```text
%s has weak-def (or flat lookup) symbol used by %s, so cannot be delayed
has interposing tuples so cannot be delayed: %s
```

弱定义符号、flat namespace 查找、`__DATA,__interpose` 里的 interposing 表。三样里出现任何一样，这个库就失去被延迟的资格。fishhook 这类框架靠的正是 interposing。

版本出处就在同一个头文件里，`dylib_use_command` 上方的注释：

```c
/*
 * An alternate encoding for: LC_LOAD_DYLIB.
 * The flags field contains independent flags DYLIB_USE_*
 * First supported in macOS 15, iOS 18.
 */
```

这和我 `strings` 出来的结果对得上：三个 iOS runtime 的 `dyld_sim` 里都有 `move loaded to delayed`，最老的那个是 iOS 18.3 的 dyld-1245.1。iOS 18 起支持，18.3 上有，一致。

链接器选项我一开始按 `-delay_init` 去搜，搜不到，差点写成"没有对应选项"。名字不在 `ld -help` 里，在 `man ld` 里：

```text
-delay-lx
        This is the same as the -lx but specifies that the dylib should
        be delay initialized.

-delay_library path_to_library
        This is the same as listing a file name path to a library on the
        link line except that will mark the dylib to be delay initialized.
```

还有一个 `-delay_framework`。苹果的叫法是 "delay initialized"，落到 Mach-O 上就是那个 `DYLIB_USE_DELAYED_INIT` 位。至于苹果有没有在哪场 WWDC 上讲过它，我没找到。

对"减少动态库数量"那条建议的影响：dyld 已经替你把用不上的库摘掉了大半。上一节测出的 93 μs 是那些你真的会用到的库的成本，这部分 dyld 摘不掉。

### page-in linking

前面那行 `Setting up kernel page-in linking` 说明这台机器上的 chained fixups 不是在启动时一次性走完的，而是交给内核在缺页调入时按页应用。这对启动时间和 dirty memory 都有影响。具体多少，我在 macOS 上没有观察手段。

> 待真机补测：在 iOS 16+ 真机上用 `vmmap` 或 Xcode Memory Report 看 `__DATA_CONST` 是否计入 dirty memory。[[iOS Mach-O：结构、符号绑定与 chained fixups]] 里那条待核实说的是同一件事，那边可以把"没有直接观察手段"改成"`DYLD_PRINT_BINDINGS` 能看到 dyld 设置了它"。

---

## 六、二进制重排：页数能压，时间在 macOS 上压不出来

原理只有一句。`__TEXT` 按 16 KB 一页调入。启动期真正会执行的函数如果散在几十个页上，就要触发几十次缺页。把它们排到相邻的几个页里，缺页次数按比例下降。

先确认链接器这一侧听不听话。写一个 order file，每行一个符号名，带下划线前缀。

```text
_fn7
_fn3
_fn9
_fn1
_fn5
_main
```

```shell
$ clang -o app_ordered app.c -Wl,-order_file,order.txt
$ nm -n app_plain   | grep -E "_fn|_main" | awk '{print $3}'
_fn1 _fn2 _fn3 _fn4 _fn5 _fn6 _fn7 _fn8 _fn9 _fn10 _main
$ nm -n app_ordered | grep -E "_fn|_main" | awk '{print $3}'
_fn7 _fn3 _fn9 _fn1 _fn5 _main _fn2 _fn4 _fn6 _fn8 _fn10
```

列进 order file 的按给定顺序排在最前，没列的按原顺序跟在后面。体积一字节不差。

### order file 从哪来

很多文章说用 Clang PGO 生成。我跑了一遍。

```shell
clang -fprofile-instr-generate -fcoverage-mapping -o app_prof app.c
LLVM_PROFILE_FILE=app.profraw ./app_prof
xcrun llvm-profdata merge -output=app.profdata app.profraw
xcrun llvm-profdata show --all-functions app.profdata
```

```text
  fn5:  Function count: 1
  fn1:  Function count: 1
  fn8:  Function count: 0
  fn10: Function count: 0
```

它给的是调用次数，不是调用顺序。次数能告诉你哪些函数是热的，排不出"先执行 fn7 再执行 fn3"。

要拿到顺序得换一条路：clang 的 SanitizerCoverage。编译期在每个函数入口插一次回调，运行期第一次命中就记下来，天然带顺序。

这个我也跑通了。

```c
void __sanitizer_cov_trace_pc_guard_init(uint32_t *start, uint32_t *stop){
    static uint32_t n; if(start==stop||*start) return;
    for(uint32_t *x=start; x<stop; x++) *x = ++n;
}
void __sanitizer_cov_trace_pc_guard(uint32_t *guard){
    if(!*guard) return;          // 每个函数只记第一次
    *guard = 0;
    void *pc = __builtin_return_address(0);
    Dl_info info; if(dladdr(pc,&info) && info.dli_sname)
        fprintf(stderr, "ORDER %03d %s\n", atomic_fetch_add(&gIdx,1), info.dli_sname);
}
```

```shell
$ clang -fsanitize-coverage=func,trace-pc-guard -o app_cov app.c cov.c
$ ./app_cov
ORDER 000 main
ORDER 001 fn7
ORDER 002 fn3
ORDER 003 fn9
ORDER 004 fn1
ORDER 005 fn5
```

顺序和源码里的调用顺序完全一致。把这份输出去重、加下划线前缀，就是可以直接喂给链接器的 order file。

真实工程里有两个细节：要在 App 启动完成的那一刻停止记录；ObjC 方法的符号名是 `+[Class method]` 这种形式，别漏了。

这里要更正一个我自己原本也信了的说法。国内讲二进制重排最有名的是抖音团队 2019 年那篇，它常被说成"用 SanitizerCoverage 抓顺序"。原文写的是"最后选择了静态扫描+运行时 trace 结合的解决方案"。静态扫 linkmap 拿到全部符号，运行时用手写 arm64 汇编 hook `objc_msgSend`，抓 ObjC 方法的调用顺序。SanitizerCoverage 是另一套方案，两套被社区混成了一套。我上面跑的是 SanitizerCoverage 这套。

### 收益测出来了吗

造一个 4000 个函数、`__text` 465 KB 的程序。随机挑 200 个当"启动期热函数"。一份默认布局，一份用 order file。

静态上效果是确定的。

```text
big_plain  : 200 个热函数，落在 29 个 16KB 页上
big_ordered: 200 个热函数，落在  2 个 16KB 页上
```

29 页压到 2 页。运行时的缺页统计也对得上：

```text
big_plain   : 272 / 272 / 270 / 272 / 270  page reclaims，major page faults 均为 1
big_ordered : 244 / 246 / 244 / 244 / 245  page reclaims，major page faults 均为 1
```

少了约 27 次 page reclaim，正好等于省下的 27 个页。

时间呢？

```text
第一轮 差值  +28 us
第二轮 差值  -62 us
第三轮 差值  -32 us
```

正负都有。在 macOS 上我测不出时间差异，而且这个结果是符合预期的：major page fault 两边都是 1，省掉的 27 次全是 soft fault。文件早就在统一缓冲区缓存里，"缺页"只是建立一次页表映射，不读磁盘，一次不到一微秒。

二进制重排真正省的是 major fault，也就是真的要从 NAND 里读一页的那种。macOS 上内存充裕、页缓存热，这个场景做不出来。

一次 major fault 到底值多少毫秒，网上的数字差一个量级。抖音那篇说"优化一个 Page Fault，启动速度提升 0.6~0.8ms"，整体"启动速度提高了约 15%"。Emerge Tools 那篇实测的平均值是 0.06 ms。两边的设备、系统版本、测量方式都不同，谁对谁错我没法裁。这个数字只能自己在目标机型上测一次，不能当常数用。 抖音那篇还提了一句方法上的事。不要用 Time Profiler 量这个：它是采样式的，而发生 page fault 时线程处于 blocked 状态，采不到。该用 System Trace。

> 待真机补测：真机冷启动（重启设备后第一次打开）才有大量 major fault。用 Instruments 的 System Trace 模板，看 Virtual Memory 通道里的 File Backed Page In 事件数量和累计时间，重排前后各录一次。也可以在 App 启动完成时读 `task_info(mach_task_self(), TASK_EVENTS_INFO, ...)` 拿 `faults` / `pageins` 两个计数，比 Instruments 好自动化。29 → 2 这个页数比例是静态可算的，可以先用 `nm -n` 加上面的脚本估一遍上限，再决定值不值得做。

---

## 七、Apple 自己的说法变过几轮

写这篇的时候我把几场启动相关的 WWDC 拿出来对了一遍。它们之间互相推翻的地方比我想象的多，而且推翻从来没有被公告过。

### 「400 毫秒」两代都不是 pre-main 的预算

这个数字被引用得最多，也被引错得最多。

2016 年 session 406 的原话是 `"a good rule of thumb, is 400 milliseconds is a good launch time"`。同一场里紧接着交代了这 400 毫秒包含什么：`"I'm mentioning these last two because those are counted in those 400 milliseconds times that I just mentioned."` 那 last two 指的是 `UIApplicationMain` 加 nib 加载、以及 app delegate 的回调。

2019 年 session 423 换了说法。400 毫秒从"启动时间"变成了 `"rendering our first frame within 400 milliseconds"`。而且明说系统自己要先花掉一部分：`"That leaves you as developers about 300 milliseconds."`

苹果现行的那篇 `Reducing your app's launch time` 里，这个数字根本不再出现。

所以拿 400 毫秒当 pre-main 的预算，对着 2016 那版是错的，对着 2019 那版也是错的。两版说的都是到首帧为止的全部时间。

### 动态库数量：三个时期三种建议

2016 年是 `"a good target's about a half a dozen"`，鼓励把动态库合并掉。

2019 年反过来：`"you should be hard linking all of your dependencies, as it's now even faster than it was before."`

Xcode 15 之后又推 mergeable dynamic libraries，等于回到"开发期分开、发布期合并"。

第四节那组数据站 2016 那一版：每多一个镜像约 93 μs，而且这个成本和镜像里有多少代码无关。2019 那句话的语境是 iOS 13 起系统会缓存运行时依赖，成本被摊薄了，但摊薄不等于消失。

### 测量工具：推荐过，然后连同实现一起删掉

2016 年 session 406 是明着推的：`"please use dyld print statistics to measure your times."`

2019 年 session 423 整场没再提过它一次，改推 Instruments 的 App Launch 模板。

到 macOS 12 的 dyld-940，它从 man page 和实现里一起消失。第一节那三条 curl 就是这件事的时间线的另一头。一个被官方在大会上点名推荐过的调试手段，三年后不再提，五年后删掉，全程没有一条 deprecation 通知。

### 惰性绑定与 rebase 成本

2016 年讲 rebase 时说 `"rebasing can sometimes be expensive because of all the iO"`。惰性绑定是当时的默认机制。2017 年宣布 `"we are going to force all symbol resolution to be up front"`。2022 年 chained fixups 把 rebase 和 bind 合成一遍扫描，再往后 page-in linking 干脆不在启动时做了。[[iOS Mach-O：结构、符号绑定与 chained fixups]] 里那一整节讲的就是这条线的终点。

一条老优化建议今天还成不成立，取决于它写作时对应的是这条线上的哪一段。

### 一句需要还原语境的话

社区常说"iOS 13 起第三方 App 也走 dyld3"。这句话有出处，是 2019 年 session 423：`"in iOS 13, we're bringing these optimizations to your apps. That means we are now caching your runtime dependencies, or warm launches."`

苹果的措辞是"缓存运行时依赖"和"热启动"，没有说第三方 App 完整跑在 dyld3 上。传到今天变成了"第三方 App 从 iOS 13 起走 dyld3"，语气强了不少。

> 这一节引的 transcript，2016 和 2017 两场来自 asciiwwdc 镜像（苹果官方页已经改成 SPA，老 session 的 transcript 不再内嵌在 HTML 里），2019 和 2022 两场是从官方页面的 HTML 抽的。前两场我没能拿到官方页原文，引用时请注意这个来源差异。

---

## 八、今天该怎么测启动

按可信度从高到低排。

Instruments 的 App Launch 模板。真机、冷启动、多次取中位数。它把 pre-main 拆到 dyld 的子阶段，并且能一直画到首帧。这是唯一一个能覆盖 `main()` 之后的手段，而 iOS 上 `main()` 之后往往比 pre-main 更长。

`DYLD_PRINT_INITIALIZERS` 加自己的时间戳。想知道"我这个库的初始化跑了多久"，这个组合最直接，代价是要改代码。

`dyld_info -inits` 做静态盘点。不用跑起来就能列出一个二进制的全部初始化器，包括 `+load`。做启动治理时先拿它列一张表，比到处 grep `+load` 靠谱。

`sysctl(KERN_PROC)` 的 `p_starttime`。本文所有数字的口径，好处是不依赖任何工具，坏处是只能测到 `main()`。

`DYLD_PRINT_STATISTICS`。不要用了，它不打印任何东西。

最后一条不是工具，是纪律。任何启动数字都必须先丢掉首跑。 前面那个 519 ms 对 2 ms 的例子，比这一节所有工具加起来都值钱。

---

## 九、几个已经不准的说法

- "设 `DYLD_PRINT_STATISTICS=1` 看 pre-main 耗时。" 它在 dyld-940（macOS 12）随 dyld4 重写一起被删掉了，man page 里的出现次数从 3 变成 0，实现里也搜不到。苹果没有公告过。替代品是 `DYLD_PRINT_LOADERS` 和 Instruments 的 App Launch 模板。
- "dyld4 是 macOS 13 / iOS 16 引入的。" `distribution-macOS` 的 `release.json` 显示 macOS 12.0.1 用的是 dyld-940，而 dyld-940 是第一个含 `namespace dyld4` 和 `doc/dyld4.md` 的 tag。分界在 macOS 12。
- "pre-main 应该控制在 400 毫秒以内。" 400 毫秒这个数字在 2016 年指的是含 `UIApplicationMain` 和 app delegate 的整个启动，在 2019 年指的是到首帧渲染完成，两版都不是 pre-main 的预算。苹果现行文档里已经不提这个数字。
- "抖音那套二进制重排是用 SanitizerCoverage 抓函数顺序的。" 原文写的是静态扫 linkmap 加运行时 hook `objc_msgSend`。SanitizerCoverage 是另一套方案。
- "延迟初始化的链接器选项是 `-delay_init`。" `man ld` 里叫 `-delay-l<x>` / `-delay_library` / `-delay_framework`。
- "静态初始化器越多启动越慢。" 2000 个 C++ 全局对象一共 96 μs，拆到 200 个独立编译单元也测不出差异。慢的是构造函数里干的事，不是它的数量。
- "`+load` 会拖慢启动。" 一次空 `+load` 调用 0.25 μs，200 个的差值是 0。慢的是 `+load` 里干的事，同样的 200 个 `+load` 里各建一个 `NSDateFormatter`，代价是 13.7 ms。
- "启动初始化器的入口是 `__DATA,__mod_init_func`。" 当前默认参数下生成的是 `__TEXT,__init_offsets`，4 字节相对偏移，不需要 rebase。
- "主二进制的 `+load` / `constructor` 是 `main()` 之前最后执行的代码。" upward link 涉及的那批库排在它之后，本文那个 hello world 里是 389 个 initializer、约 400 μs。
- "动态库会在第一次用到时才加载。" 依赖图在启动时就全部解析完（本文的 hello world 是 600 个镜像），dyld 之后把用不到的移入 delayed。这和"按需加载"是两件事：解析已经付过了，省下的是初始化。
- "系统库在磁盘上，dyld 去磁盘读。" 磁盘上没有独立文件了，3645 个 dylib 全在 6.1 GB 的 shared cache 里，`otool -L` 里那些路径今天只是查询用的名字。
- "二进制重排能省下几十毫秒。" 省的是 major page fault。本文实验里页数从 29 压到 2、soft fault 少 27 次，而 macOS 上的时间差异在噪声里。收益全部取决于目标设备上有多少次真实的磁盘读。

---

## 总结

写完这篇最大的感受：启动优化的经典清单，和它诞生时的那个 dyld 已经不是一回事了。

四条建议里只有"减少动态库数量"经得起锁死自变量之后的重测。它的成本按镜像数收，跟里面有多少代码无关：同样 100 个函数，1 个 dylib 是 2.26 ms，100 个 dylib 是 11.5 ms。另外三条的开销都被摊平到了噪声底下，需要从"减少数量"改写成"减少每一次里干的事"。

启动顺序上有两件事和流传的图不同。一是同一镜像内 `+load` 先于 `constructor`，这和 dyld4 源码里 `notifyObjCInit` 排在 `runInitializers` 前面完全对应。二是主二进制的 initializer 并不是最后一批。upward link 那一环会在它之后再跑一轮，在我的 hello world 里是 389 个。

`DYLD_PRINT_STATISTICS` 那件事最后查清楚了。答案比"它没输出"有意思得多：它是在 dyld-940 随 dyld4 重写被删掉的，那一版就是 macOS 12。同一个 tag，一边删掉旧的调试手段，一边引入 `PrebuiltLoader` / `JustInTimeLoader`。苹果没为此发过任何通知，`dyld4` 这个词至今没在任何面向开发者的材料里出现过。

方法论上这篇有三处我自己差点栽进去。

一处是 `strings` 搜不到 `DYLD_PRINT_STATISTICS`，我差点直接写"它被删了"。幸好临时补了个对照组 `DYLD_LIBRARY_PATH`。它同样搜不到，同样活得好好的。真正的证据是 man page 的跨版本 diff，不是本机的字符串表。

一处是"100 个 dylib 比 1 个慢"。第一版实验里代码量跟着 dylib 数一起变。不锁死它，我就会把"代码变多了"记在"镜像变多了"的账上。

还有一处是这一节本身。我第一版写到这里，写的是"dyld 三代对应哪个系统版本我没找到一手出处"。查不到不等于不存在，只等于我还没找对地方。`distribution-macOS` 这个仓库的存在，一条 curl 就把整张表变成了确定的事。**下判断说"查不到"之前，先想想还有哪类一手材料我根本没去看。**

## 参考资料

### 一手

- [apple-oss-distributions/distribution-macOS](https://github.com/apple-oss-distributions/distribution-macOS)：按 OS 版本打 tag，`release.json` 给出该系统用的 dyld 版本。第二节那张对照表全部出自这里
- [apple-oss-distributions/dyld](https://github.com/apple-oss-distributions/dyld)：`doc/dyld4.md`（设计文档，第二节引文的出处）、`dyld/Loader.h`（`namespace dyld4`）、`doc/man/man1/dyld.1`（跨版本 diff 定位 `DYLD_PRINT_STATISTICS` 的消失）
- SDK 头文件 `mach-o/loader.h`：`dylib_use_command`、`DYLIB_USE_DELAYED_INIT`，以及那句 "First supported in macOS 15, iOS 18."
- `man ld`：`-delay-l<x>` / `-delay_library` / `-delay_framework`
- SDK man page `dyld(1)`（MacOSX15.4.sdk 与 MacOSX26.5.sdk 两份）：苹果自己维护的 `DYLD_*` 变量清单
- `/usr/lib/dyld` 本体（dyld-1378）与三个模拟器 runtime 里的 `dyld_sim`（dyld-1245.1 / 1284.10 / 1378）

### WWDC

第七节的引文出自这四场。2016 和 2017 两场的 transcript 来自 asciiwwdc 镜像。苹果官方页已经改成 SPA，老 session 的文字稿不再内嵌在 HTML 里。2019 和 2022 两场是从官方页面直接抽的。

- WWDC16 session 406 — Optimizing App Startup Time
- WWDC17 session 413 — App Startup Time: Past, Present, and Future
- WWDC19 session 423 — Optimizing App Launch
- WWDC22 session 110362 — Link fast: Improve build and launch times
- Apple — [Reducing your app's launch time](https://developer.apple.com/documentation/xcode/reducing-your-app-s-launch-time)：注意它至今仍在用 dyld3 时期的 launch closure 说法

### 工具

- `dyld_info`：`-inits` 列初始化器、`-dependents` 看 upward link、`-platform` 读 shared cache 里的库
- `DYLD_PRINT_INITIALIZERS` / `DYLD_PRINT_LIBRARIES` / `DYLD_PRINT_SEARCHING` / `DYLD_PRINT_BINDINGS`
- `nm -n`、`otool -l`、`/usr/bin/time -l`、`llvm-profdata`
- `-Wl,-order_file,<file>`、`-fsanitize-coverage=func,trace-pc-guard`

### 本地

- [[dyld]]：dyld4 源码主干，本文第三节的顺序结论是它的实测对照
- [[iOS Mach-O：结构、符号绑定与 chained fixups]]：chained fixups、`__DATA_CONST`、page-in linking
- [[iOS 从源码到可执行文件：四个阶段与符号]]：这些 section 是怎么被编译器和链接器造出来的
- [[iOS Method Swizzling：正确姿势、+load 时机与那些坑]]：`+load` 内部的父类 / 分类 / 链接顺序规则

### 我查了但没查到的

- 苹果在哪里公告过移除 `DYLD_PRINT_STATISTICS`。答案很可能是没有：release note、WWDC、deprecation 标记，一处都没找到。man page 的跨版本 diff 是我能拿到的最强证据。
- iOS 版本到 dyld 版本的官方对照表。`distribution-macOS` 只覆盖 macOS，iOS 侧只能从模拟器 runtime 反推。
- delay initialized 有没有在哪场 WWDC 上讲过。头文件里的 "First supported in macOS 15, iOS 18." 是硬证据，但它的官方介绍我没找到。
- 一次 major page fault 到底值多少毫秒。抖音那篇说 0.6~0.8 ms，Emerge Tools 那篇说 0.06 ms，差一个量级，两边条件都不同，我没有设备去裁。

---

实验环境：macOS 26.5.2（25F84），Apple Silicon（arm64），Xcode 26.6（clang 21.0.0 / ld 1267）。`/usr/lib/dyld` 为 dyld-1378，`vm_page_size = 16384`，SIP 开启。

全部实验在 macOS 上原生编译运行，没有启动任何模拟器。涉及模拟器 runtime 的只有一处：直接 `strings` 磁盘上的 `dyld_sim` 文件，读版本号和字符串表。这是静态文件读取，不需要 boot。

启动耗时的绝对数字在 iOS 上完全不可比。本文所有毫秒/微秒数只用来回答"有没有差异"和"差异的量级关系"，不能当作 iOS 上的优化收益。凡是需要真机才能得到结论的地方，都标了 `> 待真机补测`。
