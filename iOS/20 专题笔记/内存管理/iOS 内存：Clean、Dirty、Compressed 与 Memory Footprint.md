---
title: 【iOS】内存：从页面状态、Memory Footprint 到 OOM
published: 2026-07-24
description: 从“地址在哪里”走向“现在占用多少”：梳理页面如何成为 Clean、Dirty 与 Compressed，Memory Footprint 如何增长，以及 iOS 如何处理内存压力与 OOM。
tags:
  - iOS
  - Memory
  - OOM
  - Jetsam
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 2
draft: true
---
# iOS 内存：从页面状态、Memory Footprint 到 OOM

## 前言

系列上一篇[《iOS 内存地图：从虚拟地址空间到 VM Region》](/blog/ios-memory-map-virtual-address-space-vm-region/)回答的是"一段地址属于哪里、从哪里来"：五大分区给出用途分类，Mach-O、堆和线程栈解释这些区域怎样形成，VM Region 是内核描述它们的真实方式。

但"申请了多大的虚拟地址范围"和"App 当前要为多少物理内存负责"是两个问题。这一篇把观察单位从变量和 Region 切换到内存页，继续追问：这些页面能否重新取得、是否已经被写入、系统怎样计算 Memory Footprint，以及内存压力下系统会做什么。

## 本文主线

全文按照下面的顺序展开：

```text
申请或映射虚拟地址
        ↓
实际访问和写入页面
        ↓
页面成为 Clean、Dirty 或 Compressed
        ↓
Dirty 与 Compressed 推高 Memory Footprint
        ↓
系统出现内存压力
        ↓
回收 Clean Page／压缩 Dirty Page／发送 Memory Warning／Jetsam
        ↓
区分 OOM、内存泄漏与瞬时峰值，并选择对应工具排查
```

这篇文章不再重复"这段地址用来做什么、从哪里来"——那是上一篇的内容。这里只回答一个问题：这些页面此刻是否真的占用物理内存，系统怎么为它记账。

---

## 一、从虚拟地址空间到物理内存

### 为什么虚拟地址不等于物理内存

