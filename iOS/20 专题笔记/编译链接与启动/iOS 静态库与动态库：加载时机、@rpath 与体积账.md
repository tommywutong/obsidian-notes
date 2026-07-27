---
title: 【iOS】静态库与动态库：加载时机、@rpath 与体积账
published: 2026-07-27
description: 50 个类打进一个 dylib，启动耗时和一个 dylib 都不加完全一样。同样这 50 个类摊到 50 个 dylib 里，启动多花 7.7 毫秒。启动的计价单位是镜像个数，跟你写了多少代码没关系。
tags:
  - iOS
  - Objective-C
  - Link
  - dyld
  - Mach-O
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 26
draft: true
---
# 静态库与动态库：加载时机、@rpath 与体积账

先看一组交错测出来的数字。同一个探针程序，只改它依赖几个动态库：

```text
0 个 dylib                        中位 5.006 ms
50 个类全打进 1 个 dylib          中位 5.147 ms
50 个类摊成 50 个 dylib           中位 12.728 ms
```

中间那行和第一行几乎一样。代码量、类的个数、方法的个数，三者在第二行和第三行完全相同。唯一的区别是这些东西被切成了几个文件。启动多出来的 7.7 毫秒，全部由"多了 49 个镜像"这件事本身产生。

这不是我预期的结果。我原本以为 ObjC 类的注册（`+load`、`map_images`）是主要开销，于是准备了一组纯 C 的对照库。跑完发现纯 C 的 50 个 dylib 也要 10.310 ms，占了 ObjC 版本涨幅的三分之二。**动态库的启动开销按镜像个数计价，和里面装了多少代码基本无关。**

上一篇 [[iOS 从源码到可执行文件：四个阶段与符号]] 把符号解析那一层讲完了：链接器怎么从归档里挑成员、重复符号怎么仲裁、`-ObjC` 和 `-all_load` 各拉什么。那些都发生在链接期，一次性的。这篇往后挪一格，讲运行期。库是被谁、在什么时候、按什么规则找到的，找不到会怎样，以及这套机制在启动时间和包体积上分别收多少钱。

实验全部在 macOS arm64 原生跑，没开过模拟器。凡是 iOS 上可能不同的地方，我在正文里单独标了。

---

## 一、install_name：记在库里，抄进使用者里

先把一个最容易搞混的问题钉死。可执行文件运行起来要去找 `libfoo.dylib`，它去哪儿找？这条路径存在哪里？

答案有两半。库自己有一条 `LC_ID_DYLIB`，记着"我应该被装在哪儿"，这就是 install name。使用者有一条 `LC_LOAD_DYLIB`，记着"我要加载哪个库"。链接的时候，链接器把前者原样抄进后者。抄一次，之后不管了。

Apple 的 Dynamic Library Programming Topics 里是这么写的：

> The static linker records the filenames of each of the dependent libraries at the time the app is linked. This filename is known as the dynamic library's install name.

"at the time the app is linked"是关键。抄完就断了，之后各改各的。

```shell
clang -dynamiclib -install_name '@rpath/libv.dylib' -o libv.dylib lib.c
clang -o app app.c libv.dylib -Wl,-rpath,'@executable_path'
```

```text
$ otool -D libv.dylib
@rpath/libv.dylib
$ otool -L app | grep libv
	@rpath/libv.dylib
```

现在把库的 install name 改成一个不存在的绝对路径：

```text
$ install_name_tool -id '/nonexistent/libv.dylib' libv.dylib
$ otool -D libv.dylib
/nonexistent/libv.dylib
$ otool -L app | grep libv
	@rpath/libv.dylib        ← 没变
$ ./app
val=7                        ← 照样跑
```

已经链好的 `app` 一点感觉都没有。但拿改过的库重新链一个，就不一样了：

```text
$ clang -o app2 app.c libv.dylib
$ otool -L app2 | grep libv
	/nonexistent/libv.dylib
$ ./app2
dyld[21471]: Library not loaded: /nonexistent/libv.dylib
  Reason: tried: '/nonexistent/libv.dylib' (no such file),
          '/System/Volumes/Preboot/Cryptexes/OS/nonexistent/libv.dylib' (no such file),
          '/nonexistent/libv.dylib' (no such file)
```

这条错误信息的形状值得记住。`Library not loaded:` 后面是 `LC_LOAD_DYLIB` 里那条原始字符串，`tried:` 后面是 dyld 实际去磁盘上试过的完整路径。两者不一样的时候，中间那层替换就是下一节的内容。

修的办法有两个方向。改库再重链。或者直接改使用者：

```text
$ install_name_tool -change '/nonexistent/libv.dylib' '@executable_path/libv.dylib' app2
$ ./app2
val=7
```

`install_name_tool -id` 改的是库自己那条，只影响之后新链的人。`-change` 改的是使用者里的那条，立刻生效。CocoaPods、Carthage 那些脚本在集成期做的修补，绝大多数是后者。

## 二、三个 `@` 前缀，差别只在三层依赖里才显出来

`man dyld` 把三个前缀的定义写得很完整，我抄两段原文：

> `@executable_path/` This variable is replaced with the path to the directory containing the main executable for the process.
>
> `@loader_path/` This variable is replaced with the path to the directory containing the mach-o binary which contains the load command using @loader_path. Thus, in every binary, @loader_path resolves to a different path, whereas @executable_path always resolves to the same path.

