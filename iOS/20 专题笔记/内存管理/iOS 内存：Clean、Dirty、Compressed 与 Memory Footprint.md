---
title: 【iOS】内存记账：Clean、Dirty、Compressed 与 Memory Footprint
published: 2026-07-24
description: 从'地址在哪里'切换到'现在占用多少'：梳理 Clean、Dirty、Compressed 页面状态，Copy-on-Write 如何让页面变脏，以及 Memory Footprint、OOM、内存警告与 Jetsam 的关系。
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
 # iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint

## 前言

系列上一篇 [[iOS 内存：从虚拟地址空间到堆与栈]] 回答的是"一段地址属于哪里、从哪里来"：五大分区给出用途分类，Mach-O、堆和线程栈解释这些区域怎样形成，VM Region 是内核描述它们的真实方式。

但"申请了多大的虚拟地址范围"和"App 当前要为多少物理内存负责"是两个问题。这一篇把观察单位从变量和 Region 切换到内存页，继续追问：这些页面能否重新取得、是否已经被写入、系统怎样计算 Memory Footprint，以及内存压力下系统会做什么。

## 本文主线

全文按照下面的顺序展开：

```text
申请虚拟地址不等于立刻产生等量的物理内存占用
        ↓
Clean、Dirty、Compressed 描述页面此刻的内存状态
        ↓
Copy-on-Write 是页面从"可丢弃"变成"进程私有"的触发点
        ↓
iOS 没有传统磁盘 Swap，内存压力下依赖压缩与终止进程
        ↓
Memory Warning 与 Jetsam 是两套独立机制，不是一条因果链
        ↓
准确区分 OOM、内存泄漏、内存峰值与 Jetsam
```

这篇文章不再重复"这段地址用来做什么、从哪里来"——那是上一篇的内容。这里只回答一个问题：这些页面此刻是否真的占用物理内存，系统怎么为它记账。

---

## 申请内存不等于立刻产生等量的 Memory Footprint

首先推荐阅读：[当你加载一字节内存会发生什么](https://github.com/Biscoffee/apple-docs-vault/blob/main/blogs/zh/mikeash/friday-q-a-2012-12-28-what-happens-when-you-load-a-byte-of-memory.md)

Apple 在 WWDC18《iOS Memory Deep Dive》中以典型的 16 K B 页面说明：内存系统以页为粒度管理和统计资源，一个页面可以容纳多个堆对象，一个对象也可能跨越多个页面。因此，哪怕程序只修改其中一个字节，受影响的仍可能是整个页面。

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

### **Clean memory**

可以简单理解为能够被写入数据的干净内存。对开发者而言是read-only，而iOS系统可以写入或移除。

1. System Framework、Binary Executable占用的内存
2. 可以被释放（Page Out，iOS上是压缩内存的方式）的文件，包括内存映射文件Memory mapped file（如image、data、model等）。内存映射文件通常是只读的。
3. 系统中可回收、可复用的内存，实际不会立即申请到物理内存，而是真正需要的时候再给。
4. 每个framework都有_DATA_CONST段，当App运行时使用到了某个framework，该framework对应的_DATA_CONST的内存就由clean变为dirty了。

注意：如果通过文件内存映射机制memory mapped file载入内存的，可以先清除这部分内存占用，需要的时候再从文件载入到内存。所以是Clean Memory。


### **Dirty memory**：

主要强调不可被重复使用的内存。对开发者而言，已经写入数据。

1. 被写入数据的内存，包括所有heap中的对象、图像解码缓冲(ImageIO, CGRasterData，IOSurface)。
2. 已使用的实际物理内存，系统无法自动回收。
3. heap allocation、caches、decompressed images。
4. 每个framework的_DATA段和_DATA_DIRTY段。

iOS中的内存警告，只会释放clean memory。因为iOS认为dirty memory有数据，不能清理。所以，应尽量避免dirty memory过大。

要清楚地知道Allocations和Dirty Size分别是因为什么？

> 值得注意的是，在使用 framework 的过程中会产生 Dirty Memory，使用单例或者全局初始化方法是减少 Dirty Memory 不错的方法，因为单例一旦创建就不会销毁，全局初始化方法会在 class 加载时执行。

下方有测量实验，如+50dirty的操作，在release环境不生效，因iOS系统自动做了优化。


>Example:
申请一块长度为 80000 字节的内存空间，按照一页 16KB 来计算，就需要 6 页内存来存储。
>- 当这些内存页开辟出来的时候，它们都是 `Clean` 的
>- 当向处于第一页的内存写入数据时，第一页内存会变成 `Dirty`
>- 当向处于最后一页的内存写入数据时，这一页也会变成 `Dirty`
>![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260729181948297.png)



### **Compressed memory**：
iOS设备没有swapped memory，而是采用Compressed Memory机制，一般情况下能将目标内存压缩至原有的一半以下。对于缓存数据或可重建数据，尽量使用NSCache或NSPurableData，收到内存警告时，系统自动处理内存释放操作。并且是线程安全的。

这里要注意，压缩内存机制，使得内存警告与释放内存变得稍微复杂一些。即，对于已经被压缩过的内存，如果尝试释放其中一部分，则会先将它解压。而解压过程带来的内存增大，可能得到我们并不期待的结果。如果选用NSDictionary之类的，内存比较紧张时，尝试将NSDictionary的部分内存释放掉。但若NSDictionary之前是压缩状态，释放需要先解压，解压过程可能导致内存增大而适得其反。

所以，我们平常开发所关心的内存占用其实是 _**Dirty Size和Compressed Size两部分**_，也应尽量优化这两部分。而Clean Memory一般不用太多关注。


这三者不是三个互相独立的分类，而是同一个页面在压力下可能经历的一条状态链：

```text
共享/文件映射（可能是 clean） --首次写入触发 COW--> 私有 dirty 页 --内存压力--> compressed
```

### 内存占用

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260729183653223.png)
对于 app 来说，我们主要关心的内存是 dirty memory，当然其中也包含 compressed memory。而对于 clean memory，作为开发者通常可以不必关心。

