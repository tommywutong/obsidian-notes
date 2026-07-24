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
在之前的文章中，笔者完成过
- https://www.tommywutong.cn/blog/csdn-import/csdn-154609757-ios-/
- https://www.tommywutong.cn/blog/csdn-import/csdn-152130856-ios-/
这两篇初级的文章，但是碍于当时知识面的浅薄，理解较浅 且表述不够全面，现在重新回头梳理iOS相关知识，借此机会完成一篇新博客。


## 虚拟内存

在没有操作系统的时候，CPU直接操作内存的物理地址，在这种情况下，想要在内存中同时运行两个程序是不可能的。
如果第一个程序在某位置写入一个新的值，就会擦掉第二个程序放在相同位置的所有内容。所以，同时运行两个程序是根本行不通的。

因此，在操作系统中，我们引入了虚拟地址。我们通过它把进程里使用的地址隔离开来，互不干涉，但前提是每个进程都不能访问物理地址。除此以外，操作系统会提供一种机制，让不同进程的虚拟地址和不同内存的物理地址映射起来。


- 虚拟内存：是操作系统提供给每个运行中程序的一种地址空间这是操作系统为了统一管理和保护内存而提供的一个**抽象层**。在 iOS 中，每个 App 启动时，系统都会为其分配一个独立的、连续的虚拟地址空间。其大小可以远远大于物理内存的大小。虚拟内存通过将程序的地址空间划分成若干个固定大小的页或段，并将这些页或者段映射到物理内存中的不同位置，从而使得程序在运行时可以更高效地利用物理内存。

- 物理内存：物理内存是计算机实际存在的内存，是计算机中的实际硬件部件。比如芯片上集成的 8GB 或 16GB 统一内存。它是供 CPU 快速读写数据的真实物理空间。


在操作系统中，**内存分段（Segmentation）和内存分页（Paging）** 是实现虚拟内存最核心的两种内存管理方式。两者区别在于，内存分段更多是从“程序员和逻辑”的视角出发，而分页是从“系统和硬件物理资源”的视角出发。

### 1，内存分段

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260724154844318.png)


该机制的核心思想是按照程序的逻辑结构来划分内存。编译器在编译器在编译程序时，会将程序按照逻辑功能划分为不同的段。例如：代码段（Text Segment）、数据段（Data Segment）、堆栈段（Stack Segment）等。每个段的长度是不固定的，取决于该段的实际逻辑大小。


### 2，内存分页

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260724154856696.png)


操作系统通过将内存划分为固定大小的页 (Page)（在目前的 iOS 设备中通常是 16KB）来进行管理。  

- **页表 (Page Table):** 操作系统维护着一张映射表。当你的代码尝试读取或写入某个虚拟地址时，CPU 内部的 **MMU (内存管理单元)** 会查找这张页表，将虚拟地址“翻译”成真实的物理地址。  

- **缺页中断 (Page Fault):** 虚拟内存并不一定都存在于物理内存中。当 App 访问一个虚拟地址，但发现这个页还没有被加载到物理内存时，就会触发 Page Fault。此时，系统会暂停你的 App，将所需的数据从闪存（磁盘）加载到物理内存的空闲页中，更新页表，然后再恢复 App 的运行。App 冷启动时的大量耗时，很大一部分就来源于密集的 Page Fault。




**3. iOS 内存管理的特殊性**  
桌面级操作系统（如 macOS 或 Windows）在物理内存不足时，会使用 Swap (交换空间) 机制，将内存中不常用的数据写回到硬盘上，腾出物理内存给需要的程序。但是，iOS 没有 Swap 机制。  
主要原因是移动设备的闪存读写寿命有限，且频繁的 I/O 操作会导致严重的卡顿和高耗电。为了在没有 Swap 的情况下维持系统的流畅，iOS 采用了以下机制：  
**A. 内存压缩 (Memory Compression)**  
当物理内存开始紧张时，iOS 不会将数据写入磁盘，而是将被占用的、但最近没有被访问的内存页压缩起来。  

- 当 App 再次需要访问这块内存时，系统会在将其解压。  
    
- **代价：** 压缩和解压的过程会消耗 CPU 资源，因此在内存吃紧时，设备往往会发热或耗电增加。  
    

**B. Jetsam 机制 (OOM Killer)**  
如果内存压力持续增大，连压缩内存也无法缓解，iOS 内部的 Jetsam 守护进程就会介入。它会根据优先级监控每个进程的内存占用。  

- 当系统可用物理内存低于阈值时，Jetsam 会直接向 App 发送内存警告 (`didReceiveMemoryWarning`)。  
    
- 如果 App 无法及时释放内存，Jetsam 就会无情地将该进程强制杀死，这就是开发者经常遇到的 **OOM (Out of Memory)** 崩溃。  
    

**4. 内存分类与 Objective-C 开发的联系**  
在 iOS 中，虚拟内存的页主要被分为两大类。了解这两种分类，对你在日常使用 Objective-C 进行底层开发或排查内存泄漏至关重要。  

- **Clean Memory (干净内存):**  
    是指那些可以被系统随时丢弃，并在需要时重新从磁盘加载的内存。  
    • **例子：** App 的可执行文件 (Mach-O)、加载的 Framework（比如 Objective-C Runtime 库、Foundation 等）、内存映射文件。  
    • 这类内存不会触发 Jetsam，因为系统随时可以回收它们。  
    
- **Dirty Memory (脏内存):**  
    是指被 App 写入过数据，且无法被系统回收的内存。**由于 iOS 没有 Swap，Dirty Memory 一旦产生，就必须一直死死占据着物理内存，直到你的代码主动释放它。**  
    • **例子：** 使用 `alloc` / `init` 在堆区 (Heap) 创建的所有对象实例。  
    • **开发隐患：** 如果你在编写 Block 闭包时没有正确使用 `__weak`，导致了 **Retain Cycle (循环引用)**，这部分对象所占用的内存就会变成永远无法释放的 Dirty Memory，引发内存泄漏，最终导致 OOM。  
    • 解压后的图像数据也是巨大的 Dirty Memory 开销。  
    

**总结**  
虚拟内存为 iOS App 提供了一个安全的、隔离的运行沙盒；而物理内存则是有限且极其珍贵的硬件资源。在 iOS 这种没有 Swap 机制的系统里，内存优化的核心本质，其实就是尽可能地降低 App 的 Dirty Memory 峰值，并确保对象的生命周期被正确管理。


## 参考资料

### 官方资料

- [Apple Kernel Programming Guide — Memory and Virtual Memory](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/vm/vm.html)

### 操作系统资料

- [小林 Coding — 图解操作系统](https://www.xiaolincoding.com/os/)
- [小林 Coding — 为什么要有虚拟内存？](https://www.xiaolincoding.com/os/3_memory/vmem.html)
- [小林 Coding — malloc 是如何分配内存的？](https://www.xiaolincoding.com/os/3_memory/malloc.html)

### iOS 与 Objective-C 延伸

- [Size Matters: An Exploration of Virtual Memory on iOS](https://alwaysprocessing.blog/2022/02/20/size-matters)
- [Mike Ash — Stack and Heap Objects in Objective-C](https://www.mikeash.com/pyblog/friday-qa-2010-01-15-stack-and-heap-objects-in-objective-c.html)
- [Mike Ash — Intro to the Objective-C Runtime](https://www.mikeash.com/pyblog/friday-qa-2009-03-13-intro-to-the-objective-c-runtime.html)
- 本地：[[20 专题笔记/编译链接与启动/Mach-O|Mach-O]]