差别全在这一句里：in every binary, @loader_path resolves to a different path。如果整个工程只有"可执行文件依赖若干 dylib"这一层，两者永远等价，怎么写都对。我第一次设计这个实验就是两层。跑出来两种写法结果一模一样，差点得出"没区别"的结论。

至少得三层。我搭的结构是这样：

```text
bin/main                    ← 可执行文件
mid/libmid.dylib            ← main 依赖它
mid/leafdir/libleaf.dylib   ← libmid 依赖它
```

`libleaf` 的 install name 写成 `@loader_path/leafdir/libleaf.dylib`。加载它的是 `libmid.dylib`，所以 `@loader_path` 展开成 `mid/`，拼出来正好是 `mid/leafdir/libleaf.dylib`：

```text
$ otool -L mid/libmid.dylib
mid/libmid.dylib:
	@rpath/libmid.dylib
	@loader_path/leafdir/libleaf.dylib
	/usr/lib/libSystem.B.dylib
$ otool -l bin/main | grep -A2 LC_RPATH
          cmd LC_RPATH
      cmdsize 40
         path @executable_path/../mid
$ ./bin/main
mid_fn = 43
```

把 `libleaf` 的 install name 换成 `@executable_path/leafdir/libleaf.dylib`，别的一个字不动。链接照过。运行炸：

```text
dyld[16455]: Library not loaded: @executable_path/leafdir/libleaf.dylib
  Referenced from: /private/tmp/libexp/rp/B/mid/libmid.dylib
  Reason: tried: '/private/tmp/libexp/rp/B/bin/leafdir/libleaf.dylib' (no such file)
```

`Referenced from` 是 `mid/`，`tried` 是 `bin/`。`@executable_path` 认的是进程主可执行文件，中间隔着几层依赖都不改变这一点。把 `leafdir` 复制一份到 `bin/` 旁边，立刻就跑通了——这是验证，不是修复方案。

`@rpath` 是第三种东西。它不给路径，给的是一个占位符。`man dyld`：

> Dyld maintains a current stack of paths called the run path list. When @rpath is encountered it is substituted with each path in the run path list until a loadable dylib if found. The run path stack is built from the LC_RPATH load commands in the dependency chain that lead to the current dylib load.

"until a loadable dylib is found"意味着它是按顺序试的。链三条进去，前两条指向不存在的目录：

```text
$ otool -l bin/main2 | grep -A2 LC_RPATH | grep path
         path @executable_path/nowhere1
         path @executable_path/nowhere2
         path @executable_path/../mid
$ ./bin/main2
mid_fn = 43
```

三条全指向不存在的地方时，dyld 会把试过的路径全打出来：

```text
Reason: tried: '/private/tmp/libexp/rp/bin/nowhere1/libmid.dylib' (no such file),
        '/private/tmp/libexp/rp/bin/nowhere2/libmid.dylib' (no such file), …
```

（它把每条路径打了两遍，我没查出确切原因，不影响读法。）

`@loader_path` 那套方案的好处，在把整棵目录树搬家的时候能看出来。`bin/` 和 `mid/` 一起挪到别的地方，一行配置不用改：

```text
$ cp -R bin mid /tmp/libexp/moved/ && /tmp/libexp/moved/bin/main
mid_fn = 43
```

只挪走 `mid/`，立刻就断：

```text
dyld[16674]: Library not loaded: @rpath/libmid.dylib
  Reason: tried: '/private/tmp/libexp/rp/mid/libmid.dylib' (no such file)
```

### Xcode 自己怎么填

这三个前缀不是理论问题，Xcode 的工程模板里就直接写着答案。我把几个模板的 `LD_RUNPATH_SEARCH_PATHS` 默认值抄出来：

| 模板 | 默认值 |
| --- | --- |
| iOS App | `@executable_path/Frameworks` |
| macOS App | `@executable_path/../Frameworks` |
| App Extension | `@executable_path/Frameworks @executable_path/../../Frameworks` |
| Framework | `@executable_path/Frameworks @loader_path/Frameworks` |

iOS 的可执行文件就在 `App.app/` 根下，macOS 的在 `App.app/Contents/MacOS/`，所以后者多一个 `..`。Extension 的二进制在 `App.app/PlugIns/Ext.appex/` 里，`../../Frameworks` 正好指回宿主 App 的框架目录。extension 能复用宿主已经加载的框架，靠的就是这一条。

只有 Framework 模板同时写了两个前缀。一个 framework 可能被 App 加载，也可能被另一个 framework 加载。它无法预知自己在依赖链的哪一层，所以两条都挂上。这正是第二节那个三层实验的现实版本。

## 三、什么时候加载

给一个 dylib 同时加上 `__attribute__((constructor))` 和一个实现了 `+load` 的 ObjC 类，主镜像里也各放一份，另外准备一个只靠 `dlopen` 加载的库。跑出来的顺序是：

```text
libA +load
libA constructor
主镜像 +load
主镜像 constructor
---- main() 第一行 ----
libA useA() called
---- 准备 dlopen ----
libLate +load
libLate constructor
---- dlopen 返回 0x77dd2f10 ----
```

三件事一目了然。依赖库整体排在主镜像前面，这是自底向上的初始化顺序。同一个镜像内部，`+load` 排在 C 的 constructor 前面。`dlopen` 的库在调用那一刻才走这套流程，两条打印夹在 `dlopen` 的前后之间。这条初始化链的源码路径在 [[dyld]] 里有，`runInitializersBottomUp` 到 `notifyObjCInit` 那一段。

