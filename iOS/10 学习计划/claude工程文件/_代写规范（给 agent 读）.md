---
title: 代写规范（供 agent 读取）
draft: true
---
# 代写规范

这是《2026 暑假 iOS 底层学习计划》代写系列的统一规范。动笔前完整读一遍。
已完成的专题笔记在 `iOS/20 专题笔记/` 下（对象模型 / 内存管理 / Block 三个目录）。写作时应先确认文章只承担一条主线，不要把多篇独立专题机械合并。

---

## 一、最重要的一条：证据纪律

这个系列九篇下来，审查 agent 抓出二十多条硬错误。它们的分布有极强的规律：

- **凡是来自编译产物、真实运行输出、一手源码原文的断言 —— 一条都没错。**
- **凡是来自记忆、二手转述、样本不足就推广、或者跨篇没对账的 —— 反复出错。**

所以：

1. **任何量化断言必须有自己跑出来的数据**（多少位、多少倍、哪个更快、什么时候触发）。拿不到就写定性，不给数字。绝不编。
2. **样本不足时不要下方向性结论，更不要为它补机制。** 曾经只跑两次就写下"atomic 丢得比 nonatomic 多"并编了一套解释，跑满六次发现是噪声。
3. **引用源码必须抄原文**，不要凭印象复述。函数签名、常量值、返回类型都要核对——曾经把 `objc_storeStrong` 和 `objc_storeWeak` 的返回类型写反。
4. **列进参考资料不等于读过。** 曾经引了两篇标题写着 "Swift 4" 的文章，正文却抄了 Swift 4 之前的实现。
5. **警惕测量假象。** 观察手段可能改变被观察对象——曾经用 `id` 类型的参数去读 Block，ARC 在传参时插了 copy，于是"测出"ARC 下没有栈 Block。遇到结果和预期不符，**先怀疑仪器，再怀疑结论**。
6. **跨篇对账。** 写第 N 篇时如果结论和前面某篇的数据有关，回去对一遍。曾经隔一小时写出两条互相矛盾的结论。
7. **要下"网上说的不对"这种结论，先列一遍这个实验有几个自变量。** 曾经为了定 chained fixups 的切换阈值，把部署目标从 12.0 编到 17.0 一共八次，测出 13.4，于是写下"流传的 iOS 15 是记混了"。加一个 `-simulator` 重跑就发现模拟器的阈值真的是 15.0——版本这个维度扫了八遍，平台这个维度一次没动。**跑了实验只能保证你测的那个组合是对的，保证不了它覆盖了读者会遇到的组合。**

## 二、可用的实验手段

### ⚠️ 先看这条：默认不要开模拟器

**绝大多数实验根本不需要模拟器。** Objective-C runtime、Foundation、Block、GCD、锁、KVC/KVO、内存管理——这些直接编成 macOS 原生二进制跑就行，结果和模拟器上一致（已实测对照过 KVC setter 搜索链、集合运算符返回类型、isa 位域与 extra_rc 溢出，输出逐字相同）。

**默认写法**（不碰模拟器）：

```shell
clang -fobjc-arc -framework Foundation -o out prog.m && ./out
clang -fno-objc-arc -framework Foundation -o out_mrc prog.m && ./out_mrc   # MRC 对照
clang -fobjc-arc -S -emit-llvm -O0 -o out.ll prog.m                        # 看 IR
clang -fobjc-arc -S -O1 -o out.s prog.m                                    # 看汇编
```

### UIKit 也未必需要模拟器：先试 Mac Catalyst

这条是 2026-07-27 验出来的，能省掉大部分模拟器需求。**Mac Catalyst target 编出来的是原生 macOS 二进制，但链接和加载的是真正的 `UIKitCore`**，直接 `./out` 就跑，不 boot 任何设备：

```shell
SDK=$(xcrun --sdk macosx --show-sdk-path)
clang -fobjc-arc -target arm64-apple-ios17.0-macabi -isysroot "$SDK" \
      -iframework "$SDK/System/iOSSupport/System/Library/Frameworks" \
      -framework Foundation -framework UIKit -o out prog.m && ./out
```

已经用它做成的事：`class_getIvarLayout` / `class_getWeakIvarLayout` 直接问 runtime 拿到 `UIGestureRecognizerTarget._target` 是 weak（`weakIvarLayout=01`、`ivarLayout` 为 null），比手工解析 Mach-O 的 `class_ro_t` 可靠得多，也不用管 chained fixups 的指针高位。

