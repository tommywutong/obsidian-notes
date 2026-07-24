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

> [!note] 写作状态
> 这是第一周 Day 1 的专题文章草稿。完成正文后，删除各节的 `TODO` 提示，补齐实验数据与引用，确认结论边界，再把 frontmatter 中的 `draft` 改为 `false`。

## 文章目录

1. 从一道代码题开始
2. 先分清三个层级
3. 为什么需要虚拟内存
4. 虚拟地址如何走到物理内存
5. 一个进程的虚拟地址空间里有什么
6. 从通用操作系统模型回到 iOS 与 Mach VM
7. Mach-O 与运行时内存区域是什么关系
8. 用 LLDB 观察变量与对象地址
9. 如何正确理解“内存五大分区”
10. 常见误区与边界
11. 面试回答
12. 总结
13. 仍未解决的问题
14. 参考资料

---

## 1. 从一道代码题开始

> [!todo] 写作提示
> 用一小段 Objective-C 代码引出问题：局部变量、全局变量、静态变量、字符串字面量、对象指针和对象本体分别在哪里？先写学习前的预测，不急着给最终答案。

```objc
// 在这里放置今天用于分析和实验的最小代码
```

### 1.1 学习前的预测

- 
- 
- 

### 1.2 这篇文章要解决的问题

- 
- 
- 

## 2. 先分清三个层级

> [!todo] 写作提示
> 分别解释“源代码中的变量”“进程看到的虚拟地址”“机器上的物理内存”。强调三者不是同一个层级。

### 2.1 源代码中的变量和对象

### 2.2 进程虚拟地址空间

### 2.3 物理内存

### 2.4 三个层级之间的关系

```text
源代码中的变量或对象
        ↓
进程的虚拟地址
        ↓
虚拟内存区域与页表映射
        ↓
物理内存中的页面
```

## 3. 为什么需要虚拟内存

> [!todo] 写作提示
> 不要只写“可以使用比物理内存更大的空间”。从多进程地址隔离、访问保护、灵活映射和按需使用物理页几个角度展开。

### 3.1 如果程序直接操作物理地址会发生什么

### 3.2 每个进程拥有独立地址空间

### 3.3 隔离、保护与灵活映射

### 3.4 虚拟内存不等于物理内存的扩容盘

## 4. 虚拟地址如何走到物理内存

> [!todo] 写作提示
> 先讲最小链路，再补术语。今天不必展开所有页表级数和硬件细节。

### 4.1 Page：固定大小的地址区间

### 4.2 页号与页内偏移

### 4.3 页表与 MMU

### 4.4 Page Fault

### 4.5 多级页表与 TLB 解决什么问题

```text
虚拟地址
  ↓ 拆分
虚拟页号 + 页内偏移
  ↓ 查询 TLB / 页表
物理页号 + 页内偏移
  ↓
物理地址
```

## 5. 一个进程的虚拟地址空间里有什么

> [!todo] 写作提示
> 先画教学用的简化地图，再说明真实进程由多个 VM region 组成，具体布局会受到架构、系统版本、ASLR、动态库和分配器影响。

### 5.1 可执行映像与只读内容

### 5.2 全局变量与静态变量

### 5.3 堆与动态分配

### 5.4 每个线程自己的栈

### 5.5 动态库、共享缓存与其他映射区域

### 5.6 简化内存地图

```text
进程虚拟地址空间
├── 可执行映像
│   ├── 代码
│   ├── 只读常量
│   ├── 已初始化全局/静态数据
│   └── 未初始化全局/静态数据
├── 动态库与其他映射
├── 堆与其他动态分配区域
└── 各线程自己的栈
```

## 6. 从通用操作系统模型回到 iOS 与 Mach VM

> [!todo] 写作提示
> 区分小林文章中的 Linux 示例与 Apple 平台。说明 Mach task、memory map、VM region 和 protection 的基本关系；只写今天能够确认的稳定概念。

### 6.1 Mach task 与 memory map

### 6.2 稀疏地址空间与区域权限

### 6.3 文件映射与匿名内存

### 6.4 iOS 的内存压力与 No Page Outs 边界

## 7. Mach-O 与运行时内存区域是什么关系

> [!todo] 写作提示
> 说明 Mach-O 是文件格式和装载描述，VM region 是运行时地址空间中的映射。二者相关，但不能把 Mach-O segment、section 与“内存五区”一一机械对应。

### 7.1 Mach-O 的 Header、Load Commands 与 Segment

### 7.2 `__TEXT` 与 `__DATA`

### 7.3 文件中的组织与运行时映射

## 8. 用 LLDB 观察变量与对象地址

> [!todo] 写作提示
> 先写预测，再记录地址，最后解释。区分 `object` 和 `&object`；不要根据一次地址前缀总结成所有设备都成立的规律。

