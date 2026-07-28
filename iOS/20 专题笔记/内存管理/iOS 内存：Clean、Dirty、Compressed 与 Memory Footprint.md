---
title: 【iOS】内存记账：Clean、Dirty、Compressed 与 Memory Footprint
published: 2026-07-24
description: 从'地址在哪里'切换到'现在占用多少'：梳理 Clean、Dirty、Compressed 页面状态，Copy-on-Write 如何让页面变脏，以及内存压力下的压缩、警告与 Jetsam。
tags:
  - iOS
  - Memory
  - Jetsam
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 2
draft: true
---
# iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint

## 前言

系列上一篇 [[iOS 内存：从虚拟地址空间到堆与栈]] 回答的是"一段地址属于哪里、从哪里来"：五大分区给出用途分类，Mach-O、堆和线程栈解释这些区域怎样形成，VM Region 是内核描述它们的真实方式。

但"申请了多大的虚拟地址范围"和"App 当前要为多少物理内存负责"是两个问题。这一篇把观察单位从变量和 Region 切换到内存页，继续追问：这些页面能否重新取得、是否已经被写入、系统怎样计算 Memory Footprint，以及内存压力下系统会做什么。

## 本文主线

全文按照下面的顺序展开：

```text
申请虚拟地址不等于立刻产生等量的物理内存占用
        ↓
Clean、Dirty、Compressed 描述页面此刻的记账状态
        ↓
Copy-on-Write 是页面从"可丢弃"变成"进程私有"的触发点
        ↓
iOS 没有传统磁盘 Swap，内存压力下依赖压缩与终止进程
        ↓
Memory Warning 与 Jetsam 是两套独立机制，不是一条因果链
```

这篇文章不再重复"这段地址用来做什么、从哪里来"——那是上一篇的内容。这里只回答一个问题：这些页面此刻是否真的占用物理内存，系统怎么为它记账。

---

## 申请内存不等于立刻产生等量的 Memory Footprint

Apple 在 WWDC18《iOS Memory Deep Dive》中以典型的 16 KB 页面说明：内存系统以页为粒度管理和统计资源，一个页面可以容纳多个堆对象，一个对象也可能跨越多个页面。因此，哪怕程序只修改其中一个字节，受影响的仍可能是整个页面。

这里不能把"内存使用量"笼统地理解为所有虚拟地址范围。虚拟地址空间大小、当前驻留的物理页、堆分配量和系统用于限制 App 的 Memory Footprint 是不同指标。后文所说的 footprint，重点关注 App 需要系统保留的 dirty 与 compressed 页面，而 clean 页面可以从原始来源重新建立，因此记账方式不同。

### 先区分进程级汇总与逐个 VM Region

上一篇通过 VM Region 观察进程地址空间中的一段段连续范围；这一篇转向整个进程的内存汇总指标。`task_basic_info`、`mach_task_basic_info` 描述的是整个任务（进程）的虚拟内存和驻留内存汇总信息，不是单个 VM Region：

```c
/* 当前 SDK 的注释已明确建议改用 MACH_TASK_BASIC_INFO */
struct task_basic_info {
    integer_t       suspend_count;
    vm_size_t       virtual_size;
    vm_size_t       resident_size;
    time_value_t    user_time;
    time_value_t    system_time;
    policy_t        policy;
};

struct mach_task_basic_info {
    mach_vm_size_t  virtual_size;
    mach_vm_size_t  resident_size;
    mach_vm_size_t  resident_size_max;
    time_value_t    user_time;
    time_value_t    system_time;
    policy_t        policy;
    integer_t       suspend_count;
};
```

这段代码只是两个返回结构的定义，本身不会主动统计内存。程序调用 Mach 的 `task_info()`，并选择 `MACH_TASK_BASIC_INFO`，系统才会把任务级数据写入 `mach_task_basic_info`。当前 SDK 已把 `task_basic_info` 标为旧接口，新的代码应使用始终采用 64 位大小字段的 `MACH_TASK_BASIC_INFO`。