适合 Catalyst 的：UIKit 私有类的结构与 ivar 语义、`UIView` / `CALayer` 的树结构和 `frame`/`bounds`/`transform` 换算、`hitTest:` 与响应者链的算法、`UIViewController` 的生命周期回调顺序、`UITableView` 的复用与代理调用顺序。这些都是 UIKitCore 里同一份代码。

**需要真 `UIWindow` 时**：直接 `[[UIWindow alloc] init]` 会抛 `NSApplication has not been created yet`。做一个最小 `.app` bundle 绕过——写个 `Info.plist`（`CFBundleExecutable` / `CFBundleIdentifier` / `CFBundlePackageType=APPL`），`codesign -s -` 签一下，再用 `UIApplicationMain` 起进程，就能拿到真 window。`convertPoint:toView:nil` 这类依赖 window 的语义因此可以实测而不是推测。

**已知在 Catalyst 上不可信的具体项**：`layer.contentsScale` 实测是 1.00，而同一进程里 `screen.scale` 和 `traitCollection.displayScale` 都是 2.00；`UIScreen.mainScreen.bounds` 返回的是 Mac 显示器尺寸（同一台机器两次运行可能是 960×600 和 1920×1080）。**任何依赖屏幕尺寸或 scale 的数字都不要写进正文。**

不适合 Catalyst 的：触摸事件的真实输入路径、屏幕 scale 与安全区、iOS 专属的系统行为、任何和 AppKit 桥接层相关的表现。

**另外：绝对不要用 `CGEvent` 合成鼠标/键盘事件来"制造"触摸。** 它是系统级的，会真的移动用户光标、真的按下鼠标，落点不保证在你的窗口里，还会弹辅助功能授权框。要验命中测试就直接调 `[view hitTest:point withEvent:nil]`（`event` 传 nil 完全合法），要验滚动就直接改 `contentOffset`。构造不出真实 `UITouch` 的实验，如实写做不了。**凡是结论可能受 Catalyst 桥接影响的，要么标注"Catalyst 上实测，未在 iOS 复核"，要么按下面的纪律开一台模拟器复核一次。**

注意：直接 `dlopen` `/System/iOSSupport/.../UIKitCore` 会报 `wrong platform to load into process`，必须整个二进制编成 macabi target。

**剩下这几类才真的要模拟器**：iOS 专属的系统行为、版本门控差异、Catalyst 上表现存疑需要复核的结论。

需要时的纪律，三条都是硬性的：

1. **先查有没有已经开着的**，有就复用，不要再 boot：
   ```shell
   xcrun simctl list devices booted | grep Booted
   ```
2. **同一时刻只允许存在一台。** 不要 boot 第二台。
3. **用完立刻关**，不要留给下一个人：
   ```shell
   xcrun simctl shutdown <udid>
   ```

违反这三条的后果是真实发生过的：一批 agent 各自 boot 一台、谁也不清理，堆到 18 GB 内存、540 个进程，把用户的机器逼到可用内存 39%，整批工作被迫中止。**开模拟器之前先问一句：这个实验真的需要 UIKit 吗？**

写作时如果实验是 macOS 原生跑的，在「实验环境」一节如实写明（`clang -framework Foundation`，macOS arm64），不要谎称在模拟器上跑。涉及 iOS 与 macOS 可能有差异的结论（分片数、tagged pointer 布局、系统版本相关行为），要么去模拟器复核一次，要么标注"未在 iOS 上复核"。

### 编译选项对照

```shell
clang -fobjc-runtime=ios-12.0 ...          # runtime 版本门控对照
clang -target x86_64-apple-macos13 ...     # 换架构对照
# 确实需要 iOS 环境时：
SDK=$(xcrun --sdk iphonesimulator --show-sdk-path)
clang -fobjc-arc -isysroot "$SDK" -target arm64-apple-ios17.0-simulator -framework Foundation -o out prog.m
xcrun simctl spawn booted ./out
```

常用招式：
- 照着一手头文件手写一份布局一致的 struct，强转过去直接读内存（读 isa 位域、Block_layout、Block_byref 都是这么干的）
- ARC / MRC 两份分别编译做对照
- `-O0` / `-O1` 对照，看哪些插桩被优化掉
- 换 target 架构对照（arm64 vs x86_64）
- 同一程序连跑多次，判断某个现象是稳定的还是噪声
- grep 当前 SDK 的头文件验证 API 契约（`NS_NOESCAPE`、`NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE` 等）