### 8.1 实验环境

- 设备或模拟器：
- 系统版本：
- 架构：
- Xcode 版本：
- Debug / Release：
- 是否开启优化：

### 8.2 实验代码

```objc
// 在这里放置完整的可复现实验代码
```

### 8.3 LLDB 命令

```text
# 在这里记录实际使用的 LLDB 命令
```

### 8.4 地址观察结果

| 表达式 | 它代表什么 | 观察到的地址 | 初步所属区域 | 生命周期 | 备注 |
| --- | --- | --- | --- | --- | --- |
| `&globalInitialized` |  |  |  |  |  |
| `&globalUninitialized` |  |  |  |  |  |
| `&localNumber` |  |  |  |  |  |
| `&staticNumber` |  |  |  |  |  |
| `literal` | 字符串字面量对象地址 |  |  |  |  |
| `&literal` | 局部指针变量自身的地址 |  |  |  |  |
| `object` | 普通 Objective-C 对象地址 |  |  |  |  |
| `&object` | 局部指针变量自身的地址 |  |  |  |  |

### 8.5 实验结果与预测对照

### 8.6 哪些现象不能推广为平台永真结论

## 9. 如何正确理解“内存五大分区”

> [!todo] 写作提示
> 先承认这个模型的教学价值，再写它省略了哪些真实情况。不要简单宣布“五区模型是错的”。

### 9.1 五区模型帮助我们理解什么

### 9.2 它省略了什么

### 9.3 面试中应该怎样表述

## 10. 常见误区与边界

> [!todo] 写作提示
> 每个误区采用“错误说法 → 为什么不严谨 → 更准确的表达”三段式。

### 10.1 虚拟地址就是物理地址

### 10.2 指针变量和对象本体在同一个地方

### 10.3 所有 Objective-C 对象都一定在普通堆上

### 10.4 地址前缀可以判断所有设备上的内存区域

### 10.5 Linux 的 Swap、brk 与 glibc 结论可以直接套到 iOS

### 10.6 Mach-O section 就等于运行时“内存分区”

## 11. 面试回答

> [!todo] 写作提示
> 先写 30 秒结论版，再写 2～3 分钟展开版。回答中区分稳定概念、教学模型和当前实现。

### 11.1 30 秒版本

### 11.2 2～3 分钟版本

### 11.3 可能的追问

- 为什么需要虚拟内存？
- 虚拟地址怎样转换成物理地址？
- 局部指针和它指向的对象分别在哪里？
- 栈和堆有什么区别？
- iOS 的内存管理和 Linux Swap 有什么不同？
- “内存五大分区”哪里不严谨？

## 12. 总结

> [!todo] 写作提示
> 用三到五条结论回收全文，不引入新概念。

- 
- 
- 

## 13. 仍未解决的问题

> [!todo] 写作提示
> 记录今天有意识暂缓的问题，并标注计划在哪一天补上，避免为了“写完整”而在 Day 1 无限扩张范围。

- [ ] `isa`、类对象和元类：第一周 Day 2。
- [ ] Tagged Pointer：第一周 Day 3。
- [ ] MRC / ARC 与对象生命周期：第一周 Day 4～5。
- [ ] Mach-O 完整结构与 dyld 装载：第七周。
- [ ] 

## 14. 参考资料

### 官方资料

- [Apple Kernel Programming Guide — Memory and Virtual Memory](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/vm/vm.html)

### 操作系统资料

- [小林 Coding — 图解操作系统](https://www.xiaolincoding.com/os/)
- [小林 Coding — 为什么要有虚拟内存？](https://www.xiaolincoding.com/os/3_memory/vmem.html)
- [小林 Coding — malloc 是如何分配内存的？](https://www.xiaolincoding.com/os/3_memory/malloc.html)
- 本地：[[10 学习计划/计算机操作系统（持续更新版本）_副本.docx|计算机操作系统 Word]]

### iOS 与 Objective-C 延伸

- [Size Matters: An Exploration of Virtual Memory on iOS](https://alwaysprocessing.blog/2022/02/20/size-matters)
- [Mike Ash — Stack and Heap Objects in Objective-C](https://www.mikeash.com/pyblog/friday-qa-2010-01-15-stack-and-heap-objects-in-objective-c.html)
- [Mike Ash — Intro to the Objective-C Runtime](https://www.mikeash.com/pyblog/friday-qa-2009-03-13-intro-to-the-objective-c-runtime.html)
- 本地：[[20 专题笔记/编译链接与启动/Mach-O|Mach-O]]

### 学习计划

- [[10 学习计划/2026 暑假 iOS 底层学习计划#Day 1｜先分清“地址空间”，不要一上来背 isa（对应 W1-10）|第一周 Day 1 学习计划]]