其中与本篇最相关的字段是：

| 字段 | 回答的问题 | 不能直接推出什么 |
| --- | --- | --- |
| `virtual_size` | 进程建立或保留了多大的虚拟地址范围 | 不代表已经占用同等大小的物理 RAM |
| `resident_size` | 当前有多少进程页面驻留在物理内存中 | 不等于 Xcode 展示的 Memory Footprint |
| `resident_size_max` | 驻留内存曾经达到的最大值 | 不等于 footprint 峰值 |

因此要区分两种观察尺度：

```text
VM Region / vmmap
    → 逐段观察地址范围、权限和内容来源

MACH_TASK_BASIC_INFO
    → 观察整个进程的 virtual_size 与 resident_size 汇总

TASK_VM_INFO.phys_footprint
    → 观察更接近 iOS Memory Footprint 口径的进程责任内存
```

`virtual_size`、`resident_size` 与 `phys_footprint` 不能互相替代。后面的真机实验使用 `task_info(TASK_VM_INFO)` 读取 `phys_footprint`，正是因为本篇要研究的是 App 当前承担的内存责任，而不只是地址空间有多大或多少页面暂时驻留。

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260724211423574.png)

以上一篇实验代码中的 `RunMemoryExperiment`（见 [[iOS 内存：从虚拟地址空间到堆与栈|上一篇]]）为例：`malloc(32 * 1024)` 申请一段空间，与 `memset(heapBuffer, 0x5A, 1)` 真正触碰其中一个字节，并不是同一个动作。分配器可能先为程序准备可用的虚拟地址；当程序首次访问相应页面时，系统才通过按需分页机制建立页面支持。实际变化还会受到页面是否已经存在、写入是否落在同一页以及分配器元数据等因素影响，所以不能机械地推导出"每写一个字节，内存就一定增加 16 KB"。

### iPhone 15 真机实测：`malloc` 与首次写入

我在 iPhone 15（iOS 26.5，16 KB 页面）上连续运行两次同一份 Debug 实验。程序分别在 `malloc(32 * 1024)` 之前、申请之后和写入第一个字节之后，通过 `task_info(TASK_VM_INFO)` 读取 `phys_footprint`：

| 采样点 | 第一轮 | 第二轮 |
| --- | ---: | ---: |
| `malloc` 之前 | 5.25 MB | 5.38 MB |
| `malloc(32 KB)` 之后 | 5.25 MB | 5.38 MB |
| `memset(..., 1)` 写入一个字节之后 | 5.27 MB | 5.38 MB |

这两轮数据首先验证了一条稳定结论：**申请 32 KB 堆空间没有立刻表现为 footprint 增加 32 KB**。`malloc` 返回了可用地址，但虚拟地址范围、分配器可用空间和进程当前承担的物理内存不是同一个指标。

第一轮首次写入后，显示值增加约 0.02 MB，与一个 16 KB 页的量级接近；但第二轮在保留两位小数的瞬时采样中没有显示变化。因此，不能把第一轮结果直接总结成“写一个字节必然增加一页”：

1. 分配器可能复用已经存在并已承担 footprint 的页面，而不是为本次 32 KB 请求建立全新 Region；
2. 采样值保留两位小数，16 KB 只约等于 0.0156 MB，显示结果会受到舍入影响；
3. `task_info` 读取的是进程总体 footprint，同一时刻其他运行时活动也可能产生变化；
4. `phys_footprint` 的三次总量采样不能单独证明究竟是哪一页变 dirty，也不能直接给出 clean、dirty、compressed 各自的页数。


所以这次实验支持的是“分配和首次触碰必须分开观察”，而不是“由一组三次采样精确归因一个页面”。如果要进一步验证每次写入影响了哪些页面，应把缓冲区按 16 KB 步长逐页写入，并结合 VM Tracker、Memgraph 或更细的 VM 统计重复对照。