**做不到的**：连真机、开 Instruments GUI、跑 UIKit 界面交互、真实网络环境。这类实验写清设计和预期观察点，数字留 `> 待真机补测` 占位，**不要假装测过**。

## 三、文风约束

用户两次提出"AI 味太重"，以下是据此定的硬指标，都被审查逐条量化检查过：

- **加粗全文不超过 5 处**，只留读者只看粗体也能拿到全文结论的句子。整句加粗 = 加粗失效。
- **"不是 X，而是 Y" 这个句式全文不超过 3 次。** 它曾经在一篇里出现 17 次，审查评语是"一个句式被当成结论生成器用了十次，人类写作者会在第三次的时候烦"。
- **破折号 `——` 不超过 5 个。** 它容易变成"抛概念 + 解释"的固定模具。改用句号断句。
- **短句占比**：10 字以内的句子要占到 15% 左右。长句（45 字以上）不超过 25%。
- **段落长短要参差**：允许一句成段，也要有 5-6 句的长段。平均 2 句一段是最像 AI 的节奏。
- **不要每节都以总结句收尾。** 允许某些小节直接停在数据或代码上。
- **总结压到 5 条以内，或者直接写成段落。** 不要把正文复述一遍。
- **禁用词**：值得注意的是 / 总的来说 / 换句话说 / 事实上 / 不难看出 / 众所周知 / 综上 / 顺带 / 顺手 / 值得说一句 / 挺有意思 / 值得留意 / 仅此而已。
- **不要"三个 X"式的强行编号**，也不要为对称而对称的排比。
- **表格只用于真正的多维对照**，两句话能说清的不列表。

## 四、必须有作者在场

审查反复指出的一条：AI 腔的核心不是词汇，是**没有人称在场**。

- 用第一人称写判断：「我的判断是这种写法基本不该用」「我自己的做法是：只要 block 里有分支或者多于一条语句，就写」。给**具体、可执行的个人阈值**，这是真人写作的标志。
- **写自己踩的坑**。「我一开始也是这么测的」「我第一次跑这个实验，以为 forwarding 没生效」——这类段落是全系列可信度最高的部分。
- **该下判断就下判断**，不要和稀泥。遇到"我不知道"，先把手上的资料读完再说这句话（曾经写"我不知道 `BLOCK_IS_NOESCAPE` 什么时候置位"，答案就在自己列在参考资料第一位的文档里）。
- 允许写「这个说法我不同意」。

## 五、结构

不套固定模板，但以下几点是有效的：

- **开头三段之内进入正题**，最好用一个具体的反常现象当钩子，而不是"本文将介绍……"。
- **最有价值的部分不要埋太深**。曾经把唯一的原创实验放在第四节，读者要翻过 40% 才看到。
- **核心断言必须有证据**。审查抓到过一次：全文最重要的那条论断是唯一没有实验支撑的，而已有实验的两节反而在重复前后篇的内容——篇幅分配和信息价值成反比。
- 结尾指向系列下一篇，用 `[[双链]]`。

## 六、frontmatter 格式

```yaml
---
title: 【iOS】<标题>
published: 2026-07-27
description: <一句话，最好点出本文最反直觉的那个发现>
tags: [iOS, ...]
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: <序号>
draft: true
---
```

## 七、本地可引用的已有笔记

- `iOS/20 专题笔记/Runtime/Part 1 - 对象与类的本质.md`、`Part 2 - 消息发送与转发.md`、`Part 3 - Category：加载、覆盖与关联对象.md` —— 用户自己写的，质量高，**引用不重写**
- `iOS/20 专题笔记/内存管理/` 下六篇、`对象模型/` 一篇、`Block/` 三篇 —— 本系列已完成部分，可双链引用
- `iOS/20 专题笔记/编译链接与启动/dyld.md` —— 可用，需补充
- `iOS/90 素材/csdn-import/` —— 用户早年文章，KVC / MVC / MVVM / SDWebImage 几篇可引用

## 八、交付

写完直接落到指定路径。不要在正文里留 TODO；实验留白用 `> 待真机补测：…` 的引用块形式，并说明复现方法。
