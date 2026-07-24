# _Inbox


## 2026-05-15 12:07 ARC 下 Block 的内存类型与 __block 作用
**来源**: 未知　**confidence**: 0.90

- Block 在 ARC 下默认是 __NSStackBlock__。
- 捕获外部变量后，Block 自动 copy 到堆区变为 __NSMallocBlock__。
- __block 修饰的变量被包装成堆上结构体，实现 Block 内外内存共享。

### 整理后内容

Block 在 ARC 下默认是 __NSStackBlock__，捕获外部变量后会自动 copy 到堆区变成 __NSMallocBlock__。__block 修饰的变量会被包装成结构体放到堆上，让 block 内外共享同一份内存。

<details><summary>原文</summary>

Block 在 ARC 下默认是 __NSStackBlock__，捕获外部变量后会自动 copy 到堆区变成 __NSMallocBlock__。__block 修饰的变量会被包装成结构体放到堆上，让 block 内外共享同一份内存。

</details>

---



## 2026-05-16 14:51:30 图片 ^aa4e18
topic:: iOS-Inbox
date:: 2026-05-16 14:51:30
source:: 未知
confidence:: 0.00
tags:: iOS-Inbox
**来源**: 未知　**confidence**: 0.00


![[sec-01756e1b1aa5.png]]

<details><summary>原文</summary>

(图片，无 OCR 文本)

</details>

---


## 2026-05-25 11:32:51 const 关键字基础介绍 ^49af5c
topic:: iOS-Inbox
date:: 2026-05-25 11:32:51
source:: 未知
confidence:: 0.70
tags:: iOS-Inbox
summary:: const 用于声明常量，表示变量的值不可被修改。；在 C/Objective-C 中，const 可以修饰变量、指针等。；const 有助于提高代码的安全性和可读性。
**来源**: 未知　**confidence**: 0.70

- const 用于声明常量，表示变量的值不可被修改。
- 在 C/Objective-C 中，const 可以修饰变量、指针等。
- const 有助于提高代码的安全性和可读性。

### 整理后内容

**const（常量）**

`const` 是编程语言中的一个关键字，用于声明一个常量，即其值在初始化后不可被修改。

在 C 和 Objective-C 语言中，`const` 可以修饰基本类型的变量、指针等，例如：
- `const int MAX = 100;` 声明一个整型常量 `MAX`。
- `NSString * const kMyConstant = @"value";` 声明一个指向 `NSString` 的常量指针（指针本身不可变）。

使用 `const` 有助于防止意外修改，增强代码的安全性和可读性。

<details><summary>原文</summary>

const（常量）

</details>

---


## 2026-05-25 11:46:29 NSNotificationCenter与NSNotificationQueue的区别 ^e0a747
topic:: iOS-Inbox
date:: 2026-05-25 11:46:29
source:: 自述
confidence:: 0.80
tags:: iOS-Inbox, 自述
summary:: 主题是 NSNotificationCenter 与 NSNotificationQueue 的区别；被标记为‘重要’，可能是一个学习重点或面试题；内容属于 iOS 开发中的通知机制，但不属于 Runtime 等已有主题
**来源**: 自述　**confidence**: 0.80

- 主题是 NSNotificationCenter 与 NSNotificationQueue 的区别
- 被标记为‘重要’，可能是一个学习重点或面试题
- 内容属于 iOS 开发中的通知机制，但不属于 Runtime 等已有主题

### 整理后内容

NSNotificationCenter 与 NSNotificationQueue 的区别（重要）

<details><summary>原文</summary>

NSNotificationCenter与NSNotificationQueue的区别（重要）

</details>

---


## 2026-05-25 12:22:29 iOS 中的模型树、动画树与渲染树 ^3e8339
topic:: iOS-Inbox
date:: 2026-05-25 12:22:29
source:: 未知
confidence:: 0.70
tags:: iOS-Inbox
summary:: CALayer 作为渲染树的节点；模型树管理属性状态；动画树处理动画过渡；渲染树负责最终像素输出
**来源**: 未知　**confidence**: 0.70

- CALayer 作为渲染树的节点
- 模型树管理属性状态
- 动画树处理动画过渡
- 渲染树负责最终像素输出

### 整理后内容

模型树、动画树、渲染树（节点是 CALayer）。

<details><summary>原文</summary>

模型树、动画树、渲染树（节点是CALayer）

</details>

---


## 2026-05-25 12:34:09 UICollectionView的布局与流式布局 ^1711e4
topic:: iOS-Inbox
date:: 2026-05-25 12:34:09
source:: 博客
confidence:: 0.80
tags:: iOS-Inbox, 博客
summary:: 主题聚焦于UICollectionView的布局系统；包含UICollectionViewLayout基类；包含UICollectionViewFlowLayout流式布局；属于iOS开发中的视图组件布局内容
**来源**: 博客　**confidence**: 0.80