要提醒的是"启动时加载"这个说法本身在现代 dyld 上已经不完全准确。`DYLD_PRINT_LIBRARIES=1` 打出来的日志里，除了镜像列表还有 253 行这样的东西：

```text
dyld[52114]: move loaded to delayed: libcmark-gfm.dylib
dyld[52114]: move loaded to delayed: XPCSupport
dyld[52114]: move loaded to delayed: GenerationalStorage
```

dyld 会把一部分依赖推迟到真正用到时再处理。我这个探针只 `NSLog` 了一行。600 个镜像里有 253 个被推迟了。

### dlopen 比启动期加载更贵

按需加载听起来是个省启动时间的办法，但要算清楚单价。同样 50 个库，在 `main` 里连续 `dlopen`：

```text
dlopen 50 个: 46.624 ms (平均 0.932 ms/个)   ← 首次，冷页缓存
dlopen 50 个: 26.162 ms (平均 0.523 ms/个)
dlopen 50 个: 25.263 ms (平均 0.505 ms/个)
dlopen 50 个: 25.062 ms (平均 0.501 ms/个)
dlopen 50 个: 24.861 ms (平均 0.497 ms/个)
```

热态下 0.50 ms 一个。同样这 50 个库挂在 `LC_LOAD_DYLIB` 上由 dyld 在启动时批量处理，第五节会算出来是 0.15 ms 一个。三倍多。

所以 `dlopen` 省时间的前提是"这个库这次运行根本不会被用到"。如果它迟早要加载，改成 `dlopen` 只是把账挪了个位置，而且总额变大了。我自己的阈值是：只有确定某个功能的触发率明显低于一半，才考虑 `dlopen`。纯粹为了让启动数字好看而做的延迟加载，多半是自欺。

## 四、弱链接：`-weak_library` 只改了符号表

弱链接的用途很清楚。库可能不在，不在的时候符号是 `NULL`，代码自己判断降级。写法上流传最广的就是 `if (&someSymbol)`。

我照着测了一遍。第一次的结果是崩溃：

```text
$ ./app3_weak
进入 main
&optional_fn = 0x0
[1]    139 segmentation fault
```

地址确实是 `0x0`，dyld 那一半干得没问题。崩在下一行的 `if (&optional_fn) { optional_fn(); }` 上——判断没生效。

看汇编就明白了。同一份代码，声明加不加 `__attribute__((weak_import))`，`-O0` 编出来的 `main` 差这么多：

```text
无 weak_import：
	adrp	x8, _optional_fn@GOTPAGE
	bl	_optional_fn              ← 直接调，没有任何检查

有 weak_import：
	adrp	x8, _optional_fn@GOTPAGE
	ldr	x8, [x8, _optional_fn@GOTPAGEOFF]
	cbz	x8, LBB0_2                ← 空则跳过
	bl	_optional_fn
```

没有 `weak_import` 的那版里，`cbz` 根本不存在。C 语言里函数的地址是常量且非空，编译器有权把 `&f != 0` 直接折成真，`-O0` 也照折不误。而且一句警告都不给。同样的写法用在全局变量上会报 `-Wpointer-bool-conversion`，函数则悄无声息。

更容易误导的是符号表。两版的 `nm -m` 输出一模一样：

```text
                 (undefined) weak external _optional_fn (from libw)
```

`-weak_library` 确实生效了，`LC_LOAD_WEAK_DYLIB` 也在，运行时也确实填了 0。**弱链接需要链接期和编译期两边都点头，`-weak_library` 只负责后一半。** 只看 `nm` 和 `otool -l` 验收，会得出一切正常的结论。

加上 `weak_import` 之后是这样：

```text
$ ./app2_weak            # 库在
&optional_fn = 0x1003b04b0
optional_fn 来自 libw
gOptional = 99
main 正常结束

$ mv libw.dylib libw.dylib.bak && ./app2_weak    # 库不在
&optional_fn = 0x0
&gOptional   = 0x0
optional_fn 不在，降级
gOptional 不在
main 正常结束
```

对照一下强链接版本，库一挪走就是 dyld 在 `main` 之前直接终止进程：

```text
dyld[23922]: Library not loaded: @executable_path/libw.dylib
  Reason: tried: '/private/tmp/libexp/weak/libw.dylib' (no such file)
```

日常写 iOS 代码的人很少手写 `weak_import`，因为 SDK 头文件已经替你写好了。验证一下这条路径：

```c
extern void future_fn(void) __attribute__((availability(macos,introduced=99.0)));
```

用 `-target arm64-apple-macos14.0` 编，部署目标低于 `introduced`，编译器自动按弱符号处理：

```text
	ldr	x8, [x8, _future_fn@GOTPAGEOFF]
	cbz	x8, LBB0_2
	bl	_future_fn
$ nm -m avail.o | grep future
                 (undefined) weak external _future_fn
```

`API_AVAILABLE` 那一套宏底下就是这个 attribute。所以判断"这个 API 在旧系统上存在吗"，`if (@available)` 和 `if (&SomeAPI)` 都能工作。前提是声明来自正确标注了 availability 的头文件。自己手写 `extern` 声明去调新 API，这层保护就没了。

