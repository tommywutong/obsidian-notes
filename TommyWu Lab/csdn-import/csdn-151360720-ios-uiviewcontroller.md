---
title: "【iOS】UIViewController"
slug: "ios-uiviewcontroller"
published: 2025-09-14
description: "目录 ​编辑 视图加载： 视图可见性 viewWillAppear： viewDidAppear： viewWillDisappear： 总结 视图加载： 视图初始化会设计两个方法：loadView和ViewDidload 当新添加一个视图控制器时，通过xcode生成的代码模版只"
tags: ["iOS", "UIViewController"]
category: "iOS"
draft: false
---

**目录**


[​编辑](#%E2%80%8B%E7%BC%96%E8%BE%91)


[视图加载：](#%E8%A7%86%E5%9B%BE%E5%8A%A0%E8%BD%BD%EF%BC%9A)


[视图可见性](#%E8%A7%86%E5%9B%BE%E5%8F%AF%E8%A7%81%E6%80%A7)


[viewWillAppear：](#viewWillAppear%EF%BC%9A)


[viewDidAppear：](#viewDidAppear%EF%BC%9A)


[viewWillDisappear：](#viewWillDisappear%EF%BC%9A)


[总结](#%E6%80%BB%E7%BB%93)


##


## 视图加载：


视图初始化会设计两个方法：loadView和ViewDidload


![图片](./csdn-151360720-ios-uiviewcontroller-images/01-0ee63ea2effa47a59ec0ec6f9cbe3600.png)


当新添加一个视图控制器时，通过xcode生成的代码模版只有viewDidLoad代码。当视图控制器的view被请求时，loadView方法会被调用，但因为他还没背创建，所以会是nil。


> **视图通常会通过以下三种方式加载：** 从nibs 从故事板（UIStoryboardSegue） 使用自定义代码创建UI


> **如果通过覆写loadview方法创建了自定义UI，需要牢记：** 将view视图设置到视图层级的根上 确保视图正在被其他视图控制器共享 不要调用[super loadView]


![图片](./csdn-151360720-ios-uiviewcontroller-images/02-ad8efab32ec742a18d2b5bb4235b497b.png)


在视图层次结构准备就绪后，视图呈现给用户之前，viewDidload会被调用一次。可以在方法中做一些一次性的初始化操作。


## 视图可见性


视图控制器提供了四个生命周期方法，以接收有关视图可视性的通知。


![图片](./csdn-151360720-ios-uiviewcontroller-images/03-0d8cd842c7b64d0a9ae727e53ed76c33.png)


![图片](./csdn-151360720-ios-uiviewcontroller-images/04-994b0a3a00f1405a8f858d88ecc764cc.png)


### viewWillAppear：


当视图层级已经准备好，且视图即将被放入视图窗口时，此方法会被调用。在即将展示视图控制器或之前入栈（modal或者其他）的视图控制器弹出时，这种情况就会发生。


在这个时刻，过渡动画还未开始，视图对终端用户也是不可见的。不要启动任何视图动画，因为没有任何作用。


### viewDidAppear：


当视图在视图窗口展示出来，且过渡动画完成后，此方法会被调用。


因为动画会耗费约300毫秒，所以，对比viewWillAppear：和viewDidLoad：，viewDidAppear：和viewWillDisappear：之间的时间差可能会比较大。


### viewWillDisappear：


该方法表示视图将要从屏幕上隐藏起来。这可能是因为其他视图控制器想要接管屏幕，或该视图控制器将要出栈。


你可能会注意到，当此方法被调用时，没有办法能直接够判断这是由当前视图控制器要出栈还是其他视图控制器入栈导致的。


下文摘自【高性能iOS应用开发】，笔者会在以后慢慢理解


> **以下有一些高效使用生命周期事件的最佳实践：** ** **不要重写loadView 如果每次都需要展示最新的信息，那么就在viewWillAppear中更新UI元素。 viewDidappear中开始动画，如果有视频流内容，也可以开始播放了 viewWillDisappear来暂停或停止动画，不做多余操作 viewDid Disappear 销毁内存中的复杂数据结构。


## 总结


UIViewController生命周期


- `+ (void)initialize`：函数并不会每次创建对象都调用，只有在第一次初始化的时候才会调用，再次创建将不会调用`initialize`方法。
- `init`方法和`initCoder`方法相似，知识被调用的环境不一样。如果用代码初始化，会调用`init`方法，从nib文件或者归档(`xib`、`storyboard`)进行初始化会调用`initCoder`。`initCoder`是`NSCoding`协议中的方法，`NSCoding`是负责编码解码，归档处理的协议。
- `loadView`：是开始加载`view`的起始方法，除非手动调用，否则在`ViewController`的生命周期中只调用一次。
- `viewDidLoad`：是我们最常用的方法，类成员对象和变量的初始化我们都会放在这个方法中。在创建类后无论视图展现还是消失，这个方法也只会在布局是调用一次。
- `viewWillAppear:(BOOL)animated`：方法 是在视图将要展现出来的时候调用。
- `viewWillLayoutSubviews`：方法是在将要布局子视图的时候调用。
- `viewDidLayoutSubviews`：方法是在子视图布局完成后调用。
- `viewDidAppear:(BOOL)animated`：方法是视图已经出现。
- `viewWillDisappear:(BOOL)animated`：方法是视图即将消失。
- `viewDidDisappear:(BOOL)animated`：视图已经消失。
- `dealloc`：`ViewController`被释放时调用。

---

原文发布于 CSDN：[【iOS】UIViewController](https://blog.csdn.net/2402_86720949/article/details/151360720)
