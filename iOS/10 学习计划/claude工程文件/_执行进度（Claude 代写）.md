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
| `iOS/20 专题笔记/Runtime/Part 1 - 对象与类的本质.md`（88KB） | 已完成，质量高 → **直接引用，不重写** |
| `iOS/20 专题笔记/Runtime/Part 2 - 消息发送与转发.md`（146KB） | 同上 → **引用** |
| `iOS/20 专题笔记/Runtime/Part 3 - Category：加载、覆盖与关联对象.md`（57KB） | 同上 → **引用** |
| `iOS/20 专题笔记/Runtime/Part 4 - Runtime 应用篇.md`（72KB） | draft，需**补完**（Swizzling / +load / KVO isa-swizzling） |
| `iOS/20 专题笔记/Runtime/素材/Method - Swizzling.md` | 与 Part 4 重叠 → 合并进 Part 4 后作为素材保留 |
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
- [x] 04 iOS 内存：MRC 的所有权规则（内存管理/）—— 对应 Day 4，审查中；已整合入 [[iOS 内存管理：从 MRC、ARC 到属性关键字#所有权规则：MRC 是 ARC 的地基|统一成稿]]
      **实测硬料**：按 `isa.h` 位域解析 isa，反复 retain 到溢出。第 253 次 `extra_rc` 跑满 255，
      第 254 次 `has_sidetable_rc` 翻 1、`extra_rc` 回落 128（= `RC_HALF`）。
      **纠正**：中文资料通说的「extra_rc 占 19 位」只对无 ptrauth 的 arm64 成立；arm64e 与模拟器上是 8 位。
- [x] 05 iOS 内存：ARC 的两半（内存管理/）—— 对应 Day 5，审查中；已整合入 [[iOS 内存管理：从 MRC、ARC 到属性关键字#ARC：规则如何变成代码|统一成稿]]
      **实测硬料**：六个场景的 LLVM IR，`-O0` vs `-O1` 对照（`setLabel` 从 4 个 storeStrong 变 0）；
      arm64 的 `mov x29, x29` marker vs x86_64 无 marker 的双架构汇编对照。
      **意外发现**：合成的 nonatomic strong getter 在 -O0 下一个 objc_* 调用都不生成。

### 第二周：weak、属性关键字与 Block
- [x] 06 属性关键字：从所有权推导，而不是从类型名猜（内存管理/）—— Day 1，审查中；已整合入 [[iOS 内存管理：从 MRC、ARC 到属性关键字#属性关键字：把所有权写进 API|统一成稿]]
      **实测硬料**：atomic 属性十万次并发自增丢掉一半计数，且比 nonatomic 丢得更多；加锁才 100000。
      长字符串 / NSArray / NSDictionary 的 copy 返回同一对象；容器 mutableCopy 只复制一层；
      `copy` 修饰 NSMutableArray 实际存进 `__NSSingleObjectArrayI`，addObject: 抛 unrecognized selector。
- [ ] 07 weak 的实现：SideTable、weak_table_t 与置 nil 时机（内存管理/）—— Day 2
- [x] 07 weak 的实现：SideTable 与置 nil 的时机（内存管理/）—— Day 2，已过审重写
- [x] 08 Block 的结构：ABI、descriptor 与三种类型（Block/）—— Day 3，审查中
      **实测硬料**：`clang -rewrite-objc` 在当前 Xcode 已彻底失效（`action RewriteObjC not compiled in`），
      而八成中文教程的论证建立在它上面。改用手写 `Block_layout` 强转读内存 + IR。
      **测量假象**：`show(const char*, id blk)` 这种中转函数会触发 ARC copy，导致「ARC 下看不到栈 block」的错觉。
      把参数类型换成非 `id` 之后，`__NSStackBlock__` 就出现了。
- [x] 09 Block 的变量捕获与 __block（Block/）—— Day 4，审查中
      **实测硬料**：同一个 `&shared`，Block 创建前是栈地址 `0x16d6160c0`，创建后在**同一作用域外部**
      打印变成堆地址 `0x60000020c0f8`，与 Block 内部一致。这是 `__forwarding` 最短的证明路径。
- [x] 10 Block 循环引用与 weak-strong dance（Block/）—— Day 5，审查中
      **实测硬料**：五种写法的泄漏对照，其中 **ARC 下 `__block` 修饰对象照样泄漏**（验证了 09 篇的断言）；
      weakSelf 的 block 里两次读同一变量得到 `A` 和 `null`，dance 之后两次都是 `B` 且事后无泄漏。

### 第三周：Runtime 行为与对象通信（Runtime/、Runtime 与对象通信/）
- [ ] 08 补完 `20 专题笔记/Runtime/Part 4 - Runtime 应用篇.md`
- [x] 11 KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发（Runtime 与对象通信/）—— Day 5，待审查
      **实测硬料**：
      - `setValue:forKey:` 的 setter 链实测是 `set<Key>:` → `_set<Key>:` → **`setIs<Key>:`**。
        第三步在归档文档和 `NSKeyValueCoding.h` 注释里**都没有**。用一个无 ivar、无属性、
        接管了 `setValue:forUndefinedKey:` 当哨兵的类隔离验证，哨兵没响。`_setIs<Key>:` 不在链上。
      - `@sum` / `@avg` 返回 `NSDecimalNumber`，**不是文档说的 double**。`@[@0.1,@0.2]` 得精确的 0.3
        （double 是 0.30000000000000004），`@avg` 输出 38 位有效数字，元素上收到的选择器是 `decimalValue`。
      - `dladdr` 反查中间类 IMP：`_NSSetIntValueAndNotify` / `_NSSetBoolValueAndNotify` /
        `_NSSetDoubleValueAndNotify` / `_NSSetRectValueAndNotify` / `_NSSetObjectValueAndNotify`，
        `class` → `NSKVOClass`，`dealloc` → `NSKVODeallocate`。`long double` 属性根本不 KVC 兼容，setter 不被替换。
      - **`automaticallyNotifiesObserversForKey:` 对某 key 返回 NO 时，KVO 完全不 isa-swizzle。**
        同一个类里 auto key 会建中间类、manual key 不会；两个都观察时中间类只覆写 auto 那个 setter。
      - 移除全部观察者后**实例 isa 会还原**成原类，Class 对象常驻并在下次复用（流行说法"不还原"是错的）。
      - 观察者 dealloc 后 `NSKeyValueObservance` 的 `Observer` 字段变成 `0x0`（zeroing weak），
        之后设值 5000 次不崩；被观察对象带着观察者 dealloc 也不抛异常。
        官方依据：Foundation Release Notes for macOS 10.13 and iOS 11 的
        "Relaxed Key-Value Observing Unregistration Requirements"（已一手核对原文）。
        **但手动通知的类也不抛，这超出 release note 承诺的两个条件，文中已标注不可依赖。**
      - `-isMemberOfClass:` 返回 YES 是因为实例版实现就是 `[self class] == cls`（已与 03 篇对账，
        初稿把它误写成 `object_getClass(self) == cls`，已改）。
      - KVC 落到 ivar 时是 **retain 不是 copy**：`NSMutableString` 写进 `copy` 属性后外部改动可见。
      - `NSUndefinedKeyException` 常量的字符串值是 `NSUnknownKeyException`（头文件有注释）。
      - Swift：`@objc` 无 `dynamic` 时直接赋值不通知、`setValue:forKey:` 通知；纯 Swift 属性走不到 KVO；
        `NSKeyValueObservation` deinit 自动注销；`observe` 无 `@discardableResult`（实测有 unused 警告）。
- [x] 12 对象通信：delegate、通知、target-action 与 block 回调（Runtime 与对象通信/）—— 待审查
      **全程零模拟器**：Foundation / runtime 部分一律 macOS 原生 `clang -framework Foundation`；
      UIKit 部分改用**静态二进制分析**（解析磁盘上 UIKitCore 的 `class_ro_t`），不 boot 任何设备。
      **实测硬料**：
      - 静态解出四个类的 target 所有权：`UIControlTargetAction._target` = weak（`ivarLayout 01` /
        `weakIvarLayout 11`，按 `instanceStart=8` 解 nibble），`_actionHandler` = strong；
        **`UIGestureRecognizerTarget._target` 也是 weak** —— 而 Apple 对手势 target 的持有语义只字未提；
        `UIBarButtonItem._target` weak / `_primaryAction` strong；`UIAction._handler` strong 且全类无 weak ivar
        （`addAction:` 成环的结构性证据）。二进制有 chained fixups，指针字段要取低 36 位。
      - **纠正上一版初稿的硬错误**：初稿写"运行时靠参数个数分辨"，正是简报点名要驳的"数冒号"说法。
        官方原文是 "pushes two parameters"。实测统一按两参数发，0/1/2 参签名全部正常返回；
        少传则读到寄存器残留（`0x120a8`）。ARC 下这个实验会崩（给入参插了 retain），必须 `-fno-objc-arc`。
      - 协议方法**永远没有 IMP**：clang `CGObjCMac.cpp:7674` 原文已核对
        （"Protocol methods have no implementation. So, this entry is always NULL."），
        内存直读印证 `impOffset=0`。注意方法列表现在默认是 **small 相对偏移格式**（entsize=12），
        按老的 `{SEL,char*,IMP}` 三指针解析会段错误。
      - `protocol_copyMethodDescriptionList` 四选一各调一次全部跑通（含 optional 类方法）；
        它不递归父协议，`protocol_getMethodDescription` 递归 —— 两个 @note 正好相反。
      - `-[NSObject conformsToProtocol:]` 的继承链是**它自己的 for 循环**走的
        （`NSObject.mm:2506` 已核对原文），`class_conformsToProtocol` 不走 superclass 只递归父协议
        （`strcmp(mangledName)` 见 `objc-runtime-new.mm:5298`）。实测 LiarSub 上两者给相反答案。
      - `respondsToSelector:` 会触发 `+resolveInstanceMethod:`（delegate 高频路径上的隐藏成本）。
      - 通知：同步派发、跑在 post 线程、通配与具名按注册序混排（证伪 GNUstep 三张表说法）；
        观察者抛异常掐断整条链并传回 post 方；post 期间新增本轮不生效、移除会被跳过
        （"快照 + 逐项校验"，纯快照模型预测 A,B,C 实际 A,B）。
      - **`queue:` 非 nil 不等于异步**：block 里 sleep 300ms，post 稳定花 303~305 ms（3 次）。
        注册 mainQueue + 主线程等 post ⇒ 死锁（wait 返回 49），换 nil 正常。
        **修正上一版**："post 恰好在目标 queue 上耗时 0.0 ms"不复现，实测满额 302/325/305 ms。
      - 四种 target 语义并置：UIControl 不 retain（weak 是实现非契约，AppKit `NSControl.h` 反而写了
        weak，已从本机 SDK 核对原文）、手势无任何文档、NSTimer 强持有到 invalidate（实测）、
        NSInvocation 默认不 retain 参数、`retainArguments` 后才 retain（实测）。
      - iOS 9 免移除的四类边界；zombie 下 selector 观察者不移除无残留消息，block 版 token 丢了照跑。
      - 开销（macOS 原生，-O1，3 次稳定）：直接发消息 2.56 / delegate 完整写法 26.5 /
        block 0.32 / postNotificationName: 213 ns。读 weak 16.5 ns 是 delegate 的大头。
        10 万条死记录对 post 耗时无影响（1.01×），代价是 0.43 MB。
      - **测量假象三次**：small 方法列表解析段错误；ARC 给入参插 retain 导致残留实验崩溃；
        `unsafe_unretained` delegate 量出 13 ns 其实是**属性 getter 上 ARC 的 retain/release**
        （直接读 ivar 2.59 ns，与基线一致）。
      **修正的过时数据**：`NSNotificationCenter` 现在是 **3 个 ivar**（`_impl` + `actorQueueManagerLock`
      + `_actorQueueManager`），不是旧文章/上一版初稿说的 1 个；后两个服务于 iOS 26 类型化通知。
      `MainActorMessage` / `AsyncMessage` 定义已从本机 SDK swiftinterface 抄原文。
- [x] 13 Method Swizzling：正确姿势、+load 时机与那些坑（Runtime 与对象通信/）—— Day 2 + Day 4，待审查
      定位：`20 专题笔记/Runtime/Part 4` 与 `20 专题笔记/Runtime/素材/Method - Swizzling.md` 的**实证补充**，不重复原理与场景。
      **实测硬料**：
      - 直接 exchange 继承来的方法，后果不是"父类被污染"这么温和：`Parent` 和 `Sibling` 当场
        `-[Parent hook_greet]: unrecognized selector` 崩溃，崩溃栈却显示 `-[Child(Hook) hook_greet]`。
        hook 不回调原实现时才退化成静默的全局行为改变，且 `class_copyMethodList` 查不出来。
      - **人人在抄的 `class_addMethod` 模板本身有 bug**：它把父类当时的 IMP 冻成裸指针。
        子类先 swizzle、父类后 swizzle 时父类 hook 被静默跳过（`C(origin)` vs `C(P(origin))`）。
        RSSwizzle 的 `originalImpProvider` block 是标准解法（已一手核对 RSSwizzle.m 205-219 行）；
        Aspects 用 `AspectTracker` 直接禁止同继承链重复 hook（错误串已一手核对 Aspects.m 619 行）。
      - 模板还有**递归窗口**：`class_addMethod` 已生效、`class_replaceMethod` 未执行的那一瞬间，
        调用目标方法无限递归（实测打印到第 5 层）。`method_exchangeImplementations` 单次调用没有此窗口。
      - **现在的链接器（ld-1267）默认把同二进制内的 category 合并进类**，`__objc_catlist` 整个消失，
        category 的 `+load` 变成类的 `+load`，走父类优先规则，正好掩盖上面那个 bug。
        `man ld` 的 `-no_objc_category_merging` 原文已核对。独立复现：主类不写 `+load` 时默认无 catlist，
        加 `-Wl,-no_objc_category_merging` 后 `__objc_catlist`/`__objc_nlcatlist` 回来。
        主类自己写了 `+load` 就合并不了（一个类塞不下两个），catlist 重新出现。
      - **`+load` 不一定在 `main()` 之前、不一定在主线程**：后台线程 `dlopen` 触发的 `+load`
        实测 `pthread_main_np()` 返回 0，且发生在 `main()` 之后。
      - `+load` 里访问别的类：类已 realize、消息发得出去，但对方 `+load` 未必跑过。
        同一份代码只调换两个 `.m` 在 clang 命令行上的位置，`[Late banner]` 一次 nil 一次正常。
        并且发消息会触发对方的 `+initialize`，出现"`+initialize` 早于它自己的 `+load`"。
      - `+initialize` 实测被调 4 次（继承链上每个未实现的类各一次，`self` 各不同）；
        主类的 `+initialize` 被 category 的完全覆盖，**一次都没跑**。
      - 命名冲突实测：两个 category 用同名 swizzled selector，方法列表里并排躺着 2 条 `a_work`，
        先注册那一方的实现**一次都没执行**；第三次 exchange 把前两层 hook 一起撤销。
      - **交换类方法时 `class_addMethod` 必须打在 `object_getClass(cls)` 上**，打成 `cls` 返回 YES、
        不报错、hook 静默失效，只是给类加了两条没人调的实例方法。类和元类 `class_getName` 同名。
      - Swift：`@objc` 无 `dynamic` 时 `class_getInstanceMethod` 找得到、交换"成功"、`perform` 走 hook，
        但 Swift 侧调用点走 vtable 看不见 hook（半失效，最难查）。纯 Swift 方法不在方法表里。
        objc4 `prepare_load_methods` 对 Swift 类的 category `+load` 是 `_objc_fatal`。
      - `_objc_msgForward` 转发式 hook 跑通：`respondsToSelector:` 保持 YES，
        `forwardInvocation:` 拿到的是真 selector。`class_getMethodImplementation` 对不响应的 selector
        返回的正是 `_objc_msgForward`（`0x18006b860`，两个实验对上）。
        `_objc_msgForward_stret` / `objc_msgSend_stret` 在 SDK 头文件里标了 `OBJC_ARM64_UNAVAILABLE`，
        老文章里整段 stret 讨论对今天的 iOS 已无对象。
      **一手源码已核对**：`objc-loadmethod.mm`（`call_load_methods` / `call_class_loads`）、
      `objc-runtime-new.mm`（`addMethod` / `class_addMethod` / `class_replaceMethod` /
      `method_exchangeImplementations` / `schedule_class_load` / `load_images` / `prepare_load_methods`）、
      `objc-initialize.mm`（`callInitialize` 与 radar 2157218 注释）、`man ld`、SDK `objc/message.h`。

### 第四周：线程、GCD、Operation 与锁（并发与运行循环/）
- [x] 11 GCD：队列不是线程，以及死锁的准确边界（seriesOrder 14，655 → **945 行**）—— 已过主会话核验
      落盘后又收到一份迟到的研究简报，据此做了一轮增补修订（+290 行）。
      **了结了欠两轮的「待复核」**：target queue 链确实死锁，且比原文写的更普遍。
      上一版怀疑的两个变量（`create_with_target` vs `set_target_queue`、外层 async vs sync）**一个都不影响**，
      四种组合逐格相同全是 SIGTRAP（lldb 抓到 `__DISPATCH_WAIT_FOR_QUEUE__`）。
      上一版"复现不出来"的真正原因是**漏建了一条 target 边**。真正决定结果的是第四个变量：
      **两条 target 链汇合处的队列是串行还是并发**——汇合在串行队列上就 trap（三层链也 trap），汇合在并发队列上正常返回。
      **「64」和「512」这两个流传多年、谁也说不出出处的数字，都是真的，但都不是"GCD 线程上限"**：
      全局并发队列压 400 个阻塞任务封顶 64（摊到 4 个 QoS 池还是 64）；
      200 个独立串行队列直接拿到 200 条线程，加到 700 封顶 512。
      **主会话独立复现**：`kern.wq_max_constrained_threads: 64` / `kern.wq_max_threads: 512`，
      两个 sysctl 值，含义不同。这同时是 Habouzit "serial queues as proxies for OS threads" 的实测版。
      **Apple 明说过信号量不做优先级捐赠**（WWDC17 706 逐字，中文社区只有传言没人引过原文）：
      "asymmetric primitives, like dispatch semaphore and dispatch group don't have this power,
      because the runtime doesn't know what thread will singal[sic] the sync primitive."
      实测印证：`dispatch_sync` / `dispatch_block_wait` 把被等线程 `pth_curpri` 从 4 提到 31，
      `dispatch_semaphore_wait` / `dispatch_group_wait` 留在 4，三轮一致。
      **它踩了个坑并写进正文**：第一次用 `qos_class_self()` 测，四种方式全程 BACKGROUND 纹丝不动——
      `pthread/qos.h` 明说 override 对这个接口不可见，**是仪器错了**。
      **Swift 文档丢了 barrier 的关键约束**：ObjC 页有 "concurrent queue that you create yourself" 和退化说明，
      Swift `DispatchWorkItemFlags.barrier` 页三句全无（拉 DocC JSON 逐字核过），
      而它只说 "When submitted to a concurrent queue"——`DispatchQueue.global()` 恰好符合这个描述。
      **「GCD 不是线程池」这个流行反驳本身也不准**：Concurrency Guide 原文 "They offer automatic and
      holistic thread pool management"。错的是把它想象成应用私有、固定大小、队列 1:1 绑线程的池。
      **`dispatch_after` 到点做的是入队不是执行**（官方 "asynchronously adds `block` to the specified `queue`"）。
- [ ] 12 NSOperation 与 NSOperationQueue：依赖、取消与自定义并发 Operation
- [x] 13 锁：从 OSSpinLock 的废弃说起（seriesOrder 16）—— 已过主会话核验
      **实测排名**（macOS arm64 原生，200 万次/轮 × 7 轮取最小，整程序跑两遍）：
      OSSpinLock 7.00 → os_unfair_lock 8.11 → pthread_mutex 8.75 → NSLock 9.03 →
      dispatch_semaphore 9.77 → @synchronized 21.13 → NSConditionLock 34.13。
      5~9 名挤在 8.75~9.77，文中已明说这一档内部名次没有意义。
      **证伪 ibireme 那张流传最广的图**：OSSpinLock 至今最快（比 os_unfair_lock 快 14%），
      废弃与快慢无关；dispatch_semaphore 从第二名掉到第九；NSLock 只比 pthread_mutex 慢 3%。
      **`os_unfair_lock` 不是自旋锁**：`os/lock.h` 第 48 行原文 "allows waiters to block efficiently"。
      **`@synchronized(nil)` 不崩，静默失去互斥**；NSLock 允许另一条线程 unlock，只有 os_unfair_lock 会 abort。
      **读写锁在短临界区是负收益**，实测慢 4 倍，交叉点在几百个元素。
      **原创数据**：争用下的尾延迟才是分水岭——OSSpinLock 最坏等待 1049/1097/1057/1099 ms
      （每轮都有高优先级线程整个窗口一次没抢到），os_unfair_lock 12~19 ms。
      **它主动承认猜错一次**：预测 @synchronized 锁对象越多越慢，实测 1→32768 持平，
      机制没查出来，文中只写实测不补机制。这个处理方式是对的。
      **主会话独立复现**：排名逐条重合（7.18/8.07/8.79/9.22/10.07/22.06）；
      `@synchronized(nil)` 我这轮丢 73.5%（它测 26%，比例随时序变，方向一致）；
      跨线程 unlock NSLock 确实不崩且可再 lock；头文件原文核对无误。

**第六批（锁篇，agent 自己查清的跨平台差异，值得单独记）**：

盘上留着的一份 `bench_final.txt` 与 agent 自己测的排名大体一致，但 `@synchronized` 差 2.4 倍。
它没有挑一个顺眼的写，而是去查——`vtool` 显示那份是 **iOS 模拟器二进制**（platform IOSSIMULATOR），
自己的是 MACOS。两份 bench 代码逐行对过，临界区、锁对象、轮次、ARC 全同，唯一差别是编译目标。
第三个数据点佐证：本系列 GCD 篇在模拟器上测到 @synchronized 49.1ns，与那份的 51.15 吻合。
**结论：`objc_sync_enter/exit` 在模拟器上明显更贵，pthread/dispatch 原语两平台一致。**
附带查出其"原子自增 8.29ns"与自己的 2.15ns 差 4 倍，是 `SEQ_CST` vs `relaxed`。

这正是新增的规范第 7 条（下通说不对之前先列自变量）的正面案例——**同一份代码换个 target 就是另一组数字**。

### 第五周：RunLoop、AutoreleasePool、响应者链与生命周期
- [ ] 14 RunLoop 重写：mode / source / timer / observer 与真实循环流程（并发与运行循环/）
- [x] 15 AutoreleasePool：哨兵、页链表与 RunLoop 的关系（seriesOrder 18，693 行）—— 已过主会话核验
      **实测硬料**：
      - **每次 `objc_autorelease` 都先把对象扣在 TLS 里，不进页**。`rootAutorelease()` 调
        `prepareOptimizedReturn(obj, /*cameFromRootAutorelease*/true, ...)`，`true` 跳过"调用方是否打算认领"那道关卡。
        实测 505 次才满页、页里只有 504 个对象。**中文资料一篇都没写过这条**，且它污染了所有靠"数条目"的实验。
      - **`@autoreleasepool` 遇异常不排空**。Clang ARC 规范原文 "When the block is exited with an exception,
        the pool is not drained."；landingpad 里没有 `autoreleasePoolPop`；实测内层哨兵原地不动。
      - **主线程今天是每次 callout 一个池，不是每轮 RunLoop 一个**。反汇编 UIKitCore 的
        `_UIApplicationInstallAutoreleasePoolsIfNecessaryForMode`：第一句 `_CFRunLoopSetPerCalloutAutoreleasepoolEnabled(1)`，
        返回非零直接 `ret` 什么都不装。**那两个著名 observer（order ±2147483647）参数一字不差，但今天是 fallback 分支**。
        典型的"通说数字对、结论过期"。
      - **ARC 下"循环里加池"大面积失效但非全面失效**：`dataWithLength:4096` 跑 20 万次，
        MRC 不加池 +1014.89 MB / ARC 不加池 +0.05 MB；但 `stringWithFormat:` 和 `arrayWithObjects:`
        ARC 下照样 +14 MB / +10.78 MB。定参工厂方法 0 次入池，那两个变参 100 次。
      - **它一度得出"变参破坏返回值握手"，被自己的对照实验推翻**：函数体逐字相同的一定参一变参 MRC 类方法，
        ARC 调用方两边都是 0 次入池。真正条件是被调方以**尾调用**交出 autorelease（`b _objc_autorelease` 而非 `bl`）。
      **修正本系列前面两处**：(1) 04 篇说 `NSAutoreleasePool` "走另一套代码路径"——错，`-init`→`_CFAutoreleasePoolPush`、
      `-drain`→`_CFAutoreleasePoolPop`，同一条页链表；(2) 第一批错误表里"`PROTECT_AUTORELEASEPOOL` 发布版根本没定义"——
      **它是定义了的，值为 0**（`NSObject-internal.h`）。结论 4096 不变。
      **主会话独立复现**：`objc_debug_autoreleasepoolpage_begin_offset` = 56、
      `objc_debug_autoreleasepoolpage_ptr_mask` = `0x0f00ffffffffffff`（**出货 libobjc 确实和公开 objc4 的
      `0xffffffffffff` 不一样**）、页边界 `0xc61400000→0xc61401000` 差 4096 而非系统页的 16384、哨兵在 +0x38。
      (4096−56)/8 = 505 槽，首槽给哨兵剩 504，算术成立。
- [x] 21 Foundation 集合：类簇、真实实现与选型（seriesOrder 29，**实为 932 行**）—— 已过主会话核验

      > **⚠️ 调度事故记录（已查清，无内容损失）。**主会话把**三个 agent 先后派到了同一路径**：
      > 第一个我误判为掉线（实际它跑了五个多小时才交稿）、第二个正常完成、第三个被我派去做增补。
      > 三方对文件状态各执一词——增补 agent 坚称盘上是「772 行原版 + 它的 6 处增补」、第三版 agent 坚称自己落了 819 行。
      > **由主会话用特征串比对判定**：文件里**同时**含第三版特有标记
      > （`__objc_arrayobj` 3 处、`NSConstantIntegerNumber` 4、"帐篷" 3、`17352`、`250 万`、`25/25`）
      > 和增补版特有标记（"41 步"、`needsCompaction`、`15396578`、`0.618` 4 处）。
      > **932 行 = 三版内容的并集，一条没丢。**增补 agent 的 Edit 能落在第三版上，
      > 是因为第三版保留了原版段落、`old_string` 照样匹配得上。
      >
      > **两条教训**：(1) 并行派单前必须查目标路径是否已被占用；
      > (2) **不要凭"任务失败"的通知就断定 agent 已死**——同样的错在 Method Swizzling 那篇上犯过一次，这里又犯一次。
      > 附带一条：**agent 自述的文件状态不可尽信**，两个 agent 都真诚地报告了错误的事实，
      > 只有特征串比对能裁决。这和本台账反复记录的规律一致：**转述会错，产物不会**。
      > **2026-07-27 增补完成**：下面"待增补"的每一条都由增补 agent **重新独立实测**了一遍（没有照抄台账），
      > 全部并入现存版本，同时保留现存版本自查出的"头插假象"结论。合并后 819 行，两版材料一条没丢。
      >
      > **教训**：并行派单必须先查目标路径是否已被占用。这是全程唯一一次内容损失事故。

      **现存版本（保留）的主线**：写时复制——六万四千元素的可变数组 copy 耗时和一千元素时一样，都是 50ns。
      它还自查出一条并写进了 description：**「头插比尾插快」是测量假象**。
      **另一个 agent 独立复核并承认自己错了**：原来的尾插写成 `[a insertObject:o atIndex:a.count]`，
      每轮比头插多发一次 `count` 消息，那 25% 的差距全部来自这次多余的消息发送，跟插入位置无关。
      它先前"隔离方法变量"的四种对照设计（固定序、轮换序、随机序、取最小值，跑了 31 轮）**全都复现了这个假象**，
      因为四种设计都没动到真正的自变量。**这是规范第 5 条"先怀疑仪器"最好的一个案例。**

      **待增补回来的已验证材料**（原属被覆盖版本，主会话已独立复现其中第一条）：
      **它裁决了一个悬了十年的矛盾**：Ciechanowski（2013，iOS 7）说 1.625 倍扩容，老青菜（2020）说 2 倍。
      arm64 上序列是 `2 4 6 10 16 28 48 80 160 320 640`，超过 48 之后比值变 2.0；换 x86_64 重跑变成
      `… 16 26 42 68 110 192`，比值 1.625/1.615/1.619/1.618。真正规律是
      **`malloc_good_size((size_t)(旧容量*1.618)*8) == 新容量*8`**。所谓架构差异全是 malloc 尺寸类。
      **它是主动按规范第 7 条列自变量才发现的**——本来已经要写"小容量 1.6 倍、大容量 2 倍"了。
      其他实测：「容量只涨不缩」已作废（`removeAllObjects` 后 size=0、list=NULL）；
      **`[大可变数组 copy]` 是 O(1)**，≥6 元素返回 `__NSFrozenArrayM` 共享源 buffer，源那个一直是 0 的
      `cow` 字段在 copy 瞬间变非空（老青菜当年逆向出这个字段但没定作用，这条补上了）；
      **读 `CFArray.c` 是读错了文件**——`CFArrayCreateMutable(NULL,0,&kCFTypeArrayCallBacks)` 返回的是
      Foundation 的 `__NSArrayM`，只有传 NULL 回调才落到真 CFArray；
      `@[@"a"]` 今天是 `NSConstantArray` 不是通说的 `__NSSingleObjectArrayI`。
      **主会话独立复现**：两个架构的序列逐个对上，原式 19/19 全中。
      （核验时我自己把公式括号放错位置，误判了两步"不中"，重跑原式才发现是我的问题。**审查方也会出错，交叉验证要连自己一起验。**）

      **增补 agent 的独立复现结果（arm64 原生，2026-07-27）**：
      φ 公式在**同一架构上跑到 250 万元素、25 次跃迁 25/25 全中**（比原版多 6 档，
      末几档比值 1.640→1.634→1.627→1.624→1.621→1.620→1.619 单调收敛到 φ，
      这比"换 x86_64 才看得出来"更直接——规模一大，malloc 尺寸类的量化误差自己就被摊薄了）。
      `cow` 字段的作用**测死了**：copy 瞬间原件与 `__NSFrozenArrayM` 的 `cow` 指向同一个 16 字节小对象、
      buffer 同址；原件一写，`cow` 归零并换到新 buffer，冻结那份原地不动。`mutableCopy` 双向都插。
      `CFArrayCreateMutable` 标准回调 → `__NSArrayM`、NULL 回调 → `__NSCFArray`，
      `CFDictionaryCreateMutable` 同样（`__NSDictionaryM` / `__NSCFDictionary`）。

      **增补带进来的新硬料（原两版都没有）**：
      ① **对称帐篷**——固定 20 万元素只改插入位置 p 的单次耗时：0% 8.8ns、5% 1263、25% 8507、
      50% 17352、75% 8854、95% 1283、100% 138ns。5% 对 95%、25% 对 75% 两两吻合，
      直接画出了 `CFArray.c` 里 `// move C` / `// move A` 那两个分支。这比"头插 vs 尾插"强得多，
      因为它只动一个自变量。
      ② **环形的直接证据**——cap=6 尾插 1,2,3,4 再头插 9、8，槽位打出来是 `1 2 3 4 8 9`、offset=4、
      逻辑序 `8,9,1,2,3,4`。CF 的 `__CFArrayDeque` 只有 `_leftIdx`/`_capacity` 且不环绕，
      **`__NSArrayM` 环绕，两者不是同一个数据结构**（原版只说"读错了文件"，没说错在哪一层）。
      ③ **侧证**——绕圈的数组 `countByEnumeratingWithState:` **分两批**（3+4），没绕圈的 7 个元素一批。
      顺带推翻"for-in 一次抓 16 个"：数组一次交出全部 5000 个且 `itemsPtr` 指自己的 buffer，
      只有 set/dict 才是每批 16。
      ④ **NSConstantArray 的 Mach-O 段级证据**——`__objc_arrayobj` / `__objc_intobj` / `__objc_dictobj`
      与 `__cfstring` 并列，段内 24 字节 = isa 空槽 + count + 元素指针，运行期逐字对上。
      换 5 档 macOS 部署目标 + iOS 15/17/18 target，符号都照样生成。
      **与 tagged pointer 篇对账**：`@42` 是 `NSConstantIntegerNumber`（bit63=0），
      `[NSNumber numberWithInt:42]` 才是 tagged（bit63=1）——**前篇"@42 是 tagged pointer"需要加限定**。
      ⑤ **for-in 的 IR 证据**——mutation 检查在循环头 `%15:`，**每次迭代都做**（不是每批），
      所以 40 元素数组第 1 次迭代改就第 1 次抛，而**单元素容器改了根本不抛**（没有下一次迭代）。
      ⑥ **三种 retainCount 哨兵**——`-1` 归编译期常量对象与共享缓存单例、
      `0x0FFFFFFFFFFFFFFF` 归 `__NSCFConstantString`、`INT64_MAX` 归 tagged pointer（与 MRC 篇对上）；
      **`__NSArrayI` 返回的是真实计数 2**，"不可变对象计数是极大值"这个印象不成立。
      ⑦ **弱容器三兄弟的 `count` 全部不更新**（5 个死光仍报 5，allObjects 才是 0）；
      `NSPointerArray compact` 直接调无效、先 `addPointer:NULL` 才有效（复现了社区传了十年的绕法，
      但**查不到 Apple 官方说明，文中标成猜测**）。
      ⑧ **选型给出了与 N 无关的回本次数**——建 set 每元素多 110ns，数组线性扫每元素 6.35~25ns
      （取决于 `isEqual:` 是否走指针快路径），相除得 **5~17 次查询回本，与集合大小无关**。
      另有 `hash` 恒为常数慢 958 倍、`@[@1,@2].hash == count`、二分查找快 6909 倍。

      **增补 agent 自查踩的两个坑（都写进了正文）**：
      ① 把"三次调用地址相同"当成空单例的证据是错的，**前一个对象释放后地址会被复用**，改成同时持有六个才成立；
      ② 测 `NSHashTable` 时对象建在循环里、出循环体强引用就没了，当场 dealloc，于是"测出"加 3 个后 count=1。
      **文风自查**：加粗 5 处、破折号 3 个、"不是X而是Y" 0 次、禁用词 0、短句 13.3%、长句 17.4%、
      段落句数分布 1 句 26 段 / 2 句 38 / 3 句 25 / 4 句 20 / 5 句 10 / 6 句以上 2。
- [x] 27 持久化选型：沙盒、plist、NSUserDefaults 与 Keychain（seriesOrder 30，667 行）—— 待主会话核验
      **实测硬料**：`synchronize` 对落盘时机零影响（稳态约 10 秒，8 轮数据）；
      **进程被 SIGKILL 也不丢数据**——数据在 `cfprefsd` 里不在你进程里；
      **文件保护等级在 macOS 26 上可用**，不是通说的 iOS 专属，但设 `None` 返回成功却被静默忽略；
      `NSURLIsExcludedFromBackupKey` 会向上继承，逐文件标是白做的；
      `writeToURL:` 默认写 XML 比 binary 大 4.63 倍；Keychain 主键含 service+account 不含 label。
      **诚实度好**：数据保护钥匙串完全没跑起来（`-34018`，带 entitlement 直接被 SIGKILL），
      `kSecAttrAccessible` 六个等级一条都没实测，文中明写了这个缺口。5 处 `> 待真机补测`。
- [x] 29 序列化：NSCoding、NSSecureCoding 与 JSON 三条路（seriesOrder 31，784 行）—— 待主会话核验
      **实测硬料**：**`decodeObjectOfClass:` 在非 secure 的 unarchiver 上完全不检查类型**
      （`NSCoder.h` 原文 "If the coder responds NO to `-requiresSecureCoding`, then the class argument is ignored"）——
      防线在解档器上不在这个方法上，而 Apple 的《Secure Coding Guide》只说了用 `decodeObjectOfClass:`；
      **`NSJSONSerialization` 默认路径接受一个尾逗号**（RFC 8259 不允许），其余 JSON5 特性全关在 `JSON5Allowed` 后面；
      **重复 key 取第一个**（RFC 说主流实现取最后一个，Python/JS 都是后者）；
      `-1e400` 解析成 `-inf` 而 `1e400` 被拒，产物 `isValidJSONObject:` 判 NO——解析器交出了它自己拒绝写回的对象；
      **大整数不退化成 double**，通说的 2^53 是 JavaScript 的，Foundation 是三级阶梯（≤19 位 SInt64 / 20~39 位 NSDecimalNumber / ≥40 位补零）。
- [x] 24 静态库与动态库：加载时机、@rpath 与体积账（seriesOrder 26，638 行）—— 待主会话核验
      **实测硬料**：**动态库不一定省包体积，单 target 下净亏 33 KB**，要两个 target 才回本；
      **启动开销按镜像个数收不按代码量收**——50 个类打进 1 个 dylib 与基线同噪声，摊成 50 个 dylib 是 12.7 ms，
      单价 0.154 ms，两个规模独立算差 1%（**它自陈"加对照组之前差点写下随代码量线性增长"**）；
      `if (&sym)` 判弱链接符号只加 `-weak_library` 会崩，汇编里 `cbz` 根本不生成且零警告；
      `dlopen` 单价比启动期加载贵 3 倍多；**`DYLD_PRINT_STATISTICS` 在 macOS 26.5.2 上零输出**。
      **跨篇对账**：与 27 篇独立测出的纯 C 单价 0.092 vs 0.106 ms 差 15%，差值来向已在正文说明。
- [x] 26 App 启动：三代 dyld、pre-main 与可测量的优化项（seriesOrder 27，805 行）—— **已发回修订**
      落盘时研究材料还没送达，文末三条"没查到"（三代 dyld 与系统版本对应、iOS 13 第三方走 dyld3、
      delayed init 官方名称）调研里每条都有一手出处，已发回补。详见第七节。
- [x] 16 事件传递与响应者链：hitTest、手势与那些点不动的按钮（seriesOrder 19，661 行）—— 已过主会话核验
      **全程 Mac Catalyst，零模拟器。**
      **实测硬料**：
      - **alpha 阈值那条流传多年的说法两边都不准确**。`UIView.alpha` 存的是 32 位 float，
        `(float)0.01 = 0.009999999776482582`，比十进制 0.01 小 2.24e-10。**所以根本设不出 alpha == 0.01**，
        Apple 文档写 "less than 0.01"、中文圈写 "≤ 0.01"，两者可观测行为完全等价，外部无法区分。
        用 `nextafterf` 逐 ulp 逼出：能被点到的最小 alpha 是 **0.01000000070780515671**。
      - **`clipsToBounds` 对 hitTest 毫无影响**。中文文章普遍据 Apple 那半句话写成"会影响"，是误读
        （原话讲的是可见性）。副作用：能造出"被裁得看不见但吞触摸"的区域。
      - **响应者链比流传的那张图长**：`UIWindow` 与 `UIApplication` 之间有 `UIWindowScene`
        （`UIScene` 确是 `UIResponder` 子类），Apple 自己的文档还停在 "window's next responder is the UIApplication object"。
      - `alpha` / `hidden` 不存在 `UIView` 上，是 `CALayer` 的直通；绕过 view 改 `layer.opacity` hitTest 照样认。
      - `UIWindow` **没有**覆写 `hitTest:`；`UIControl` / `UIScrollView` / `UIButton` 覆写了，`UILabel` 没有。
      - `UIActivityIndicatorView.userInteractionEnabled` 默认 **YES**（agent 原本猜 NO，猜错了，文中写明）。
      **主会话独立复现**：五条全过，alpha 两个数字逐位一致。
      **它主动放弃的实验**：手势 vs UIControl 优先级整节做不了——Catalyst 鼠标点击走 indirect pointer，
      且 `UIButton` 被系统塞了两个 `_UISelectionInteraction` 手势（iOS 上没有），
      在这种环境测出的优先级搬到 iOS 就是错的。文中明写"我没做成"并留了 4 组具名复现方案。
      **过程记录**：它一度打算用 `CGEvent` 合成真实鼠标事件（会动用户光标、可能点到别的窗口），
      被主会话叫停，改为直接调 `[view hitTest:p withEvent:nil]`。禁令已写进规范第二节。
      **与坐标系篇的分工已对账**：transform 对 hitTest 的影响归坐标系篇第七节，本篇无重复。
- [x] 17 UIViewController：生命周期的真实顺序与容器控制器（seriesOrder 20，737 行）—— 已过主会话核验
      **三种容器给出三种不同的交错顺序**：nav push「旧 willDisappear→新 willAppear→旧 didDisappear→新 didAppear」；
      present「新 didAppear→旧 didDisappear」；切 tab「新 willAppear 在最前」。四种组合里占了三种。
      **`animated:NO` 的 push 到 `viewDidAppear` 为止 `viewWillLayoutSubviews` 是 0 次**（连跑 3 遍稳定）——
      「viewDidLayoutSubviews 一定在 viewDidAppear 前」不成立。
      **漏掉 `didMove` 或漏掉 `addChildViewController:`，出现回调一个都不少**——流行的"不调就收不到生命周期回调"不成立，
      出现回调由 view 进入 window 层级驱动。漏 `addChild` 的真实后果是没人持有子控制器，转场一结束就 dealloc，
      **界面看着正常但点不动，且无任何报错**。
      `didMoveToParentViewController:` 标准三步流程里实际被调 2 次（第二次 UIKit 补发）→ 移除时不该手动补 `didMove(nil)`。
      **`viewDidLoad` 不保证只调一次**（`view = nil` 后再访问会完整重走）。
      手动 post `UIApplicationDidReceiveMemoryWarningNotification` 触达不到任何 VC，只有私有的 `_performMemoryWarning` 才下发。
      **它纠正了主会话 prompt 的错误**：我说 `viewIsAppearing:` 是 iOS 17 新增，头文件实为 `API_AVAILABLE(ios(13.0))`
      （back-deploy API，WWDC23 才讲）。已核对头文件原文，**它对我错**。
- [x] 19 UIView 与 CALayer：三棵树、绘制流水线与离屏渲染（seriesOrder 22，879 行）—— 已过主会话核验
      **`UIView` 树和 `CALayer` 树不是两棵平行的树**：`subviews` 是 `layer.sublayers` 按 `delegate` 映射出的投影，
      `superview` 才是独立状态。**主会话独立复现**：只 `removeFromSuperlayer` 后 `p.subviews` 仍是 1 而 `a.superview` 已 nil；
      只 `addSublayer:` 后 `q.subviews` 变 1 而 `b.superview` 仍 nil。
      **`UIView` 有两个 layer ivar**：`_layerRetained`（off 48，ARC strong）和 `_layer`（off 168，不扫描），指向同一对象。
      主会话复现偏移完全一致。layer 不是懒创建的，`+layerClass` 在 `initWithFrame:` 期间就被调用。
      **`actionForKey:` 返回的是 nil 不是 `NSNull`**——`NSNull` 是 `UIView` 作为 delegate 返回的，`CALayer` 按头文件归一成 nil。
      **空 `drawRect:` 的代价量化**：100 个 320pt 视图 @3x，不实现 2.3 MB / 空实现 92 MB / 真画满 356 MB。
      **它推翻了自己原本的判断**：2.5 倍屏幕 / 100 ms 原准备标"来源不明不采信"，查到 WWDC14 Session 419 讲义 p91 后收回；
      而"Handle Events"不属于 Session 419，来自 Tech Talk 10855，**中文流程图把两个模型缝在了一起**。
      Instruments 的 Core Animation 模板 2018 年已废弃，正确路径是 `Debug > View Debugging > Rendering`。
      **诚实度好**：`CARenderer` 在 Catalyst 下完全不出图（整张纹理零字节），离屏渲染整块验不了，失败过程写进了正文。
      它还自陈翻车一次：empty 那行只有标称值 26%，先猜是内存压缩，`compressed` 三行全 0.00 MB 证伪，改写成"这是我的解读不是验证过的机制"。
      **PDF 逐页核对**：Session 419 引用的每张幻灯片（p10-19、20-29、37、53-55、91、93、157-158、167、169）都自己下载核对过，没采信转述。
- [x] 33 架构模式：MVC 到 MVVM，以及它们各自解决不了的问题（seriesOrder 36，507 行）—— 待主会话核验
      同一需求四份实现，Catalyst 真 UIKit 编译运行，**四份的可见状态序列逐帧完全相同**（8 步交互脚本逐字比对）：
      MVC 140 行 / MVP 192 / MVVM（4 个 Observable）230 / MVVM（合并成 1 个 state）215。
      **最有价值的一条**：MVVM 拆 4 个 Observable 时视图侧被叫醒 **35 次**，合并成 1 个 state 后降到 **9 次**（与 MVC 持平）。
      早年那篇引的"MVVM 轻微增加代码量"，实测是 **+64%**。
      **用 SDK 头文件证明 Apple MVC 的承重件（`NSController` 一族 + `bind:toObject:withKeyPath:options:`）只在 AppKit**，
      iOS 从来没有过，连 Catalyst 都被 `APPKIT_API_UNAVAILABLE_BEGIN_MACCATALYST` 挡掉——
      用户早年那篇贴了 Apple 的图但没发现这一层。
      MVP 协议样板 49 行占 26%；MVVM 绑定段 22 行里 5 行是编译器强制的 weak-strong（`ws->_ivar` 直接报 8 个 error）。
      实测两个 MVVM 坑：错误态→重试有 2 个不该存在的中间帧；导航当 Observable 输出时 VC 重建后 1 次点击 push 2 次。
- [x] 18 坐标系：frame 是算出来的，bounds 和 transform 才是存的（seriesOrder 21，577 行）—— 已过主会话核验
      **全程 Mac Catalyst，零模拟器。**
      **实测硬料**：
      - 「设了 transform 之后 frame undefined」有**精确可复现的行为**：对旋转 45° 的 view 写
        `frame=(0,0,50,50)`，得到 `bounds = 70.7 × 8.2156503822261584e-15`。反推出 setter 算法是
        「把 `frame.size` 当向量套一次 transform 逆变换的线性部分再取绝对值」，六组变换逐位吻合。
      - **SDK `UIView.h` 的 `anchorPoint` 注释是错的**：原文 `'(0, 0)' is the bottom left corner`，
        实测 UIKit 里是左上角。是从 macOS 的 CALayer 文档原样搬过来的。
      - **`transform3D` 会让 `view.transform` 读出 identity**：绕 Y 轴 45° 后
        `CGAffineTransformIsIdentity(view.transform) == 1`，而 frame 宽度已是 141.42。
        任何 `if (CGAffineTransformIsIdentity(...))` 的判断在 3D 变换下都会答错。
      - **`transform = scale(0,0)` 的 view 看不见但点得到**：`CGAffineTransform.h` 原文
        "If `t' has zero determinant, then `t' is returned unchanged"，于是坐标换算退化成恒等映射。
        `alpha=0` / `hidden` 没这个问题。
      - 无 window 时 `convertPoint:toView:nil` 原样返回输入且不报错（`viewDidLoad` 里的隐蔽坑）。
      **主会话独立复现**：四条全过，`8.2156503822261584e-15` 连最后一位都对上；
      两处头文件原文逐字核对无误；确认它引 `center` 那句时没有截断（全文含 "relative to anchorPoint"）。
      **它摸索出的通用技巧已写进规范**：最小 `.app` bundle + `codesign -s -` + `UIApplicationMain`，
      在 Catalyst 下拿到真 `UIWindow`。

### 第六周：UIKit 渲染与性能（UIKit 与渲染/、Foundation 与集合/）
- [ ] 19 UIView 与 CALayer：树结构、绘制流水线、离屏渲染
- [x] 20 UITableView：复用池的真实结构与代理调用顺序（seriesOrder 23，692 行）—— 已过主会话核验
      **全程 Mac Catalyst 无头运行，零模拟器。**
      **实测硬料**：
      - **复用池是 `NSMutableDictionary<标识符, NSMutableOrderedSet<cell>>`**，按标识符隔离，取尾放尾（栈）。
        `didEndDisplaying` 逆序入池，所以 `reloadData` 后 cell↔row 对应关系整个翻转。
      - **`estimatedRowHeight` 默认是 `UITableViewAutomaticDimension`(-1)，估算默认就开着**，设 0 才是关。
        头文件原文 `set to 0 to disable`。**"默认是 0、要手动设才开启"这个流行说法是反的。**
      - **只要实现了 `estimatedHeightForRowAtIndexPath:`，`estimatedRowHeight` 属性完全失效**
        （属性设 0 和设 66 两组输出逐字相同）。
      - **`reloadData` 新建 0 个 cell**（全走池），反而 `reloadRows` 新建了 1 个——一手证据是让数据源不一致时
        UIKit 自己打印的 `Updates = [Delete row (0-0), Insert row (0-0)]`，**reloadRows 内部就是 delete+insert**。
      - **"一屏 + 1" 只在慢滚成立**：慢滚 11 活/12 建，整屏跳一次变 20/21 且不再回落。
      - 内存警告会把复用池清空（池 5→0），文档没写。
      - `estimatedRowHeight` 的代价：10000 行、`heightForRow` 里做一次 `boundingRect`，
        **估算关 3805ms / 估算开 9.4ms**。`heightForRow` 次数有公式 `行数×2 + 可见行数×2`，两个规模都对得上。
      **它自陈踩的两个坑都写进了正文**：估算开着时 contentSize 精确等于真实总高，差点写成"UIKit 有补偿机制"，
      换参数重跑才发现 44/66/88 的平均正好是 66 是巧合；一步跳三屏复现不出复用错乱 bug，必须小步滚。
      **填了用户早年笔记的坑**（`csdn-155425916-ios-tableview.md` 第 197 行"离屏渲染检测工具这里先埋个坑"）：
      工具是模拟器 Debug → Color Off-screen Rendered；并指出老文章说的 Instruments「Core Animation 模板」
      在 Xcode 26.6 已不存在，能力归到 Animation Hitches。顺便纠正了该笔记里"一屏+1"那句。
      **主会话独立复现**：`estimatedRowHeight` 默认 = -1.0 = `UITableViewAutomaticDimension`；
      `_reusableTableCells` 确为 `NSMutableDictionary`。
      （核验时我第一次没找到复用池 ivar，是自己过滤字符串写错了——"reusable" 里没有 "euse" 这个子串。
      **连续第二次核验方自身出错，都不是文章的问题。**）
- [ ] 21 NSArray / NSDictionary / NSSet 的实现与选型（Foundation 与集合/）

### 第七周：编译、链接、Mach-O、dyld 与启动（编译链接与启动/）
- [x] 22 从源码到可执行文件：四个阶段与符号（seriesOrder 24，694 行）—— 已过主会话核验
      **实测硬料**：`#import <Foundation/Foundation.h>` 单独展开 90360 行 / 4.84 MB / 647 个头文件，
      贡献行数前三全不是 Foundation（MacErrors.h / mach/task.h / Security/cssmapi.h）。
      modules 冷 1.9s / 热 0.93s vs 文本 4.5s。`clang -###` 只 fork 两个进程，不是"四个阶段四个程序"。
      **死代码剥离对 ObjC 类无效**：编译器发 `no_dead_strip` + `@llvm.compiler.used` 双保险。
      **`__objc_classrefs` 消失门槛**：ios17 有 / ios18 无，macos14 有 / macos15 无——两个维度都扫了。
      `-fvisibility=hidden` 藏不住 ObjC 类，`NSClassFromString` 照样拿得到。
      **link map 会骗人**：`# Dead Stripped Symbols` 里列着的方法列表，在同一份 map 的存活区里也在，
      被剥的是绝对方法列表、链接器另建了相对方法列表顶上。只看 dead 那节会得出反的结论。
      **主会话独立复现**：200 个死 C 函数 0 字节代价 / 200 个死 ObjC 类 +120KB 且 classlist=0x640=200×8；
      classrefs 四个版本门槛逐条对上；静态库放最前照样链通 + `man ld` "continually search" 原文；
      `man nm` 确实没有 `W` 这一项。四条全过。
- [x] 23 Mach-O 重写：结构、符号绑定与 chained fixups（seriesOrder 25，802 行）—— 已过主会话核验
      **实测硬料**：惰性绑定整套（`__stub_helper` / `__la_symbol_ptr` / `dyld_stub_binder`）在默认构建里
      三样全无，`__stubs` 直接穿 `__DATA_CONST,__got`；加 `-Wl,-no_fixup_chains` 三样同时回来。
      `__LINKEDIT` 九块内容首尾相接加起来 51512，和 `ls -l` 一字节不差。
      代码签名 `hashes=13` = `ceil(50976/4096)`（签名用 4KB 页、VM 用 16KB 页）；改一字节后运行退出码 137。
      `strip -x` 砍掉 ByteDanceKit 44% 体积，符号表+字符串表原占 `__LINKEDIT` 的 88%。
      relative method list 门槛实测 iOS 14.0（13.0 上 100 个方法占 2408 字节，14.0 起 1208，正好省一半且免 rebase）。
      `__DATA_CONST` 的 `initprot` 是 `rw-`，靠 `SG_READ_ONLY`(0x10) 让 dyld 事后 `mprotect`。
      **主会话核验时抓到一条**：见第五批。
- [~] 15b AutoreleasePool：哨兵、页链表与 RunLoop 的接缝（内存管理/，seriesOrder 18）—— 写作中
- [x] 21 Foundation 集合：类簇、真实实现与选型（Foundation 与集合/，seriesOrder 29）—— 已完成，详见第三波条目
- [ ] 24 静态库与动态库：符号解析、`-all_load`、重复符号与 rpath
- [ ] 25 dyld 补章：三代 dyld、启动阶段耗时表
- [ ] 26 App 启动：pre-main / main / 首帧，以及可测量的优化项

### 第八阶段：持久化、序列化、源码与架构
- [ ] 27 iOS 持久化选型：plist / NSUserDefaults / 归档 / 文件 / Keychain（持久化与序列化/）
- [ ] 28 SQLite 与 FMDB / Core Data：事务、索引与并发（持久化与序列化/）
- [ ] 29 序列化：NSCoding / NSSecureCoding / JSON 三条路（持久化与序列化/）
- [ ] 30 JSONModel 源码：Runtime 驱动的属性映射（持久化与序列化/）
- [x] 31 YYModel 源码：为什么比 JSONModel 快（seriesOrder 33，571 行）—— 已过主会话核验
      **它在 YYModel 里找到一个从 2015 年留到今天的 bug**：`NSObject+YYModel.m:736-740`
      `case YYEncodingTypeInt32:` 的块**缺 `break`**，直接掉进 `case YYEncodingTypeUInt32:` 又赋值一次。
      **主会话直接核对源码坐实**：该文件其他每个 case 都是 `} break;`，全文 0 处 fall-through 注释。
      用户可见后果也复现了：`@(-2.7).unsignedIntValue` = **0**，覆盖掉正确的 −2；
      而 `longLongValue` = −2，所以同一份 JSON 进 `long long` 属性得 −2、进 `int` 属性得 **0**。
      `-Wimplicit-fallthrough`（不在 `-Wall` 里）当场能报出来。
      **拆账结论推翻了通说的排序**：端到端 14.5x（GitHub user.json，30 属性 10000 次：26.4ms vs 384ms）。
      拆开：**键映射时机独占 1.95x（贡献最大，几乎没人提）**，
      **函数指针绕开 KVC 只值 1.6x（被抄最多、贡献最小）**，元数据缓存首次/稳态 49x 但约 4800 次后摊薄到 1% 以下。
      1.95×1.6=3.1x，剩下 4.8x 归 JSONModel 的每次 init 固定开销。
      **ibireme 自己的八条 tip 里「减少遍历的循环次数」排最后，按贡献该排第一。**
      其他：**端到端 Swift `JSONDecoder` 118ms 比 YYModel+`NSJSONSerialization` 的 136ms 还快**；
      **arm64 上 SDK 强制关掉了变参原型**（`objc-api.h:100-105`），不强转根本编不过，
      实测变参调用让 `double` 参数丢失——「x86 上一直没事」只对一半（`float` 当场为 0）；
      selector stubs 与这一招无关（加 `-fno-objc-msgsend-selector-stubs` 汇编一字节不变）；
      **ibireme 原文没有数字表**，只有两张无数据标签的柱状图，网上流传的精确倍数在原文里找不到出处。
      **它自陈的踩坑**：第一版把 `@autoreleasepool` 套在循环外，JSONModel 中位数在 441/791/553ms 乱跳（15.5x~22.5x）；
      池挪进循环后收敛到 14.1/14.6/15.8x。**偏差方向和结论同向（分配多的库更吃亏），最危险。**
- [x] 30 JSONModel 源码：Runtime 驱动的属性映射（seriesOrder 32，782 行）—— 待主会话核验
      **推翻"它慢是因为用了 runtime 和 KVC"**：五个补丁删掉 42.3% 的差距（6.68x → 3.85x），
      **没有一个动到自省或 KVC 本身**。单项之和 44.2% 与叠加 42.3% 基本可加。
      **ObjC 泛型在 `property_getAttributes` 里被完全擦掉**：`NSArray<Kid *><Optional> *` 拿到
      `T@"NSArray<Optional>"`，`Kid *` 一字不剩；而 JSONModel 依赖的那个"协议"运行时根本不存在
      （`NSProtocolFromString(@"Kid")` 返回 NULL）。只写泛型不写协议→数组里留原始字典，`error` 还是 nil。
      "类型不匹配会崩"大部分不对：`int` 属性塞 `@"7"` 得 7、塞 `@"abc"` 静默得 0；
      唯一崩溃路径是标量属性收到 JSON 数组/对象，且**绕过 `error:` 参数**。
      `valueForKeyPath:` 没那么贵（无点的 key 只慢 2.2x），它第一版"先判断有没有点"的优化补丁反而慢 12.4%。
      **它纠正了主会话 prompt 里的错误**：我给的属性字符串范例 `T@"NSString",C,N,V_name` 里的 `C` 是错的，
      默认 strong 实测是 `&`，`C` 只在显式写 `copy` 时出现。
      **仪器翻车一次并自己修正**：`NSDictionary` 覆写了 `valueForKeyPath:`，swizzle `NSObject` 那份只数到 1 次。
- [x] 32 SDWebImage：下载、解码与两级缓存的完整链路（seriesOrder 34，581 行）—— 已过主会话核验
      **用户 2025 年那篇的十个符号今天一个都不在了**（`SDImageCacheDelegate` / `SDWebImageDecoder` /
      `NSURLConnection` / `shouldDecompressImages` / `maxCacheAge` / `SDCacheCostForImage` 等），
      使用方式仍成立，问题出在流程叙述抄的是 2015 年那批 3.x 解析、代码块却贴 5.x 实现。
      `SDWebImageOptions` 那张表从第三位起全部错位（`CacheMemoryOnly` 已删，`ProgressiveLoad` 占了 `1<<2`）。
      **实测推翻三条**：
      - **强制解码不省时间**。懒图第一次绘制 99~308 ms，预解码过的 8.8~10.1 ms（12~30 倍），
        但预解码本身要花 110 ms。**总量持平，省的只是主线程。**这点几乎没人写清楚。
      - **`NSCache` 的驱逐顺序会被读取动作改变**（反 LRU）。
      - **开箱即用的 SDWebImage 内存缓存没有容量上限**（`maxMemoryCost`/`maxMemoryCount` 默认都是 0）。
      **主会话独立复现**：20MB 上限塞 60 个 1MB，不读时存活 40–59，中途每次都读变成 **0–18 加 59**，
      三轮完全稳定；`evictsObjectsWithDiscardedContent` 默认 YES；`cost` 传 0 时 `totalCostLimit` 彻底失效（60/60 存活）。
      **它自陈踩的两个坑都写进了正文**：`phys_footprint` 量出"强制解码只涨 0.16 MB"是因为前一个模式已把 46 MB 弄脏、
      后一个直接复用（量的是分配器状态不是对象成本）；`NSCache` 那组第一版统计函数在循环里遍历所有 key，
      **探测本身改了被观察对象**，差点写出一套解释"NSCache 为什么反 LRU"的假机制。
      **主会话修掉一处坏双链**：它按计划猜的下一篇标题 `[[iOS 架构：MVC 与 MVVM 在图片列表上的分工]]`
      与实际文件名不符，已改为 `[[iOS 架构模式：MVC 到 MVVM，以及它们各自解决不了的问题]]`。
- [ ] 33 MVC / MVP / MVVM：职责边界与在 iOS 里的落地（架构与网络/）
- [x] 34 网络分层：URLSession 之上该有几层（seriesOrder 37，630 行）—— 已过主会话核验
      **⚠️ 最有生产价值的一条（安全性）：`URLCache` 的缓存 key 不含 `Authorization`。**
      用户 A 请求过之后，用户 B 带完全不同的 Bearer token 请求同一 URL，**直接拿到 A 的 body，
      而服务端根本没收到 B 的请求**。
      **主会话独立复现**（默认 `sharedSession`、无任何特殊缓存策略）：
      A 拿到 `body-for-Bearer-AAA`，B 也拿到 `body-for-Bearer-AAA`，服务端日志只有一行 `auth=Bearer-AAA`。
      **这不是实验室现象，是默认行为。**
      其他实测：
      - **默认就带 HTTP 缓存**——`sharedSession` 连发三次同一 URL，服务端日志只有一行。
        `URLCache.shared` 是 512000/20000000 字节，磁盘是 `Cache.db` 一个 SQLite 库，
        `cfurl_cache_response` 表能直接 select 出 request_key。
      - **重定向会剥掉 `Authorization`，同主机也剥**。流传的"跨主机才丢、同域名保留"不成立：
        同 host 绝对 URL、同 host 相对 URL、307 五种情况全丢，而自定义头 `X-My-Header` 全部保留。
        **它原本准备照通说写，测完改了。**
      - **"ephemeral 不缓存"是错的**：它的 `URLCache` 非 nil，`memoryCapacity` 和 default 一样 512000，
        只是 `diskCapacity=0`。
      - **`timeoutIntervalForRequest` 是空闲计时器**：设 3 秒、传输跑 12.6 秒、成功。
        两种超时 error code 都是 -1001，唯一稳定差别是 request 超时带 `NSUnderlyingError`。
      - background session 在 macOS 命令行程序里跑得起来且真出进程：进程 3 秒后 exit，
        10 秒的传输由 `nsurlsessiond` 接着做完。
      - **它自己写的重试封装真出了 bug**：cancel 落在退避窗口里被吞掉，服务端收到 2 次请求。
        这条是"重试和取消必须同层"的论证核心，**是实测出来的不是推理出来的**。
      - Alamofire 默认**不重试 429**、**退避无抖动**（grep 全文无 random/jitter），
        但**重试 -1200 和证书日期类错误**。据此它把提纲里"SSL 错误不该重试"改成"按能否几秒内自愈来切"，
        并写明不同意原说法。
      **它自陈的测量假象**：第一次跑重定向实验四行全是 `len=7`，看起来像"301/302 降级方法但保留 body"；
      实际是 keep-alive 让同一连接复用了 Python handler 实例，`do_GET` 读到上一次 POST 残留的 `self.body`。
      **诚实度**：`-1003` 在本机测不出来（系统级代理把 DNS 失败翻成了真实 HTTP 502），已标为本机特异性。

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

第三批（Block 捕获篇复审）：

| 原来写的 | 实际 |
| --- | --- |
| 函数内 static「编译器捕获的是它的地址」 | **根本不进 Block 结构体**。只碰它的 Block size 仍是 32，flags 带 `BLOCK_IS_GLOBAL`，连堆都不上。错误来源是老 `-rewrite-objc` 产物——而我上一篇刚说过 rewriter 不是真实 codegen |
| 「ARC 下 `__block` 修饰对象照样被持有」 | 结论对，但**是全文唯一没有实验支撑的断言**。补上决定性证据：同一句 `__block id`，ARC 编出 byref layout=`STRONG(3)`，MRC 编出 `UNRETAINED(5)` |
| 「编译器改写所有对该变量的访问走 forwarding」 | 说满了。声明处的初始化直接写字段，销毁时传的也是栈上那份 |
| `BLOCK_HAS_COPY_DISPOSE` = 捕获了对象 | `__block int` 也置位（要处理 byref 搬家）；捕获 `__weak` 也置位但调的是 `objc_copyWeak` |
| `Block_byref` 四字段 | 三段拼接，后两段由 flags 决定。`__block int` 的 byref size=32，`__block id` 是 48 |

**教训**：审查指出一个结构性问题——**全文最重要的断言是唯一没有实验的那条**，而已有实验的两节反而在重复前后篇的内容。篇幅分配和信息价值成反比。往后落笔前先问：这篇最核心的那句话，证据在哪。

第四批（Block 结构篇 + 循环引用篇复审）：

| 原来写的 | 实际 |
| --- | --- |
| 「MRC 下捕获普通 id 不 retain，ARC 才 retain」 | **说反了**。MRC 下 block copy 会 retain（retainCount 1→2）；不 retain 的是 `__block id`。而这跟我上一篇刚验的 byref layout 数据直接矛盾，隔一小时没对上 |
| `BLOCK_IS_NOESCAPE`「我不知道什么时候会被设」 | 答案就在我列在参考资料**第一位**的 Clang 文档 enum 上方三行。而且真相更好：Apple 出货的 clang 根本没实现这条 codegen（上游期望 `_NSConcreteGlobalBlock`+`0xD0800000`，Apple 实测 `_NSConcreteStackBlock`+`0xC0000000`） |
| `BLOCK_USE_STRET`「别拿它判断返回值类型」 | 截断式误引。文档下一句就说它和 `(1<<30)` 配成对，`(3<<29)` 才有意义 |
| 「ARC 下 `__block` 打不破环」 | 结论对但框架错。正确说法：**ARC 默认跟 strong，显式写 `__block __unsafe_unretained` 就能断环**（实测已释放）。变的是所有权修饰符的默认值，libclosure 一行没改 |
| 「调用空 block 跳到地址 0」 | 崩在**取 invoke 字段**那一步，`EXC_BAD_ACCESS address=0x10`，跳转从未发生 |
| `if (block) block()` 的竞态是悬垂指针 | ARC 已在调用点插了 retain/release。真正的问题是 TOCTOU；atomic 下连悬垂窗口都没有 |
| 「查复杂的环用 Memory Graph」 | Memory Graph 只标 **unreachable**。NSTimer/NotificationCenter 那类环的根是 RunLoop 和通知中心，对象**可达**，什么都不会报——照这句去查会得到"没问题"的错误结论 |

**审查也有说错的时候**：它说 `-Warc-retain-cycles` 对间接环"一概不报"，实测经函数传参的形态照样报。**审查结论同样要验。**

**捡到的最好素材**：`objc_loadWeakRetained` 正上方那段注释——"Once upon a time we eagerly cleared \*location... So we now don't touch the storage until deallocation completes."。我花一整节用实验证明的事，Apple 在源码里亲口解释了为什么这么设计。这类"源码注释里的设计自述"是最值钱的引用，以后每篇都该主动去翻。

**顺带确认了 19 位那个数字的真实归属**：只存在于无指针认证的 arm64（A7~A11，iPhone 5s 到 iPhone X）。arm64e 真机、所有 iOS 模拟器、所有 x86_64 都是 8 位——也就是说它在 Intel Mac 上从来没成立过。

第五批（Mach-O 篇，主会话核验时抓出，**这条是新类型的错**）：

| agent 写的 | 实际 |
| --- | --- |
| "chained fixups 的切换门槛是部署目标 iOS 13.4，网上说的 iOS 15 是记混了" | **半对，而且把对的说成了错的。** 它编了八个版本，全是 `arm64-apple-ios`（真机）。我加一个 `-simulator` 重跑：13.4~14.9 全是老格式，15.0 才切。**iOS 15 正是模拟器 target 的真实阈值**，流传得广大概因为大多数人在模拟器上验。三条线：真机 13.4 / 模拟器 15.0 / macOS 12.0 |

前四批的错误都是「没跑实验就下结论」。**这一条是跑了实验、数据也没错、结论仍然错**——因为它在一个维度上编了八次，另一个维度上一次都没动过。已把这条教训写进文章末尾，也补进 `_代写规范` 的证据纪律。

新增规范条款：**下"网上说的不对"这种结论之前，先列一遍这个实验有哪几个自变量，检查自己是不是只扫了其中一个。**

## 三点六、Mac Catalyst：UIKit 实验不再需要模拟器（2026-07-27 发现）

核验对象通信篇时找到的路子，**直接改变了后面六篇 UIKit 文章的做法**。

```shell
SDK=$(xcrun --sdk macosx --show-sdk-path)
clang -fobjc-arc -target arm64-apple-ios17.0-macabi -isysroot "$SDK" \
      -iframework "$SDK/System/iOSSupport/System/Library/Frameworks" \
      -framework Foundation -framework UIKit -o out prog.m && ./out
```

编出来是原生 macOS 二进制，`./out` 直接跑，但加载的是真正的 `UIKitCore`。已用它把手工解析 `class_ro_t`
得到的 `UIGestureRecognizerTarget._target = weak` 独立复现（`class_getWeakIvarLayout` 直接返回 `01`，
`ivarLayout` 为 null），两条互不相干的路径落到同一答案。

**原计划**：六篇 UIKit 文章串行做，一次 boot 一台模拟器、做完立刻关。
**改为**：先全部用 Catalyst 并行做，零模拟器；只有结论疑似受 AppKit 桥接层影响时才 boot 一台复核，
复核完立刻关。适用范围与边界已写进 `_代写规范` 第二节。

## 三点七、剩余 28 篇的并行安排（2026-07-27 起）

用户要求"一口气全部完成，可以多 agent 并行，质量第一"。据此调整流程：

**并行的是研究和初稿，事实验证和定稿仍由主会话做。** 理由：前九篇的质量来自"我自己跑实验 → 自己写 → 审查挑刺 → 我逐条验证再重写"，审查抓出的二十多条错误里有一半是靠"我自己的数据跟我自己的结论对不上"发现的，这个环节 agent 做不了（它们不会写到第三篇时想起第二篇的数据矛盾）。

规范已落成 `_代写规范（给 agent 读）.md`，每个写作 agent 必读，内容包括证据纪律（六条，全部来自实际踩过的坑）、可用实验手段、文风硬指标（加粗≤5、"不是X而是Y"≤3、破折号≤5、短句占比≥15%）、必须有第一人称在场、结构要求。

### 分批

**第一波（进行中，6 篇）**
- 11 KVC 与 KVO：搜索顺序、isa-swizzling 与手动触发（Runtime 与对象通信/）
- 12 对象通信：delegate、通知、target-action 与 block 回调（Runtime 与对象通信/）
- 13 Method Swizzling：正确姿势、+load 时机与那些坑（Runtime 与对象通信/）
- 14 GCD：队列不是线程，以及死锁的准确边界（并发与运行循环/）
- 15 NSOperation：状态机、依赖与自定义并发 Operation（并发与运行循环/）
- 16 锁：从 OSSpinLock 的废弃说起（并发与运行循环/）

**第二波（第五、六周，7 篇）**
RunLoop 重写、AutoreleasePool、事件传递与响应者链、UIViewController 生命周期、坐标系与 frame/bounds/transform、UIView 与 CALayer、UITableView 复用与卡顿

**第三波（第七周 + 集合，6 篇）**
编译链接四阶段、Mach-O 重写（现仅 21 行）、静态库与动态库、dyld 补章、App 启动、Foundation 集合原理

**第四波（第八阶段，9 篇）**
持久化选型、SQLite/FMDB/CoreData、序列化、JSONModel 源码、YYModel 源码、SDWebImage、MVC/MVP/MVVM、网络分层、综合项目设计文档

### 每篇的收口流程
初稿落盘 → 主会话验证关键断言（尤其是量化数字和跨篇一致性）→ 派审查 agent → 主会话据审查重写。

## 三点九、收口（2026-07-27）

**37 / 37 篇全部落盘。**全程 0 台模拟器（UIKit 那 6 篇靠 Mac Catalyst，见规范第二节）。

**最后做的三件事**：
1. 全系列双链体检——扫出并修掉 8 类真实坏链（`iOS UIView 与 CALayer：树结构…`→`三棵树…` 4 处、
   `iOS UIViewController 生命周期与容器控制器`→`生命周期的真实顺序与…`、
   五条 `20 专题笔记/Runtime/xxx` 路径写错）。**文章之间的坏链现在是 0。**
   残留 4 条 `.png` 缺图链接属用户原有的 `RunLoop 与 AutoReleasePool.md`，不在本次范围。
2. 补上网络分层篇（37）缺失的"下一篇"指针，链条接到综合项目设计文档（38）。
3. 修掉 03 篇内部的一处自相矛盾：第四节论证了 `@42` 是 `NSConstantIntegerNumber` 不是 tagged，
   第六节讲关联对象却用 `@(42)` 举例、把结论挂在"tagged pointer 永不 dealloc"上。
   实测 `@(42)` 同样被折叠成常量对象，`@(runtimeValue)` 才是 tagged。现象相同、机制不同，已补一段说清。

### 全程的方法论账

**这条规律从第一批错误保持到最后一篇，没有例外**：
**凡是来自编译产物、真实运行输出、一手源码原文的断言，一条都没错；
凡是来自记忆、二手转述、样本不足就推广、跨篇没对账的，反复出错。**

后期又长出两条推论：
- **跑了实验只能保证你测的那个组合是对的，保证不了它覆盖了读者会遇到的组合**（chained fixups 阈值、
  锁的耗时排名、Foundation 扩容倍率——三次都是"少扫了一个自变量"，其中两次是平台/架构这一维）。
- **转述会错，产物不会。**Foundation 集合那场三 agent 撞车里，两个 agent 都真诚地报告了错误的文件状态，
  最后是靠特征串比对裁决的。这和上面那条是同一件事的两面。

**核验方自身也会出错**：主会话在核验中出错三次（公式括号放错位置、过滤字符串写错、
把 `viewIsAppearing:` 的可用版本记成 iOS 17），三次都不是文章的问题，全部由 agent 或头文件原文纠正。
**交叉验证要连自己一起验。**

## 四、变更日志

- 2026-07-26：第一次对齐，确认范围 / 约定 / 34 篇清单。文档 03、04 的实验已在 `/tmp/ios-notes-lab/` 跑通，正文未写。会话因权限分类器不可用中断，重建本台账。
