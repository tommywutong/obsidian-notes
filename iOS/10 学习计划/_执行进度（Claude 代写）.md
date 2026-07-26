---
title: 2026 暑假 iOS 底层学习计划 · 代写执行进度
published: 2026-07-26
description: Claude 按《2026 暑假 iOS 底层学习计划》代写笔记的文档清单、约定与进度台账。会话中断后从这里恢复上下文。
tags:
  - iOS
  - Meta
category: iOS
draft: true
---

# 代写执行进度台账

> 用途：这份文件是**执行状态的唯一真相源**。会话中断 / 换窗口后，先读这里恢复上下文，不要凭记忆推进。
> 母计划：[[2026 暑假 iOS 底层学习计划]]

## 一、已确认的范围与约定

- **范围**：第一周 ~ 第七周主线 + 第八阶段，约 37 篇。**不含**全程并行线（操作系统 / 网络 / 数据库 / 算法）——2026-07-26 再次确认。
- **第八阶段的「综合小项目」**：只写设计文档（模块划分、数据流、类职责、每个技术点回指哪一周的笔记），**不建 Xcode 工程**。
- **粒度**：相近内容合并成篇，但比"一周一篇"更精细——一个大主题拆成 2~5 篇，避免覆盖不全。
- **内容来源（2026-07-26 调整）**：不再套固定模板写作，改为「广读 + 提炼」。三层来源：
  1. 计划正文里的 **290 个链接**（去重后），按主题分配到每篇文章，真读真提炼，不是列在参考资料里充数；
  2. 自行搜索补充：objc4 / libclosure / dyld 源码、WWDC session、结论在新系统上是否仍成立；
  3. 本地已有笔记（见第二节的复用判定）。
- **抓取工具的覆盖差异**：`WebFetch` 拿英文站没问题；简书、CSDN 会返回 403，需改用 `mcp__web-fetcher__fetch_url`。两者互补。
- **写作流程（三段式）**：
  1. **研究 agent**（每篇一个）——读计划里分配给该篇的链接 + 自行检索补充，产出高密度技术简报，要求标注矛盾点、过时说法、可跑的实验设计；
  2. **主会话落笔**——统一由主会话写，避免多 agent 各写各的导致文风散；能真跑的实验主会话亲自跑，用来验证简报里的关键论断；
  3. **审查 agent**（每篇一个）——事实核查（逐条比对 objc4 等一手源码）、找覆盖缺口、抓「人机味」到具体句子、评结构，产出「必须改 / 建议改 / 可选」三档清单，主会话据此修订。
- **文风约束**（2026-07-26 用户两次提出「人机味」后定的硬指标）：
  - 加粗全文控制在个位数，只用于真正反直觉的结论
  - 表格只用于真正的多维对照，两句话能说清的不列表
  - 允许一句话成段，允许段落长短悬殊
  - 不强制每节收尾，避免「引入—展开—小结」的模板节奏
  - 总结压到五条以内，或直接写成段落
  - 该下判断就下判断，允许写「这个说法我不同意」
- **实验纪律**（重要，这是母计划的红线）：
  - 命令行 / iOS 模拟器能跑的实验 → **真跑真贴输出**（clang、`clang -S -emit-llvm`、`clang -rewrite-objc`、lldb、otool、nm、xcrun simctl、xctrace）。
  - 真机专属数值（内存压力阈值、Jetsam、真机 tagged pointer 位布局、性能耗时）→ 标注 `> 待真机补测` 占位，**不伪造数据**。
  - 模拟器性能数字只能用来验证**行为逻辑**，不能拿来下"优化有效"的结论。
- **写作约定**（跟随 `20 专题笔记/内存管理/` 已有两篇）：
  - 文件名：`iOS <主题>：<副标题>.md`
  - frontmatter：`title: 【iOS】...`、`series: 2026 暑假 iOS 底层学习`、`seriesSlug: ios-internals-2026-summer`、`seriesOrder: <递增>`、`draft: true`
  - 结构：前言 → 主线推进 → 实验（含命令 + 真实输出）→ 稳定语义 vs 当前实现 → 总结 → 参考资料