当 App 开始写入页面时，相关页面可能从 clean 变为 dirty。下面这张图描述的是"已分配"和"已写入"在页面状态上的区别，而不是上一篇五大分区中的新区域。

![Snapzy_2026-07-24_21-14-54_871.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/Snapzy_2026-07-24_21-14-54_871.png)

## Clean、Dirty 与 Compressed

- **Clean memory**：内容可以从原始来源重新构造或重新读取的页面。例如，未被修改的 Mach-O、Framework 和其他文件映射页面通常可以在需要时丢弃。Clean page 当前也可能驻留并占用 RAM；它的关键特点是可重新建立，而不是"永远不占物理内存"。
- **Dirty memory**：内容已经被进程修改，不能直接丢弃并靠原文件恢复的页面。堆上的对象、解码后的图像缓冲区等通常会贡献 dirty memory，但内存按页记账，分配与实际写入仍要分开观察。
- **Compressed memory**：系统压缩后暂时保留的页面。它减少当前物理占用，但数据没有被释放，仍应纳入 App footprint 的分析。

这三者不是三个互相独立的分类，而是同一个页面在压力下可能经历的一条状态链：

```text
共享/文件映射（可能是 clean） --首次写入触发 COW--> 私有 dirty 页 --内存压力--> compressed
```

### Copy-on-Write：页面怎样从"可共享"变成"进程私有"

上一篇提到，Mach-O 的 `__DATA` 段这类可写数据可能通过 Copy-on-Write 形成进程私有页面——这里补上具体机制：

1. `__DATA` 段刚被映射时，以**共享、只读**的方式接入进程，可能与文件本身或 dyld shared cache 共享同一块物理页。这时它是 clean 的：内容和文件一致，丢了可以重新读回来。
2. 当程序第一次**写入**这个页面，硬件发现页表标记为只读，触发 Page Fault。内核判断这是一次合法写入，于是执行 COW：分配一块新的私有物理页，把原内容拷贝过去，再把这个进程的页表项指向新页并改成可写，最后完成这次写入。
3. 从这一刻起，这个页面的内容已经和原文件产生分歧，不再能靠重新读文件恢复——这正是 dirty 的定义。它变成了一个进程私有的、系统必须负责保管的页面。

Compressed 只会发生在 dirty 页面身上：clean 页面在压力下直接丢弃即可（反正能重建，压缩反而浪费 CPU），只有 dirty 页面因为丢了就真丢数据，系统才值得花 CPU 压缩它、腾出物理内存，等再次访问时解压。

新获得的匿名页面可能最初仍是 clean（尚未被真正触碰）；当程序真正写入页面时，页面才会变 dirty。非可丢弃的 dirty page 不能像 clean 文件页那样直接回收，但系统可以压缩它，或者在压力无法缓解时终止相关进程。

## iOS 内存压力与回收


![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260725093622870.png)


### iOS 没有传统意义上的磁盘 Swap

桌面操作系统通常可以把一部分匿名内存换出到磁盘。Apple 的 iOS 虚拟内存资料和 WWDC18《iOS Memory Deep Dive》强调：iOS 不把普通 App 的 dirty page 当作传统磁盘 swap 的后备存储来使用。

这不等于"iOS 的虚拟内存从不读取存储设备"。App 的可执行文件、动态库和内存映射文件本来就可以按需 page in；clean 的文件映射页被丢弃后，也可以在需要时重新读取。对于 App 开发，关键区别是：不能期待系统把不断增长的匿名 dirty memory 像桌面 swap 一样长期换出，从而替应用兜底。

系统实现会随版本演进，因此比起把原因简单归结为"保护闪存寿命"，更稳妥的结论是：iOS 的内存策略优先考虑移动设备的性能、能耗和系统响应，并通过回收可重建页面、内存压缩以及必要时终止进程来控制压力。

### 内存压缩

当物理内存紧张时，系统可以压缩近期不活跃的页面，以减少它们占用的物理空间；再次访问时再进行解压。压缩和解压会消耗 CPU，说明内存压力也可能转化为性能与能耗开销。