当内存占用的部分过大，就会发生前文所说的内存警告以及 OOM 崩溃等情况，所以我们应该尽可能的减少内存占用，并对内存警告以及 OOM 崩溃做好防范。减少内存占用也能侧面提升启动速度，要加载的内存少了，自然启动速度会变快。

按照正常的思路，app 监听到内存警告时应该主动清理释放掉一些优先级低的内存，这本质上是没错的。不过由于 compressed memory 的特殊性，所以导致内存占用的实际大小考虑起来会有些复杂。

![Snapzy_2026-07-29_18-33-54_314.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/Snapzy_2026-07-29_18-33-54_314.png)

比如上面这种情况，当我们收到内存警告时，我们尝试将 Dictionary 中的部分内容释放掉，但由于之前的 Dictionary 由于未使用，所以正处于被压缩状态；而解压、释放部分内容之后，Dictionary 处于未压缩状态，可能并没有减少物理内存，甚至可能反而让物理内存更大了。

所以，进行缓存更推荐使用 NSCache 而不是 NSDictionary，就是因为 NSCache 不仅线程安全，而且对存在 compressed memory 情况下的内存警告也做了优化，可以由系统自动释放内存。

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

### iOS 语境中的 OOM

OOM 是 **Out of Memory** 的缩写，指的是在 iOS 设备上当前应用因为内存占用过高而被操作系统强制终止，在用户侧的感知就是 App 一瞬间的闪退，与普通的 Crash 没有明显差异。但是当我们在调试阶段遇到这种崩溃的时候，从设备`设置->隐私->分析与改进`中是找不到普通类型的崩溃日志，只能够找到`Jetsam`开头的日志，这种形式的日志其实就是 OOM 崩溃之后系统生成的一种专门反映内存异常问题的日志。

但在 iOS 开发讨论中，它经常被用来概括几种并不完全相同的结果：

1. **某次内存申请失败**：从通用编程语义看，这是最直接的 OOM。例如 `malloc` 在无法满足申请时可能返回 `NULL`，其他 API 也可能按照自己的约定返回错误或抛出异常。
2. **进程超过自身的 Memory Footprint 限制**：App 持续增加 dirty 与 compressed memory，逼近或超过当前设备、进程类型和运行状态对应的限制，系统可能通过资源异常或 Jetsam 终止进程。
3. **系统整体内存压力下被选中终止**：App 未必单独达到一个固定数字，但系统需要迅速回收资源时，内核可能根据进程优先级、状态和内存责任选择它作为 Jetsam 对象。

真实 iOS App 往往在一次普通分配明确返回失败之前，就已经因为内存压力被系统从外部终止。因此，日常所说的“App 发生 OOM”，通常指的是**高内存使用导致的系统终止**，但分析报告时仍应确认实际终止原因，不能看到“突然消失”就直接下结论。

OOM 分为两大类，Foreground OOM / Background OOM，简写为 FOOM 以及 BOOM。而其中 FOOM 是指 app 在前台时由于消耗内存过大，而被系统杀死，直接表现为 crash。

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

### 如何确认和排查一次 OOM

排查时先确认“是不是高内存终止”，再分析“哪类内存造成增长”，最后才回到具体对象和分配调用。不要只凭用户描述的“闪退”或 Xcode 中看到的某个瞬时数字判断。