- **实验工作区**：`/tmp/ios-notes-lab/<wNdM>/`（临时，不进 vault）

## 二、复用 / 重写判定（探查结论）

| 已有内容 | 判定 |
| --- | --- |
| `iOS/Runtime/Part 1 - 对象与类的本质.md`（88KB） | 已完成，质量高 → **直接引用，不重写** |
| `iOS/Runtime/Part 2 - 消息发送与转发.md`（146KB） | 同上 → **引用** |
| `iOS/Runtime/Part 3 - Category.md`（57KB） | 同上 → **引用** |
| `iOS/Runtime/Part 4 - Runtime 应用篇.md`（72KB） | draft，需**补完**（Swizzling / +load / KVO isa-swizzling） |
| `iOS/Runtime/Method - Swizzling.md` | 与 Part 4 重叠 → 合并进 Part 4 后作为素材保留 |
| `20 专题笔记/内存管理/…从虚拟地址空间到堆与栈.md`（700 行） | 基本完成 → 只做收尾 / 翻 draft |
| `20 专题笔记/内存管理/…Clean、Dirty、Compressed….md`（206 行） | **已成文**（含真机实测 + 总结 + 参考），与"刚开始"的印象不符 → 保留，只补链接与 draft 状态 |
| `20 专题笔记/并发与运行循环/GCD.md`（28KB） | 旧碎片笔记 → **重写为正式长文** |
| `20 专题笔记/并发与运行循环/RunLoop 与 AutoReleasePool.md`（7.5KB） | 偏薄 → **重写** |
| `20 专题笔记/编译链接与启动/Mach-O.md`（980B） | stub → **重写** |
| `20 专题笔记/编译链接与启动/dyld.md`（34KB） | 可用 → 补"三代 dyld 历史 / 启动阶段表" |
| `csdn-import/` KVC、MVC、MVVM、SDWebImage | 早年自写，质量可用 → **引用不重写** |
| `csdn-import/` KVO、UITableView | 自己标了"这里先埋个坑" → 新文档**填坑** |

## 三、文档清单（32 篇待写 + 2 篇已有 = 34）

`[ ]` 未开始 · `[~]` 进行中 · `[x]` 完成

### 第一周：对象、类与所有权
- [x] 01 iOS 内存：从虚拟地址空间到堆与栈（内存管理/）—— 已有，待收尾
- [x] 02 iOS 内存：Clean、Dirty、Compressed 与 Memory Footprint（内存管理/）—— 已有
- [x] 03 iOS 对象模型：类型判断、内存对齐与 Tagged Pointer（对象模型/）
      对应计划 Day 3。实验全部真跑：10 组类型判断、ivar 偏移与两种对齐、Tagged Pointer 探针、
      三次启动的混淆器对照、字面量常量折叠对照、`CFGetRetainCount` = INT64_MAX。
      **修正**：arm64 标记位在 **bit63**，不是网上常说的最低位；`malloc_size == 0` 不足以判定 tagged。
- [x] 04 iOS 内存：MRC 的所有权规则（内存管理/）—— 对应 Day 4，审查中
      **实测硬料**：按 `isa.h` 位域解析 isa，反复 retain 到溢出。第 253 次 `extra_rc` 跑满 255，
      第 254 次 `has_sidetable_rc` 翻 1、`extra_rc` 回落 128（= `RC_HALF`）。
      **纠正**：中文资料通说的「extra_rc 占 19 位」只对无 ptrauth 的 arm64 成立；arm64e 与模拟器上是 8 位。
- [x] 05 iOS 内存：ARC 的两半（内存管理/）—— 对应 Day 5，审查中
      **实测硬料**：六个场景的 LLVM IR，`-O0` vs `-O1` 对照（`setLabel` 从 4 个 storeStrong 变 0）；
      arm64 的 `mov x29, x29` marker vs x86_64 无 marker 的双架构汇编对照。
      **意外发现**：合成的 nonatomic strong getter 在 -O0 下一个 objc_* 调用都不生成。