再记一个我踩过的坑，和弱链接无关。最早跑这个实验时，第一版程序什么都没打印就 segfault，我以为是 dyld 在 `main` 之前就崩了。实际上输出被重定向到管道，`stdout` 变成全缓冲，崩溃时缓冲区还没刷。加一行 `setvbuf(stdout, NULL, _IONBF, 0)` 之后所有打印都出来了。测崩溃相关的东西，先把缓冲关掉。

## 五、回到开头：启动的计价单位是镜像个数

现在把第一段那组数字摊开。

探针程序在 `main` 第一行读 `KERN_PROC_PID` 拿到进程的 `p_starttime`，和当前时间相减，得到"从 exec 到 main 第一行"的墙钟耗时。七组配置交错跑，每组 30 次，避免系统负载漂移把某一组整体抬高：

| 配置 | min (ms) | 中位 (ms) |
| --- | --- | --- |
| 不依赖任何自制 dylib | 4.212 | 5.006 |
| 1 个 dylib | 4.019 | 5.493 |
| 10 个 dylib | 5.413 | 6.294 |
| 50 个类打进 1 个 dylib | 4.176 | 5.147 |
| 50 个纯 C dylib | 8.925 | 10.310 |
| 50 个含 ObjC 类的 dylib | 10.640 | 12.728 |
| 100 个含 ObjC 类的 dylib | 18.897 | 20.634 |

单价算出来很稳：`(12.728 − 5.006) / 50 = 0.154`，`(20.634 − 5.006) / 100 = 0.156`。两个规模独立算出来差 1%，线性得干净。纯 C 版本是 0.106 ms。ObjC 类的注册只占单价的三分之一，剩下三分之二是"多一个镜像"本身。

第四行是这张表里最重要的一行。它和第一行的差别在噪声范围内，而它和第六行的代码内容完全相同。我加这一组的原因是：前三行里"dylib 个数"和"总代码量"绑着一起涨，两个自变量搅在一起，测出来的斜率不知道该算在谁头上。把 50 个类塞进一个 dylib，代码量维持不变、镜像数降到 1。斜率立刻归零。

再看一个横向的量。用 `DYLD_PRINT_LIBRARIES=1` 数一下，那个"不依赖任何自制 dylib"的探针进程一共加载了 600 个镜像，其中在磁盘上真实存在的文件只有 4 个：

```text
$ ls -l /usr/lib/libobjc.A.dylib
ls: /usr/lib/libobjc.A.dylib: No such file or directory
$ ls -lh /System/Volumes/Preboot/Cryptexes/OS/System/Library/dyld/
-rwxr-xr-x  1 root  admin   838M  aot_shared_cache.0
-rwxr-xr-x  1 root  admin   909M  aot_shared_cache.1
-rwxr-xr-x  1 root  admin   890M  aot_shared_cache.2
```

剩下 596 个来自 dyld 共享缓存，它们在系统构建时就被合并、预链接、预排布好了。596 个缓存镜像连同 exec、内核映射、Foundation 初始化一共 5 毫秒，而 50 个躺在磁盘上的自制 dylib 就要 7.7 毫秒。系统库多不要紧。自己的库多才要紧。

### 这个数字能推广到 iOS 吗

按规范第七条，先把自变量列一遍。这个实验里我扫了两个维度：dylib 个数（0/1/10/50/100）、代码在镜像间的分布（50 个类 1 个库 vs 50 个库）。一个字没动的维度有：

- 平台。macOS arm64 原生，没在 iOS 上跑过。
- 签名。我的 dylib 是链接器打的 adhoc 签名（`flags=0x20002(adhoc,linker-signed)`）。iOS 上每个嵌入 framework 都要真正签名，启动时要验签，这部分开销我完全没测到。
- 库的大小。每个测试 dylib 只有一个类、50 KB 左右。真实 framework 大几个数量级，fixup 数量和页数都不是一个级别。
- 磁盘与页缓存。全部热态。第一次冷跑的时候我拿到过 486 ms，是热态的近百倍。

> 待真机补测：iOS 真机上"每个嵌入 framework 的启动成本"。复现方法是造 N 个空 framework（N 取 0/10/50）嵌进同一个 App，用 `MetricKit` 的 `MXAppLaunchMetric` 或者 Instruments 的 App Launch 模板取 pre-main 时间，冷启动至少跑 10 次取中位。我的预期是单价比 macOS 更高（多了验签），但方向一致；如果真机测出来单价反而更低，说明共享缓存或者预热策略在 iOS 上做了额外的事，那时候这一节要重写。

和下一篇 [[iOS App 启动：三代 dyld、pre-main 与可测量的优化项]] 对了一次账。那边用不同的探针（只量 pre-main，不含 exec 和内核映射）、不同的库内容（纯 C 函数）独立测了一遍，100 个 dylib 从 2.3 ms 涨到 11.5 ms，单价 0.092 ms。我这边纯 C 的 50 个是 0.106 ms。两个数差 15%，方向和量级一致，差值的来向也说得通：我的口径多算了 exec 到 dyld 接手那一段。两篇独立测出同一个结论，我对这条的信心比只有一组数据时高不少。

> 待真机补测：`DYLD_PRINT_STATISTICS=1` 在 macOS 26.5.2 上已经不输出任何内容（我核对过 stderr 是 0 字节，而同一个进程 `DYLD_PRINT_LIBRARIES=1` 正常工作，说明 dyld 确实读了 `DYLD_*` 环境变量）。`DYLD_PRINT_STATISTICS_DETAILS` 同样没有输出。这条只在 macOS 上验过，Xcode 里给 iOS scheme 加这个变量是否还有效，需要真机确认。