- 主题聚焦于UICollectionView的布局系统
- 包含UICollectionViewLayout基类
- 包含UICollectionViewFlowLayout流式布局
- 属于iOS开发中的视图组件布局内容

### 整理后内容

UICollectionView的Layout和FlowLayout

<details><summary>原文</summary>

UICollectionView的Layout和FlowLayout

</details>

---


## 2026-05-25 12:38:34 手势识别器 cancelsTouchesInView 属性解析 ^633ae6
topic:: iOS-Inbox
date:: 2026-05-25 12:38:34
source:: 未知
confidence:: 0.60
tags:: iOS-Inbox
summary:: cancelsTouchesInView 是 UIGestureRecognizer 的一个属性；该属性默认值为 YES（对于大多数手势识别器）；当设为 YES 时，识别手势后会取消触摸事件的传递；当设为 NO 时，触摸事件会继续传递给视图
**来源**: 未知　**confidence**: 0.60

- cancelsTouchesInView 是 UIGestureRecognizer 的一个属性
- 该属性默认值为 YES（对于大多数手势识别器）
- 当设为 YES 时，识别手势后会取消触摸事件的传递
- 当设为 NO 时，触摸事件会继续传递给视图

### 整理后内容

cancelsTouchesInView（默认对大多数手势为 YES）

<details><summary>原文</summary>

cancelsTouchesInView（默认对大多数手势为 YES）

</details>

---


## 2026-05-25 12:39:22 delaysTouchesBegan 属性解析 ^565ba5
topic:: iOS-Inbox
date:: 2026-05-25 12:39:22
source:: 未知
confidence:: 0.50
tags:: iOS-Inbox
summary:: delaysTouchesBegan 是 UIControl 的属性，默认值为 NO；当设置为 YES 时，会延迟触摸开始事件的传递，以等待手势识别器确认；此行为常用于避免与手势识别器（如 UIPanGestureRecognizer）的冲突
**来源**: 未知　**confidence**: 0.50

- delaysTouchesBegan 是 UIControl 的属性，默认值为 NO
- 当设置为 YES 时，会延迟触摸开始事件的传递，以等待手势识别器确认
- 此行为常用于避免与手势识别器（如 UIPanGestureRecognizer）的冲突

### 整理后内容

delaysTouchesBegan（默认 NO）

<details><summary>原文</summary>

delaysTouchesBegan（默认 NO）

</details>

---


## 2026-05-25 12:58:24 使用NSUserDefaults存储用户偏好设置 ^cb9b5c
topic:: iOS-Inbox
date:: 2026-05-25 12:58:24
source:: 未知
confidence:: 0.70
tags:: iOS-Inbox
summary:: NSUserDefaults用于存储轻量级的用户偏好设置；可通过键值对形式存储基本数据类型；常用于保存应用配置、用户选项等
**来源**: 未知　**confidence**: 0.70

- NSUserDefaults用于存储轻量级的用户偏好设置
- 可通过键值对形式存储基本数据类型
- 常用于保存应用配置、用户选项等

### 整理后内容

用NSUserDefaults存储用户偏好

<details><summary>原文</summary>

用NSUserDefaults存储用户偏好

</details>

---


## 2026-05-26 13:01:16 iOS 沙盒机制与核心目录详解 ^e3f973
topic:: iOS-Inbox
date:: 2026-05-26 13:01:16
source:: 博客
confidence:: 0.90
tags:: iOS-Inbox, 博客
summary:: 每个 App 拥有独立的沙盒目录，确保 App 间隔离与安全。；沙盒包含四个核心目录：Documents、Library/Caches、Library/Preferences 和 tmp。；Documents 存储用户重要数据，会被 iCloud 备份；Library/Caches 存储可再生缓存，不备份且可被清理。；Library/Preferences 存储用户偏好设置，通过 NSUserDefaults 以 plist 文件存储，会被备份。；tmp 存储临时文件，不备份且系统可随时清空。
**来源**: 博客　**confidence**: 0.90

- 每个 App 拥有独立的沙盒目录，确保 App 间隔离与安全。
- 沙盒包含四个核心目录：Documents、Library/Caches、Library/Preferences 和 tmp。
- Documents 存储用户重要数据，会被 iCloud 备份；Library/Caches 存储可再生缓存，不备份且可被清理。
- Library/Preferences 存储用户偏好设置，通过 NSUserDefaults 以 plist 文件存储，会被备份。
- tmp 存储临时文件，不备份且系统可随时清空。

### 整理后内容