### 第二周：weak、属性关键字与 Block
- [x] 06 属性关键字：从所有权推导，而不是从类型名猜（内存管理/）—— Day 1，审查中
      **实测硬料**：atomic 属性十万次并发自增丢掉一半计数，且比 nonatomic 丢得更多；加锁才 100000。
      长字符串 / NSArray / NSDictionary 的 copy 返回同一对象；容器 mutableCopy 只复制一层；
      `copy` 修饰 NSMutableArray 实际存进 `__NSSingleObjectArrayI`，addObject: 抛 unrecognized selector。
- [ ] 07 weak 的实现：SideTable、weak_table_t 与置 nil 时机（内存管理/）—— Day 2
- [ ] 08 Block 的结构：ABI、descriptor 与三种类型（Block/）—— Day 3
- [ ] 09 Block 的变量捕获与 __block（Block/）—— Day 4
- [ ] 10 Block 循环引用与 weak-strong dance（Block/）—— Day 5

### 第三周：Runtime 行为与对象通信（Runtime/、Runtime 与对象通信/）
- [ ] 08 补完 `Runtime/Part 4 - Runtime 应用篇.md`
- [ ] 09 KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发
- [ ] 10 Cocoa 对象通信四件套：delegate / notification / target-action / block 回调选型

### 第四周：线程、GCD、Operation 与锁（并发与运行循环/）
- [ ] 11 GCD 重写：队列 / 同步异步 / 死锁 / group / barrier / semaphore
- [ ] 12 NSOperation 与 NSOperationQueue：依赖、取消与自定义并发 Operation
- [ ] 13 iOS 锁全景：从 OSSpinLock 到 os_unfair_lock、@synchronized、pthread、NSLock 家族

### 第五周：RunLoop、AutoreleasePool、响应者链与生命周期
- [ ] 14 RunLoop 重写：mode / source / timer / observer 与真实循环流程（并发与运行循环/）
- [ ] 15 AutoreleasePool：哨兵页、嵌套与 RunLoop 的关系（内存管理/）
- [ ] 16 事件传递与响应者链：hitTest → 响应链 → 手势冲突（UIKit 与渲染/）
- [ ] 17 UIViewController 生命周期与容器控制器（UIKit 与渲染/）
- [ ] 18 frame / bounds / center / transform 与坐标系换算（UIKit 与渲染/）

### 第六周：UIKit 渲染与性能（UIKit 与渲染/、Foundation 与集合/）
- [ ] 19 UIView 与 CALayer：树结构、绘制流水线、离屏渲染
- [ ] 20 UITableView：复用机制、代理调用顺序与卡顿排查（填 csdn-import 的坑）
- [ ] 21 NSArray / NSDictionary / NSSet 的实现与选型（Foundation 与集合/）

### 第七周：编译、链接、Mach-O、dyld 与启动（编译链接与启动/）
- [ ] 22 从源码到可执行文件：预处理 / 编译 / 汇编 / 链接四段
- [ ] 23 Mach-O 重写：header / segment / section / 符号表与 otool 解剖
- [ ] 24 静态库与动态库：符号解析、`-all_load`、重复符号与 rpath
- [ ] 25 dyld 补章：三代 dyld、启动阶段耗时表
- [ ] 26 App 启动：pre-main / main / 首帧，以及可测量的优化项

### 第八阶段：持久化、序列化、源码与架构
- [ ] 27 iOS 持久化选型：plist / NSUserDefaults / 归档 / 文件 / Keychain（持久化与序列化/）
- [ ] 28 SQLite 与 FMDB / Core Data：事务、索引与并发（持久化与序列化/）
- [ ] 29 序列化：NSCoding / NSSecureCoding / JSON 三条路（持久化与序列化/）
- [ ] 30 JSONModel 源码：Runtime 驱动的属性映射（持久化与序列化/）
- [ ] 31 YYModel 源码：为什么比 JSONModel 快（持久化与序列化/）
- [ ] 32 SDWebImage：完整下载 → 解码 → 缓存链路（持久化与序列化/）
- [ ] 33 MVC / MVP / MVVM：职责边界与在 iOS 里的落地（架构与网络/）
- [ ] 34 网络分层：URLSession → 请求层 → 业务层，以及缓存与重试（架构与网络/）