## 六、体积账：两次翻转

"静态库让包变大、动态库让包变小"这个说法，我实测下来至少有两个位置会翻转。

一份 200 个 ObjC 类的模块，分别做成静态库和动态库，各链进一个 App：

```text
mod.o          232,920
libmod.a       245,288
libmod.dylib   170,496

app_static     170,792         总计 170,792
app_dynamic     33,880  + dylib 170,496 = 总计 204,376
```

静态方案反而小 33,584 字节。原因不复杂。动态方案里，那 170 KB 的代码要住在一个独立的 Mach-O 里，自带 header、load commands、`__LINKEDIT`、导出符号表、代码签名，同时可执行文件那 33 KB 也一分没省。**一个 dylib 的固定开销大约 30 KB 起，只有一个使用者的时候永远是亏的。**

加第二个 target 就翻过来了：

```text
两个 target 各自链静态库： 170,792 × 2                    = 341,584
两个 target 共用动态库：    33,880 × 2 + 170,496          = 238,256
                                                    差值   103,328
```

`nm` 验证过重复是真的：`t1_static` 和 `t2_static` 里 `Mod199` 的符号各有 6 个，`t1_dyn` 里 0 个，全在 dylib 那一份里。

第二次翻转出现在"库很大但你只用一小部分"的时候。把同样的代码拆成 20 个 `.o` 打包，只调用其中一个成员：

| 方案 | 可执行文件 | 库 | 总计 |
| --- | --- | --- | --- |
| 静态库，只用 1/20 | 53,440 | — | 53,440 |
| 动态库，只用 1/20 | 33,880 | 167,568 | 201,448 |
| 静态库 + `-ObjC` | 167,208 | — | 167,208 |

静态方案只拉进来 10 个类（`nm` 数的），另外 190 个根本没进二进制。动态方案里那 190 个类必须整份带着走，因为 dylib 是一个不可分割的映像。差 3.8 倍。

第三行是同一个静态库加上 `-ObjC`。200 个类全进来，53 KB 变成 167 KB。静态库按需拉取的优势被这一个 flag 吃干净。上一篇讲过 `-ObjC` 为什么必要（category 不产生 external 符号），这里是它在体积账上的标价：如果你的库靠 `-ObjC` 才能正常工作，那么"静态库只带需要的部分"这条好处基本不成立。

镜像切得太碎在体积上也有账。第五节那 50 个各含一个类的 dylib，加起来 2,517,528 字节；同样这 50 个类打成一个 dylib，64,144 字节。39 倍。纯 C 版本单个 16,760 字节，也是几乎全部由固定结构撑起来的。

### 至于"动态库省内存"

这条是真的。但它省的是物理内存，不是包体积。两个进程同时加载同一个 dylib，`vmmap` 里看到的是这样：

```text
__TEXT      104fdc000-104ffc000  [128K  32K  0K  0K] r-x/rwx SM=COW  /tmp/libexp/shm/libbig.dylib
__TEXT      100d7c000-100d9c000  [128K  32K  0K  0K] r-x/rwx SM=COW  /tmp/libexp/shm/libbig.dylib
```

两个进程各自映射到不同的基址（ASLR），但 `SM=COW`、由同一个文件支撑，只读的 `__TEXT` 页在物理内存里是同一份。iOS 上一个 App 通常只有一个进程。这条好处主要体现在系统框架上，你自己的 framework 享受不到。

## 七、`.tbd`：链接期根本不需要真的 dylib

数一下 iPhoneOS 26.5 SDK 的 `usr/lib` 目录：

```text
.tbd    283 个
.dylib    3 个
```

那 3 个还是 `libLLVM.dylib` 一类的工具库，不是系统运行时。`Foundation.framework` 里更干脆，一个二进制都没有：

```text
$ ls -l $SDK/System/Library/Frameworks/Foundation.framework/
-rw-r--r--  6865722  Foundation.tbd
drwxr-xr-x           Headers
drwxr-xr-x           Modules
```

`.tbd` 是文本的，打开就能读：

```yaml
--- !tapi-tbd
tbd-version:     4
targets:         [ arm64e-ios ]
install-name:    '/usr/lib/libz.1.dylib'
current-version: 1.2.12
exports:
  - targets:         [ arm64e-ios ]
    symbols:         [ _adler32, _adler32_combine, _adler32_z, _compress, _compress2,
                       _compressBound, _crc32, _crc32_combine, _crc32_combine_gen, … ]
```

链接器对一个动态库只需要两样东西。一是它导出了哪些符号，用来判断引用能否解析；二是它的 install name，用来写进 `LC_LOAD_DYLIB`。这个 YAML 文件里两样都全了。代码一个字节都不需要。

所以交叉编译 iOS 目标不需要设备，也不需要从设备里掏系统库出来：

```shell
clang -isysroot $(xcrun --sdk iphoneos --show-sdk-path) \
      -target arm64-apple-ios18.0 -fobjc-arc -framework Foundation -o ios_bin ios.m
```

```text
$ otool -L ios_bin
	/System/Library/Frameworks/Foundation.framework/Foundation
	/usr/lib/libobjc.A.dylib
	/usr/lib/libSystem.B.dylib
	/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation
```

