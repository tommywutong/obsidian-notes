---
title: "【iOS】折叠cell"
slug: "ios-collapsible-cell"
published: 2025-09-04
description: "先给出实现效果： 折叠cell的实现 折叠cell的关键是通过按钮控制tableView的高度： 如此以来，点击按钮可以实现tableView高度的变化。 实现选中的单元格移到最前： 我们先获取Array中被选中的项目 然后在数株中移除，并插到头部，然后重新刷新tableView"
tags: ["iOS", "UITableView", "Cell"]
category: "iOS"
draft: false
---

## 先给出实现效果：


![图片](./csdn-151195540-ios-cell-images/01-e71ff9c89abb4f008d87d41a1352ab78.gif)


## 折叠cell的实现


折叠cell的关键是通过按钮控制tableView的高度：


```objective-c
- (void)press:(UIButton *)btn {
    [self.foldTableView mas_updateConstraints:^(MASConstraintMaker *make) {
        if (btn.selected) {
            make.height.mas_equalTo(30);
        } else {
            make.height.mas_equalTo(150);
        }
    }];
    btn.selected = !btn.selected;
}
```


如此以来，点击按钮可以实现tableView高度的变化。


实现选中的单元格移到最前：


```objective-c
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    NSString *str = self.array[indexPath.section];
    [self.array removeObjectAtIndex:indexPath.section];
    [self.array insertObject:str atIndex:0];
    [self.foldTableView reloadData];
    [self press:self.btn];
}
```


我们先获取Array中被选中的项目


然后在数株中移除，并插到头部，然后重新刷新tableView。这样就可以实现效果如下：


![图片](./csdn-151195540-ios-cell-images/02-06f6608dc3d3475281af92d1e351b541.gif)

---

原文发布于 CSDN：[【iOS】折叠cell](https://blog.csdn.net/2402_86720949/article/details/151195540)