## iOS 沙盒机制

每个 App 拥有独立的**沙盒目录（Sandbox）**，App 之间互相隔离，确保安全。

### 沙盒的四个核心目录

| 目录 | 存储内容 | 备份策略 | 特点 |
|------|---------|----------|------|
| **Documents** | 用户重要数据（离线文件、视频草稿等） | ✅ iCloud 备份 | 长期持久化存储 |
| **Library/Caches** | 可再生缓存（图片/视频离线缓存） | ❌ 不备份，系统空间不足时可被清理 | 缓存数据 |
| **Library/Preferences** | 用户偏好设置（字体大小、夜间模式等） | ✅ 备份 | 底层通过 `NSUserDefaults` 以 plist 文件存储 |
| **tmp** | 临时文件（如下载中的片段） | ❌ 不备份，系统可随时清空 | 短期临时存放 |

> **关键安全原则**：每个 App 只能读写自己的沙盒目录，无法访问其他 App 的沙盒，这是 iOS 安全模型的基础。

<details><summary>原文</summary>

## iOS 沙盒机制

每个 App 拥有独立的**沙盒目录（Sandbox）**，App 之间互相隔离，确保安全。

### 沙盒的四个核心目录

| 目录 | 存储内容 | 备份策略 | 特点 |
|------|---------|----------|------|
| **Documents** | 用户重要数据（离线文件、视频草稿等） | ✅ iCloud 备份 | 长期持久化存储 |
| **Library/Caches** | 可再生缓存（图片/视频离线缓存） | ❌ 不备份，系统空间不足时可被清理 | 缓存数据 |
| **Library/Preferences** | 用户偏好设置（字体大小、夜间模式等） | ✅ 备份 | 底层通过 `NSUserDefaults` 以 plist 文件存储 |
| **tmp** | 临时文件（如下载中的片段） | ❌ 不备份，系统可随时清空 | 短期临时存放 |

> **关键安全原则**：每个 App 只能读写自己的沙盒目录，无法访问其他 App 的沙盒，这是 iOS 安全模型的基础。

</details>

---


## 2026-05-26 13:01:34 iOS 沙盒机制概述 ^e22155
topic:: iOS-Inbox
date:: 2026-05-26 13:01:34
source:: 未知
confidence:: 0.80
tags:: iOS-Inbox
summary:: iOS 沙盒（Sandbox）是应用程序的隔离运行环境，用于限制应用对文件系统、网络等资源的访问。；每个应用都有自己独立的沙盒目录，包括 Documents、Library、tmp 等子目录。；沙盒机制保障了系统的安全性，防止应用间相互干扰或恶意访问。；沙盒内应用只能读写自己的目录，访问其他应用数据需要特定权限或 API（如共享数据容器）。
**来源**: 未知　**confidence**: 0.80

- iOS 沙盒（Sandbox）是应用程序的隔离运行环境，用于限制应用对文件系统、网络等资源的访问。
- 每个应用都有自己独立的沙盒目录，包括 Documents、Library、tmp 等子目录。
- 沙盒机制保障了系统的安全性，防止应用间相互干扰或恶意访问。
- 沙盒内应用只能读写自己的目录，访问其他应用数据需要特定权限或 API（如共享数据容器）。

### 整理后内容

iOS 沙盒机制

<details><summary>原文</summary>

iOS沙盒机制

</details>

---


## 2026-05-27 12:34:13 C 语言 static 关键字的含义与用法 ^8def59
topic:: iOS-Inbox
date:: 2026-05-27 12:34:13
source:: 未知
confidence:: 0.70
tags:: iOS-Inbox
summary:: static 用于函数或变量时，限制其作用域为当前文件（内部链接）；static 局部变量在函数调用之间保持值，生命周期与程序相同；static 全局变量仅在当前文件可见，避免命名冲突；static 函数只能在定义它的文件内调用，提供模块化封装
**来源**: 未知　**confidence**: 0.70

- static 用于函数或变量时，限制其作用域为当前文件（内部链接）
- static 局部变量在函数调用之间保持值，生命周期与程序相同
- static 全局变量仅在当前文件可见，避免命名冲突
- static 函数只能在定义它的文件内调用，提供模块化封装

### 整理后内容

## static（静态）

在 C 语言中，`static` 是一个存储类关键字，主要有两种用途：

1.  **用于局部变量**：
    *   该变量的生命周期将延长至整个程序运行期间，但作用域仍然局限于定义它的函数内部。
    *   它只被初始化一次，后续函数调用将保留上一次的值。

2.  **用于全局变量或函数**：
    *   该变量或函数的作用域被限制在当前源文件内部（即具有内部链接性），其他文件无法访问。这有助于实现数据封装和避免命名冲突。