#### 第一步：确认终止类型

- 查找对应的 **Jetsam Event Report** 或系统提供的终止诊断，确认目标进程、终止原因、当时的进程状态和内存数据。
- 如果拿到的是普通 crash report，则先检查异常类型、触发线程和调用栈，不要预设它一定是 OOM。
- 没有收到 `didReceiveMemoryWarning` 不能排除 OOM；收到过警告也不能证明最终一定由高内存终止。

`Jetsam`机制清理策略可以总结为下面两点：

1. 单个 App 物理内存占用超过上限
2. 整个设备物理内存占用收到压力按照下面优先级完成清理：
3. 1. 后台应用>前台应用
    2. 内存占用高的应用>内存占用低的应用
    3. 用户应用>系统应用

##### 真机 JetsamEvent 样本：

Xcode 会把已经从设备同步的诊断日志保存在下面这类目录中：

```text
~/Library/Developer/Xcode/DeviceLogs/<设备>/Other Logs/JetsamEvent-*.ips
```

我在自己的 iPhone 15 设备日志中找到了一份 2026 年 6 月 22 日生成的真实 `JetsamEvent`。原始文件约 166 KB，记录了 301 个进程。下面只保留诊断所需字段，并对 incident、诊断标识和第三方 App 名称进行了脱敏；页数、状态和终止原因仍保留原始值：
```json
{
  "bug_type": "298",
  "timestamp": "2026-06-22 10:51:37 +0800",
  "os_version": "iPhone OS 26.5 (23F77)"
}

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

1. `pageSize` 决定页数怎样换算
```text
pageSize = 16,384 Byte = 16 KiB
```
Jetsam 报告中的 `rpages` 和 `lifetimeMax` 都以页为单位，需要乘以 `pageSize` 才能得到字节数：
```text
当前使用量 ≈ rpages × pageSize
生命周期峰值 ≈ lifetimeMax × pageSize
```

2. 用 `reason` 找出真正被移除的进程
Apple 的 Jetsam 报告说明指出，`processes` 数组中只有被移除的进程才具有 `reason` 字段。因此，这份日志中真正被移除的是：

```text
name   = assetsd
reason = per-process-limit
```

`per-process-limit` 表示该进程越过了系统为它执行的单进程内存限制。这里的 `assetsd` 是系统守护进程，所以这份日志证明的是一次真实的系统进程 Jetsam，不是本文示例 App 自己发生了 OOM。

3. 计算被移除进程的当前用量与峰值

对 `assetsd`：
```text
rpages × pageSize
= 1,454 × 16,384
= 23,822,336 Byte
≈ 22.72 MiB
```
它的生命周期峰值为：
```text
lifetimeMax × pageSize
= 4,596 × 16,384
= 75,300,864 Byte
≈ 71.81 MiB
```
`lifetimeMax` 表示进程生命周期中观测到的最高页数，不是报告直接给出的“配置阈值”。因此不能据此声称 `assetsd` 的固定限制就是 71.81 MiB；系统限制会受到进程类型和系统策略影响。

4. `largestProcess` 不等于“被杀进程”

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

5. `states` 帮助理解进程当时处境

- 第三方进程是 `active + frontmost`，说明它当时处于前台活动状态；
- `assetsd` 是 `daemon + idle`，说明它是当时处于空闲状态的系统守护进程；
- `states` 可以帮助理解调度和进程状态，但不能替代 `reason` 判断谁被移除。

因此，对这份真实日志的准确结论是：

> iPhone 15 在这次事件中记录了一次 `assetsd` 的 `per-process-limit` Jetsam。另一个第三方前台进程虽然是 `largestProcess`，但没有 `reason`，不是本次报告标记的被移除进程。诊断 Jetsam 时应先搜索 `reason`，再用 `pageSize × rpages` 计算相关进程的内存量。

Apple 的 [Identifying high-memory use with Jetsam Event Reports](https://developer.apple.com/documentation/xcode/identifying-high-memory-use-with-jetsam-event-reports) 给出的官方分析顺序也是先确认 `pageSize` 和 `largestProcess`，再在 `processes` 中搜索 `reason`，最后结合 `rpages`、`states` 与 `lifetimeMax` 判断。

#### 第二步：观察增长形态

```text
重复操作后持续上升、很少回落
    → 优先排查泄漏、缓存和对象生命周期

某个操作瞬间陡升，随后本应回落
    → 优先排查图片解码、大缓冲区、多份副本和并发峰值

进入后台后更容易被终止
    → 检查后台仍保留的大资源和后台状态下更紧张的系统预算
