---
title: "【iOS】AutoLayout与Masonry：简化iOS布局"
slug: "ios-autolayout-masonry"
published: 2025-09-04
description: "Auto Layout 与 Masonry 苹果提供的自动布局（Auto Layout）能够对视图进行灵活有效的布局。但是，使用原生的自动布局相关的语法创建约束的过程是非常冗长的，可读性也比较差。 Masonry 的目标其实就是 为了解决原生自动布局语法冗长的问题 。 其实说到本"
tags: ["iOS", "AutoLayout", "Masonry"]
category: "iOS"
draft: false
---

## Auto Layout 与 Masonry


苹果提供的自动布局（Auto Layout）能够对视图进行灵活有效的布局。但是，使用原生的自动布局相关的语法创建约束的过程是非常冗长的，可读性也比较差。


![图片](./csdn-150935318-autolayoutmasonry-ios-images/01-7b580325e618460b9cc1b84972772360.png)


Masonry 的目标其实就是 **为了解决原生自动布局语法冗长的问题**。


其实说到本质，它和手动布局是一样的。对一个控件放在哪里，我们依然只关心它的`(x, y, width, height)`。但手动布局的方式是，一次性计算出这四个值，然后设置进去，完成布局。但当父控件或屏幕发生变化时，子控件的计算就要重新来过，非常麻烦。


因此，在自动布局中，我们不再关心`(x, y, width, height)`的具体值，我们只关心`(x, y, width, height)`四个量对应的约束。


### 约束


#### 添加约束的规则：


如果两个控件是父子控件，则添加到父控件中。


如果两个控件不是父子控件，则添加到层级最近的共同父控件中。


![图片](./csdn-150935318-autolayoutmasonry-ios-images/02-f0d681dfd62541788d27b1feee3f1543.png)


**我们查看源码：**


### 基本属性


![图片](./csdn-150935318-autolayoutmasonry-ios-images/03-0d519883628a4cb9a5f226a89b0ab221.png)


![图片](./csdn-150935318-autolayoutmasonry-ios-images/04-2416cd34736242988691df8d6dfc1a94.png)


### 添加约束的三种方法：


![图片](./csdn-150935318-autolayoutmasonry-ios-images/05-0698cdef639c4be8af811881b486b5af.png)


Masonry是基于AutoLayout实现计算视图Frame的


```objective-c
// 定义这个常量，就可以不用在开发过程中使用"mas_"前缀。
#define MAS_SHORTHAND
// 定义这个常量，就可以让Masonry帮我们自动把基础数据类型的数据，自动装箱为对象类型。
#define MAS_SHORTHAND_GLOBALS
### Masonry的使用：


mas_equalTo和equalTo的区别


**用法** **常见场景** **例子** equalTo(view) 对齐某个 view 的同属性 make.left.equalTo(self.view) equalTo(view.mas_xxx) 对齐某个 view 的特定属性 make.left.equalTo(otherView.mas_right) equalTo(@数值) 固定数值（必须手动包对象） make.width.equalTo(@100) mas_equalTo(数值) 固定数值、结构体（推荐写法） make.width.mas_equalTo(100


```objective-c
[button mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.equalTo(self.view).offset(20)
        make.bottom.equalTo(self.view).offset(0);
        make.right.equalTo(self.view).offset(10);
        make.left.equalTo(self.view).offset(200);
        make.centerX.equalTo(self.view);//水平居中
        make.centerY.equalTo(self.view);//垂直居中
        make.width.height.mas_equalTo(60);
        make.edges.equalTo(self.view).insets(UIEdgeInsetsMake(20, 20, 20, 20));//四周都留20
        make.width.equalTo(self.view).multipliedBy(0.5); //宽度 = 父视图的一半
    }];
```


下图为使用Masonry实现的折叠cell demo。


![图片](./csdn-150935318-autolayoutmasonry-ios-images/06-b36fb3ebb8d14f00ac104f65205bc3e5.png)


```objective-c
- (void)viewDidLoad {
    [super viewDidLoad];
    self.foldTableView = [[UITableView alloc] initWithFrame:CGRectZero style:UITableViewStylePlain];
    self.foldTableView.delegate = self;
    self.foldTableView.dataSource = self;
    [self.view addSubview:self.foldTableView];
    [self.foldTableView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.view).offset(200);
        make.top.equalTo(self.view).offset(300);
        make.width.mas_equalTo(100);
        make.height.mas_equalTo(30);
    }];

    self.array = [NSMutableArray arrayWithObjects:@"1", @"2", @"3", @"4", @"5", nil];

    self.btn = [UIButton buttonWithType:UIButtonTypeCustom];
    [self.btn setImage:[UIImage imageNamed:@"kaiqi"] forState:UIControlStateNormal];
    [self.btn setImage:[UIImage imageNamed:@"guanbi"] forState:UIControlStateSelected];
    [self.btn addTarget:self action:@selector(press:) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:self.btn];
    [self.btn mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.view).offset(270);
        make.top.equalTo(self.view).offset(300);
        make.width.mas_equalTo(40);
        make.height.mas_equalTo(40);
    }];
}
```

---

原文发布于 CSDN：[AutoLayout与Masonry：简化iOS布局](https://blog.csdn.net/2402_86720949/article/details/150935318)