写进去的是设备上的绝对路径，而这些路径在我这台机器上一个都不存在。`install-name` 那一行直接从 `.tbd` 里抄过来的。

顺着这条线往回看第五节那 596 个共享缓存镜像，就串上了：设备上这些路径同样不是文件，是共享缓存里的一段。链接期看 `.tbd`，运行期看共享缓存，中间靠 install name 这个字符串对上。整条链上"真正的 dylib 文件"从头到尾没出现过。

## 八、framework 就是个有固定结构的文件夹

Apple 论坛上 Quinn 那篇 An Apple Library Primer 的定义是：

> A framework is a bundle structure with the `.framework` extension that has both compile-time and run-time roles: At compile time, the framework combines the library's headers and its stub library. At run time, the framework combines the library's code, as a Mach-O dynamic library, and its associated resources.

我用 `mkdir` 和 `ln -s` 手工造了一个，没有 Xcode，没有 `Info.plist`：

```shell
mkdir -p Greet.framework/Versions/A/{Headers,Resources}
clang -dynamiclib -install_name '@rpath/Greet.framework/Versions/A/Greet' \
      -o Greet.framework/Versions/A/Greet g.m
cp Greet.h Greet.framework/Versions/A/Headers/
(cd Greet.framework/Versions && ln -s A Current)
(cd Greet.framework && ln -s Versions/Current/Greet Greet \
                    && ln -s Versions/Current/Headers Headers)
```

```text
$ clang -F. -framework Greet -framework Foundation -o use use.m -Wl,-rpath,'@executable_path'
$ ./use
2026-07-27 07:11:05.116 use[45282] framework 就是个文件夹
```

`-F` 加上目录，`-framework` 加上名字，链接器自己去 `Greet.framework/Greet` 找二进制、去 `Greet.framework/Headers/` 找头文件。约定就这两条。剩下的都是它自己的事。

那个 `Versions/A` 加两层符号链接的结构是 macOS 特有的。Apple 的 Bundle Programming Guide 说得很直白：

> The bundle's `Versions` subdirectory contains the individual framework revisions while symbolic links at the top of the bundle directory point to the latest revision.

同一份文档对 iOS 的说法是：

> The bundle structure of iOS applications is geared more toward the needs of a mobile device. It uses a relatively flat structure with few extraneous directories in an effort to save disk space and simplify access to the files.

iOS SDK 里的 framework 确实是扁平的，连 `Versions` 都没有：

```text
$ ls $SDK/System/Library/Frameworks/CoreVideo.framework/
CoreVideo.tbd    Headers/    Modules/
```

我用 `xcodebuild -create-xcframework` 建出来的 iOS framework 也是扁平的，`Kit.framework/{Kit, Headers/, Info.plist}` 三项，没有符号链接。这一点对签名有实际影响。iOS 的 bundle 里出现符号链接会在签名或者上传阶段出问题，是这类"从 macOS 抄结构"事故里最常见的一种。

### 静态 framework

把 `Greet.framework/Greet` 那个位置的 dylib 换成一个 `ar` 归档，其余不变：

```text
$ file Static.framework/Static
Static.framework/Static: current ar archive random library
$ clang -F. -framework Static -framework Foundation -o use_static use2.m && ./use_static
2026-07-27 07:11:06.412 use_static[45322] framework 就是个文件夹
$ otool -L use_static | grep -c Static
0
```

链接过了，跑起来了，`otool -L` 里一条引用都没有。代码全进了可执行文件。"静态 framework"这个词描述的是二进制槽位里放了什么，framework 这个壳只管目录结构和头文件搜索。

### xcframework 的 `Info.plist`

`.xcframework` 是一层更外面的容器。它的 `Info.plist` 只有三个顶层键，其中 `AvailableLibraries` 是一张索引表：

```xml
<key>AvailableLibraries</key>
<array>
    <dict>
        <key>BinaryPath</key>          <string>Kit.framework/Kit</string>
        <key>LibraryIdentifier</key>   <string>ios-arm64</string>
        <key>LibraryPath</key>         <string>Kit.framework</string>
        <key>SupportedArchitectures</key> <array><string>arm64</string></array>
        <key>SupportedPlatform</key>   <string>ios</string>
    </dict>
    <dict>
        <key>BinaryPath</key>          <string>Kit.framework/Kit</string>
        <key>LibraryIdentifier</key>   <string>ios-arm64-simulator</string>
        <key>LibraryPath</key>         <string>Kit.framework</string>
        <key>SupportedArchitectures</key> <array><string>arm64</string></array>
        <key>SupportedPlatform</key>   <string>ios</string>
        <key>SupportedPlatformVariant</key> <string>simulator</string>
    </dict>
</array>
<key>CFBundlePackageType</key>   <string>XFWK</string>
<key>XCFrameworkFormatVersion</key> <string>1.0</string>
```

两条记录的 `SupportedArchitectures` 都是 `arm64`，`SupportedPlatform` 都是 `ios`，区别只在第二条多了 `SupportedPlatformVariant: simulator`。这一个键就是 xcframework 存在的全部理由。Apple Silicon 之后，iOS 真机和 iOS 模拟器都是 arm64，`lipo` 那种按架构切片的胖二进制没法区分它们。xcframework 把切片依据从"架构"换成"平台 + 变体 + 架构"，用目录隔离代替胖二进制。