> 母计划第八阶段还要求一个"综合小项目"作为收口作品——这需要在 Xcode 里跑通完整 App，超出纯文档范围，暂列为待议项。

## 三点五、审查轮发现的错误（存档，避免重犯）

第一批四篇过审查后改掉的硬错误。共同点很明显：**凡是来自记忆或转述的断言都出过错，凡是来自编译产物的断言都对。**

| 文章 | 原来写的 | 实际 |
| --- | --- | --- |
| Tagged Pointer | ivar 重排能省内存（照抄流行说法） | `{char a; id b;}` 和 `{id b; char a;}` 都是 24，**一个字节不省**；要两个以上小成员能合并才见效 |
| Tagged Pointer | 除标记位外所有位每次启动都变 | `bit62` 三次纹丝不动且按类分组；低 3 位是标签索引，跨启动被置换而非异或。**我自己的数据就能证伪自己的结论** |
| Tagged Pointer | 对象最少 16 字节来自 malloc 粒度 | 来自 objc4 的 `if (size < 16) size = 16;`，注释写着 CF 要求。malloc 粒度解释的是 24→32 那档 |
| Tagged Pointer | arm64 用 `OBJC_MSB_TAGGED_POINTERS` | 用 `OBJC_SPLIT_TAGGED_POINTERS`，标签索引在低位。LSB 只出现在 Intel Mac 的原生 macOS 进程，iOS 模拟器不算 |
| Tagged Pointer | 关联对象"语义有问题" | 能设能读不崩；真正的问题是两个同值对象共享同一份 + 永不 dealloc 导致永久泄漏 |
| Tagged Pointer | 并发赋值崩溃的例子 | 必须写 `nonatomic`，atomic 的 setter 在锁内换槽位恰恰不会过度释放 |
| MRC | `extra_rc` = retainCount − 1 | **就是 retainCount**（`initIsa` 里 `extra_rc = 1`）。正因如此旧版的 `deallocating` 位被删了 |
| MRC | isa 里有 `deallocating` 位 | 没有，靠 `extra_rc == 0 && has_sidetable_rc == 0` 推 |
| MRC | 快速路径四个条件都在 isa 位里 | `has_cxx_dtor` 在 arm64e/模拟器上不在 isa 里（`ISA_HAS_CXX_DTOR_BIT` 为 0），要查类标志 |
| MRC | `PROTECT_AUTORELEASEPOOL` 是 Debug 开关 | 发布版 objc4 里**根本没定义**，要自己重编 libobjc |
| ARC | `objc_alloc_init` 出自 WWDC20 | 是 iOS 13：`-fobjc-runtime=ios-12.0` 只有 `objc_alloc`，`ios-13.0` 才有 |
| ARC | 返回值优化靠读调用方指令流 | arm64 已换成 TLS 暂存 + 返回地址差值（4/8），读指令降级为兜底。x86_64 仍走老路 |
| ARC | 合成 getter 不插桩是因为编译器判定 ivar 不会被释放 | 是 CodeGen 对合成访问器的专门捷径：手写同形状代码照样生成 `retainAutoreleaseReturnValue` |
| ARC | `objc_claimAutoreleasedReturnValue` 在 Clang ARC 规范里 | 规范里没有，是 objc4 自己加的入口点 |
| ARC | ARC 完全不追踪引用关系 | 强引用不追踪；`__weak` 恰恰是在 SideTable 里记"谁指向我" |

第二批（属性关键字复审 + weak）：