需要注意：被压缩的内存并没有从 App 的内存责任中消失。理解和分析 footprint 时，应同时关注 dirty memory 与 compressed memory，不能把"已压缩"等同于"已释放"。

### Memory Warning 与 Jetsam

Memory Warning 和 Jetsam 是**两套独立机制**，不是一条"先警告、不处理再杀"的因果链：

- **Memory Warning**（`didReceiveMemoryWarning`）是 UIKit 层面的建议性通知，发给还活着、还能执行代码的进程，本质是系统在说"压力大了，你自己看着办、主动释放点缓存"。它不保证发生在终止之前，也不保证每个即将被终止的进程都收到过。
- **Jetsam** 是内核层面的强制终止机制，持续监控每个进程的 footprint 和系统整体压力，一旦超过阈值——可能是某进程自己的硬限制，也可能是系统整体压力排序后选中它——就会直接终止，不需要也不保证先经过任何 App 代码。尤其是后台进程，往往根本没有机会运行代码去响应警告，会被直接 Jetsam。

开发者事后能看到的是 **Jetsam Event Report**，它不同于由异常或信号产生的常规 crash report：进程是被内核从外部终止的，而不是自己触发了某个信号。

所以准确的说法是：忽略内存警告*可能*增加被 Jetsam 的概率，但 Jetsam 完全可以在没有任何前置警告的情况下发生，比如一个后台进程的 footprint 悄悄涨过了线。

循环引用会让对象生命周期超出预期，持续占用它们关联的内存；这属于内存泄漏和对象所有权问题。解码后的大图也可能快速抬高 dirty memory 峰值。二者都值得优化，但不能由此推出"所有 Objective-C 对象都一直占据物理内存，直到手动释放"。

---

## 总结

到这里可以得出四个结论，衔接上一篇的六条：

1. 虚拟地址范围、堆分配量、驻留物理页和 Memory Footprint 是不同指标；内存按页管理，哪怕只写入一个字节，受影响的也可能是整个页面。
2. Clean、Dirty、Compressed 描述的是页面的可重建性和系统记账状态：clean 页面能从原始来源重建，因此可以直接丢弃；dirty 页面不能，只能压缩或转移风险；compressed 页面数据仍在，只是暂时不占物理内存。
3. Copy-on-Write 是页面从"共享、可丢弃"变成"私有、必须记账"的触发点：首次写入才会真正产生 dirty page，仅仅"映射"或"分配"不会。
4. iOS 不依赖传统磁盘 Swap 为不断增长的匿名 dirty memory 兜底；内存压缩、Memory Warning 与 Jetsam 是三种不同强度的应对手段，其中 Memory Warning 和 Jetsam 是两套独立机制，不存在严格的先后因果关系。

结合上一篇的六条结论，两篇文章合起来回答了同一个问题的两个层面：一段内存"在哪里、从哪里来"，以及它"现在是否真的占用物理内存、系统怎样为它记账"。地址空间部分见系列上一篇 [[iOS 内存：从虚拟地址空间到堆与栈]]。

## 参考资料

- [Apple — Gathering information about memory use](https://developer.apple.com/documentation/xcode/gathering-information-about-memory-use)
- [Apple — Reducing your app's memory use](https://developer.apple.com/documentation/xcode/reducing-your-app-s-memory-use)
- [Apple — Responding to memory warnings](https://developer.apple.com/documentation/uikit/responding-to-memory-warnings)
- [Apple — Identifying high-memory use with Jetsam Event Reports](https://developer.apple.com/documentation/xcode/identifying-high-memory-use-with-jetsam-event-reports)
- [Apple — Reduce terminations in your app](https://developer.apple.com/documentation/xcode/reduce-terminations-in-your-app)
- [WWDC18 — iOS Memory Deep Dive](https://developer.apple.com/videos/play/wwdc2018/416/)
