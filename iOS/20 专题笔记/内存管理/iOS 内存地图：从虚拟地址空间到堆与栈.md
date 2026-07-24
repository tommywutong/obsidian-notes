---
title: "【iOS】内存地图：从虚拟地址空间到堆与栈"
published: 2026-07-24
description: "从操作系统虚拟内存出发，结合 Mach VM、Mach-O 与 LLDB 实验，梳理 iOS App 中变量、对象、堆、栈和映射区域之间的关系。"
tags: ["iOS", "Operating System", "Virtual Memory", "Memory", "Mach-O", "LLDB"]
category: "iOS"
series: "2026 暑假 iOS 底层学习"
seriesSlug: "ios-internals-2026-summer"
seriesOrder: 1
draft: true
---
# iOS 内存地图：从虚拟地址空间到堆与栈

## 前言

在之前的文章中，笔者完成过两篇内存管理的入门文章：

- [【iOS】内存五大分区](https://www.tommywutong.cn/blog/csdn-import/csdn-154609757-ios-/)
- [【iOS】内存管理初级](https://www.tommywutong.cn/blog/csdn-import/csdn-152130856-ios-/)

受限于当时的知识面，这两篇文章对很多概念的理解较浅，部分表述也不够严谨。现在重新梳理 iOS 底层知识，希望从操作系统的虚拟内存出发，把“变量在哪里”“对象在哪里”“堆和栈是什么”“Mach-O 的 Segment 又是什么”这些容易混在一起的问题逐层分开。

## 从一道代码题开始

> [!todo]
> 放入一段最小的 Objective-C 代码，包含局部变量、全局变量、静态变量、字符串字面量和通过 `alloc/init` 创建的对象。先记录自己的判断：指针变量在哪里，对象本体又在哪里？

这道题不能只靠“内存五大分区”作答。要解释清楚它，首先必须分清三个层级：

1. **源代码层**：变量是什么类型、具有怎样的存储期和所有权关系。
2. **进程虚拟地址空间层**：地址落在哪个虚拟内存区域，具有什么保护权限。
3. **物理内存层**：对应的页当前是否驻留在 RAM 中，是 clean、dirty，还是已经被压缩。

下面先从第二、第三层开始。

## 虚拟内存

### 为什么需要地址转换与隔离

在没有内存保护和地址转换机制的环境中，程序直接使用物理地址。多个程序如果使用了相同的地址，就可能互相覆盖数据。虽然系统也可以依靠人工规划固定的物理地址来运行多个程序，但这种方式难以做到可靠隔离、灵活装载和高效共享。

现代操作系统因此让进程主要使用虚拟地址，并由硬件和内核共同完成从虚拟地址到物理页的映射。不同进程中的同一个虚拟地址可以映射到不同的物理页；内核也可以通过权限控制，阻止进程访问不属于自己的映射。

- **虚拟内存（Virtual Memory）**：操作系统提供给进程的一层地址空间抽象。一个 iOS App 看到的是自己独立的虚拟地址空间，其中包含多个离散的虚拟内存区域（VM Region）。虚拟地址空间很大，但并不意味着其中每个地址都有效，也不意味着所有已映射页面都同时驻留在物理内存中。
- **物理内存（Physical Memory）**：设备真实存在、可由 CPU 访问的 RAM。进程访问虚拟地址时，CPU 的 MMU 会依据页表把它转换为对应的物理地址。

### 教材中的分段

![分段示意图](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260724154844318.png)

分段（Segmentation）是一种经典的内存管理思想：按照代码、数据、栈等逻辑单元描述地址空间，各段长度可以不同。它有助于理解“程序可以由不同用途、不同权限的区域组成”。

但这里必须区分三个名字相似、含义不同的概念：

- 操作系统教材中的 **Segmentation** 是一种地址转换和内存管理模型。
- Mach-O 中的 `__TEXT`、`__DATA` 等 **Segment** 是可执行文件及其装载映射的组织方式。
- 堆和栈是进程运行时使用的虚拟内存区域，不是“编译器创建的 CPU 分段”。

现代 arm64 iOS 的内存管理重点是分页和 Mach VM。后文讨论 Mach-O 时，还需要进一步观察 `__TEXT`、`__DATA` 等文件布局如何被映射为进程中的 VM Region。

### 现代 iOS 的重点：分页

![分页示意图](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260724154856696.png)

分页（Paging）把虚拟地址空间和物理内存都按固定大小的页进行管理。Apple 当前文档指出，iOS 中典型的页大小是 16 KB；具体值仍应以设备和运行环境为准，可以通过 `vm_page_size`、`getpagesize()` 或 Jetsam Event Report 中的 `pageSize` 字段确认，而不应在程序逻辑中写死。

- **页表（Page Table）**：记录虚拟页到物理页的映射及读、写、执行等权限。代码访问虚拟地址时，CPU 内部的 **MMU（Memory Management Unit）** 会依据页表完成地址转换。
- **虚拟内存区域（VM Region）**：一段具有相同属性的连续虚拟地址范围。一个进程拥有许多 VM Region，但整个虚拟地址空间并非从头到尾连续有效。



### Page Fault 不一定意味着读取闪存

当 CPU 访问某个虚拟地址，而当前页表状态不能直接完成这次访问时，会触发 Page Fault。之后发生什么取决于页面的来源和访问是否合法：

- 页面可能已经在 RAM 中，只是当前映射尚未建立或需要更新，这类情况不需要读取闪存。
- 首次访问匿名内存时，系统可能提供一个零填充页。
- 写入共享页面时，系统可能通过 Copy-on-Write 为进程建立私有副本。
- 访问文件映射但尚未驻留的页面时，系统可能从可执行文件、动态库或其他映射文件中 page in。
- 如果地址无效或权限不允许，这次访问无法被正常修复，最终可能表现为 `EXC_BAD_ACCESS`。

因此，不能把 Page Fault 简化成“每次都从闪存读数据”。在 App 冷启动中，可执行文件和依赖库的 page-in 确实可能带来开销，但它只是启动耗时的组成部分之一，不能脱离实际测量断言它总是占据大部分时间。

## 一个 iOS 进程的虚拟地址空间里有什么

> [!todo]
> 结合 `vmmap` 或 Xcode Memory Report，画出当前实验 App 的内存地图。至少标出：主程序与动态库映射、全局与静态数据、堆、各线程的栈、共享缓存和其他内存映射。

这里需要提前建立一个边界：常说的“代码区、数据区、堆区、栈区”是一张便于入门的教学地图，并不是 Darwin/iOS 对全部 VM Region 的完整分类。真实进程还会包含动态库、共享缓存、匿名映射、内存映射文件、线程相关区域等；同一个大类内部也可能因为权限和用途不同而被拆成多个 Region。

## 指针变量和对象本体分别在哪里

> [!todo]
> 回到开头的代码，分别解释变量本身保存在哪里、变量中保存的值是什么，以及该值指向的对象本体在哪里。不要只回答“对象在堆上”，还要说明字符串字面量、Tagged Pointer 等例外。

例如，在一个函数中声明 `NSObject *object = [[NSObject alloc] init];` 时：

- 局部指针变量 `object` 具有自动存储期，编译器通常会让它位于当前栈帧中，但经过优化后也可能只存在于寄存器中。
- `object` 保存的是对象地址；普通 Objective-C 对象通常由运行时在动态分配区域中创建。
- `&object` 表示指针变量本身的地址，`object` 表示其保存的对象地址，两者不是同一个概念。
- 并非所有看起来像对象的值都对应一次普通堆分配，例如字符串字面量和 Tagged Pointer。

## 用 LLDB 验证地址分布

> [!todo]
> 在 Debug 构建中记录实验环境，再使用 LLDB 比较局部变量、全局变量、静态变量、字符串字面量、普通对象及其指针变量的地址。将“预期—命令—结果—解释”整理成表格，并说明编译优化可能改变局部变量的实际位置。

可以从这些命令开始：

```lldb
p/x &localVariable
p/x &globalVariable
p/x object
p/x &object
image list
```

地址本身只是一条线索。还需要结合 `image list`、`vmmap`、Mach-O 布局和 Region 权限，才能解释一个地址属于哪类映射。

## iOS 内存压力与回收

### iOS 没有传统意义上的磁盘 Swap

桌面操作系统通常可以把一部分匿名内存换出到磁盘。Apple 的 iOS 虚拟内存资料和 WWDC18《iOS Memory Deep Dive》强调：iOS 不把普通 App 的 dirty page 当作传统磁盘 swap 的后备存储来使用。

这不等于“iOS 的虚拟内存从不读取存储设备”。App 的可执行文件、动态库和内存映射文件本来就可以按需 page in；clean 的文件映射页被丢弃后，也可以在需要时重新读取。对于 App 开发，关键区别是：不能期待系统把不断增长的匿名 dirty memory 像桌面 swap 一样长期换出，从而替应用兜底。

系统实现会随版本演进，因此比起把原因简单归结为“保护闪存寿命”，更稳妥的结论是：iOS 的内存策略优先考虑移动设备的性能、能耗和系统响应，并通过回收可重建页面、内存压缩以及必要时终止进程来控制压力。

### 内存压缩

当物理内存紧张时，系统可以压缩近期不活跃的页面，以减少它们占用的物理空间；再次访问时再进行解压。压缩和解压会消耗 CPU，说明内存压力也可能转化为性能与能耗开销。

需要注意：被压缩的内存并没有从 App 的内存责任中消失。理解和分析 footprint 时，应同时关注 dirty memory 与 compressed memory，不能把“已压缩”等同于“已释放”。

### Memory Warning 与 Jetsam

系统处于内存压力时，UIKit 可以通过 App Delegate、View Controller 或通知等途径向 App 传递低内存警告。这个警告反映的是系统级压力，不一定仅由当前 App 导致，也不能保证每次被终止前都严格经历同一条回调链。

如果压力持续、进程超过相应内存限制，或系统需要迅速回收资源，系统可能终止一个或多个进程。开发者随后可能看到 **Jetsam Event Report**。它不同于由异常或信号产生的常规 crash report，因此把这种终止简单写成“Jetsam 先发送 `didReceiveMemoryWarning`，App 不处理后再杀死”并不准确。

### Clean、Dirty 与 Compressed

- **Clean memory**：内容可以从原始来源重新构造或重新读取的页面。例如，未被修改的 Mach-O、Framework 和其他文件映射页面通常可以在需要时丢弃。它更容易回收，但不代表完全没有 page-in 成本，也不能据此断言它与 Jetsam 无关。
- **Dirty memory**：内容已经被进程修改，不能直接丢弃并靠原文件恢复的页面。堆上的对象、解码后的图像缓冲区等通常会贡献 dirty memory，但“调用一次 `alloc/init` 就必然让全部相关页面变成 dirty”仍然过于绝对：内存按页记账，分配与实际写入也要分开观察。
- **Compressed memory**：系统压缩后暂时保留的页面。它减少了物理占用，但仍应纳入 App footprint 的分析。

新分配的页面可能最初仍是 clean；当程序真正写入页面时，页面才会变 dirty。非可丢弃的 dirty page 不能像 clean 文件页那样直接回收，但系统可以压缩它，或者在压力无法缓解时终止相关进程。

循环引用会让对象生命周期超出预期，持续占用它们关联的内存；这属于内存泄漏和对象所有权问题。解码后的大图也可能快速抬高 dirty memory 峰值。二者都值得优化，但不能由此推出“所有 Objective-C 对象都一直占据物理内存，直到手动释放”。

## 如何正确理解“内存五大分区”

> [!todo]
> 在完成 LLDB 实验后，重新审视旧文章中的“五大分区”。分别说明它作为教学模型解决了什么问题、遗漏了哪些 VM Region，以及为什么 Mach-O Segment、进程运行时区域和物理页状态不能混为一谈。

## 当前阶段小结

到这里可以先得出四个结论：

1. 进程使用的是由多个 VM Region 组成的虚拟地址空间，虚拟地址并不等于物理地址。
2. 现代 iOS 的内存管理重点是分页与 Mach VM；Mach-O Segment 不等同于教材中的 CPU 分段。
3. Page Fault 有多种原因，并不必然发生闪存读取；clean、dirty、compressed 描述的则是页面可回收性与记账状态。
4. 回答“变量和对象在哪里”时，必须分别讨论变量本身、变量保存的地址、对象本体以及字面量或 Tagged Pointer 等例外。


## 参考资料

### 官方资料

- [Apple — About the Virtual Memory System](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/AboutMemory.html)
- [Apple — Reducing your app's memory use](https://developer.apple.com/documentation/xcode/reducing-your-app-s-memory-use)
- [Apple — Responding to memory warnings](https://developer.apple.com/documentation/uikit/responding-to-memory-warnings)
- [Apple — Identifying high-memory use with Jetsam Event Reports](https://developer.apple.com/documentation/xcode/identifying-high-memory-use-with-jetsam-event-reports)
- [Apple — Reduce terminations in your app](https://developer.apple.com/documentation/xcode/reduce-terminations-in-your-app)
- [Apple — Reducing your app's launch time](https://developer.apple.com/documentation/xcode/reducing-your-app-s-launch-time)
- [WWDC18 — iOS Memory Deep Dive](https://developer.apple.com/videos/play/wwdc2018/416/)
- [Apple Kernel Programming Guide — Memory and Virtual Memory](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/vm/vm.html)

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