首先推荐阅读：[Friday Q&A 2012-12-28: What Happens When You Load a Byte of Memory](https://www.mikeash.com/pyblog/friday-qa-2012-12-28-what-happens-when-you-load-a-byte-of-memory.html)
中译版：
[当你加载一字节内存会发生什么](https://github.com/Biscoffee/apple-docs-vault/blob/main/blogs/zh/mikeash/friday-q-a-2012-12-28-what-happens-when-you-load-a-byte-of-memory.md)

Apple 在 WWDC18《iOS Memory Deep Dive》中以典型的 16 KiB 页面说明：内存系统以页为粒度管理和统计资源，一个页面可以容纳多个堆对象，一个对象也可能跨越多个页面。因此，哪怕程序只修改其中一个字节，受影响的仍可能是整个页面。

这里不能把"内存使用量"笼统地理解为所有虚拟地址范围。虚拟地址空间大小、当前驻留的物理页、堆分配量和系统用于限制 App 的 Memory Footprint 是不同指标。后文所说的 footprint，重点关注 App 需要系统保留的 dirty 与 compressed 页面，而 clean 页面可以从原始来源重新建立，因此记账方式不同。

### 进程级汇总与逐个 VM Region

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

## 二、页面为什么会真正占用内存

### 一：`malloc`、首次访问与 Dirty Page

以上一篇实验代码中的 `RunMemoryExperiment`（见[上一篇](/blog/ios-memory-map-virtual-address-space-vm-region/)）为例：`malloc(32 * 1024)` 申请一段 32 KiB 空间，与 `memset(heapBuffer, 0x5A, 1)` 真正触碰其中一个字节，并不是同一个动作。分配器可能先为程序准备可用的虚拟地址；当程序首次访问相应页面时，系统才通过按需分页机制建立页面支持。实际变化还会受到页面是否已经存在、写入是否落在同一页以及分配器元数据等因素影响，所以不能机械地推导出"每写一个字节，Memory Footprint 就一定增加 16 KiB"。

### iPhone 15 真机实测：`malloc` 与首次写入

我在 iPhone 15（iOS 26.5.2，16 KiB 页面）上连续运行两次同一份 Debug 实验。程序分别在 `malloc(32 * 1024)` 之前、申请之后和写入第一个字节之后，通过 `task_info(TASK_VM_INFO)` 读取 `phys_footprint`：

| 采样点 | 第一轮 | 第二轮 |
| --- | ---: | ---: |
| `malloc` 之前 | 5.25 MiB | 5.38 MiB |
| `malloc(32 KiB)` 之后 | 5.25 MiB | 5.38 MiB |
| `memset(..., 1)` 写入一个字节之后 | 5.27 MiB | 5.38 MiB |

这两轮数据共同支持一个结论：**申请 32 KiB 堆空间没有立刻表现为 footprint 增加 32 KiB**。`malloc` 返回了可用地址，但虚拟地址范围、分配器可用空间和进程当前承担的物理内存不是同一个指标。

第一轮首次写入后，显示值增加约 0.02 MiB，与一个 16 KiB 页的量级接近；但第二轮在保留两位小数的瞬时采样中没有显示变化。因此，不能把第一轮结果直接总结成“写一个字节必然增加一页”：
1. 分配器可能复用已经存在并已承担 footprint 的页面，而不是为本次 32 KiB 请求建立全新 Region；
2. 采样值保留两位小数，16 KiB 只约等于 0.0156 MiB，显示结果会受到舍入影响；
3. `task_info` 读取的是进程总体 footprint，同一时刻其他运行时活动也可能产生变化；
4. `phys_footprint` 的三次总量采样不能单独证明究竟是哪一页变 dirty，也不能直接给出 clean、dirty、compressed 各自的页数。


所以这次实验支持的是“分配和首次触碰必须分开观察”，而不是“由一组三次采样精确归因一个页面”。如果要进一步验证每次写入影响了哪些页面，应把缓冲区按 16 KiB 步长逐页写入，并结合 VM Tracker、Memgraph 或更细的 VM 统计重复对照。

当 App 开始写入页面时，相关页面可能从 clean 变为 dirty。下面这张图描述的是"已分配"和"已写入"在页面状态上的区别，而不是上一篇五大分区中的新区域。

![Snapzy_2026-07-24_21-14-54_871.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/Snapzy_2026-07-24_21-14-54_871.png)

### 路径二：Copy-on-Write 与 Private Dirty Page

上一篇提到，Mach-O 的 `__DATA` 段这类可写数据可能通过 Copy-on-Write 形成进程私有页面——这里补上具体机制：
1. `__DATA` 对程序通常表现为可写的 VM Region，但它的共享模式可以是 Copy-on-Write。页面内容尚未被当前进程修改时，可以继续复用文件支持或共享的物理页，此时内容仍是 clean。
2. 当程序第一次**写入**相应页面时，底层只读保护会触发一次可以修复的 Page Fault。内核识别出这是合法的 COW 写入，于是为当前进程建立私有副本，再完成这次写入。这里的“底层保护”不等于 App 看到的 VM Region 是 `r--`；在 `vmmap` 或 `memory region` 中，它仍可以显示为 `rw-`。
3. 从这一刻起，这个页面的内容已经和原文件产生分歧，不再能靠重新读文件恢复——这正是 dirty 的定义。它变成了一个进程私有的、系统必须负责保管的页面。

在 `vmmap` 的描述中，这类 Region 可以同时显示为可写权限 `rw-` 和共享模式 `COW`；Apple 的 [Viewing Virtual Memory Usage](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/VMPages.html) 也专门区分了访问权限和共享模式。

在本文关注的 iOS App 内存模型里，未修改的文件支持 clean page 可以在压力下丢弃并重新载入；不能直接丢弃的 private／dirty page 才是内存压缩主要处理的对象。被压缩后，数据仍保存在物理内存中，只是占用空间变小，后续访问时需要解压。

新申请的匿名虚拟地址可能尚未获得对应的驻留物理页；当程序真正写入时，系统才按需建立页面支持，并使相关页面进入 dirty 状态。非可丢弃的 dirty page 不能像 clean 文件页那样直接丢弃，但 App 可以主动释放它，系统也可以压缩它；压力仍无法缓解时，系统可能终止进程。

### 两条路径的区别

```text
匿名堆内存：
malloc → 获得可用虚拟地址 → 首次写入 → 匿名 Dirty Page

文件支持内存：
Clean／共享页面 → 首次写入触发 COW → Private Dirty Page
```

两条路径的起点不同：前者从新申请的匿名地址开始，后者从已有的文件支持或共享页面开始；但实际写入以后，它们都可能形成需要当前进程负责的 dirty memory。下面再分别说明 Clean、Dirty 与 Compressed 到底代表什么。

## 三、Clean、Dirty 与 Compressed

### Clean memory

Clean 描述的是页面内容可以从原始来源重新取得，而不是页面“没有占用物理内存”，也不是说这段地址只能读、不能写。访问权限由 VM Region 的 `r--`、`rw-` 等属性决定，Clean／Dirty 描述的是页面内容是否已经发生需要保留的修改，这两个维度不能混在一起。

常见的 clean memory 包括：

1. 尚未被修改的可执行文件和系统 Framework 代码页；
2. 尚未被修改的只读数据和文件映射页面，例如按需载入的图片文件、模型文件；
3. 内容仍与原始文件一致、在压力下可以丢弃并在以后重新读取的页面。

Clean page 既可能已经驻留并占用 RAM，也可能尚未进入物理内存。系统在压力下可以丢弃可重建的 clean page，需要时再从文件载入，而不是把这类页面先压缩保存。

内存映射文件也不一定永远是 clean 或只读：只读且未修改的文件映射通常保持 clean；如果映射允许写入并且内容被修改，相应页面就可能变成 dirty。Framework 的 `__DATA_CONST` 也不会因为 App “使用了这个 Framework”就自动变 dirty，真正的判断依据仍然是页面是否发生了需要私有保存的修改。

### Dirty memory

Dirty 表示页面内容已经被修改，不能在不保存内容的情况下直接丢弃并依靠原始文件恢复。它不是“永远不能重复使用”的内存：App 主动释放对象后，相关空间仍可以被分配器重新利用；在内存压力下，系统也可以压缩近期不活跃的 dirty page。

常见来源包括：

1. 已经实际写入的堆对象和缓冲区；
2. 图片解码后的像素缓冲，例如 ImageIO、Core Graphics 或 IOSurface 相关内存；
3. 缓存、可变集合，以及 Framework 中实际需要写入的 `__DATA`、`__DATA_DIRTY` 页面；
4. 文件支持页面经过 Copy-on-Write 后形成的进程私有副本。

这里不能简单地说“所有 heap allocation 都是 dirty”。`malloc` 返回地址并不代表整段空间已经获得并写入了对应物理页；尚未触碰的部分、分配器复用的页面和分配器元数据都可能让实际结果与申请字节数不同。

Memory Warning 也不会替 App “只释放 clean memory”。它是系统发给仍能运行的 App 的通知，提醒 App 主动释放可以重建的缓存、图片和临时对象，或者减少并发任务。系统回收 clean page、压缩 dirty page，以及 App 自己解除强引用，是三条不同的路径。Apple 的 [Responding to low-memory warnings](https://developer.apple.com/documentation/xcode/responding-to-low-memory-warnings) 还特别提醒，不要在收到警告后临时遍历整个对象图寻找可释放内容。

理解工具结果时还要区分：

```text
Allocations
→ 关注堆上发生了哪些分配

Dirty Size
→ 关注哪些页面已经被写入并需要进程负责
```

> **例子：80,000 字节会覆盖多少页？**
>
> 如果缓冲区恰好从页边界开始，按 16 KiB 一页计算：
>
> ```text
> ceil(80,000 / 16,384) = 5 页
> ```
>
> 如果起始地址没有页对齐，它也可能跨越 6 个页面。通过 `malloc(80000)` 申请时，分配器还可能复用已有 Region，所以这个计算只表示地址范围最多跨过多少页，不能直接证明系统新分配了同样数量的物理页。
>
> 当程序第一次写入某个尚未承担实际内容的页面时，该页才可能进入 dirty 状态。只写第一页和最后一页，不代表中间所有页面都会同时变 dirty。
>
> ![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260729181948297.png)

### Compressed memory

iOS 不为普通 App 的匿名 dirty memory 提供传统磁盘 Swap，而是可以把近期不活跃、又不能直接丢弃的页面压缩后保留在内存中。压缩比例取决于页面内容，没有一个可以通用于所有数据的固定比例；再次访问这些页面时，系统需要先解压。

Compressed memory 并没有消失：它仍占用物理内存，只是压缩后的占用通常比未压缩时更小，而且仍属于 App 的 Memory Footprint。压缩和解压还会消耗 CPU，因此内存压力也可能表现为性能与能耗问题。

![Snapzy_2026-07-29_18-33-54_314.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/Snapzy_2026-07-29_18-33-54_314.png)

对于可以重新创建的缓存，`NSCache` 通常比无上限的 `NSMutableDictionary` 更合适：它支持自动淘汰策略，也允许从多个线程查询和修改。但淘汰时机与顺序不是业务代码可以依赖的确定保证，仍应合理设置 `countLimit` 或 `totalCostLimit`，具体行为见 Apple 的 [`NSCache`](https://developer.apple.com/documentation/foundation/nscache) 文档。`NSPurgeableData` 则用于内容可以被丢弃的数据，只有在内容没有被标记为正在使用时，系统才可能在低内存情况下丢弃它。

释放缓存时也要考虑压缩页面。遍历很大的对象图或逐项访问一个长期未使用的字典，可能先把相应页面从 compressor 中解压，短时间内反而增加内存压力。这并不是说“删除 Dictionary 一定让物理内存变大”，而是说明收到 Memory Warning 后不应临时扫描所有对象寻找可释放内容；更好的做法是提前设计好有边界、可整体放弃的缓存策略。

在本文采用的 App 内存模型中，三种状态可以用下面这条常见路径连接起来：

```text
共享/文件映射（可能是 clean） --首次写入触发 COW--> 私有 dirty 页 --内存压力--> compressed
```

### Memory Footprint：App 真正负责的内存

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260729183653223.png)

前面已经出现过三个容易混淆的进程级指标，它们回答的不是同一个问题：

| 指标                            | 回答的问题                   |
| ----------------------------- | ----------------------- |
| `virtual_size`                | 进程建立或保留了多大的虚拟地址范围       |
| `resident_size`               | 当前有多少进程页面驻留在物理内存中       |
| `TASK_VM_INFO.phys_footprint` | 当前有多少内存被系统记作这个进程需要负责的资源 |

Xcode 中关注的 Memory Footprint 更接近第三种，而不是把虚拟地址范围或所有驻留页面简单相加。在 Apple 面向开发者的内存模型中，分析 Footprint 时主要关注 **Dirty Memory 与 Compressed Memory**：dirty 页面不能依靠原始文件直接恢复；它被压缩以后虽然体积可能变小，内容仍需要系统替 App 保存。

Clean Memory 即使已经驻留，也可能占用物理 RAM；但它的内容能够从可执行文件、Framework 或映射文件重新取得，系统可以在压力下丢弃并在以后重新载入。因此，Clean Memory 通常不是降低 App Footprint 时的首要目标。

这里的“Dirty + Compressed”用于理解和优化，不是 `phys_footprint` 内核实现的完整数学公式。实际记账还会涉及其他系统内存类别，本文不把开发者模型写成一条绝对等式。

## 四、iOS 如何应对内存压力


![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260725093622870.png)

当 Memory Footprint 持续增长或瞬时峰值过高时，App 可能收到内存警告，也可能在来不及响应警告时被系统终止。下面继续看 iOS 面对内存压力时可以采取哪些措施。


### iOS 没有传统意义上的磁盘 Swap

桌面操作系统通常可以把一部分匿名内存换出到磁盘。Apple 的 iOS 虚拟内存资料和 WWDC18《iOS Memory Deep Dive》强调：iOS 不把普通 App 的 dirty page 当作传统磁盘 swap 的后备存储来使用。

这不等于"iOS 的虚拟内存从不读取存储设备"。App 的可执行文件、动态库和内存映射文件本来就可以按需 page in；clean 的文件映射页被丢弃后，也可以在需要时重新读取。对于 App 开发，关键区别是：不能期待系统把不断增长的匿名 dirty memory 像桌面 swap 一样长期换出，从而替应用兜底。

系统实现会随版本演进，因此比起把原因简单归结为"保护闪存寿命"，更稳妥的结论是：iOS 的内存策略优先考虑移动设备的性能、能耗和系统响应，并通过回收可重建页面、内存压缩以及必要时终止进程来控制压力。

### 系统怎样处理不同状态的页面

前面已经解释了 Compressed memory 的定义。把它放进内存压力场景后，系统面对不同页面时大致有两种选择：

```text
仍能从原始文件重新取得的 Clean Page
    → 可以直接回收，需要时重新载入

不能直接丢弃的 Dirty Page
    → 可以压缩近期不活跃的内容，后续访问时再解压
```

因此，压缩不是释放，只是系统在有限 RAM 中继续保留 dirty 内容的一种方式。如果回收 clean page、压缩 dirty page，以及 App 自己释放可重建资源以后仍不足以缓解压力，系统才可能需要终止进程来快速回收内存。

### Memory Warning 与 Jetsam

![Snapzy_2026-07-29_19-37-09_257.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/Snapzy_2026-07-29_19-37-09_257.png)


Memory Warning 和 Jetsam 是**两套独立机制**，不是一条"先警告、不处理再杀"的因果链：

- **Memory Warning**（`didReceiveMemoryWarning`）是 UIKit 层面的建议性通知，发给还活着、还能执行代码的进程，本质是系统在说"压力大了，你自己看着办、主动释放点缓存"。它不保证发生在终止之前，也不保证每个即将被终止的进程都收到过。

- **Jetsam** 是内核层面的强制终止机制，持续监控每个进程的 footprint 和系统整体压力，一旦超过阈值——可能是某进程自己的硬限制，也可能是系统整体压力排序后选中它——就会直接终止，不需要也不保证先经过任何 App 代码。尤其是后台进程，往往根本没有机会运行代码去响应警告，会被直接 Jetsam。

开发者事后能看到的是 **Jetsam Event Report**，它不同于由异常或信号产生的常规 crash report：进程是被内核从外部终止的，而不是自己触发了某个信号。

App 收到内存警告后，应该释放已经明确标记为可重建、当前又不需要的资源，并减少后续分配，而不是临时遍历整个对象图寻找“看起来能删”的内容。后者可能访问并解压长期未使用的页面，在压力最大的时刻制造额外峰值。

所以准确的说法是：忽略内存警告*可能*增加被 Jetsam 的概率，但 Jetsam 完全可以在没有任何前置警告的情况下发生，比如一个后台进程的 footprint 悄悄涨过了线。

## 五、iOS 语境下的 OOM

OOM 是 **Out of Memory** 的缩写，分为两大类，Foreground OOM / Background OOM，简写为 FOOM 以及 BOOM。而其中 FOOM 是指 app 在前台时由于消耗内存过大，而被系统杀死，直接表现为 crash。
从最直接的语义看，它表示系统无法继续满足内存需求；但在 iOS 开发讨论中，“OOM”经常被用来概括几种并不完全相同的结果：

1. **某次内存申请失败**：从通用编程语义看，这是最直接的 OOM。例如 `malloc` 在无法满足申请时可能返回 `NULL`，其他 API 也可能按照自己的约定返回错误或抛出异常。
2. **进程超过自身的 Memory Footprint 限制**：App 持续增加 dirty 与 compressed memory，逼近或超过当前设备、进程类型和运行状态对应的限制，系统可能通过资源异常或 Jetsam 终止进程。
3. **系统整体内存压力下被选中终止**：App 未必单独达到一个固定数字，但系统需要迅速回收资源时，内核可能根据进程优先级、状态和内存责任选择它作为 Jetsam 对象。

真实 iOS App 往往在一次普通分配明确返回失败之前，就已经因为内存压力被系统从外部终止。因此，日常所说的“App 发生 OOM”，通常指的是**高内存使用导致的系统终止**，但分析报告时仍应确认实际终止原因，不能看到“突然消失”就直接下结论。

前台发生高内存终止时，用户可能感知到 App 突然退出；后台进程被 Jetsam 时，用户通常不会立即察觉，只会在下次进入时看到 App 重新启动。设备的分析数据或 Xcode 同步的设备日志中可能出现 `JetsamEvent` 报告，它与带有触发线程和调用栈的普通 crash report 不同。

OOM、Memory Warning、Jetsam 和内存泄漏之间的关系可以整理为：

```text
持续分配、内存泄漏或瞬时内存峰值
                    ↓
更多页面被实际写入
                    ↓
Dirty / Compressed Memory 增长
                    ↓
Memory Footprint 上升
                    ↓
       ┌────────────┴────────────┐
       ↓                         ↓
系统可能发送 Memory Warning     达到进程限制或系统压力无法缓解
       │                         ↓
App 有机会调整策略              Jetsam / 内存资源终止
       │                         ↓
但不保证一定先发生              开发者通常称为 OOM
```

这张图表示可能出现的路径，不表示每次终止都必须依次经历所有节点。尤其要避免下面三个等号：

```text
OOM ≠ 内存泄漏
OOM ≠ 一定收到过 Memory Warning
OOM ≠ 普通异常崩溃
```

Jetsam Event Report 与普通 crash report 的诊断重点不同。普通崩溃通常从异常类型、触发线程和调用栈入手；高内存终止发生在进程外部，报告的重点是当时的内存状态、进程状态和终止原因，不一定存在一条能够直接指向业务代码的“崩溃调用栈”。

### App 为什么会被推向 OOM

OOM 不等于内存泄漏。泄漏可能让 footprint 长期单调上升，但没有泄漏的程序也可能因为瞬时峰值超过限制而被终止。常见来源可以分成两类。

#### 持续增长

- **循环引用或所有权错误**：对象已经失去业务价值，却仍被强引用，相关堆页面长期保持 dirty。
- **无上限缓存**：图片、网络响应、模型结果或页面对象只进不出；即使其中一部分被压缩，compressed memory 仍属于 App 的 footprint。
- **不断累积的集合和历史数据**：数组、字典、日志、消息列表或离线数据没有分页、淘汰或持久化边界。
- **生命周期过长的全局对象和单例**：内容一旦创建便长期存活，容易把临时需求变成常驻内存。

这里的“字典缓存”通常具有散列表一类容器的共同成本。除了业务数据本身，容器还需要保存槽位和管理信息；为了降低哈希冲突，内部容量通常不能简单等于当前元素数量。扩容时还可能短暂同时存在旧表和新表，并对已有元素重新定位。因此，无上限字典缓存既会因为持续持有 key 和 value 造成长期增长，也可能在扩容过程中产生额外内存峰值。

散列表并不是一种新的内存分区。它描述的是数据如何组织和查找；容器对象、动态槽位以及它持有的 key、value 最终仍由堆、Mach-O 映射或 Tagged Pointer 等现有存储形式承载。这里关注它对 Footprint 的影响，哈希函数、冲突处理和 rehash 的完整实现留到数据结构或 Runtime 专题展开。

这类问题的典型曲线是：重复同一个业务流程后，Memory Footprint 不断建立新高，并且返回初始页面后也不能接近原来的基线。

#### 瞬时峰值

- **大图解码**：图片文件在磁盘上可能很小，但解码后的位图成本主要取决于像素尺寸和像素格式，而不是 JPEG、HEIF 或 PNG 的文件大小。
- **同时处理多份大数据**：下载数据、解压结果、解析模型和最终对象在某个阶段同时存活，形成短时间的多份副本。
- **并发任务过多**：多个图片解码、视频处理、数据库查询或模型推理同时进行，各自都合理，叠加后却超过限制。
- **大块缓冲区和图形资源**：`Data`、`malloc` 缓冲区、Core Graphics 位图、Metal 资源等都可能快速抬高进程的内存责任。
- **切换实现时产生峰值**：先创建新缓存、新页面或新模型，再释放旧内容，会让两代资源短暂共存。

这类问题可能没有任何泄漏：任务结束后内存本来能够下降，但系统在峰值结束之前已经终止了进程。Apple 在 WWDC18《iOS Memory Deep Dive》中专门强调，应按解码后的尺寸分析图片成本，并通过下采样避免为了生成小图而完整解码原图。

![Snapzy_2026-07-29_16-51-15_745.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/Snapzy_2026-07-29_16-51-15_745.png)

## 六、如何确认和排查一次 OOM

排查时先确认“是不是高内存终止”，再分析“哪类内存造成增长”，最后才回到具体对象和分配调用。不要只凭用户描述的“闪退”或 Xcode 中看到的某个瞬时数字判断。

### 第一步：确认终止类型

- 查找对应的 **Jetsam Event Report** 或系统提供的终止诊断，确认目标进程、终止原因、当时的进程状态和内存数据。
- 如果拿到的是普通 crash report，则先检查异常类型、触发线程和调用栈，不要预设它一定是 OOM。
- 没有收到 `didReceiveMemoryWarning` 不能排除 OOM；收到过警告也不能证明最终一定由高内存终止。

Jetsam 既可能由单进程限制触发，也可能发生在系统整体内存压力下。但不能把它简化成“后台一定先于前台、占用高的一定先于占用低的、用户 App 一定先于系统进程”的固定排序。内核还会考虑进程的 priority band、活跃状态、具体限制、coalition 和本次事件的 `reason`。下面这份真机日志就记录了系统守护进程因 `per-process-limit` 被移除，而当时占用更高的第三方前台进程没有被移除。

#### iPhone 15 真机 JetsamEvent 样本
Xcode 会把已经从设备同步的诊断日志保存在下面这类目录中：

```text
~/Library/Developer/Xcode/DeviceLogs/<设备>/Other Logs/JetsamEvent-*.ips
```

我在自己的 iPhone 15 设备日志中找到了一份 2026 年 6 月 22 日生成的真实 `JetsamEvent`。原始文件约 166 KB，记录了 301 个进程。`.ips` 文件开头是一条元数据 JSON，后面才是报告正文 JSON。下面只保留诊断所需字段，并对 incident、诊断标识和第三方 App 名称进行了脱敏；页数、状态和终止原因仍保留原始值。

元数据：

```json
{
  "bug_type": "298",
  "timestamp": "2026-06-22 10:51:37 +0800",
  "os_version": "iPhone OS 26.5 (23F77)"
}
```

报告正文：

```json
{
  "product": "iPhone15,4",
  "memoryStatus": {
    "pageSize": 16384
  },
  "largestProcess": "<第三方前台进程，已脱敏>",
  "processes": [
    {
      "name": "<第三方前台进程，已脱敏>",
      "states": ["active", "frontmost"],
      "rpages": 73572,
      "lifetimeMax": 87958
    },
    {
      "name": "assetsd",
      "states": ["daemon", "idle"],
      "rpages": 1454,
      "reason": "per-process-limit",
      "lifetimeMax": 4596
    }
  ]
}
```

这份日志最值得注意的地方是：`largestProcess` 和被 Jetsam 移除的进程不是同一个进程。

**1. `pageSize` 决定页数怎样换算**

```text
pageSize = 16,384 bytes = 16 KiB
```

Jetsam 报告中的 `rpages` 和 `lifetimeMax` 都以页为单位，需要乘以 `pageSize` 才能得到字节数：

```text
当前使用量 ≈ rpages × pageSize
生命周期峰值 ≈ lifetimeMax × pageSize
```

**2. 用 `reason` 找出真正被移除的进程**

Apple 的 Jetsam 报告说明指出，`processes` 数组中只有被移除的进程才具有 `reason` 字段。因此，这份日志中真正被移除的是：

```text
name   = assetsd
reason = per-process-limit
```

`per-process-limit` 表示该进程越过了系统为它执行的单进程内存限制。这里的 `assetsd` 是系统守护进程，所以这份日志证明的是一次真实的系统进程 Jetsam，不是本文示例 App 自己发生了 OOM。

**3. 计算被移除进程的当前用量与峰值**

对 `assetsd`：

```text
rpages × pageSize
= 1,454 × 16,384
= 23,822,336 bytes
≈ 22.72 MiB
```

它的生命周期峰值为：

```text
lifetimeMax × pageSize
= 4,596 × 16,384
= 75,300,864 bytes
≈ 71.81 MiB
```

`lifetimeMax` 表示进程生命周期中观测到的最高页数，不是报告直接给出的“配置阈值”。因此不能据此声称 `assetsd` 的固定限制就是 71.81 MiB；系统限制会受到进程类型和系统策略影响。

**4. `largestProcess` 不等于“被杀进程”**

报告中的第三方前台进程在采样时拥有：

```text
rpages = 73,572
当前使用量约为 1,149.56 MiB

lifetimeMax = 87,958
生命周期峰值约为 1,374.34 MiB
```

它确实是当时占用页数最多的进程，所以出现在 `largestProcess`；但它没有 `reason` 字段，因此这份 JetsamEvent 并没有说它在本次事件中被移除。
这也是分析 Jetsam 日志时非常常见的误区：

```text
错误顺序：
看到 largestProcess → 直接认定它 OOM

正确顺序：
在 processes 中搜索 reason
        ↓
确认被移除的进程与终止原因
        ↓
结合 pageSize、rpages、lifetimeMax 和 states 分析
```

**5. `states` 帮助理解进程当时处境**

- 第三方进程是 `active + frontmost`，说明它当时处于前台活动状态；
- `assetsd` 是 `daemon + idle`，说明它是当时处于空闲状态的系统守护进程；
- `states` 可以帮助理解调度和进程状态，但不能替代 `reason` 判断谁被移除。

因此，对这份真实日志的准确结论是：

> iPhone 15 在这次事件中记录了一次 `assetsd` 的 `per-process-limit` Jetsam。另一个第三方前台进程虽然是 `largestProcess`，但没有 `reason`，不是本次报告标记的被移除进程。诊断 Jetsam 时应先搜索 `reason`，再用 `pageSize × rpages` 计算相关进程的内存量。

Apple 的 [Identifying high-memory use with Jetsam Event Reports](https://developer.apple.com/documentation/xcode/identifying-high-memory-use-with-jetsam-event-reports) 给出的官方分析顺序也是先确认 `pageSize` 和 `largestProcess`，再在 `processes` 中搜索 `reason`，最后结合 `rpages`、`states` 与 `lifetimeMax` 判断。

### 第二步：观察增长形态

```text
重复操作后持续上升、很少回落
    → 优先排查泄漏、缓存和对象生命周期

某个操作瞬间陡升，随后本应回落
    → 优先排查图片解码、大缓冲区、多份副本和并发峰值

进入后台后更容易被终止
    → 检查后台仍保留的大资源和后台状态下更紧张的系统预算
```

不同设备、前后台状态以及 App 与 Extension 的限制可能不同，因此不要背诵一个“所有 iPhone 通用的 OOM 阈值”。调试时应在目标设备和真实业务路径上观察趋势与峰值。

### 第三步：选择对应工具

| 想回答的问题 | 更合适的工具 |
| --- | --- |
| Footprint 在哪个操作后开始上升 | Xcode Memory Gauge、埋点采样、重复业务路径对照 |
| 创建了哪些对象、分配量是否持续增长 | Instruments Allocations |
| 是否存在泄漏或循环引用 | Leaks、Xcode Memory Graph |
| Dirty、Compressed 和不同 VM 类型怎样变化 | Instruments VM Tracker；导出 Memgraph 后结合 `vmmap` 分析 |
| 某个大对象或缓冲区从哪里分配 | 开启 Malloc Stack Logging 后采集 Memgraph，再结合 `heap`、`malloc_history` 分析 |
| 线上是否发生高内存终止 | Jetsam Event Report 与系统诊断数据 |

工具之间不能互相替代：Leaks 没有发现泄漏，不代表不存在图片解码或并发任务造成的峰值；Allocations 主要观察堆分配，也不能单独解释所有 VM、图形和压缩内存。正确顺序是先用 footprint 和报告确定现象，再根据增长来源缩小范围。


### 我们能做些什么？

这部分我目前还缺少足够的大型项目治理经验，不适合总结一套“照着做就不会 OOM”的固定清单。不过从前面的页面模型和诊断方法出发，至少可以先做几件明确的事：

- 给缓存设置数量或成本边界，不让字典、图片和历史数据无限增长；
- 加载图片时关注解码后的像素尺寸，需要缩略图时优先用 ImageIO 下采样；
- 限制图片解码、数据解析和模型任务的并发数，避免多个合理任务叠出不合理峰值；
- 避免新旧两份大数据、缓冲区或模型长时间同时存在；
- 在进入后台或页面不可见后，释放当前不需要、又可以重新载入的大资源；
- 检查不必要的启动期初始化、Framework 和全局数据，确认它们是否在启动路径中过早制造长期存活的 dirty memory；
- 用同一台真机重复同一条业务路径，比较基线、峰值、操作结束后的回落和多轮执行后的趋势。

下面两篇工程案例涉及线上监控、归因和治理，适合作为实践层面的延伸阅读：

- [iOS 性能优化实践：头条抖音如何实现 OOM 崩溃率下降 50%+](https://juejin.cn/post/6885144933997494280)
- [你真的了解 OOM 吗？——京东 iOS App 内存优化实录](https://juejin.cn/post/6844904002203697160)

---

## 总结

到这里可以得出五个结论，衔接上一篇的六条：

1. 虚拟地址范围、堆分配量、驻留物理页和 Memory Footprint 是不同指标；内存按页管理，哪怕只写入一个字节，受影响的也可能是整个页面。
2. Clean、Dirty、Compressed 描述的是页面的可重建性和系统记账状态：clean 页面可以从原始来源重建，因此能够被丢弃；dirty 页面不能在不保存内容的情况下直接丢弃，但 App 可以主动释放它，系统也可以压缩它；compressed 页面仍占用物理内存并计入 Footprint，只是压缩后的占用通常更小。
3. 对文件支持的 Copy-on-Write 页面来说，首次写入会建立进程私有副本并使相关页面变 dirty；仅仅建立映射或申请虚拟地址，不代表已经产生等量 dirty memory。
4. iOS 不依赖传统磁盘 Swap 为不断增长的匿名 dirty memory 兜底。回收 clean page、压缩 dirty page、发送 Memory Warning 和通过 Jetsam 终止进程是不同机制，其中 Memory Warning 与 Jetsam 不存在严格的先后因果关系。
5. iOS 开发中常说的 OOM 通常指高内存导致的系统终止，但它不等于内存泄漏、Memory Warning 或某一种固定的崩溃形式；排查时必须区分持续增长与瞬时峰值，并用 Jetsam 报告、Memory Footprint 和对应工具建立证据链。

结合上一篇的六条结论，两篇文章合起来回答了同一个问题的两个层面：一段内存"在哪里、从哪里来"，以及它"现在是否真的占用物理内存、系统怎样为它记账"。地址空间部分见系列上一篇[《iOS 内存地图：从虚拟地址空间到 VM Region》](/blog/ios-memory-map-virtual-address-space-vm-region/)。

## 参考资料

- [Apple — Gathering information about memory use](https://developer.apple.com/documentation/xcode/gathering-information-about-memory-use)
- [Apple — Reducing your app's memory use](https://developer.apple.com/documentation/xcode/reducing-your-app-s-memory-use)
- [Apple — Responding to memory warnings](https://developer.apple.com/documentation/uikit/responding-to-memory-warnings)
- [Apple — Responding to low-memory warnings](https://developer.apple.com/documentation/xcode/responding-to-low-memory-warnings)
- [Apple — NSCache](https://developer.apple.com/documentation/foundation/nscache)
- [Apple — Viewing Virtual Memory Usage](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/VMPages.html)
- [Apple — Identifying high-memory use with Jetsam Event Reports](https://developer.apple.com/documentation/xcode/identifying-high-memory-use-with-jetsam-event-reports)
- [Apple — Reduce terminations in your app](https://developer.apple.com/documentation/xcode/reduce-terminations-in-your-app)
- [WWDC18 — iOS Memory Deep Dive](https://developer.apple.com/videos/play/wwdc2018/416/)
- [iOS 性能优化实践：头条抖音如何实现 OOM 崩溃率下降 50%+](https://juejin.cn/post/6885144933997494280)
- [你真的了解 OOM 吗？——京东 iOS App 内存优化实录](https://juejin.cn/post/6844904002203697160)
- [iOS Memory 内存详解（长文）](https://juejin.cn/post/6844903902169710600#heading-22)
- https://www.mikeash.com/pyblog/friday-qa-2009-03-13-intro-to-the-objective-c-runtime.html
- https://www.mikeash.com/pyblog/friday-qa-2012-12-28-what-happens-when-you-load-a-byte-of-memory.html