---
title: "【iOS】内存地图：从虚拟地址空间、堆栈到 Memory Footprint"
published: 2026-07-24
description: "从操作系统虚拟内存出发，先梳理 iOS App 的地址空间、Mach-O、五大分区与堆栈，再理解 Clean、Dirty、Compressed、Memory Footprint 和内存压力。"
tags: ["iOS", "Operating System", "Virtual Memory", "Memory", "Mach-O", "Memory Footprint", "LLDB"]
category: "iOS"
series: "2026 暑假 iOS 底层学习"
seriesSlug: "ios-internals-2026-summer"
seriesOrder: 1
draft: true
---
# iOS 内存地图：从虚拟地址空间、堆栈到 Memory Footprint

## 前言

在之前的文章中，笔者完成过两篇内存管理的入门文章：

- [【iOS】内存五大分区](https://www.tommywutong.cn/blog/csdn-import/csdn-154609757-ios-/)
- [【iOS】内存管理初级](https://www.tommywutong.cn/blog/csdn-import/csdn-152130856-ios-/)

受限于当时的知识面，这两篇文章对很多概念的理解较浅，部分表述也不够严谨。现在重新梳理 iOS 底层知识，希望从操作系统的虚拟内存出发，把“变量在哪里”“对象在哪里”“堆和栈是什么”“Mach-O 的 Segment 又是什么”“系统如何统计和回收内存”这些容易混在一起的问题逐层分开。

## 本文主线

这篇文章会同时讨论地址空间和内存占用，但二者不能混在同一个层级中。全文按照下面的顺序展开：

```text
操作系统为什么需要虚拟内存
        ↓
一个 iOS App 获得怎样的虚拟地址空间
        ↓
用“五大分区”建立一张用途地图
        ↓
Mach-O、堆和线程栈分别怎样形成这些区域
        ↓
用 VM Region、内容来源和访问权限回到真实系统
        ↓
用代码和 LLDB 验证变量、指针与对象的位置
        ↓
从“地址在哪里”过渡到“实际占用多少”
        ↓
理解 Clean、Dirty、Compressed 与 Memory Footprint
        ↓
理解内存警告、压缩与 Jetsam
```

这里涉及三个不同的观察层级：

| 观察层级 | 主要回答的问题 | 典型概念 |
| --- | --- | --- |
| 进程地址空间 | 这段地址用来做什么 | VM Region、代码、数据、堆、栈 |
| 可执行文件与运行时 | 这段内容从哪里来 | Mach-O、`malloc`、线程栈 |
| 物理页面与系统记账 | 它现在是否占用宝贵的内存资源 | Clean、Dirty、Compressed、Memory Footprint |

同一份数据可以同时被这三个层级描述。例如，一个全局变量在教学模型中属于全局/静态区，在 Mach-O 中可能来自 `__DATA` 的某个 Section，运行后所在页面还可能被记为 clean 或 dirty。它们不是互相竞争的三种答案，而是从不同角度描述同一件事。

---

## 第一部分：从操作系统到 iOS 地址空间

### 操作系统中的内存

在没有内存保护和地址转换机制的环境中，程序直接使用物理地址。多个程序如果使用了相同的地址，就可能互相覆盖数据。虽然系统也可以依靠人工规划固定的物理地址来运行多个程序，但这种方式难以做到可靠隔离、灵活装载和高效共享。

现代操作系统因此让进程主要使用虚拟地址，并由硬件和内核共同完成从虚拟地址到物理页的映射。不同进程中的同一个虚拟地址可以映射到不同的物理页；内核也可以通过权限控制，阻止进程访问不属于自己的映射。


- **虚拟内存（Virtual Memory）**：操作系统提供给进程的一层地址空间抽象。一个 iOS App 看到的是自己独立的虚拟地址空间，其中包含多个离散的虚拟内存区域（VM Region）。虚拟地址空间很大，但并不意味着其中每个地址都有效，也不意味着所有已映射页面都同时驻留在物理内存中。
![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260724210538582.png)

- **物理内存（Physical Memory）**：设备真实存在、可由 CPU 访问的 RAM。进程访问虚拟地址时，CPU 的 MMU 会依据页表把它转换为对应的物理地址。

操作系统教材通常通过“分段”和“分页”两条路线介绍地址空间管理。对现代 arm64 iOS 来说，后文真正需要继续使用的是分页、页表、权限和 VM Region；分段主要帮助理解历史模型和逻辑区域。

#### 分段

![分段示意图](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260724154844318.png)

分段（Segmentation）是一种经典的内存管理思想：按照代码、数据、栈等逻辑单元描述地址空间，各段长度可以不同。它有助于理解“程序可以由不同用途、不同权限的区域组成”。

但这里必须区分三个名字相似、含义不同的概念：

- 操作系统教材中的 **Segmentation** 是一种地址转换和内存管理模型。
- Mach-O 中的 `__TEXT`、`__DATA` 等 **Segment** 是可执行文件及其装载映射的组织方式。
- 堆和栈是进程运行时使用的虚拟内存区域，不是“编译器创建的 CPU 分段”。

现代 arm64 iOS 的内存管理重点是分页和 Mach VM。后文讨论 Mach-O 时，还需要进一步观察 `__TEXT`、`__DATA` 等文件布局如何被映射为进程中的 VM Region。

#### 分页

![分页示意图](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260724154856696.png)

为了方便映射和管理，虚拟内存和物理内存都被分割成相同大小的单位，物理内存的最小单位被称为帧（Frame），而虚拟内存的最小单位被称为页（Page）。
Apple 当前文档指出，iOS 中典型的页大小是 16 KB；具体值仍应以设备和运行环境为准，可以通过 `vm_page_size`、`getpagesize()` 或 Jetsam Event Report 中的 `pageSize` 字段确认，而不应在程序逻辑中写死。

- **页表（Page Table）**：记录虚拟页到物理页的映射及读、写、执行等权限。代码访问虚拟地址时，CPU 内部的 **MMU（Memory Management Unit）** 会依据页表完成地址转换。
- **虚拟内存区域（VM Region）**：一段具有相同属性的连续虚拟地址范围。一个进程拥有许多 VM Region，但整个虚拟地址空间并非从头到尾连续有效。

#### 按需分页与 Page Fault（缺页中断）

某个虚拟地址落在进程已经建立的合法 VM Region 中，不等于对应页面已经驻留在物理 RAM。系统可以先建立地址范围和映射关系，等程序真正访问某一页时，再提供零填充页、从 Mach-O 或其他映射文件取得内容，或者完成 Copy-on-Write，然后继续执行原来的指令。

CPU 发现现有页表不能直接完成访问时，会触发 Page Fault 并交给内核处理。能够由内核补全映射的 Page Fault 是按需分页的正常机制，并不等同于崩溃；只有地址无效或访问权限冲突且内核无法修复时，才可能最终表现为 `EXC_BAD_ACCESS`。`EXC_BAD_ACCESS` 是非法内存访问的结果之一，不能简单等同于“野指针”。

这里先把 Page Fault 当作一个连接点：它解释了为什么“已经分配或映射虚拟地址”仍不代表“对应物理页已经到位”。在第二部分讨论 `malloc`、首次写入和 Memory Footprint 时还会再次用到这个结论。Apple 的 [About the Virtual Memory System](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/AboutMemory.html) 对 soft fault、hard fault 和 page-in 有进一步说明。

关于分页、页表和地址转换的更多细节，可以继续阅读：[小林 Coding：为什么要有虚拟内存？](https://www.xiaolincoding.com/os/3_memory/vmem.html)。本文暂时停在能够支撑后续 iOS 地址空间分析的程度。


### 从操作系统过渡到 iOS App

一个运行中的 iOS App，本质上就是一个进程。系统会为它提供独立的虚拟地址空间。前文讨论的分页、页表和 VM Region，在这里不再只是操作系统教材中的抽象概念，而是 App 代码实际运行的环境。

```mermaid
flowchart TB
    APP["一个正在运行的 iOS App<br/>拥有自己的虚拟地址空间"]

    APP --> MACHO["启动时映射进来的内容<br/>程序代码、全局变量、静态变量"]
    APP --> HEAP["运行过程中动态申请的内存<br/>堆 Heap"]
    APP --> STACK["函数调用时使用的临时空间<br/>线程栈 Stack"]
```

这里先建立一个边界：常说的“代码区、数据区、堆区、栈区”是一张便于入门的教学地图，并不是 Darwin/iOS 对全部内存区域的完整分类。暂时不展开动态库、共享缓存、匿名映射和访问权限，先用五大分区回答“不同内容大概用哪类空间”，等理解 Mach-O、堆和线程栈之后，再统一落到真实的 VM Region。

### “五大分区”

我们常说的内存“五大分区”，可以先整理为下面这张表格：

| 教学分区   | 主要内容          | 在真实 iOS 中继续追问                   |
| ------ | ------------- | ------------------------------- |
| 代码区    | 编译后的机器指令、函数代码 | 通常来自 Mach-O 的可执行、只读映射           |
| 常量区    | 字符串字面量、部分只读常量 | 可能位于 Mach-O 的只读 Section         |
| 全局/静态区 | 全局变量、静态变量     | 根据是否初始化、是否可写，进入不同的数据 Section    |
| 堆区     | 运行时动态申请的数据    | 通常由 `malloc` 等分配器管理多个 VM Region |
| 栈区     | 函数调用期间的临时状态   | 每个线程拥有自己的线程栈                    |

这张表先解决“源代码里的东西大概放在哪里”。接下来再分别追问：前三类如何由 Mach-O 带入进程，堆如何在运行时增长，栈为什么属于线程。

下面用贯穿全文的实验代码把五类内容放在一起：

```objc
int globalInitialized = 11;
int globalZeroInitialized;
static int staticInitialized = 22;
static NSString * const globalStringLiteral = @"global literal";

__attribute__((noinline, optnone))
static void RunMemoryExperiment(void) {
    volatile int localNumber = 33;
    NSObject *object = [[NSObject alloc] init];
    NSString *localLiteral = @"local literal";

    volatile int runtimeValue = 42;
    NSNumber *taggedNumber = @(runtimeValue);

    void *heapBuffer = malloc(32 * 1024);
    memset(heapBuffer, 0x5A, 1);

    // 在这里暂停并通过 LLDB 观察

    free(heapBuffer);
}
```

先用五大分区回答，再保留必要的边界：

| 代码中的内容 | 教学模型中的位置 | 更准确的说明 |
| --- | --- | --- |
| `RunMemoryExperiment` 的机器指令 | 代码区 | 本次构建位于 Mach-O 的 `__TEXT,__text`，运行时区域权限为 `r-x` |
| `@"global literal"`、`@"local literal"` | 常量区 | 本次构建中的字符串对象位于只读的 Mach-O 映射，运行时权限为 `r--` |
| `globalInitialized`、`staticInitialized` | 全局/静态区 | 本次构建位于 `__DATA,__data` |
| `globalZeroInitialized` | 全局/静态区 | 本次构建位于零填充的 `__DATA,__common`；不同工具链也可能显示为 `__bss` 等零填充 Section |
| `localNumber` | 栈区 | Debug、`-O0` 下观察到它位于当前线程栈；优化后可能进入寄存器或被消除 |
| 局部变量 `object` 本身 | 栈区 | 它是保存对象地址的局部强引用；优化后位置也可能变化 |
| `[[NSObject alloc] init]` 创建的普通对象 | 堆区 | 本次运行中对象地址落在可读写的分配区域 |
| 局部变量 `heapBuffer` 本身 | 栈区 | 它只保存 `malloc` 返回的地址 |
| `malloc(32 * 1024)` 返回的缓冲区 | 堆区 | 分配器管理的可读写区域；只写入第一个字节不代表 32 KB 每一页都已被触碰 |
| `taggedNumber` 指向的值 | 不对应普通堆对象 | 本次运行得到 Tagged Pointer，其值编码在指针中，不能把它当作普通对象地址查询 VM Region |

“五大分区”在这里完成了第一轮分类，但它没有说明 Mach-O、VM Region、ASLR、页面是否驻留以及 Clean/Dirty。后面的内容就是在不推翻这张教学地图的前提下，逐步补上真实系统的信息。

### Mach-O：解释启动时映射进来的代码和数据

Mach-O 不是“五大分区”之外的第六个分区。它是 iOS 可执行文件的组织格式，用来解释 App 启动前代码和全局数据保存在什么地方，以及启动后如何被映射进虚拟地址空间。

| 五大分区中的说法 | Mach-O 与虚拟内存视角                               |
| -------- | -------------------------------------------- |
| 代码区      | 机器指令通常来自 `__TEXT` Segment 中的相关 Section       |
| 常量区      | 字符串和部分只读常量通常来自只读 Section                     |
| 全局/静态区   | 已初始化数据、零填充数据等来自 `__DATA` 及相关 Segment/Section |

Mach-O 的 Segment 描述可执行文件及装载映射的组织方式；操作系统教材中的 Segmentation 描述的是一种地址管理模型。二者名字相似，但不能混为一谈。

#### Segment 与 Section

Mach-O 使用两层结构组织内容：

- **Segment** 是装载和权限管理的大单位，例如 `__TEXT`、`__DATA_CONST`、`__DATA`。
- **Section** 位于 Segment 内部，用于继续区分具体内容，例如 `__text`、`__cstring`、`__data`、`__common`。

本次实验二进制的关键布局如下：

| Segment / Section | 实验中的内容 | 典型运行权限 |
| --- | --- | --- |
| `__TEXT,__text` | `RunMemoryExperiment` 的机器指令 | `r-x` |
| `__TEXT,__cstring` | C 字符串等字面量数据 | 随 `__TEXT` 映射；内容不可写，Region 可能因同一 Segment 包含代码而显示 `r-x` |
| `__DATA_CONST,__cfstring` | Objective-C 字符串常量对象 | `r--` |
| `__DATA_CONST,__const` | 指向全局字符串对象的常量指针等 | `r--` |
| `__DATA,__data` | 已初始化的可写全局、静态变量 | `rw-` |
| `__DATA,__common` | 本次构建中的未初始化外部全局变量，装载时零填充 | `rw-` |

这里还有两个容易忽略的区域：

- `__PAGEZERO` 在 64 位 Mach-O 中保留低地址范围，不映射为可访问内存，有助于让空指针附近的访问尽早失败。
- `__LINKEDIT` 保存符号、字符串表、重定位等链接信息，服务于装载、符号解析和调试，但它不属于“五大分区”里的业务数据。

Mach-O 中记录的是链接时虚拟地址、大小和权限等装载信息。启动时，内核和 dyld 根据这些信息建立 VM Region，并继续映射依赖的 Framework 与 dyld shared cache。最终进程地址空间远大于主程序自己的 Mach-O。

#### ASLR：为什么每次运行的地址可能不同

为了避免二进制每次都出现在固定地址，系统会在加载时引入随机的 **ASLR Slide**。可以先记住下面的关系：

```text
运行时地址 = Mach-O 中的链接地址 + ASLR Slide
```

在最终实验二进制中，`RunMemoryExperiment` 的链接地址是 `0x10000103c`。同一份二进制连续启动两次，记录到的运行时地址分别为：

```text
第一次：0x10472d03c
第二次：0x102b9503c
```

函数没有“搬到另一个 Section”，改变的是整份映像的加载基址。比较地址时应该先判断它属于哪个映像，并结合 `image list`、`image lookup` 或 `vmmap` 获取加载地址，不能把不同运行中的绝对地址直接比较。Apple 在 WWDC21 [Symbolication: Beyond the basics](https://developer.apple.com/videos/play/wwdc2021/10211/) 中完整演示了 Linker Address、Load Address 与 ASLR Slide 的关系。

这一篇只需要掌握以上桥梁。Mach Header、Load Commands 的字段、dyld Rebase/Bind、符号解析和共享缓存内部结构，继续放在本地 [[20 专题笔记/编译链接与启动/Mach-O|Mach-O]] 专题中学习。

### 堆与线程栈：解释运行过程中出现的区域

Mach-O 主要解释启动时已经存在的代码和数据。App 开始运行后，动态申请的数据通常进入由内存分配器管理的堆区域；函数调用则会使用当前线程自己的栈。

- **堆**：不是一整块固定且永远连续的区域。`malloc` 等分配器会管理一个或多个虚拟内存区域，并把合适的空间交给调用方。
- **线程栈**：每个线程都有自己的栈，用来支撑函数调用和临时状态。多个线程共享进程的代码、全局数据和堆，但不共享同一个线程栈。
- **局部变量**：在未优化的直觉模型中经常位于栈帧；经过编译优化后，也可能只存在于寄存器中。

### 指针变量和对象本体分别在哪里

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

在本次 `-O0` 实验中：

```text
object  = 0x60000001c020    // 普通 NSObject 对象所在的分配区域
&object = 0x16c24ad30       // 后台实验线程的栈区域
```

这正好证明“指针变量在栈上”和“对象在堆上”可以同时成立。但这句话仍有三个边界：

1. **编译器优化**：局部变量可能只存在于寄存器中，或者被完全消除，因此“局部变量一定在栈上”不严谨。
2. **字符串字面量**：`@"local literal"` 在本次实验中落在主程序的只读映射内，并不是函数每执行一次就在堆上创建一个新字符串对象。
3. **Tagged Pointer**：运行时值 `@(runtimeValue)` 在本次实验中得到 `0x8a8f68c2cc3e21bc`。使用 VM Region 查询该值失败，因为它不是指向普通已映射对象内存的地址，而是把值及类型信息编码在指针位中。Tagged Pointer 的具体编码属于 Runtime 实现细节，不应依赖某一位模式编写业务逻辑。

因此，面试中回答“对象在哪里”时，至少需要先确认讨论的是普通动态对象、字面量对象，还是 Tagged Pointer；回答“变量在哪里”时，还要区分变量本身和变量保存的值。

### 用 VM Region 回到真实 iOS

到这里已经知道：

- 代码、常量和部分全局数据由 Mach-O 带入进程；
- 堆在运行过程中承接动态分配；
- 每个线程拥有自己的栈；
- 指针变量和对象本体可能位于不同区域。

现在再引入 VM Region，就不会把它误认为“五大分区之外的第六个分区”。**五大分区按照用途分类，VM Region 则是内核描述一段连续虚拟地址范围的实际方式。**一个 Region 具有起止地址、访问权限、内容来源等属性；同一教学分区可能由多个 Region 组成，一个 Mach-O Segment 在实际映射和保护过程中也不能简单等同于整张进程内存地图。

#### 两种常见来源：文件映射与匿名内存

从“页面内容可以去哪里重新找”这一角度，VM Region 可以先粗分为两类：

- **文件映射（File-backed Mapping）**：内容来自 Mach-O、Framework、dyld shared cache 或通过 `mmap` 映射的文件。未被修改的页面可以丢弃，需要时再从原文件或系统缓存取得，因此通常更容易保持为 clean。
- **匿名内存（Anonymous Memory）**：没有一个可直接重新读取的原始文件，常见于堆、线程栈以及运行时申请的数据。程序真正写入后，相关页面通常成为 dirty；iOS 不能依赖传统磁盘 swap 为不断增长的匿名 dirty memory 兜底。

这不是说“文件映射永远 clean、匿名内存永远 dirty”。Mach-O 中的可写数据可能通过 Copy-on-Write 形成进程私有页面，文件映射也可能被修改；新申请的匿名地址也可能尚未真正触碰。这里的分类是为了回答内容来源，并为后文的 Clean、Dirty 与 Memory Footprint 建立桥梁。

#### VM Region 的访问权限

调试工具经常用三个字母表示 Region 当前允许的操作：

| 权限 | 含义 | 常见例子 |
| --- | --- | --- |
| `r--` | 可读，不可写，不可执行 | 字符串对象和其他只读数据映射 |
| `rw-` | 可读、可写，不可执行 | 全局可写数据、堆、线程栈 |
| `r-x` | 可读、不可写、可执行 | Mach-O 中的机器指令 |
| `---` | 当前不可读写执行 | `__PAGEZERO`、Stack Guard 等保护区域 |

权限让同一进程中的区域承担不同职责，也贯彻了“数据不可随意执行、代码不可随意写入”的安全边界。Apple 在 [Investigating memory access crashes](https://developer.apple.com/documentation/xcode/investigating-memory-access-crashes) 中展示了 `r-x` 的 `__TEXT`、`rw-` 的线程栈以及无权限的 Stack Guard。

把五大分区、来源和权限放到一起，可以得到更接近真实系统的关系：

| 教学用途 | 常见形成方式 | 常见权限 | 后续页面状态线索 |
| --- | --- | --- | --- |
| 代码 | Mach-O / Framework 文件映射 | `r-x` | 未修改时通常可从文件重新取得 |
| 常量 | Mach-O 只读数据映射 | `r--`，或随 `__TEXT` 显示为 `r-x` | 通常更容易保持 clean |
| 全局/静态数据 | Mach-O 数据映射、零填充页、Copy-on-Write | `rw-` | 被写入后可能贡献 dirty memory |
| 堆 | 分配器管理的匿名内存 | `rw-` | 实际写入的页面通常贡献 dirty memory |
| 线程栈 | 每个线程各自的匿名 Region | `rw-`，边界附近可有 `---` Guard | 被使用的栈页面属于进程需要负责的内存 |

这一步完成了文章前半篇最重要的转换：

```text
五大分区告诉我们“用来做什么”
        ↓
Mach-O、堆和线程栈告诉我们“怎样形成”
        ↓
VM Region 告诉我们“系统怎样描述”
        ↓
文件/匿名来源和页面状态继续解释“系统怎样回收与记账”
```

### 用 LLDB 与 VM API 验证地址分布

本次记录的实验环境为：

| 项目 | 环境 |
| --- | --- |
| Xcode | 26.6 |
| 运行环境 | iPhone 16 Pro Simulator，iOS 18.4，Apple Silicon Mac |
| 构建方式 | Objective-C、Debug 信息、`-O0`，关键函数额外使用 `optnone` |
| 页面大小 | `vm_page_size = 16384`，即 16 KB |
| 验证方式 | 程序日志记录地址，并使用 `vm_region_64` 查询 Region 范围与权限；在 Xcode 中可用下列 LLDB 命令交叉验证 |

Simulator 与真机不完全相同，尤其是系统共享缓存、分配器实现、地址编码和内存压力行为。下面的结果用于验证“地址属于哪类 Region”和“权限有什么差异”，不能用来推导所有 iPhone 的固定地址。

当程序停在 `RunMemoryExperiment` 内部时，可以执行：

```lldb
p/x &globalInitialized
p/x &globalZeroInitialized
p/x &staticInitialized
p/x &localNumber
p/x object
p/x &object
p/x localLiteral
p/x taggedNumber
p/x heapBuffer

image list
image lookup --address 0x10472d03c
memory region 0x10472d03c
image dump sections MemoryMapExperiment
```

其中：

- `p/x expression` 以十六进制打印变量值；
- `&variable` 查看变量本身的地址；
- `image list` 查看主程序和依赖映像的加载地址；
- `image lookup --address` 把运行时地址定位到映像、Segment/Section、符号和源码；
- `memory region address` 查看该地址所在 VM Region 的范围和权限；
- `image dump sections` 查看 LLDB 识别到的 Mach-O Section。

本次最终运行的关键结果如下：

| 观察对象 | 实际地址 | Region 与权限 | 解释 |
| --- | --- | --- | --- |
| `RunMemoryExperiment` | `0x10472d03c` | `0x10472c000–0x104730000 r-x` | 主程序机器指令 |
| 全局字符串字面量 | `0x104730140` | `0x104730000–0x104734000 r--` | 主程序只读数据映射 |
| `globalInitialized` | `0x104734740` | `0x104734000–0x104738000 rw-` | 已初始化的全局数据 |
| `globalZeroInitialized` | `0x10473480c` | 同一 `rw-` Region | 零填充全局数据 |
| `staticInitialized` | `0x104734808` | 同一 `rw-` Region | 已初始化静态数据 |
| 主线程局部变量 | `0x16b6cf7dc` | `0x16aed8000–0x16b6d4000 rw-` | 主线程自己的栈区域 |
| 后台线程 `localNumber` | `0x16c24ad3c` | `0x16c1c8000–0x16c250000 rw-` | 另一个线程的栈区域 |
| 局部指针变量 `&object` | `0x16c24ad30` | 与后台线程局部变量相同 | 指针变量本身位于当前线程栈 |
| 普通对象 `object` | `0x60000001c020` | 可读写分配区域 | 普通 Objective-C 对象本体 |
| `heapBuffer` | `0x10880d400` | `0x108800000–0x109000000 rw-` | `malloc` 管理的区域 |
| `taggedNumber` | `0x8a8f68c2cc3e21bc` | VM Region 查询失败 | 值编码在 Tagged Pointer 中，不是普通对象地址 |

最有价值的不是记忆这些十六进制数，而是观察到：

- 代码、只读常量、可写全局数据具有不同权限；
- 主线程与后台线程的局部变量落在不同栈 Region；
- `object` 和 `&object` 位于完全不同的地址范围；
- Tagged Pointer 不能按照普通堆对象理解；
- 再次运行时绝对地址会因 ASLR 改变。

#### 从单个地址扩展到整张内存地图

LLDB 适合追踪“这个地址是什么”，`vmmap` 更适合回答“整个进程有哪些 Region”。对运行中的调试进程或导出的 Memgraph，可以从下面的命令开始：

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

```mermaid
flowchart TB
    APP["MemoryMapExperiment 进程的虚拟地址空间"]

    APP --> TEXT["主程序代码映射<br/>Mach-O __TEXT / __text<br/>r-x"]
    APP --> CONST["主程序只读数据<br/>字符串、__DATA_CONST 等<br/>r--"]
    APP --> DATA["主程序可写数据<br/>全局变量、静态变量<br/>rw-"]
    APP --> HEAP["动态分配区域<br/>普通对象、malloc buffer<br/>rw-"]
    APP --> MAINSTACK["主线程栈<br/>局部状态<br/>rw-"]
    APP --> WORKERSTACK["后台线程栈<br/>局部状态<br/>rw-"]
    APP --> IMAGES["Framework 与 dyld shared cache<br/>代码和数据映射"]
    APP --> OTHER["其他 VM Region<br/>匿名映射、mmap 文件、Guard 等"]
```

这张图刻意没有画成五个连续盒子，也没有固定地址高低关系。它表达的是“一个进程包含多种用途的 VM Region”；完整数量和顺序应以当次运行的 `vmmap`、Memory Report 或 Memgraph 为准。

---

## 第二部分：从“在哪里”到“实际占多少”

前半篇已经从五大分区走到真实 VM Region，并区分了文件映射和匿名内存。但“申请了多大的虚拟地址范围”和“App 当前为多少物理内存负责”仍不是同一个问题。下面开始把观察单位从变量和 Region 切换为内存页，继续回答这些页面能否重新取得、是否已被写入，以及系统怎样计算 Memory Footprint。

### 申请内存不等于立刻产生等量的 Memory Footprint

Apple 在 WWDC18《iOS Memory Deep Dive》中以典型的 16 KB 页面说明：内存系统以页为粒度管理和统计资源，一个页面可以容纳多个堆对象，一个对象也可能跨越多个页面。因此，哪怕程序只修改其中一个字节，受影响的仍可能是整个页面。

这里不能把“内存使用量”笼统地理解为所有虚拟地址范围。虚拟地址空间大小、当前驻留的物理页、堆分配量和系统用于限制 App 的 Memory Footprint 是不同指标。后文所说的 footprint，重点关注 App 需要系统保留的 dirty 与 compressed 页面，而 clean 页面可以从原始来源重新建立，因此记账方式不同。

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260724211423574.png)

以 `malloc` 为例，申请一段空间与真正触碰其中的每一个页面并不是同一个动作。分配器可能先为程序准备可用的虚拟地址；当程序首次访问相应页面时，系统才通过前文提到的按需分页机制建立页面支持。实际变化还会受到页面是否已经存在、写入是否落在同一页以及分配器元数据等因素影响，所以不能机械地推导出“每写一个字节，内存就一定增加 16 KB”。

当 App 开始写入页面时，相关页面可能从 clean 变为 dirty。下面这张图描述的是“已分配”和“已写入”在页面状态上的区别，而不是五大分区中的新区域。

![Snapzy_2026-07-24_21-14-54_871.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/Snapzy_2026-07-24_21-14-54_871.png)

### Clean、Dirty 与 Compressed

- **Clean memory**：内容可以从原始来源重新构造或重新读取的页面。例如，未被修改的 Mach-O、Framework 和其他文件映射页面通常可以在需要时丢弃。Clean page 当前也可能驻留并占用 RAM；它的关键特点是可重新建立，而不是“永远不占物理内存”。
- **Dirty memory**：内容已经被进程修改，不能直接丢弃并靠原文件恢复的页面。堆上的对象、解码后的图像缓冲区等通常会贡献 dirty memory，但内存按页记账，分配与实际写入仍要分开观察。
- **Compressed memory**：系统压缩后暂时保留的页面。它减少当前物理占用，但数据没有被释放，仍应纳入 App footprint 的分析。

新获得的页面可能最初仍是 clean；当程序真正写入页面时，页面才会变 dirty。非可丢弃的 dirty page 不能像 clean 文件页那样直接回收，但系统可以压缩它，或者在压力无法缓解时终止相关进程。

### iOS 内存压力与回收

#### iOS 没有传统意义上的磁盘 Swap

桌面操作系统通常可以把一部分匿名内存换出到磁盘。Apple 的 iOS 虚拟内存资料和 WWDC18《iOS Memory Deep Dive》强调：iOS 不把普通 App 的 dirty page 当作传统磁盘 swap 的后备存储来使用。

这不等于“iOS 的虚拟内存从不读取存储设备”。App 的可执行文件、动态库和内存映射文件本来就可以按需 page in；clean 的文件映射页被丢弃后，也可以在需要时重新读取。对于 App 开发，关键区别是：不能期待系统把不断增长的匿名 dirty memory 像桌面 swap 一样长期换出，从而替应用兜底。

系统实现会随版本演进，因此比起把原因简单归结为“保护闪存寿命”，更稳妥的结论是：iOS 的内存策略优先考虑移动设备的性能、能耗和系统响应，并通过回收可重建页面、内存压缩以及必要时终止进程来控制压力。

#### 内存压缩

当物理内存紧张时，系统可以压缩近期不活跃的页面，以减少它们占用的物理空间；再次访问时再进行解压。压缩和解压会消耗 CPU，说明内存压力也可能转化为性能与能耗开销。

需要注意：被压缩的内存并没有从 App 的内存责任中消失。理解和分析 footprint 时，应同时关注 dirty memory 与 compressed memory，不能把“已压缩”等同于“已释放”。

#### Memory Warning 与 Jetsam

系统处于内存压力时，UIKit 可以通过 App Delegate、View Controller 或通知等途径向 App 传递低内存警告。这个警告反映的是系统级压力，不一定仅由当前 App 导致，也不能保证每次被终止前都严格经历同一条回调链。

如果压力持续、进程超过相应内存限制，或系统需要迅速回收资源，系统可能终止一个或多个进程。开发者随后可能看到 **Jetsam Event Report**。它不同于由异常或信号产生的常规 crash report，因此把这种终止简单写成“Jetsam 先发送 `didReceiveMemoryWarning`，App 不处理后再杀死”并不准确。

循环引用会让对象生命周期超出预期，持续占用它们关联的内存；这属于内存泄漏和对象所有权问题。解码后的大图也可能快速抬高 dirty memory 峰值。二者都值得优化，但不能由此推出“所有 Objective-C 对象都一直占据物理内存，直到手动释放”。

### 回到“五大分区”：把三个观察层级重新合并

完成实验后，可以更准确地评价“五大分区”：

- 它解决的是**用途分类**问题，适合快速判断函数代码、全局变量、动态对象和局部状态大概属于哪一类。
- 它遗漏了 Framework、dyld shared cache、`mmap` 文件、Guard Page、匿名映射和图形相关映射等真实 VM Region。
- 一个教学分区不一定只对应一个连续 Region。例如堆可以由多个分配区域组成，每个线程也都有自己的栈。
- Mach-O Segment 描述的是可执行文件及其装载布局；堆和线程栈是运行时形成的区域；Clean、Dirty、Compressed 描述的是页面状态。三者回答不同问题。
- 仅凭“对象在堆上”无法推出它当前占用多少物理内存，更无法直接推出 App 的 Memory Footprint。

最终可以选择几个具体对象，从三个层级同时描述：

| 例子 | 地址空间用途 | 来源或形成方式 | 页面状态与 footprint |
| --- | --- | --- | --- |
| 函数机器指令 | 代码区域 | 从 Mach-O 的只读、可执行映射而来 | 通常是可重新加载的 clean 文件页 |
| 可写全局变量 | 全局/静态区域 | 从 Mach-O 数据 Section 映射而来 | 写入后相关页面可能变为 dirty |
| 普通 Objective-C 对象 | 通常位于堆管理的区域 | 运行时通过分配器创建 | 被实际写入的相关页面通常贡献 dirty memory |
| 函数局部状态 | 当前线程的栈或寄存器 | 随函数调用建立和销毁 | 被写入的栈页面属于 App 需要负责的页面 |

这张表才是全文真正的终点：先判断“在哪里”，再追问“从哪里来”，最后分析“系统怎样为它记账”。  

### 总结

到这里可以得出八个结论：

1. 进程使用的是由多个 VM Region 组成的虚拟地址空间，虚拟地址并不等于物理地址。
2. 合法地址对应的页面尚未驻留也可能触发 Page Fault；内核能够修复的 Page Fault 是按需分页的正常过程，不等于崩溃。
3. 文件映射和匿名内存说明页面内容从哪里来，`r--`、`rw-`、`r-x` 等权限说明程序能对该 Region 做什么。
4. “五大分区”是一张用途地图，适合建立直觉，但不是 Darwin/iOS 对所有 VM Region 的完整分类。
5. Mach-O 解释启动时的代码和数据如何进入地址空间；堆和线程栈则解释运行过程中出现的动态区域；ASLR 会改变映像每次运行的加载地址。
6. 回答“变量和对象在哪里”时，必须分别讨论变量本身、变量保存的地址、对象本体以及字面量或 Tagged Pointer 等例外。
7. 虚拟地址范围、堆分配量、驻留物理页和 Memory Footprint 是不同指标，不能只看到一个数字就互相替代。
8. Clean、Dirty、Compressed 描述页面的可重建性和系统记账状态；内存压缩、警告与 Jetsam 则描述系统在压力下采取的策略。


## 参考资料

### 官方资料

- [Apple — About the Virtual Memory System](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/AboutMemory.html)
- [Apple — Overview of the Mach-O Executable Format](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/CodeFootprint/Articles/MachOOverview.html)
- [Apple — Viewing Virtual Memory Usage](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/VMPages.html)
- [Apple — Investigating memory access crashes](https://developer.apple.com/documentation/xcode/investigating-memory-access-crashes)
- [Apple — Gathering information about memory use](https://developer.apple.com/documentation/xcode/gathering-information-about-memory-use)
- [Apple — Reducing your app's memory use](https://developer.apple.com/documentation/xcode/reducing-your-app-s-memory-use)
- [Apple — Responding to memory warnings](https://developer.apple.com/documentation/uikit/responding-to-memory-warnings)
- [Apple — Identifying high-memory use with Jetsam Event Reports](https://developer.apple.com/documentation/xcode/identifying-high-memory-use-with-jetsam-event-reports)
- [Apple — Reduce terminations in your app](https://developer.apple.com/documentation/xcode/reduce-terminations-in-your-app)
- [Apple — Reducing your app's launch time](https://developer.apple.com/documentation/xcode/reducing-your-app-s-launch-time)
- [WWDC18 — iOS Memory Deep Dive](https://developer.apple.com/videos/play/wwdc2018/416/)
- [WWDC21 — Symbolication: Beyond the basics](https://developer.apple.com/videos/play/wwdc2021/10211/)
- [Apple Kernel Programming Guide — Memory and Virtual Memory](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/vm/vm.html)
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
- https://juejin.cn/post/6844903902169710600?searchId=202607242005413F8E66D396B122E9CEF3
- 本地：[[20 专题笔记/编译链接与启动/Mach-O|Mach-O]]
