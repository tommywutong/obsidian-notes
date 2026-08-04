---
title: 【iOS】内存地图：从虚拟地址空间到 VM Region
published: 2026-07-29
description: 从操作系统虚拟内存出发，梳理 iOS App 的地址空间、Mach-O、五大分区、堆栈与 VM Region，用 LLDB 实验验证变量、指针与对象的真实位置。
tags:
  - iOS
  - Memory
  - Mach-O
  - LLDB
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 1
draft: false
---
# iOS 内存地图：从虚拟地址空间到 VM Region

## 前言

在之前的文章中，笔者完成过两篇内存管理的入门文章：

- [【iOS】内存五大分区](https://www.tommywutong.cn/blog/csdn-import/csdn-154609757-ios-/)
- [【iOS】内存管理初级](https://www.tommywutong.cn/blog/csdn-import/csdn-152130856-ios-/)

受限于当时的知识面，这两篇文章对很多概念的理解较浅，部分表述也不够严谨。现在重新梳理 iOS 底层知识，希望从操作系统的虚拟内存出发，把"变量在哪里""对象在哪里""堆和栈是什么""Mach-O 的 Segment 又是什么"这些容易混在一起的问题逐层分开。

这篇文章只回答"一段虚拟地址属于哪里、由谁在什么时候放进去的"。这些地址背后的页面此刻是否真的占用物理内存、系统怎样统计和回收，放在系列下一篇《iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint》里单独展开。

## 主线

全文按照下面的顺序展开：

```text
操作系统为什么需要虚拟内存
        ↓
一个 iOS App 获得怎样的虚拟地址空间
        ↓
用"五大分区"建立一张用途地图
        ↓
Mach-O、堆和线程栈分别怎样形成这些区域
        ↓
用 VM Region、内容来源和访问权限回到真实系统
        ↓
用代码和 LLDB 验证变量、指针与对象的位置
```

这里涉及两个不同的观察角度：

| 观察角度 | 主要回答的问题 | 典型概念 |
| --- | --- | --- |
| 用途 | 这段地址用来做什么 | 五大分区、代码、数据、堆、栈 |
| 来源 | 这段内容从哪里来、由谁在什么时候建立 | Mach-O、VM Region、`malloc`、线程栈 |

同一份数据可以同时被这两个角度描述。例如，一个全局变量在教学模型中属于全局/静态区（用途），在 Mach-O 中来自 `__DATA` 的某个 Section、运行时是一个具体的 VM Region（来源）。它们不是互相竞争的两种答案，而是从不同角度描述同一件事。至于这个 Region 现在是否真的占用物理内存、算 clean 还是 dirty，属于系列下一篇的"页面记账"视角，这里先不展开。

---

## 操作系统中的内存

在没有内存保护和地址转换机制的环境中，程序直接使用物理地址。多个程序如果使用了相同的地址，就可能互相覆盖数据。虽然系统也可以依靠人工规划固定的物理地址来运行多个程序，但这种方式难以做到可靠隔离、灵活装载和高效共享。

现代操作系统因此让进程主要使用虚拟地址，并由硬件和内核共同完成从虚拟地址到物理页的映射。不同进程中的同一个虚拟地址可以映射到不同的物理页；内核也可以通过权限控制，阻止进程访问不属于自己的映射。


- **虚拟内存（Virtual Memory）**：操作系统提供给进程的一层地址空间抽象。一个 iOS App 看到的是自己独立的虚拟地址空间，其中包含多个离散的虚拟内存区域（VM Region）。虚拟地址空间很大，但并不意味着其中每个地址都有效，也不意味着所有已映射页面都同时驻留在物理内存中。


- **物理内存（Physical Memory）**：设备真实存在、可由 CPU 访问的 RAM。进程访问虚拟地址时，CPU 的 MMU 会依据页表把它转换为对应的物理地址。

![image.png](https://cdn.jsdelivr.net/gh/tommywutong/piccbes@master/img/20260727180848288.png)


操作系统教材通常通过"分段"和"分页"两条路线介绍地址空间管理。对现代 arm64 iOS 来说，后文真正需要继续使用的是分页、页表、权限和 VM Region；分段主要帮助理解历史模型和逻辑区域。

### 分段

![分段示意图](https://cdn.jsdelivr.net/gh/tommywutong/piccbes@master/img/20260724154844318.png)

分段（Segmentation）是一种经典的内存管理思想：按照代码、数据、栈等逻辑单元描述地址空间，各段长度可以不同。它有助于理解"程序可以由不同用途、不同权限的区域组成"。

但这里必须区分三个名字相似、含义不同的概念：

- 操作系统教材中的 **Segmentation** 是一种地址转换和内存管理模型。
- Mach-O 中的 `__TEXT`、`__DATA` 等 **Segment** 是可执行文件及其装载映射的组织方式。
- 堆和栈是进程运行时使用的虚拟内存区域，不是"编译器创建的 CPU 分段"。

现代 arm64 iOS 的内存管理重点是分页和 Mach VM。后文讨论 Mach-O 时，还需要进一步观察 `__TEXT`、`__DATA` 等文件布局如何被映射为进程中的 VM Region。

### 分页

![分页示意图](https://cdn.jsdelivr.net/gh/tommywutong/piccbes@master/img/20260724154856696.png)

为了方便映射和管理，虚拟内存和物理内存都被分割成相同大小的单位，物理内存的最小单位被称为帧（Frame），而虚拟内存的最小单位被称为页（Page）。
Apple 的虚拟内存资料说明，较新的 64 位 iOS 设备通常向用户空间暴露 16 KB 页面，Jetsam Event Report 的官方示例也是 16 KB；具体值仍应以设备和运行环境为准，可以通过 `vm_page_size`、`getpagesize()` 或报告中的 `pageSize` 字段确认，而不应在程序逻辑中写死。

- **页表（Page Table）**：记录虚拟页到物理页的映射及读、写、执行等权限。代码访问虚拟地址时，CPU 内部的 **MMU（Memory Management Unit）** 会依据页表完成地址转换。
- **虚拟内存区域（VM Region）**：一段具有相同属性的连续虚拟地址范围。一个进程拥有许多 VM Region，但整个虚拟地址空间并非从头到尾连续有效。

内核建立和管理虚拟内存映射时以页为基本粒度，但这不意味着每次 `malloc` 几个字节都会单独浪费一个 16 KB 页面。用户态内存分配器会先取得并管理较大的虚拟内存区域，再把其中的小块空间分给不同对象；这就是后文讨论“堆”时需要建立的连接。


### 按需分页与 Page Fault（缺页中断）

![](https://cdn.jsdelivr.net/gh/tommywutong/piccbes@master/img/20260725092731666.png)

1. **虚拟映射与延迟分配 (Lazy Allocation)**

   当系统为进程分配虚拟内存，或通过 `mmap` 映射文件时，首先建立的是一段合法的 **VM Region**。**建立一段虚拟地址映射，不等于立即占用同等大小的物理 RAM**；页面可能按需进入物理内存，映射本身及页表等管理结构仍会产生一定开销，因此不能笼统写成“实际消耗为零”。

2. **触发调度 (Page Fault 机制)**

   当 CPU 首次真正读写某个尚未驻留在物理内存的虚拟页时，现有的页表无法完成地址转换，从而触发**缺页中断 (Page Fault)**。这是一个正常的、连接虚拟与物理空间的桥梁，并非程序错误。

3. **内核介入与数据准备**

   内核接管中断后，会根据该虚拟页的属性进行动态修复：

   - **读取数据 (Page In)：** 如果是映射文件，内核会从文件或系统缓存中取得相应内容。
   - **其他策略：** 内核也可能分配一个全零填充页 (Zero-Fill)，或者为只读共享页创建一个私有副本 (Copy-on-Write)。

   页面怎样被统计、压缩或回收，以及 iOS 在内存压力下怎样处理进程，属于系列下一篇的范围。

4. **异常边界**
    内核完成修复后，原指令可以继续执行；如果访问的是未映射地址，或者违反了 Region 的读写执行权限，进程通常会收到 `EXC_BAD_ACCESS`。因此，Page Fault 本身可以是正常的按需分页过程，只有无法修复的地址或权限错误才会成为程序可见的异常。

这里先把 Page Fault 当作一个连接点：它解释了为什么"已经分配或映射虚拟地址"仍不代表"对应物理页已经到位"。系列下一篇讨论 `malloc`、首次写入和 Memory Footprint 时还会再次用到这个结论。Apple 的 [About the Virtual Memory System](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/AboutMemory.html) 对 soft fault、hard fault 和 page-in 有进一步说明。

关于分页、页表和地址转换的更多细节，可以继续阅读：[小林 Coding：为什么要有虚拟内存？](https://www.xiaolincoding.com/os/3_memory/vmem.html)。本文暂时停在能够支撑后续 iOS 地址空间分析的程度。


## 从操作系统过渡到 iOS App

一个运行中的 iOS App，本质上就是一个进程。系统会为它提供独立的虚拟地址空间。前文讨论的分页、页表和 VM Region，在这里不再只是操作系统教材中的抽象概念，而是 App 代码实际运行的环境。

![](https://cdn.jsdelivr.net/gh/tommywutong/piccbes@master/img/20260727180848288.png)


下面先梳理一个 iOS App 从磁盘文件到运行中地址空间的过程。
### 阶段一：App还没有运行

这时候 App 的代码和数据存放在磁盘上的 Mach-O 文件中。
```
磁盘上的 Mach-O
├── __TEXT：代码、部分只读内容
├── __DATA_CONST：部分只读数据
├── __DATA：可写的全局、静态数据
└── 其他装载和链接信息
```

### 阶段二：用户启动App

系统创建一个进程，并提供一套独立的虚拟地址空间，而不是直接交给它一整块连续物理内存。随后，内核和 dyld 根据 Mach-O 的装载信息建立相关映射；依赖的系统 Framework、动态库和 dyld shared cache 也会进入进程地址空间。

所以进程的完整虚拟地址空间，远大于主程序自己的 Mach-O。

### 阶段三：用 VM Region 描述这些映射

虚拟地址空间不是一个从头到尾全部有效的大数组，而是由许多离散的 VM Region 组成。
一个 VM Region 可以先理解为：
> 一段连续的虚拟地址范围，这段范围具有相近的来源、用途和访问权限。

### 阶段四：App开始运行，堆和线程栈出现

Mach-O 主要解释启动时已经存在的代码和全局数据，但程序运行起来以后，还会不断产生新的内存需求。

- 每创建一个线程，系统都会为这个线程准备自己的栈 Region。
- 当程序执行 `malloc` 或创建普通 Objective-C 对象时，运行时需要动态分配内存。这些分配通常来自分配器管理的区域；分配器会向系统取得并管理一个或多个 VM Region，再把其中的小块空间交给对象和缓冲区。

本文到这里仍然只讨论“地址从哪里来”。页面是否驻留、系统怎样统计与回收，以及内存压力下的行为，留到系列下一篇展开。

### 阶段五：CPU真正访问这些地址

无论一个地址属于代码、全局变量、堆还是栈，程序看到的首先都是虚拟地址。CPU 访问时，MMU 和页表参与地址转换；如果映射有效且权限允许，访问最终落到相应的物理页。



## “五大分区”

这就是我们经常提到的五大分区

![image.png](https://cdn.jsdelivr.net/gh/tommywutong/piccbes@master/img/20260727183544537.png)


栈（Stack）
栈是每个线程独有的一段虚拟地址区域，用来支撑函数调用并保存部分临时状态。在未优化的直觉模型中，函数调用会建立栈帧，函数返回时相应空间被复用；但在真实的 arm64 调用约定中，参数和返回值可能通过寄存器传递，编译器也可能省略栈帧或把局部变量保存在寄存器里。

在 arm64/iOS 上，线程栈通常向低地址增长：需要新的栈空间时，栈顶指针（`SP`）向更小的地址移动；函数返回时 `SP` 往回移动。这里不要顺势背诵“堆一定向高地址增长”：现代分配器会管理多个 Region，堆不是一条与栈相向生长的单一连续线。

![Snapzy_2026-07-27_18-38-30_502.png](https://cdn.jsdelivr.net/gh/tommywutong/piccbes@master/img/Snapzy_2026-07-27_18-38-30_502.png)

函数栈也称为栈，和上文是一个东西。
我们首先理解一下"栈帧"（Stack Frame）这个概念。在便于学习的未优化模型里，一次函数调用可能在栈上保存局部变量、溢出到栈上的参数、返回地址以及需要恢复的寄存器状态。它能很好地解释递归和栈回溯，但不是“每个参数、每个局部变量都必然在栈上”的硬性布局表；最终位置取决于调用约定和编译优化。

![image.png](https://cdn.jsdelivr.net/gh/tommywutong/piccbes@master/img/20260727201729938.png)

怎么这么像微机8086呢（bushi）

![image.png](https://cdn.jsdelivr.net/gh/tommywutong/piccbes@master/img/20260727201756951.png)


这种设计非常巧妙，因为它天然支持了函数的递归调用——每一层递归都会产生一个独立的栈帧，各自拥有独立的局部变量副本,互不干扰,直到最内层递归返回时,栈帧才逐层弹出。

栈溢出简单来说就是，如果递归函数嵌套太深或递归深度过大，栈帧（Stack Frame）不断累积，就会耗尽这块内存空间，产生栈溢出，程序因此崩溃

![image.png](https://cdn.jsdelivr.net/gh/tommywutong/piccbes@master/img/20260727195521956.png)


**堆**

堆是运行时动态分配内存的教学名称。它承接普通 Objective-C 对象、`malloc` 缓冲区等分配，其生命周期不由当前函数是否返回直接决定。C 缓冲区通常由 `malloc`／`free` 显式管理；Objective-C 对象则通常由 ARC 根据所有权关系插入相应的引用计数操作。

堆不是一段简单的、按固定方向线性增长的连续区域，也不遵循先进先出（FIFO）等队列语义——具体的组织方式和增长方向取决于分配器实现（iOS 使用 libmalloc 的 nano/magazine 等策略），后文会展开说明分配器如何管理多个 VM Region、把其中的小块空间分给不同对象。

关于栈对象和堆对象的一些其他细节，可以参考[这篇文章](https://www.tommywutong.cn/resources/mike-ash-friday-qa/02-memory-management/2010-01-15-stack-and-heap-objects-in-objective-c/)

**代码区**
存放编译后的机器指令。在 iOS 中，主程序代码通常来自 Mach-O 的可执行映射。

**常量区**
“常量区”用于概括字符串字面量、部分只读常量等内容。它们通常作为 Mach-O 中的只读数据随映像进入进程地址空间，可能分布在多个 Section 和 VM Region 中，并不存在一个必须连续、统一命名为“常量区”的真实区域。

**全局/静态区**

这一类用于概括具有静态存储期的对象。它们在程序整个执行期间都存在，已初始化数据与零填充数据通常由 Mach-O 的不同 Section 描述并在装载时建立映射。
- `未初始化`的`全局变量`和`静态变量`，即BSS区（.bss）
- `已初始化`的`全局变量`和`静态变量`，即数据区（.data）

“全局”描述名字的声明位置和可见范围，“静态存储期”描述对象的生命周期，两者不是按照“值能不能修改”来区分。文件作用域变量本来就具有静态存储期；`static` 在文件作用域还会改变符号的链接可见性，在函数内部则让局部名字对应的对象具有静态存储期。

----

“五大分区”更像一份用途分类，不包括进程中的所有真实映射。Framework、dyld shared cache、`mmap` 文件和 Guard Page 等内容，后文统一用 VM Region 解释。

```objc
#import <Foundation/Foundation.h>
#include <stdlib.h>

int globalNumber = 10;
static int staticNumber = 20;

void RunSimpleMemoryTest(void) {
    int localNumber = 30;
    NSString *text = @"Hello";
    NSObject *object = [[NSObject alloc] init];
    void *buffer = malloc(100);

    NSLog(@"global = %d", globalNumber);
    NSLog(@"static = %d", staticNumber);
    NSLog(@"local = %d", localNumber);
    NSLog(@"text = %@", text);
    NSLog(@"object = %@", object);
    NSLog(@"buffer = %p", buffer); // 在这里设置断点

    free(buffer);
}
```
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    RunSimpleMemoryTest();
}
```

先用五大分区回答这段代码：

| 代码中的内容 | 教学模型中的位置 | 更准确的说明 |
| --- | --- | --- |
| `RunSimpleMemoryTest` 的机器指令 | 代码区 | 通常来自 Mach-O 的 `__TEXT,__text`，运行时映射通常可读、可执行 |
| `@"Hello"` 字符串字面量 | 常量区 | 字面量对象通常来自主程序的只读 Mach-O 映射 |
| `globalNumber`、`staticNumber` | 全局/静态区 | 二者具有静态存储期，本例均为已初始化的可写数据 |
| `localNumber` | 栈区 | Debug、`-O0` 下通常能在当前线程栈中观察到；优化后位置可能变化 |
| 局部变量 `text`、`object`、`buffer` 本身 | 栈区 | 它们是保存地址的局部指针变量；优化后也可能进入寄存器 |
| `[[NSObject alloc] init]` 创建的普通对象 | 堆区 | `object` 保存对象地址，对象本体通常来自动态分配区域 |
| `malloc(100)` 返回的缓冲区 | 堆区 | `buffer` 保存缓冲区地址，缓冲区由分配器管理，最后需要 `free` |

>五大分区是一种用途分类，但这些区域并不是在同一时刻、通过同一种方式产生的。代码、常量和全局/静态数据主要由 Mach-O 在 App 启动时带入虚拟地址空间；堆与线程栈则在 App 运行过程中由内存分配器和线程系统建立。下面分别分析这两条形成路径。

### `static`

上面代码里的 `staticNumber` 只是文件作用域的 `static`。`static` 关键字在 C/Objective-C 里其实控制两件不同的事，容易被"是不是进静态区"这一个问题混在一起：

| 位置                         | `static` 改变的是什么                                                        | 是否改变存储位置                                            |
| -------------------------- | ---------------------------------------------------------------------- | --------------------------------------------------- |
| 文件作用域（如 `staticNumber`）    | 链接可见性：从 external linkage 变成 internal linkage，即这个符号不再能被其他编译单元引用         | 不改变。它本来就具有静态存储期，`static` 只是让它变成"文件私有"               |
| 函数局部（如 `static int count`） | 声明的作用域仍在函数内，但对象具有静态存储期，程序执行期间只存在一份 | 改变。它从一开始就由静态存储区域承载，并不是函数第一次执行时才从栈“搬过去” |

第一种不会改变对象原本的静态存储期，主要影响符号能否被其他编译单元引用；第二种则让一个作用域仍在函数内的名字，对应到一份贯穿程序执行期的存储。用一段实验代码观察第二种行为：

```objc
__attribute__((noinline, optnone))
void RunStaticCallCountTest(void) {
    static int staticCallCount = 0;
    int localCallCount = 0;
    staticCallCount++;
    localCallCount++;

    NSLog(@"staticCallCount 地址 = %p，值 = %d", &staticCallCount, staticCallCount);
    NSLog(@"localCallCount 地址 = %p，值 = %d", &localCallCount, localCallCount);
}
```

连续调用两次 `RunStaticCallCountTest()`，在 iOS 模拟器（iPhone 16 Pro，iOS 26.5，Xcode 26.6，Debug，`-O0`）上实测：

| 调用次数 | `staticCallCount` 地址 | `staticCallCount` 值 | `localCallCount` 地址 | `localCallCount` 值 |
| --- | --- | --- | --- | --- |
| 第 1 次 | `0x1043a4960` | 1 | `0x16ba634a4` | 1 |
| 第 2 次 | `0x1043a4960` | 2 | `0x16ba634a4` | 1 |

用 `memory region` 查询 `0x1043a4960` 落在 `0x1043a4000–0x1043a8000 rw-`，与该次运行中 `globalNumber`／`staticNumber` 所在的可写数据 Region 是同一块——说明这个函数局部 `static` 对象由静态可写数据映射承载，而不是当前线程栈。两次调用中它的地址不变、值从 1 累加到 2，跨调用保留了状态。

`localCallCount` 的地址在这两次调用之间也没变，这是本次模拟器 Debug 构建复用了同一段栈帧的结果，不代表栈变量本身具有跨调用保留状态的能力；它的值每次都重新是 1，说明栈上的存储在每次调用时都被重新初始化，与 `staticCallCount` 的行为形成对照。

## Mach-O
Mach-O是 iOS 可执行文件的组织格式，用来解释 App 启动前代码和全局数据保存在什么地方，以及启动后如何被映射进虚拟地址空间。

Mach-O 的 Segment 描述可执行文件及装载映射的组织方式；操作系统教材中的 Segmentation 描述的是一种地址管理模型。二者名字相似，但不能混为一谈。

### Segment 与 Section

Mach-O 使用两层结构组织内容：

- **Segment** 是装载和权限管理的大单位，例如 `__TEXT`、`__DATA_CONST`、`__DATA`。
- **Section** 位于 Segment 内部，用于继续区分具体内容，例如 `__text`、`__cstring`、`__data`、`__common`。

| Segment / Section | 实验中的内容 | 典型运行权限 |
| --- | --- | --- |
| `__TEXT,__text` | `RunSimpleMemoryTest` 的机器指令 | `r-x` |
| `__TEXT,__cstring` | C 字符串等字面量数据 | 随 `__TEXT` 映射；内容不可写，Region 可能因同一 Segment 包含代码而显示 `r-x` |
| `__DATA_CONST,__cfstring` | `@"Hello"` 等 Objective-C 字符串常量对象 | `r--` |
| `__DATA,__data` | `globalNumber`、`staticNumber` 等已初始化的可写数据 | `rw-` |
| `__DATA,__bss` / `__common` | 未初始化的全局、静态变量，装载时获得零填充空间 | `rw-` |

这里还有两个容易忽略的区域：

- `__PAGEZERO` 在 64 位 Mach-O 中保留低地址范围，不映射为可访问内存，有助于让空指针附近的访问尽早失败。
- `__LINKEDIT` 保存符号、字符串表、重定位等链接信息，服务于装载、符号解析和调试，但它不属于"五大分区"里的业务数据。

Mach-O 中记录的是链接时虚拟地址、大小和权限等装载信息。启动时，内核和 dyld 根据这些信息建立 VM Region，并继续映射依赖的 Framework 与 dyld shared cache。最终进程地址空间远大于主程序自己的 Mach-O。

### ASLR：为什么每次运行的地址可能不同

为了避免二进制每次都出现在固定地址，系统会在加载时引入随机的 **ASLR Slide**。

```text
运行时地址 = Mach-O 中的链接地址 + ASLR Slide
```

同一份二进制连续启动两次时，`RunSimpleMemoryTest` 的绝对地址通常会发生变化。函数没有“搬到另一个 Section”，改变的是整份映像的加载基址。实验中不需要记忆任何固定十六进制地址，只需使用 `image lookup -n RunSimpleMemoryTest` 查看本次运行的结果。

比较地址时应该先判断它属于哪个映像，并结合 `image list`、`image lookup` 或 `vmmap` 获取加载地址，不能把不同运行中的绝对地址直接比较。

这个关系也是符号化（Symbolication）的核心。崩溃日志或 `image lookup --address` 拿到的都是运行时地址：工具先根据地址落在哪个镜像的加载范围内，确定它属于哪个镜像（崩溃报告的 "Binary Images" 段会记录每个镜像的加载基址），再用该镜像自己的 slide 把运行时地址减回链接地址，最后拿这个链接地址去 dSYM 里查函数名和行号。用错镜像的 slide，或者把不同镜像、不同进程、不同次运行的绝对地址直接比较，都会得到没有意义的结果——这也是为什么调试时要先用 `image list` 确认加载基址，而不是直接比较两次运行记下的十六进制数。Apple 在 WWDC21 [Symbolication: Beyond the basics](https://developer.apple.com/videos/play/wwdc2021/10211/) 中完整演示了 Linker Address、Load Address 与 ASLR Slide 的关系。

## 堆与线程栈：解释运行过程中出现的区域

Mach-O 主要解释启动时已经存在的代码和数据。App 开始运行后，动态申请的数据通常进入由内存分配器管理的堆区域；函数调用则会使用当前线程自己的栈。

- **堆**：不是一整块固定且永远连续的区域。`malloc` 等分配器会管理一个或多个虚拟内存区域，并把合适的空间交给调用方。
- **线程栈**：每个线程都有自己的栈，用来支撑函数调用和临时状态。多个线程共享进程的代码、全局数据和堆，但不共享同一个线程栈。
- **局部变量**：在未优化的直觉模型中经常位于栈帧；经过编译优化后，也可能只存在于寄存器中。

## 指针变量和对象本体分别在哪里

例如，在一个函数中声明 `NSObject *object = [[NSObject alloc] init];` 时：

- 局部指针变量 `object` 具有自动存储期，编译器通常会让它位于当前栈帧中，但经过优化后也可能只存在于寄存器中。
- `object` 保存的是对象地址；普通 Objective-C 对象通常由运行时在动态分配区域中创建。
- `&object` 表示指针变量本身的地址，`object` 表示其保存的对象地址，两者不是同一个概念。
- 并非所有看起来像对象的值都对应一次普通堆分配，例如字符串字面量和 Tagged Pointer。

可以把这一行代码拆成两个问题：

```text
&object  ── 指针变量自己的地址
object   ── 指针变量保存的值
              │
              └── 普通情况下指向对象本体
```

在基础实验中，`&object` 通常会和 `&localNumber` 落在同一个线程栈 Region；`object` 的值则通常落在分配器管理的可读写 Region。具体十六进制地址每次运行都可能变化，重要的是它们属于不同的地址范围。

### 实验：同一个指针变量依次指向三个对象

只观察一次 `alloc`，虽然能看到 `&object` 和 `object` 是两个地址，但还不够直观。下面让同一个指针变量 `object` 依次保存三个对象的地址：

```objc
void RunPointerAndObjectTest(void) {
    NSObject *firstObject = [[NSObject alloc] init];
    NSObject *secondObject = [[NSObject alloc] init];
    NSObject *thirdObject = [[NSObject alloc] init];

    NSObject *object = nil;

    object = firstObject;
    NSLog(@"第 1 次：&object = %p，object = %p",
          (void *)&object, (__bridge void *)object); // 断点 1

    object = secondObject;
    NSLog(@"第 2 次：&object = %p，object = %p",
          (void *)&object, (__bridge void *)object); // 断点 2

    object = thirdObject;
    NSLog(@"第 3 次：&object = %p，object = %p",
          (void *)&object, (__bridge void *)object); // 断点 3

    // 确保三个对象在整个实验过程中都保持存活，排除地址复用干扰。
    NSLog(@"keep alive: %@ %@ %@", firstObject, secondObject, thirdObject);
}
```

在 `viewDidLoad` 中调用：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    RunPointerAndObjectTest();
}
```

将工程保持在 Debug、`-O0`，在标记的三行 `NSLog` 上分别设置断点。每次暂停都执行：
```text
p/x &object
p/x object
memory region &object
memory region object
```
如果当前 LLDB 版本不能直接解析 `memory region object`，就先复制 `p/x` 输出的实际地址，再执行：
```text
memory region 0x实际地址
```
把三次结果记录成下面的表格。不要照抄示例地址，以当次运行结果为准：

| 暂停位置 | `&object` | `object` | `&object` 所在 Region | `object` 所在 Region |
| --- | --- | --- | --- | --- |
| 第 1 次赋值 | 记录实际地址 | 记录第一个对象地址 | 当前线程栈，通常为 `rw-` | 分配器管理的可写 Region |
| 第 2 次赋值 | 应与第 1 次相同 | 应变为第二个对象地址 | 应与第 1 次相同 | 分配器管理的可写 Region |
| 第 3 次赋值 | 应与前两次相同 | 应变为第三个对象地址 | 应与前两次相同 | 分配器管理的可写 Region |

由于 `firstObject`、`secondObject`、`thirdObject` 在最后仍会被使用，三个对象在实验期间同时存活，分配器不能把同一块对象内存交给其中两个。因此这个实验应当稳定观察到：

- **`&object` 三次保持不变**：它表示局部指针变量 `object` 自己的地址。在本次 Debug 实验中，它通常位于当前线程栈。
- **`object` 三次保存不同的值**：它表示 `object` 当前保存的对象地址，三次分别指向三个仍然存活的普通对象。
- **两类地址所在 Region 不同**：`&object` 通常落在栈 Region；`object` 通常落在分配器管理的可写 Region。
已连接的 iPhone 15 上运行 `MemoryMapLab`，App 启动时连续调用两次实验函数，得到：

| 调用 | `&object` 与 Region | 第 1 个对象 | 第 2 个对象 | 第 3 个对象 | 三个对象所在 Region |
| --- | --- | --- | --- | --- | --- |
| 第 1 次 | `0x16f138950`，`0x16f040000–0x16f13c000 rw-` | `0x105da0bc0` | `0x105da0bd0` | `0x105da0be0` | `0x105c00000–0x106000000 rw-` |
| 第 2 次 | `0x16f138950`，`0x16f040000–0x16f13c000 rw-` | `0x105da0f50` | `0x105da0f60` | `0x105da0f70` | `0x105c00000–0x106000000 rw-` |
这两轮真机数据验证了三个结构性结论：

1. 在每次函数调用内部，三次赋值中的 `&object` 都是同一个地址，它表示当前栈帧中指针变量自己的存储位置。
2. 三个对象同时存活时，`object` 依次保存三个不同地址；三个地址虽然属于同一个分配器 Region，却不是同一块对象内存。
3. `&object` 位于线程栈 Region，`object` 保存的对象地址位于另一个分配器 Region，因此“局部指针变量在栈上”和“普通对象本体在堆中”可以同时成立。

两次函数调用中恰好再次出现相同的 `&object`，是这次 Debug 构建复用了同一段栈帧位置，不能据此声称局部变量具有静态存储期。三个对象地址在本次运行中呈现相邻分配，也不能据此推导 `NSObject` 的固定大小或分配器在其他系统版本上的固定策略。

还可以做一个反向实验：去掉 `firstObject`、`secondObject`、`thirdObject` 这些额外强引用，只反复执行 `object = [[NSObject alloc] init]`。旧对象生命周期结束后，分配器可能复用它的地址。但仅凭地址再次出现，不能精确证明 ARC 在哪一条指令释放了对象；它只能说明对象地址可能被复用，不能用地址大小或是否重复判断对象的新旧。

这正好说明“指针变量在栈上”和“对象在堆上”可以同时成立。但这句话仍有三个边界：

1. **编译器优化**：局部变量可能只存在于寄存器中，或者被完全消除，因此“局部变量一定在栈上”不严谨。
2. **字符串字面量**：`@"Hello"` 通常落在主程序的只读映射内，并不是函数每执行一次就在堆上创建一个新字符串对象。
3. **Tagged Pointer**：部分小型 `NSNumber`、`NSDate`、`NSString` 等值可能直接编码在指针中，并不对应普通堆对象。它属于第二轮实验，本次基础代码暂不加入。

因此，回答"对象在哪里"时，至少需要先确认讨论的是普通动态对象、字面量对象，还是 Tagged Pointer；回答"变量在哪里"时，还要区分变量本身和变量保存的值。

### 从“指针变量与对象本体”继续追问的面试题

“指针变量不等于对象本体”不只用于回答对象位于哪里。对象传参、属性访问、ARC、相等性和对象大小等问题，都依赖同一张关系图：

```text
指针变量自己的地址：&object
        ↓ 这个位置中保存了一个值
指针变量保存的值：object
        ↓ 普通情况下把它解释成对象地址
对象本体：该地址处由运行时和分配器管理的内存
```

下面按理解顺序梳理常见追问。

#### 问题一：给方法传入对象，传的是对象还是指针？

```objc
static void ReplaceObject(NSObject *object) {
    object = [[NSObject alloc] init];
}

NSObject *original = [[NSObject alloc] init];
ReplaceObject(original);
```

调用 `ReplaceObject(original)` 时，复制的是 `original` 保存的对象地址，不是复制整个对象
函数内部重新执行 `object = ...`，只改变参数这个局部指针变量保存的值，不会修改调用方的 `original`。

> **面试回答**：Objective-C 方法传递对象时，参数接收的是对象指针值的一份副本。给参数重新赋值只改变参数自己的指向，不会修改调用方的指针变量。

#### 问题二：修改指针与修改对象有什么区别？

```objc
static void ChangePointer(NSMutableString *string) {
    string = [NSMutableString stringWithString:@"new"];
}

static void ChangeObject(NSMutableString *string) {
    [string appendString:@"!"];
}
```

两段代码做的不是同一件事：

```text
string = anotherString
→ 修改局部指针变量保存的地址
→ 让它改为指向另一个对象

[string appendString:@"!"]
→ 读取 string 保存的地址
→ 找到双方共同指向的对象
→ 修改该对象的内部状态
```

因此，`ChangePointer` 不会把调用方的指针改成新字符串；`ChangeObject` 则可能让调用方观察到同一个可变字符串对象已经发生变化。

> **面试回答**：重新赋值改变的是指针的指向；向对象发送修改消息，改变的是该指针所指向对象的状态。

#### 问题三：为什么 `NSError **` 可以修改调用方的指针？

```objc
NSError *error = nil;
BOOL success = [manager loadData:&error];
```

这里传入的不是 `error`，而是 `&error`：

```text
error   的类型：NSError *
error   的含义：保存 NSError 对象的地址

&error  的类型：NSError **
&error  的含义：指针变量 error 自己的地址
```

普通 `NSError *` 参数只能得到对象地址的一份副本；`NSError **` 参数拿到了调用方指针变量所在的位置，因此被调用方可以把一个新的错误对象地址写回 `error`。在 ARC 下，Cocoa 常见的错误返回参数还涉及 `__autoreleasing` 推断，具体所有权规则放到 ARC 专题继续讨论。

> **面试回答**：调用时传入 `&error`，相当于把调用方指针变量自己的地址交给方法，所以方法能够修改该指针变量保存的对象地址。

#### 问题四：`self.name`、`_name` 和局部变量 `name` 是一回事吗？

```objc
@interface Person : NSObject
@property (nonatomic, copy) NSString *name;
@end

- (void)test {
    NSString *name = @"Tommy";
    self.name = name;
}
```

三者处于不同层级：

```text
name
→ 当前函数中的局部指针变量
→ Debug、未优化实验中通常位于栈

_name
→ Person 对象内部的实例变量
→ 它的存储跟随 Person 对象本体
→ 它保存的仍然是 NSString 对象地址

self.name
→ 属性访问表达式
→ 赋值时通常调用 setName:
→ 读取时通常调用 name
```

`property` 本身不是一个新的“内存分区”。自动合成属性时，真正提供存储的是实例变量 `_name`；而 `_name` 也没有把整个字符串对象嵌入 `Person`，它通常只是保存另一个对象的地址。

> **面试回答**：局部 `name` 是函数内的指针变量，`_name` 是 `Person` 对象内部的实例变量，`self.name` 通常表示一次 getter 或 setter 调用。

#### 问题五：`strong` 和 `weak` 决定变量在栈还是堆吗？

不决定。

```objc
__strong NSObject *strongObject;
__weak NSObject *weakObject;
```

`strong` 和 `weak` 描述的是引用与对象之间的所有权关系：

- `strong` 引用会维持所指对象的生命周期；
- `weak` 引用不会延长对象生命周期；对象销毁后，weak 引用会被置为 `nil`。

栈、寄存器和对象内部实例变量讨论的是“这个指针变量存在哪里”；`strong`、`weak` 讨论的是“这条引用是否拥有对象”。一个局部 strong 指针仍可能在栈或寄存器中，一个对象内部的 weak 实例变量则跟随对象本体。

> **错误回答**：strong 在堆上，weak 在栈上。
> **正确回答**：strong/weak 决定所有权语义，不决定指针变量的内存分区。

#### 问题六：执行 `object = nil`，对象会马上销毁吗？

不一定。
```objc
NSObject *first = [[NSObject alloc] init];
NSObject *second = first;

first = nil;
```
这里直接发生的是：指针变量 `first` 不再保存原对象地址，并解除由 `first` 建立的那一条 strong 引用。但 `second` 仍然指向并强引用原对象，因此对象可以继续存活。

```text
first = nil
→ first 不再指向原对象
→ 不等于把对象所在内存直接“擦掉”

对象是否销毁
→ 取决于是否还存在其他 strong 引用
```

> **面试回答**：把 strong 指针设为 `nil` 只解除这一条强引用；只有最后一条强引用消失后，对象才具备结束生命周期的条件。


#### 问题七： `==` 和 `isEqual:` 比较的是什么？

```objc
NSString *first = [NSString stringWithFormat:@"Tommy"];
NSString *second = [NSString stringWithFormat:@"Tommy"];

BOOL samePointer = (first == second);
BOOL equalValue = [first isEqual:second];
```

对于对象指针：

```text
first == second
→ 比较两个指针变量保存的地址
→ 判断是否指向同一个对象

[first isEqual:second]
→ 询问对象定义的逻辑相等关系
→ 判断两者表示的值是否相等
```

不同类可以按照自身语义实现 `isEqual:`。如果自定义类重写 `isEqual:` 并准备把对象放进 `NSSet`、`NSDictionary` 等哈希集合，还必须维持下面的契约：

```text
[a isEqual:b] == YES
        ↓
a.hash == b.hash
```

> **面试回答**：`==` 比较对象身份，也就是指针地址；`isEqual:` 比较类所定义的逻辑相等性。

#### 问题八：`sizeof(object)` 得到的是对象大小吗？

```objc
NSObject *object = [[NSObject alloc] init];
NSLog(@"%zu", sizeof(object));
```

在当前 64 位 iPhone 上，`sizeof(object)` 通常得到 `8`，因为它测量的是对象指针的宽度，不是 `NSObject` 对象本体或分配器实际分配块的大小。

需要区分三个问题：
```text
sizeof(object)
→ 指针变量占多少字节

class_getInstanceSize([object class])
→ 该类的实例布局至少需要多少字节

malloc_size((__bridge const void *)object)
→ 分配器为这个普通堆对象实际提供的分配块有多大
```

后两个数字也不要求相同：分配器可能按照大小等级向上取整。`malloc_size` 也不能用于 Tagged Pointer 等并不对应普通堆分配的值。

> **面试回答**：`sizeof(object)` 只得到指针大小；对象实例布局与分配器实际分配大小需要分别观察。

#### 继续学习的顺序

这些问题可以沿同一条主线继续推进：

```text
指针变量与对象本体
        ↓
修改指针 vs 修改对象
        ↓
对象参数传递的是指针值副本
        ↓
NSError ** 为什么可以写回
        ↓
self.name、_name、局部 name
        ↓
strong、weak 与对象生命周期
        ↓
==、isEqual: 与 hash
        ↓
copy、集合、Block 与 Tagged Pointer
```
当前文章先把前八个问题作为从“地址与位置”走向“对象语义”的连接点。`copy` 与深浅拷贝、集合怎样持有对象、Block 捕获、`__block`、weak 实现和引用循环，应分别在 ARC、Block 或 Runtime 专题中展开，避免把“对象在哪里”和“对象如何被拥有”重新混成一个问题。

## VM Region

VM Region 是内核和调试工具观察进程虚拟地址空间的实际单位之一：一段连续的虚拟地址范围，具有相应的保护权限、映射来源和其他属性。它可以来自主程序或 Framework 的 Mach-O 映射，也可以是分配器管理的区域、线程栈、Guard Page 或 `mmap` 文件。

| 常见 Region | 可能的来源与用途 |
| --- | --- |
| 主程序／Framework 映像 | `__TEXT`、`__DATA_CONST`、`__DATA` 等 Mach-O Segment 的运行时映射 |
| 分配器管理区域 | 普通对象和 `malloc` 缓冲区所使用的动态分配空间 |
| 线程栈与 Stack Guard | 每个线程的调用栈，以及用于越界保护的无权限边界 |
| 映射文件 | 由 `mmap` 等机制建立的文件内容映射 |

查询一个具体地址所属的 Region，可以先回答三件事：它落在哪一段地址范围、当前允许读写还是执行、这段映射大致来自哪里。但 Region 本身不会直接告诉我们“这里有哪几个 Objective-C 对象”，也不能只凭 Region 大小推导已驻留物理内存或 Memory Footprint。

因此，前面的概念可以这样串起来：

- 代码、常量和部分全局数据由 Mach-O 描述，装载后成为一个或多个 VM Region；
- 堆是分配器在一个或多个 VM Region 上继续切分出的动态分配空间；
- 每个线程的栈也对应自己的 Region，并可能带有 Guard Region；
- 指针变量与它指向的对象本体可能落在不同 Region。

本篇继续观察“地址在哪里、映射从哪里来”。`virtual_size`、`resident_size`、Clean、Dirty、Compressed 与 Memory Footprint 等页面记账和回收问题，留到系列下一篇《iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint》展开。

到这里已经知道：

- 代码、常量和部分全局数据由 Mach-O 带入进程；
- 堆在运行过程中承接动态分配；
- 每个线程拥有自己的栈；
- 指针变量和对象本体可能位于不同区域。

**五大分区按照用途分类，VM Region 则是内核描述一段连续虚拟地址范围的实际方式。** 一个 Region 具有起止地址、访问权限、内容来源等属性；同一教学分区可能由多个 Region 组成，一个 Mach-O Segment 在实际映射和保护过程中也不能简单等同于整张进程内存地图。

### 两种常见来源：文件映射与匿名内存

从"页面内容可以去哪里重新找"这一角度，VM Region 可以先粗分为两类：

- **文件映射（File-backed Mapping）**：内容来自 Mach-O、Framework、dyld shared cache 或通过 `mmap` 映射的文件。未被修改的页面可以丢弃，需要时再从原文件或系统缓存取得，因此通常更容易保持为 clean。
- **匿名内存（Anonymous Memory）**：没有一个可直接重新读取的原始文件，常见于堆、线程栈以及运行时申请的数据。程序真正写入后，相关页面通常成为 dirty；具体怎样计入 Footprint、压缩或回收，留到下一篇讨论。

这不是说"文件映射永远 clean、匿名内存永远 dirty"。Mach-O 中的可写数据可能通过 Copy-on-Write 形成进程私有页面，文件映射也可能被修改；新申请的匿名地址也可能尚未真正触碰。这里的分类是为了回答内容来源，为系列下一篇的 Clean、Dirty 与 Memory Footprint 建立桥梁，具体的页面状态转换会在那一篇里展开。

### VM Region 的访问权限

调试工具经常用三个字母表示 Region 当前允许的操作：

| 权限 | 含义 | 常见例子 |
| --- | --- | --- |
| `r--` | 可读，不可写，不可执行 | 字符串对象和其他只读数据映射 |
| `rw-` | 可读、可写，不可执行 | 全局可写数据、堆、线程栈 |
| `r-x` | 可读、不可写、可执行 | Mach-O 中的机器指令 |
| `---` | 当前不可读写执行 | `__PAGEZERO`、Stack Guard 等保护区域 |

权限让同一进程中的区域承担不同职责，也贯彻了"数据不可随意执行、代码不可随意写入"的安全边界。Apple 在 [Investigating memory access crashes](https://developer.apple.com/documentation/xcode/investigating-memory-access-crashes) 中展示了 `r-x` 的 `__TEXT`、`rw-` 的线程栈以及无权限的 Stack Guard。

把五大分区、来源和权限放到一起，可以得到更接近真实系统的关系：

| 教学用途 | 常见形成方式 | 常见权限 | 后续页面状态线索 |
| --- | --- | --- | --- |
| 代码 | Mach-O / Framework 文件映射 | `r-x` | 未修改时通常可从文件重新取得 |
| 常量 | Mach-O 只读数据映射 | `r--`，或随 `__TEXT` 显示为 `r-x` | 通常更容易保持 clean |
| 全局/静态数据 | Mach-O 数据映射、零填充页、Copy-on-Write | `rw-` | 被写入后可能贡献 dirty memory |
| 堆 | 分配器管理的匿名内存 | `rw-` | 实际写入的页面通常贡献 dirty memory |
| 线程栈 | 每个线程各自的匿名 Region | `rw-`，边界附近可有 `---` Guard | 被使用的栈页面属于进程需要负责的内存 |

这一步完成了本文最重要的转换：
```text
五大分区告诉我们"用来做什么"
        ↓
Mach-O、堆和线程栈告诉我们"怎样形成"
        ↓
VM Region 告诉我们"系统怎样描述"
        ↓
文件/匿名来源和页面状态继续解释"系统怎样回收与记账"（见系列下一篇）
```

## 用 LLDB 验证基础实验

本次记录的实验环境为：

| 项目    | 环境                                                                       |
| ----- | ------------------------------------------------------------------------ |
| Xcode | 26.6                                                                     |
| 运行环境  | iPhone 16 Pro Simulator，Apple Silicon Mac（具体 iOS 版本以当次运行环境为准，不作为固定结论的依据） |
| 构建方式  | Objective-C、Debug 信息、`-O0`                                               |
| 页面大小  | `vm_page_size = 16384`，即 16 KB                                           |
| 验证方式  | 在 `RunSimpleMemoryTest` 内设置断点，通过 LLDB 比较变量地址并查询 VM Region                |

Simulator 与真机不完全相同，尤其是系统共享缓存、分配器实现、地址编码和内存压力行为。下面的基础步骤用于学习怎样观察地址；本节后半部分再用 iPhone 15 的两轮真机数据验证结论。无论哪种环境，都不能由一次运行推导所有 iPhone 的固定地址。

在 `NSLog(@"buffer = %p", buffer);` 这一行设置断点。程序停下后，先执行第一组命令，只比较“变量本身的地址”和“变量保存的地址”：

```text
p/x &globalNumber
p/x &staticNumber
p/x &localNumber
p/x &text
p/x text
p/x &object
p/x object
p/x &buffer
p/x buffer
```

这里应该先观察到三组关系：

1. `&globalNumber` 与 `&staticNumber` 通常落在主程序的可写数据映射中；
2. `&localNumber`、`&text`、`&object`、`&buffer` 在 Debug、`-O0` 下通常落在当前线程的栈 Region；
3. `text`、`object`、`buffer` 保存的地址与这些局部指针变量自己的地址不同。

第一组理解之后，再检查这些地址分别属于哪个映像和 VM Region：

```text
image lookup -n RunSimpleMemoryTest
image list
```

`image lookup` 用来确认函数属于主程序的 `__TEXT,__text`。对于其他变量，先复制 `p/x` 得到的实际地址，再执行：

```text
memory region 0x实际地址
```

例如，假设本次 `object` 打印为 `0x600000012340`，就执行：

```text
memory region 0x600000012340
```

这里的地址只是命令格式示例，不是固定结果。ASLR、分配器和每次运行状态都会改变绝对地址。

基础实验应当得到下面这些“关系结果”：

| 观察对象 | 预期的 Region 特征 | 能说明什么 |
| --- | --- | --- |
| `RunSimpleMemoryTest` | 主程序映像中的 `r-x` 区域 | 函数机器指令属于代码映射 |
| `text` 指向的 `@"Hello"` | 主程序的只读映射，具体权限以当次构建为准 | 字符串字面量不是每次调用都新建的普通堆对象 |
| `globalNumber`、`staticNumber` | 主程序映像中的 `rw-` 区域 | 已初始化的全局、静态变量来自可写数据映射 |
| `&localNumber` | 当前线程的 `rw-` 栈 Region | 局部整数在本次 Debug 实验中位于线程栈 |
| `&text`、`&object`、`&buffer` | 通常与 `&localNumber` 位于同一个栈 Region | 局部指针变量本身也属于当前函数的临时状态 |
| `object` | 分配器管理的可读写 Region | 普通 Objective-C 对象本体通常在堆中 |
| `buffer` | 分配器管理的可读写 Region | `malloc` 返回的是动态缓冲区地址 |

不要比较表中的“地址大小关系”，而要比较它们属于哪个映像、哪个 Region、具有怎样的权限。再次运行时绝对地址变化是正常现象。

### iPhone 15 真机实测
为了确认 Simulator 上的结论能否在真实 iOS 环境中复现，我另外建立了一个最小 Objective-C 实验 App `MemoryMapLab`。真机环境如下：

| 项目 | 环境 |
| --- | --- |
| 设备 | iPhone 15（iPhone15,4，arm64e） |
| 系统 | iOS 26.5.2（23F84） |
| Xcode | 26.6 |
| 构建方式 | Objective-C、Debug、`-O0`，实验函数使用 `noinline` 与 `optnone` 保持可观察性 |
| 页面大小 | `sysconf(_SC_PAGESIZE) = 16384`，即 16 KB |
| Region 数据来源 | App 对自身地址调用 `vm_region_64`，同时输出地址、Region 范围和当前权限 |

第一轮启动取得的代表性结果如下：

| 观察对象 | 运行时地址 | 所在 Region 与权限 | 说明 |
| --- | --- | --- | --- |
| `RunMemoryExperiment` | `0x102eb4e14` | `0x102eb0000–0x102eb8000 r-x` | 函数机器指令位于主程序的可执行映射 |
| `globalInitialized` | `0x102ebc958` | `0x102ebc000–0x102ec0000 rw-` | 已初始化全局变量位于可写数据映射 |
| `staticInitialized` | `0x102ebc95c` | `0x102ebc000–0x102ec0000 rw-` | `static` 改变链接可见性，不会因此形成一个独立“静态区” |
| `globalZeroInitialized` | `0x102ebc960` | `0x102ebc000–0x102ec0000 rw-` | 零初始化变量运行时同样需要可写存储 |
| `&globalStringLiteral` | `0x102eb80d8` | `0x102eb8000–0x102ebc000 r--` | 这是全局常量指针变量自己的地址 |
| `globalStringLiteral` 指向的字面量对象 | `0x102eb8120` | `0x102eb8000–0x102ebc000 r--` | 字符串字面量本体位于只读映射 |
| `&localNumber` | `0x16cf4c96c` | `0x16ce54000–0x16cf50000 rw-` | 本次 Debug 构建中局部整数位于主线程栈 |
| `&object` | `0x16cf4c960` | `0x16ce54000–0x16cf50000 rw-` | 局部指针变量本身与其他局部状态位于线程栈 |
| `object` 指向的 `NSObject` | `0x107d98ba0` | `0x107c00000–0x108000000 rw-` | 对象本体位于分配器管理的可写 Region |
| `malloc` 返回的缓冲区 | `0x107eb0000` | `0x107c00000–0x108000000 rw-` | 堆并非一个独立、连续且边界固定的“大盒子” |
| `@(runtimeValue)` | `0x9af4dd61519470e0` | 无普通 VM Region | 本次运行生成了 Tagged Pointer，没有单独的普通堆对象地址 |

> 上表 Tagged Pointer 一行显示为 16 位十六进制（完整 64 位），而其他行是 9~10 位。这不是数据错误：其他行打印的是"地址"，数值范围受当前地址空间实际使用量限制；这一行打印的是指针变量的完整位模式本身——因为 Tagged Pointer 不指向任何内存，它的值就是数据编码后直接摆放的结果，所以会占满整个 64 位宽度。

这里最重要的不是记住这些十六进制数，而是观察地址之间的关系：

1. 函数、只读字面量、可写全局数据分别落入 `r-x`、`r--`、`rw-` 的映像 Region；
2. 局部指针变量和它指向的对象本体是两个地址，前者本次位于栈，后者位于分配器管理的区域；
3. 普通对象与 `malloc` 缓冲区虽然都可归入教学模型中的“堆”，系统看到的却是分配器管理的 VM Region；
4. Tagged Pointer 的值不是一个可以交给 `memory region` 查询的普通对象地址。其具体编码属于运行时实现细节，不能依赖本次位模式。

为了验证 ASLR，完全退出并第二次启动同一份二进制，得到以下代表性变化：

| 观察对象 | 第一轮 | 第二轮 |
| --- | --- | --- |
| `RunMemoryExperiment` | `0x102eb4e14` | `0x1041a0e14` |
| `globalInitialized` | `0x102ebc958` | `0x1041a8958` |
| 字符串字面量对象 | `0x102eb8120` | `0x1041a4120` |
| `&localNumber` | `0x16cf4c96c` | `0x16bc6096c` |
| `NSObject` 对象本体 | `0x107d98ba0` | `0x1091acad0` |

主程序中的函数、全局数据和字面量跟随同一个 Mach-O 映像整体移动，权限和相对布局保持一致；栈和堆地址则由线程栈、分配器和当次运行状态分别决定，不能拿它们与 Mach-O 的 slide 做同一种推导。这正是“ASLR 改变装载基址，但不会把代码突然变成堆内存”的真机证据。

需要明确数据来源：上表的地址与 Region 是实验 App 通过 `vm_region_64` 对自身查询后记录的，不是伪装成 LLDB transcript 的输出。要在 Xcode 中独立复核，可以给 `MemoryExperimentBreakpoint` 添加符号断点；停下后先用 `up` 回到 `RunMemoryExperiment`，再执行：

```text
frame variable localNumber object localLiteral runtimeValue taggedNumber heapBuffer
p/x &globalInitialized
p/x &globalZeroInitialized
p/x &staticInitialized
p/x globalStringLiteral
memory region &globalInitialized
memory region heapBuffer
```

### iPhone 15 真机实验：地址空间有多大，真正可用多少

[Size Matters](https://alwaysprocessing.blog/2022/02/20/size-matters) 根据 2022 年公开的 XNU 源码讨论了 iOS 设备的虚拟地址空间、可用地址空间和 Extended Virtual Addressing。但文章中的设备表格、XNU 版本和“扩展到 64 GB”都是当时的结果，不能直接当作今天所有设备的固定结论。这里继续使用 `MemoryMapLab`，在当前 iPhone 15 上重新测量。

实验前先分清五个容易混在一起的数字：

| 数字 | 它真正表示什么 |
| --- | --- |
| 64 bit 指针 | 一个普通指针值的存储宽度，不代表进程能使用 \(2^{64}\) 字节 |
| `MACH_VM_MAX_ADDRESS` | 当前 SDK 向用户空间公开的 CPU 虚拟地址上界常量，不等于当前普通 task 一定可以分配到该处 |
| `task_vm_info.virtual_size` | task 的虚拟内存记账值，可能包含 GPU carveout 等特殊保留区，不能直接当作 App 可用容量 |
| 当前 task 的可寻址上界 | 通过在不同高地址临时保留一个页面测得的本次运行边界 |
| 最大连续可保留区间 | 当前地址空间中最大的一个连续空洞；它不等于全部空洞之和，也会受现有映射和地址碎片影响 |

#### 实验方法

当前 iPhoneOS 26.5 SDK 不支持 App 直接包含 `<mach/mach_vm.h>`。实验使用公开的 `<mach/mach.h>`，并调用 `vm_allocate`、`vm_deallocate`、`vm_region_64` 与 `task_info`：

1. 用 `vm_region_64` 枚举当前 CPU 地址范围内的 VM Region；
2. 用 `vm_allocate(..., VM_FLAGS_ANYWHERE)` 二分探测最大连续空洞，精度为 64 MiB；
3. 在 8、12、16、24、32、48、60 GiB 附近固定保留一个 16 KB 页面；
4. 在 16～24 GiB 之间继续二分，把当前普通 task 的上界细化到一个页面；
5. 所有成功保留的区域都不进行读写，并立刻用 `vm_deallocate` 释放；
6. 对比实验前后的 Memory Footprint、`virtual_size` 和 Region 数量，确认没有把大块虚拟地址转化成同等大小的物理内存占用。

最核心的“保留但不触碰”代码如下：

```objc
#import <mach/mach.h>
#import <unistd.h>

static kern_return_t ProbeOnePage(vm_address_t requestedAddress) {
    vm_size_t pageSize = (vm_size_t)sysconf(_SC_PAGESIZE);
    vm_address_t address = requestedAddress;

    kern_return_t result = vm_allocate(
        mach_task_self(),
        &address,
        pageSize,
        VM_FLAGS_FIXED
    );

    if (result == KERN_SUCCESS) {
        // 不读、不写，立即释放；这里只验证虚拟地址能否被保留。
        vm_deallocate(mach_task_self(), address, pageSize);
    }
    return result;
}

static BOOL ReserveAndRelease(vm_size_t size, vm_address_t *resultAddress) {
    vm_address_t address = 0;
    kern_return_t result = vm_allocate(
        mach_task_self(),
        &address,
        size,
        VM_FLAGS_ANYWHERE
    );

    if (result != KERN_SUCCESS) {
        return NO;
    }

    if (resultAddress != NULL) {
        *resultAddress = address;
    }
    vm_deallocate(mach_task_self(), address, size);
    return YES;
}
```

这里绝不能对数 GiB 的测试区域执行 `memset`。一旦真正读写页面，实验问题就会从“虚拟地址能否保留”变成“设备能否提供物理页以及何时被 Jetsam 终止”。

#### iPhone 15 实测结果

本次 App 由 Personal Team 签名，没有 `com.apple.developer.kernel.extended-virtual-addressing` entitlement。得到：

| 观察项 | 真机结果 |
| --- | --- |
| 指针宽度 | 64 bit |
| 页面大小 | 16 KB |
| SDK 的 `MACH_VM_MAX_ADDRESS` | `0xFC0000000`，即 63 GiB |
| `task_vm_info.virtual_size` | 391.12 GiB |
| 63 GiB CPU 上界以下的枚举结果 | 88 个 Region，共 6.12 GiB |
| GPU carveout | `0x1000000000–0x7000000000`，即 64～448 GiB，共 384 GiB，权限 `---` |
| 当前普通 task 的页级上界 | `0x458000000`，即 17.375 GiB |
| 当前最大连续可保留区间 | 约 5.375 GiB，成功样本起点为 `0x300000000`，即 12 GiB |
| 探测前后 Memory Footprint | 5.33 MiB → 5.33 MiB |
| 探测前后 `virtual_size` | 391.12 GiB → 391.12 GiB |
| 探测前后 Region 数 | 88 → 88 |

高地址单页探测结果为：

| 探测位置 | 结果 | 本次实验能说明什么 |
| --- | --- | --- |
| 8 GiB 附近 | `KERN_NO_SPACE`（3） | 该范围已经存在映射，失败不代表越过地址上界 |
| 12 GiB 附近 | 成功 | 普通 task 可以在此处保留页面 |
| 16 GiB 附近 | 成功 | 当前 task 可寻址范围已经超过 16 GiB |
| 24、32、48、60 GiB 附近 | `KERN_INVALID_ADDRESS`（1） | 这些候选地址超过了当前未扩展 task 的可分配边界 |

因此，这次实验中的地址关系可以简化成：

```text
指针宽度：64 bit
    ≠ App 拥有 2^64 字节可用空间

SDK 公开 CPU 上界：63 GiB
    ≠ 当前普通 task 的实际上界

当前普通 task 上界：17.375 GiB
    ≠ 17.375 GiB 全部可供堆使用

当前最大连续空洞：约 5.375 GiB
    ≠ App 可以安全写入 5.375 GiB 物理内存
```

391.12 GiB 也不是 App 突然获得了数百 GiB 可用内存。当前 Apple XNU 的 arm64 参数把 `0x1000000000–0x7000000000` 定义为 GPU carveout；真机的 `vm_region_64` 查询也看到同一段 384 GiB、权限为 `---` 的 Region。391.12 GiB 减去这段特殊保留区后只剩 7.12 GiB，其中还包含普通映射与 CPU/GPU 边界处的保留范围；严格限制在 63 GiB CPU 上界以下枚举时得到的是 6.12 GiB。它适合做 task 记账观察，却不能脱离 VM Region 明细直接解释为“App 可用地址空间”。

#### Extended Virtual Addressing 今天应该怎样理解

Apple 目前仍提供 [Extended Virtual Addressing Entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.kernel.extended-virtual-addressing)，官方只承诺它允许 App 访问扩展地址空间，并没有在文档中承诺一个适用于所有系统版本和设备的固定 GiB 数值。

2022 年文章根据当时的 XNU 把它解释为 64 GB “jumbo”模式。当前 Apple 开源 XNU 的 [`vm_param.h`](https://github.com/apple-oss-distributions/xnu/blob/main/osfmk/mach/arm/vm_param.h) 已经出现新的 `EXTENDED_USER_VA_SUPPORT` 路径：内核私有配置可以把 `MACH_VM_MAX_ADDRESS_RAW` 改为 `0x00007FFFFE000000`。这说明旧文里的“扩展后就是 64 GB”不再适合作为永久结论；但开源主分支中的上界也不能反过来当作这台 iPhone 15 已经实测得到的容量。

本次 Personal Team 的 provisioning profile 不包含该 entitlement，Xcode 当前能力元数据也不允许此团队类型启用它，所以这里只完成了“未扩展”基准。以后使用支持该 capability 的开发团队时，应当按下面的顺序复测：

1. 在 Target → Signing & Capabilities 中添加 **Extended Virtual Addressing**；
2. 检查最终签名 App 的 entitlements，不能只看工程中的 `.entitlements` 文件；
3. 在同一台设备、相同系统和近似运行状态下再次执行本实验；
4. 对比页级上界、最大连续空洞和高地址单页探测；
5. 不把扩展地址空间误写成 RAM、Memory Footprint 或 Jetsam 上限的提高。

### 从单个地址扩展

LLDB 适合追踪"这个地址是什么"，`vmmap` 更适合回答"整个进程有哪些 Region"。对运行中的调试进程或导出的 Memgraph，可以从下面的命令开始：

```shell
vmmap -summary 12345
vmmap /path/to/App.memgraph
```

Xcode 与 Instruments 中几个常见工具关注的问题并不相同：

| 工具 | 主要回答的问题 |
| --- | --- |
| LLDB | 某个变量、对象或地址当前是什么 |
| `vmmap` / VM Tracker | 进程有哪些 VM Region，各区域多大、权限和 dirty/compressed 情况如何 |
| Xcode Memory Report | App 当前的 Memory Footprint 和趋势如何 |
| Debug Memory Graph | 哪些对象彼此持有，是否存在引用环 |
| Allocations / Leaks | 堆对象何时分配，是否存在无法回收的泄漏 |

结合本次实验，可以先画出一张不强调高低地址顺序的内存地图：

```text
MemoryMapExperiment 进程的虚拟地址空间
├── 主程序代码映射：Mach-O __TEXT / __text，r-x
├── 主程序只读数据：字符串、__DATA_CONST 等，r--
├── 主程序可写数据：全局变量、静态变量，rw-
├── 动态分配区域：普通对象、malloc buffer，rw-
├── 主线程栈：局部状态，rw-
├── 后台线程栈：局部状态，rw-
├── Framework 与 dyld shared cache：代码和数据映射
└── 其他 VM Region：匿名映射、mmap 文件、Guard 等
```


---

## 总结

到这里可以得出六个结论：

1. 进程使用的是由多个 VM Region 组成的虚拟地址空间，虚拟地址并不等于物理地址。
2. 合法地址对应的页面尚未驻留也可能触发 Page Fault；内核能够修复的 Page Fault 是按需分页的正常过程，不等于崩溃。
3. 文件映射和匿名内存说明页面内容从哪里来，`r--`、`rw-`、`r-x` 等权限说明程序能对该 Region 做什么。
4. "五大分区"是一张用途地图，适合建立直觉，但不是 Darwin/iOS 对所有 VM Region 的完整分类，也遗漏了 Framework、dyld shared cache、`mmap` 文件、Guard Page 等真实存在的 Region。
5. Mach-O 解释启动时的代码和数据如何进入地址空间；堆和线程栈则解释运行过程中出现的动态区域；ASLR 会改变映像每次运行的加载地址，符号化的本质就是把这个偏移减回去。
6. 回答"变量和对象在哪里"时，必须分别讨论变量本身、变量保存的地址、对象本体以及字面量或 Tagged Pointer 等例外。

这六条结论回答的都是"在哪里"和"从哪里来"。但虚拟地址范围、堆分配量、驻留物理页和 Memory Footprint 是不同的指标——一段地址被"分配"，不代表它现在真的占用同等大小的物理内存。这句话正好是系列下一篇的起点：《iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint》会继续讨论 Clean、Dirty、Compressed 与内存压力下的回收策略。

## 参考资料

### 官方资料

- [Apple — About the Virtual Memory System](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/AboutMemory.html)
- [Apple — Overview of the Mach-O Executable Format](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/CodeFootprint/Articles/MachOOverview.html)
- [Apple — Viewing Virtual Memory Usage](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/VMPages.html)
- [Apple — Investigating memory access crashes](https://developer.apple.com/documentation/xcode/investigating-memory-access-crashes)
- [Apple — Reducing your app's launch time](https://developer.apple.com/documentation/xcode/reducing-your-app-s-launch-time)
- [WWDC21 — Symbolication: Beyond the basics](https://developer.apple.com/videos/play/wwdc2021/10211/)
- [Apple Kernel Programming Guide — Memory and Virtual Memory](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/vm/vm.html)
- [Apple — Extended Virtual Addressing Entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.kernel.extended-virtual-addressing)
- [Apple Open Source XNU — arm64 VM parameters](https://github.com/apple-oss-distributions/xnu/blob/main/osfmk/mach/arm/vm_param.h)
- [Apple — Working with Objects](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithObjects/WorkingwithObjects.html)
- [Apple — Object Ownership](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/ObjectOwnership.html)
- [Apple — NSObjectProtocol `isEqual:`](https://developer.apple.com/documentation/objectivec/nsobjectprotocol/1418795-isequal)
- [Clang — Objective-C Automatic Reference Counting](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)
- [LLDB — GDB to LLDB command map](https://lldb.llvm.org/use/map.html)
- [LLDB — Symbolication](https://lldb.llvm.org/use/symbolication.html)

### 操作系统资料

- [小林 Coding — 图解操作系统](https://www.xiaolincoding.com/os/)
- [小林 Coding — 为什么要有虚拟内存？](https://www.xiaolincoding.com/os/3_memory/vmem.html)
- [小林 Coding — malloc 是如何分配内存的？](https://www.xiaolincoding.com/os/3_memory/malloc.html)

### iOS 与 Objective-C 延伸

- [Size Matters: An Exploration of Virtual Memory on iOS](https://alwaysprocessing.blog/2022/02/20/size-matters)
- [Mike Ash — Stack and Heap Objects in Objective-C](https://www.mikeash.com/pyblog/friday-qa-2010-01-15-stack-and-heap-objects-in-objective-c.html)
- [Mike Ash — Intro to the Objective-C Runtime](https://www.mikeash.com/pyblog/friday-qa-2009-03-13-intro-to-the-objective-c-runtime.html)
- [本站译文 — Stack and Heap Objects in Objective-C](https://www.tommywutong.cn/resources/mike-ash-friday-qa/02-memory-management/2010-01-15-stack-and-heap-objects-in-objective-c/)