<details><summary>原文</summary>

static（静态）

</details>

---


## 2026-05-28 06:56:37 在run回调中通过标识名获取AST节点 ^2c67f3
topic:: iOS-Inbox
date:: 2026-05-28 06:56:37
source:: 未知
confidence:: 0.80
tags:: iOS-Inbox
summary:: 用户询问在run回调中如何根据标识名从MatchResult获取特定AST节点（如ObjCPropertyDecl）；问题涉及静态分析工具或AST匹配框架的使用；核心关注点是MatchResult对象的API或方法调用
**来源**: 未知　**confidence**: 0.80

- 用户询问在run回调中如何根据标识名从MatchResult获取特定AST节点（如ObjCPropertyDecl）
- 问题涉及静态分析工具或AST匹配框架的使用
- 核心关注点是MatchResult对象的API或方法调用

### 整理后内容

在run回调中，如何根据标识名从MatchResult中获取具体的AST节点（如ObjCPropertyDecl）？

<details><summary>原文</summary>

在run回调中，如何根据标识名从MatchResult中获取具体的AST节点（如ObjCPropertyDecl）？

</details>

---


## 2026-06-08 14:29:59 Clang MatchFinder 配置规则与回调关联 ^daabe8
topic:: iOS-Inbox
date:: 2026-06-08 14:29:59
source:: 博客
confidence:: 0.90
tags:: iOS-Inbox, 博客
summary:: 使用 `matcher.addMatcher()` 添加匹配规则，参数为 AST 节点声明器（如 `objcPropertyDecl()`）和 `MatchCallback` 对象指针；通过 `.bind("标识名")` 为匹配的节点绑定字符串标识；匹配成功时自动调用 `MatchCallback` 的 `run` 方法，传入 `MatchResult`；在 `run` 方法中，使用 `Result.Nodes.getNodeAs<NodeType>("标识名")` 获取具体的 AST 节点进行处理
**来源**: 博客　**confidence**: 0.90

- 使用 `matcher.addMatcher()` 添加匹配规则，参数为 AST 节点声明器（如 `objcPropertyDecl()`）和 `MatchCallback` 对象指针
- 通过 `.bind("标识名")` 为匹配的节点绑定字符串标识
- 匹配成功时自动调用 `MatchCallback` 的 `run` 方法，传入 `MatchResult`
- 在 `run` 方法中，使用 `Result.Nodes.getNodeAs<NodeType>("标识名")` 获取具体的 AST 节点进行处理

### 整理后内容

在 MatchFinder 中配置匹配规则和关联回调：

**配置匹配规则：**
使用 `matcher.addMatcher()` 方法添加规则。第一个参数是 AST 节点声明器（如 `objcPropertyDecl()`），并通过 `.bind("标识名")` 绑定一个字符串标识；第二个参数是 `MatchCallback` 对象的指针。

**匹配到的节点与回调关联：**
匹配成功时，会自动调用回调对象的 `run` 方法，并传入匹配结果（包含标识名）。在 `run` 方法中，可以通过 `result.Nodes.getNodeAs<NodeType>("标识名")` 获取具体的 AST 节点（如 `ObjCPropertyDecl`），其中标识名需与 `.bind()` 中一致。

```cpp
// 配置规则
matcher.addMatcher(objcPropertyDecl().bind("property"), &callback);

// 在 MatchCallback 的 run 方法中
void run(const MatchFinder::MatchResult &Result) override {
    const auto *prop = Result.Nodes.getNodeAs<ObjCPropertyDecl>("property");
    // 处理节点
}
```

<details><summary>原文</summary>

在 MatchFinder 中配置匹配规则和关联回调：

**配置匹配规则：**
使用 `matcher.addMatcher()` 方法添加规则。第一个参数是 AST 节点声明器（如 `objcPropertyDecl()`），并通过 `.bind("标识名")` 绑定一个字符串标识；第二个参数是 `MatchCallback` 对象的指针。

**匹配到的节点与回调关联：**
匹配成功时，会自动调用回调对象的 `run` 方法，并传入匹配结果（包含标识名）。在 `run` 方法中，可以通过 `result.Nodes.getNodeAs<NodeType>("标识名")` 获取具体的 AST 节点（如 `ObjCPropertyDecl`），其中标识名需与 `.bind()` 中一致。

```cpp
// 配置规则
matcher.addMatcher(objcPropertyDecl().bind("property"), &callback);

// 在 MatchCallback 的 run 方法中
void run(const MatchFinder::MatchResult &Result) override {
    const auto *prop = Result.Nodes.getNodeAs<ObjCPropertyDecl>("property");
    // 处理节点
}
```

</details>

---
