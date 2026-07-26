---
title: "【iOS】push 和 present"
slug: "ios-push-vs-present"
published: 2025-09-06
description: "转场动画是下面几个情况： 导航控制器的push和pop动画。 普通控制器的present和Dismiss动画。 pesent和dismiss : dismiss多级 present还有两个方法可以让我们实现一个跨级返回的效果。presentingViewController 和p"
tags: ["iOS", "页面跳转"]
category: "iOS"
draft: false
---

> 转场动画是下面几个情况： 导航控制器的push和pop动画。 普通控制器的present和Dismiss动画。


## pesent和dismiss :


![图片](./csdn-151260513-ios-push-present-images/01-456da7a46ca64624b9436663279db117.png)


```objective-c
ViewController3* vc2 = [[ViewController3 alloc] init];
[self presentViewController:vc2 animated:YES completion:nil];

[self dismissViewControllerAnimated:YES completion:nil];
```


![图片](./csdn-151260513-ios-push-present-images/02-7947cbc6cf9b476b8c46badd9b3f6710.gif)


### dismiss多级


present还有两个方法可以让我们实现一个跨级返回的效果。presentingViewController 和presentedViewController这两个方法分别是什么呢？这里简单解释一下返回两个的对应视图控制器。


当从1中弹出2后：


self.presentingViewController 在1中，就是nil；在2中，就是1

self.presentedViewController在1中，就是2；在2中，就是nil

 


## push和pop


`pushViewController:animated:` 方法用于将新的视图控制器推入导航栈。这意味着新控制器将显示在当前控制器的上方，同时当前控制器仍然在堆栈中。


```objective-c
SecondViewController *second = [[SecondViewController alloc] init];
    [self.navigationController pushViewController:second animated:YES];

	//返回上一级
	[self.navigationController popViewControllerAnimated:YES];

	//返回根视图
	[self.navigationController popToRootViewControllerAnimated:YES];

	//返回指定级数 （objectAtIndex:参数为想要返回的级数）
	[self.navigationController popToViewController:[self.navigationController.viewControllers objectAtIndex:0]  animated:YES];
```


pop有两个类别：


第一个类别就是返回上一层


```objective-c
[self.navigationController popViewControllerAnimated:YES];
```


第二个类别就是返回到某一层


```objective-c
[self.navigationController popToRootViewControllerAnimated:YES];//这个是返回到根视图
[self.navigationController popToViewController:viewController animated:YES];//返回指定的某一层视图控制器
```


这里我们返回某一层的视图控制器可以通过这种方式来返回self.navigationController.viewControllers[i]这里的i是你需要的viewController的层级也可以采用for循环通过判断我们的一个view是否符合isKindeOfClass这个方法来找到对应的UIView，来实现返回某一层的ViewController。

 


```objective-c
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self.navigationController popToViewController:self.navigationController.viewControllers[0] animated:YES];
}
```


## push和present的区别：


push是需要依赖导航控制器Nav的，页面有层级关系，压栈和出栈。而present不需要导航控制器，独立弹出覆盖在原页面。


在我看来，push多用来进行层级之间的导航，适用于二级页面


而present则适合于独立的任务如登陆设置等。

---

原文发布于 CSDN：[【iOS】push 和 present](https://blog.csdn.net/2402_86720949/article/details/151260513)