| 文章 | 原来写的 | 实际 |
| --- | --- | --- |
| 属性关键字 | `nonatomic copy` 的 setter 是编译器内联 | 走 `objc_setProperty_nonatomic_copy`。clang `PropertyImplStrategy` 里写死："If we have a copy property, we always have to use setProperty." |
| 属性关键字 | atomic 丢得比 nonatomic 多，因为抢锁变慢、重叠窗口变大 | **纯属编造**。标量属性的 atomic/nonatomic setter 是逐字节相同的 `str`，不碰 PropertyLocks。跑满六次两边互有胜负，是噪声。我只跑两次就下了方向性结论还配了机制 |
| 属性关键字 | weak 比直接指针访问慢一个量级 | 实测 2.7–4.1 倍（weak 46ns vs strong 11–17ns），不是 10 倍 |
| 属性关键字 | `objc_storeStrong` 返回 id、`objc_storeWeak` 返回 void | 正好写反了；且 `shouldCopy` 是 `signed char` 不是 `BOOL`（要容纳 `MUTABLE_COPY = 2`） |
| 属性关键字 | 这些访问器函数都没有公开声明 | `objc_storeWeak` / `objc_loadWeak` 在 `objc/runtime.h` 里是公开 API |
| 属性关键字 | `tintColor` 设 nil 回到系统默认色 | 是沿 superview 链找第一个非默认值，找不到才落到系统默认 |
| 属性关键字 | Block 属性 copy vs strong「实测没观察到差异」 | 不是没观察到，是 clang AST 里规定等价（`isBlockPointerType() ? Copy : Retain`）。但 `retain` 真的只 retain 不 copy，有专门警告 |

| weak | Swift 用双计数器 + 对象变僵尸 | 那是 Swift 3 的行为。Swift 4 起是三计数（strong/unowned/weak），side table 惰性分配，weak 变量指向 side table 而非对象，对象内存在 unowned 归零时就 free |
| weak | MRC 下能直接用 `objc_storeWeak`，只是没有 `__weak` 语法糖 | MRC 写 `__weak` 是硬编译错误；但 `-fobjc-weak` 一开就能用。zeroing weak 和 ARC 是可拆开的两个特性 |
| weak | 结构图里把哈希函数写成了 key | 那一层 key 是对象地址，`((addr>>4)^(addr>>9))%StripeCount` 是 `indexForPointer` 映射。**一篇立论"每层 key 都不同"的文章，最外层自己写混了** |
| weak | 「DEALLOCATING 置位」 | nonpointer isa 上没有这个位，`isDeallocating()` 就是 `extra_rc == 0 && has_sidetable_rc == 0` |
| weak | 不支持 weak 的类（照抄 2012 年名单） | iOS 26.5 SDK 上只剩 `NSMachPort` / `NSMessagePort`，UIKit 一个都没有。一行 grep 就能查 |

**教训**：两次采样凑巧支持了一个"有意思"的结论，就加粗写进正文和总结。**样本不足时不要下方向性结论，更不要为它补机制。**

**第二条教训**：weak 那篇里，我在参考资料里挂了两篇标题写着 "Swift 4" 的文章，正文却抄了 Swift 4 之前的实现。**引了勘误却抄了被勘误的内容——列进参考资料不等于读过。**

**捡到的最好素材**：`objc_loadWeakRetained` 正上方那段注释——"Once upon a time we eagerly cleared \*location... So we now don't touch the storage until deallocation completes."。我花一整节用实验证明的事，Apple 在源码里亲口解释了为什么这么设计。这类"源码注释里的设计自述"是最值钱的引用，以后每篇都该主动去翻。

**顺带确认了 19 位那个数字的真实归属**：只存在于无指针认证的 arm64（A7~A11，iPhone 5s 到 iPhone X）。arm64e 真机、所有 iOS 模拟器、所有 x86_64 都是 8 位——也就是说它在 Intel Mac 上从来没成立过。

## 四、变更日志

- 2026-07-26：第一次对齐，确认范围 / 约定 / 34 篇清单。文档 03、04 的实验已在 `/tmp/ios-notes-lab/` 跑通，正文未写。会话因权限分类器不可用中断，重建本台账。