Quinn 那篇里的说法是：

> An XCFramework is a single document package that includes libraries for any combination of platforms and architectures. An XCFramework holds either a framework, a dynamic library, or a static library. All the elements must be the same type.

所以这三个词各回答一个不同的问题：library 说的是链接形态（静态还是动态），framework 说的是打包形式（目录约定），xcframework 说的是分发容器（怎么装下多个平台）。

## 九、iOS 上的限制

这一节我尽量只抄一手文档。先看 TN2435 的原文：

> Dynamic libraries outside of a framework bundle, which typically have the file extension `.dylib`, are not supported on iOS, watchOS, or tvOS, except for the system Swift libraries provided by Xcode.

Quinn 的 Library Primer 说得更短：

> macOS supports both frameworks and standalone dynamic libraries. Other Apple platforms support frameworks but not standalone dynamic libraries.

两条合起来的结论是：iOS 上可以用自制动态库，但必须包在 framework 里；裸 `.dylib` 不行。

这是打包与分发层面的限制，链接器不参与。我拿 iPhoneOS SDK 交叉编译了一个裸 `.dylib` 并链了上去，`ld` 一声不吭，退出码 0，`otool -L` 里那条 `@rpath/libraw.dylib` 好端端地记着。所以你在本地能编过，不代表这条路走得通，卡点在别处。

嵌入的位置也是定死的。TN2435 说 framework 通过 Embedded Binaries 加入，Destination 设成 Frameworks，"Ensure Code Sign on Copy is checked"。落到磁盘上就是 `App.app/Frameworks/Foo.framework`，配合 iOS App 模板那条默认的 `@executable_path/Frameworks`，第二节那套路径解析就闭合了。

System framework 和 Embedded framework 的差别，从第五节和第七节的数据看最清楚。System framework 在 dyld 共享缓存里，磁盘上没有对应文件，已经预链接好，不进你的 IPA，加载成本接近零。Embedded framework 是你 IPA 里的一个真实目录，要单独签名、单独验签、单独 mmap、单独跑 fixup。每一个都在启动时间和包体积上各交一份钱。"控制嵌入 framework 的数量"这条建议一直有人提，理由就在这里。

至于"为什么早年 iOS 不允许动态库"，我翻了 Dynamic Library Programming Topics、TN2435、Library Primer 和 Bundle Programming Guide，没有找到 Apple 的正面解释。流传的理由（代码签名边界、沙盒隔离、防止运行期注入、共享缓存的封闭性）我认为都合理，但**这是社区共识，我没找到官方出处**。能确认的只有时间线上的事实：iOS 8 引入 Embedded Framework 之前，第三方代码只能以静态库形态进入 App，而这个能力的开放和 App Extension 是同一版一起来的。extension 需要和宿主共享代码，静态库做不到这件事。这个因果关系是我的推断，同样没有一手依据。

## 十、几个不准的说法

- "`if (&someSymbol)` 就能安全判断弱链接符号在不在。" 只在声明带 `weak_import`（或等价的 availability 标注）时成立。光加 `-weak_library`，`nm -m` 会显示 `weak external`、`LC_LOAD_WEAK_DYLIB` 也在、dyld 也确实填了 0，但编译器把判断折成了常量真，`-O0` 下汇编里连 `cbz` 都没有，库缺失时直接 SIGSEGV。
- "静态库会让包变大，动态库能瘦身。" 实测同一份 200 个类的模块，单 target 下静态方案 170,792 字节、动态方案 204,376 字节，动态大 33 KB。要到两个 target 才翻过来。库很大而你只用一小部分时，静态方案 53,440、动态方案 201,448，差 3.8 倍——除非你加了 `-ObjC`，那时候静态方案涨到 167,208。
- "`@executable_path` 和 `@loader_path` 差不多。" 只在"可执行文件直接依赖 dylib"这一层等价。三层依赖里两者指向不同目录，实测同一份代码换个前缀就是 `Library not loaded`，`Referenced from` 和 `tried` 会分别打出两个目录给你看。
- "改了动态库的 install name，用它的可执行文件就跟着变了。" `install_name_tool -id` 只改库自己的 `LC_ID_DYLIB`，已经链好的使用者里那条 `LC_LOAD_DYLIB` 是链接时抄过去的副本，不受影响。要改使用者得用 `-change`。
- "`dlopen` 按需加载能省启动时间。" 只有在这次运行确实不加载它的前提下才省。单价上 `dlopen` 更贵：同样 50 个库，热态 `dlopen` 是 0.497～0.523 ms/个，挂在 `LC_LOAD_DYLIB` 上由 dyld 批量处理是 0.15 ms/个。
- "动态库多了会拖慢启动，因为代码多了。" 代码量和启动时间基本无关。50 个类打进 1 个 dylib，中位 5.147 ms，和不加任何 dylib 的 5.006 ms 在噪声内；同样这 50 个类摊成 50 个 dylib，中位 12.728 ms。单价约 0.15 ms/镜像，其中三分之二是纯"多一个镜像"的成本，ObjC 注册只占三分之一。
- "`DYLD_PRINT_STATISTICS=1` 能看 pre-main 耗时。" macOS 26.5.2 上它一个字节都不输出，`DYLD_PRINT_STATISTICS_DETAILS` 同样。同一个进程 `DYLD_PRINT_LIBRARIES=1` 正常工作，所以不是环境变量被剥掉。iOS 上是否还有效我没验。
- "iOS 完全不能用动态库。" iOS 8 起可以，但必须包成 framework 嵌进 `App.app/Frameworks/`。TN2435 的措辞是 framework bundle 之外的 `.dylib` 不受支持。注意链接器不拦你，本地编得过不代表能上架。

