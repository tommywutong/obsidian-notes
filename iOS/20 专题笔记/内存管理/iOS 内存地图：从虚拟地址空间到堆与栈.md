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
