---
layout: post
title: "UITableView优化方法"
date: 2015-06-06 14:21:29 +0800
comments: true
categories: iOS
---
先看一下UITableView的[官方文档](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UITableView_Class/)，看看有没有什么以前不知道的细节。  

- NSIndexPath
官方文档里：
> The NSIndexPath class represents the path to a specific node in a tree of nested array collections. This path is known as an index path.

UITableView对NSIndexPath做了个类扩展所以能直接取section和row，实际就是长度为2的index path第一个指明在sections里的位置第二个指明在rows里的位置。对于indexPath1.4.2.3如下图，以前除了在tableView几乎没用过indexPath。什么时候有用？OC写个b树？  
![](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSIndexPath_Class/Art/indexpath.gif)




vvebo 缓存力度 专门针对uitableview的缓存框架

国外搜一搜 问问师兄们
用layer
runloop
http://www.atatech.org/articles/27707 圆角优化

注意iOS8中字体变大变小

iOS 7 Programming Pushing the limits


hack weixin
25 tips to optimize http://traximus.github.io/blog/2014/01/21/ios-performance-tips-and-tricksks/




像素对齐
ata 蚂蚁搜索一下