```

不同设备、前后台状态以及 App 与 Extension 的限制可能不同，因此不要背诵一个“所有 iPhone 通用的 OOM 阈值”。调试时应在目标设备和真实业务路径上观察趋势与峰值。

#### 第三步：选择对应工具

| 想回答的问题 | 更合适的工具 |
| --- | --- |
| Footprint 在哪个操作后开始上升 | Xcode Memory Gauge、埋点采样、重复业务路径对照 |
| 创建了哪些对象、分配量是否持续增长 | Instruments Allocations |
| 是否存在泄漏或循环引用 | Leaks、Xcode Memory Graph |
| Dirty、Compressed 和不同 VM 类型怎样变化 | VM Tracker、Memgraph |
| 某个大对象或缓冲区从哪里分配 | 开启 Malloc Stack Logging 后采集 Memgraph，再结合分配历史分析 |
| 线上是否发生高内存终止 | Jetsam Event Report 与系统诊断数据 |

工具之间不能互相替代：Leaks 没有发现泄漏，不代表不存在图片解码或并发任务造成的峰值；Allocations 主要观察堆分配，也不能单独解释所有 VM、图形和压缩内存。正确顺序是先用 footprint 和报告确定现象，再根据增长来源缩小范围。


#### 我们能做些什么？

这部分由于笔者展示缺乏真实的开发经验，可以阅读参考资料中的
- [# iOS 性能优化实践：头条抖音如何实现 OOM 崩溃率下降50%+](https://juejin.cn/post/6885144933997494280?searchId=20260725091107C08106C1FFC05727A26C)
- [# 你真的了解OOM吗？——京东iOS APP内存优化实录](https://juejin.cn/post/6844904002203697160?searchId=202607242005413F8E66D396B122E9CEF3)
两篇链接

---

## 总结

到这里可以得出五个结论，衔接上一篇的六条：

1. 虚拟地址范围、堆分配量、驻留物理页和 Memory Footprint 是不同指标；内存按页管理，哪怕只写入一个字节，受影响的也可能是整个页面。
2. Clean、Dirty、Compressed 描述的是页面的可重建性和系统记账状态：clean 页面能从原始来源重建，因此可以直接丢弃；dirty 页面不能，只能压缩或转移风险；compressed 页面数据仍在，只是暂时不占物理内存。
3. Copy-on-Write 是页面从"共享、可丢弃"变成"私有、必须记账"的触发点：首次写入才会真正产生 dirty page，仅仅"映射"或"分配"不会。
4. iOS 不依赖传统磁盘 Swap 为不断增长的匿名 dirty memory 兜底；内存压缩、Memory Warning 与 Jetsam 是三种不同强度的应对手段，其中 Memory Warning 和 Jetsam 是两套独立机制，不存在严格的先后因果关系。
5. iOS 开发中常说的 OOM 通常指高内存导致的系统终止，但它不等于内存泄漏、Memory Warning 或某一种固定的崩溃形式；排查时必须区分持续增长与瞬时峰值，并用 Jetsam 报告、Memory Footprint 和对应工具建立证据链。

结合上一篇的六条结论，两篇文章合起来回答了同一个问题的两个层面：一段内存"在哪里、从哪里来"，以及它"现在是否真的占用物理内存、系统怎样为它记账"。地址空间部分见系列上一篇 [[iOS 内存：从虚拟地址空间到堆与栈]]。

## 参考资料

- [Apple — Gathering information about memory use](https://developer.apple.com/documentation/xcode/gathering-information-about-memory-use)
- [Apple — Reducing your app's memory use](https://developer.apple.com/documentation/xcode/reducing-your-app-s-memory-use)
- [Apple — Responding to memory warnings](https://developer.apple.com/documentation/uikit/responding-to-memory-warnings)
- [Apple — Identifying high-memory use with Jetsam Event Reports](https://developer.apple.com/documentation/xcode/identifying-high-memory-use-with-jetsam-event-reports)
- [Apple — Reduce terminations in your app](https://developer.apple.com/documentation/xcode/reduce-terminations-in-your-app)
- [WWDC18 — iOS Memory Deep Dive](https://developer.apple.com/videos/play/wwdc2018/416/)
- [# iOS 性能优化实践：头条抖音如何实现 OOM 崩溃率下降50%+](https://juejin.cn/post/6885144933997494280?searchId=20260725091107C08106C1FFC05727A26C)
- [# 你真的了解OOM吗？——京东iOS APP内存优化实录](https://juejin.cn/post/6844904002203697160?searchId=202607242005413F8E66D396B122E9CEF3)
- [iOS Memory 内存详解 (长文)](https://juejin.cn/post/6844903902169710600?searchId=202607242005413F8E66D396B122E9CEF3#heading-22)