---

## 总结

库这件事上，链接期和运行期是两套账本。上一篇管前者，这篇管后者。

运行期这一半的枢纽是 install name 这个字符串。库把它写在自己的 `LC_ID_DYLIB` 里，链接器在链接那一刻抄进使用者的 `LC_LOAD_DYLIB`，之后两边就断了联系。`@executable_path` 认进程主可执行文件。`@loader_path` 认发出这条 load command 的那个二进制。`@rpath` 什么都不认，只是个占位符，实际路径由依赖链上所有 `LC_RPATH` 拼出来按顺序试。三者的差别只在三层以上的依赖里才显形，而 Xcode 的 Framework 模板默认同时写了前两个，原因就在这里。

两笔账都测出了和通说相反的方向。体积上，只有一个使用者的时候动态库是亏的，实测亏 33 KB，要两个 target 才回本；库大而你只用一小部分的时候静态库赢 3.8 倍，除非 `-ObjC` 把这个优势吃掉。启动上，代价按镜像个数收，不按代码量收。50 个类装一个库和不装库一样快，摊成 50 个库就多 7.7 毫秒。

方法论上最有用的一条来自第五节那个对照组。我最初只扫了"dylib 个数"这一个维度，测出一条漂亮的线性关系，差点就写下"启动开销随代码量线性增长"。加一组"代码量不变、镜像数变"的对照，斜率立刻塌了。跑出线性关系不等于找到了自变量。得先确认自己扫的那个维度上，没有第二个东西跟着一起动。

下一篇 [[iOS App 启动：三代 dyld、pre-main 与可测量的优化项]] 回到启动这条线，把 pre-main 拆成 dyld、runtime、业务三段，第五节那个"镜像数才是计价单位"的结论在那边被独立复现了一次。Mach-O 的结构和 fixup 在 [[iOS Mach-O：结构、符号绑定与 chained fixups]]，装载和初始化的源码路径在 [[dyld]]，符号解析那一层在 [[iOS 从源码到可执行文件：四个阶段与符号]]。

## 参考资料

### 官方

- `man dyld`：`@executable_path` / `@loader_path` / `@rpath` 的完整定义，本文第二节两段引用抄自这里
- [Dynamic Library Programming Topics](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/DynamicLibraries/000-Introduction/Introduction.html)：install name 是链接时记录的，以及 launch time 加载与 runtime 加载的分工
- [TN2435 Embedding Frameworks In An App](https://developer.apple.com/library/archive/technotes/tn2435/_index.html)：iOS/watchOS/tvOS 不支持 framework bundle 之外的 `.dylib`，嵌入位置与 Code Sign on Copy
- [Apple Developer Forums — An Apple Library Primer](https://developer.apple.com/forums/thread/715385)（Quinn）：static library / dynamic library / framework / stub library / XCFramework / mergeable library 的术语定义
- [Bundle Programming Guide — Bundle Structures](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFBundles/BundleTypes/BundleTypes.html)：macOS 的 versioned bundle 与 iOS 的扁平结构
- [Framework Programming Guide — Anatomy of Framework Bundles](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPFrameworks/Concepts/FrameworkAnatomy.html)：`Versions/A` 与 `Current` 两层符号链接的用意
- Xcode 26.6 工程模板（`…/Platforms/*/Developer/Library/Xcode/Templates/`）：`LD_RUNPATH_SEARCH_PATHS` 各 target 类型的默认值

### 本地

- [[iOS 从源码到可执行文件：四个阶段与符号]]：本文的前一篇，链接期的符号解析、归档成员粒度、`-ObjC` / `-all_load` 的语义
- [[iOS Mach-O：结构、符号绑定与 chained fixups]]：`LC_ID_DYLIB` / `LC_LOAD_DYLIB` 所在的 load command 体系，以及 fixup 的两代形态
- [[dyld]]：`start` → `prepare` → `runInitializersBottomUp` → `notifyObjCInit` 的初始化链，第三节那个顺序的源码依据

---

实验环境：macOS 26.5.2（arm64，Apple Silicon），Xcode 26.6，Apple clang 21.0.0，`ld-1267`，iPhoneOS 26.5 SDK。

全程没有开过任何模拟器。涉及 iOS 的部分只用到 SDK 的头文件和 `.tbd`：交叉编译加 `otool` / `nm` 静态检查就够，第七节那个 `arm64-apple-ios18.0` 的二进制和第九节那个裸 `.dylib` 都是编完直接看 load command，没有运行过。

需要标出平台维度的有三处。第五节的启动耗时全部是 macOS 原生数据，iOS 上多了每个嵌入 framework 的验签开销，方向应该一致但单价未知，正文里已经写了复现方法。第五节末尾 `DYLD_PRINT_STATISTICS` 失效那条只在 macOS 26.5.2 上验过。第八节 iOS framework 扁平结构这条，SDK 里的现成 framework 和我用 `xcodebuild -create-xcframework` 生成的都是扁平的，符号链接在 iOS 签名阶段会出问题这一点我引的是 Apple 文档对 bundle 结构的描述，没有实际做过一次签名和上传验证。
