---
layout: post
title: "《iOS设计模式解析》笔记"
date: 2015-05-21 14:14:16 +0800
comments: true
categories: iOS
---
<!--more-->
##第1章 你好，设计模式
####1.5.4 作为复合设计模式的MVC
MVC本身并不是最基本的设计模式，它包含了很多更加基本的设计模式，在MVC中，基本设计模式相互配合。  
Cocoa及Cocoa Touch的MVC中包含的基本模式有：Composite、Command、Mediator、Strategy、Observer。

- Composite：视图间的相互组合
- Command：Target-Action模式，action代码的执行被延迟到特定事件发生
- Mediator：Controller在Model和View中起的作用
- Strategy：原话是“控制器可以是视图对象的一个策略”，我理解类似代理吧
- Observer：模型对象向它所关注的控制器等对象发出内部状态变化的通知，KVO嘛，应该是模型对象向关注它的控制器等对象发出通知

####1.6 影响设计的几个问题
说了GoF提的`针对接口编程，而不是针对实现编程`以及`优先使用对象组合而不是类继承`。
##第2章 案例分析：设计一个应用程序
这一章打眼一看就感觉很亲切，因为我做过一个类似的应用。等我读了一小半后甚至发现跟我做的东西几乎一样，思考过程也很类似，不过我的功能更多点。
####设计过程中3个重要的里程碑

- 想法的概念化  
我要做一个手写记事本，大概有abcd功能。头脑风暴，罗列需求，明确需求，细化需求。
- 界面外观设计  
文件展示用collectionView，点击一个就进入编辑界面，这个界面可以切换工具实现写字、绘画、手势编辑功能，要做一个有时差效果的工具栏能四处拖拽，工具栏上还要有redo、undo按钮，设置要做在右上角popover，帮助要用webview呈现用markdown来导出html页。
- 架构设计  
主要的功能有写字、绘画、手势编辑，那么可以做一个FingerTracker的父类，做三个子View，Model层用point类和inkPoint类，存储用UIDocument。

####2.3.1 视图管理
为了避免Canvas、Palette、Thumbnail这3个viewController间的耦合引入了CoordinatingController，然后用requestViewChangeByObject:(UIBarButtonItem)来处理视图切换。  
想起以前的项目里类似的东西:
```objective-c
//TBCityNavigatorRegister
[TBNavigator registerClass:@"TBCityHomeController" withPath:TBCityURLHome];

//TBNavigator
BOOL TBOpenURLFromSourceAndParams(NSString* urlPath, id source, NSDictionary* params);
BOOL TBOpenURLFromTarget(NSString* urlPath, id target);

//UIViewController实现UIViewControllerTBNavigator协议来接参数完成初始化
//为什么要接URL我也不知道
- (id)initWithNavigatorURL:(NSURL*)URL query:(NSDictionary*)query;
```
当然这里为了从外部打开，url处理成了scheme形式，如果没有这个需求可以直接是类名。对于解耦还是有好处的。

####2.3.2 如何表现涂鸦
我的app里model有`Point`和`InkPoint`，前者有x、y坐标，后者继承自前者，多一个`strokeWidth`属性用来实现数字墨水效果，多一个`isEnd`记录是否是笔划终结点；然后有一个`CharUnit`代表字符，里面存字符单元的宽、高，一个数组存所有的inkPoint，一个数组存pointsNumPerStroke；然后`Document`对象有一个NSMutableArray记录所有的charUnit、imageUnit等等，绘制的时候判断当前对象类型来绘制，总的说就是`访问者模式`。  
这里引入`Vertex`存储location，用来记录一个笔划上的一个点，然后`Dot`继承自Vertext，多一个color和size，记录单独的点，然后一个`Stroke`实现addMark、count、lastChild等方法，有一个`Mark Protocol`，声明了addMark、removeMark、childAtIndex、`drawWithContext`等方法。**Mark协议存在的意义**就在于，Vertex、Dot、Stroke都实现了自己的drawWithContext等方法后，就可以统一的对待所有对象。Vertext就是addLineToPoint到自己的location，Dot就是以自己的frameSize等画出自身就一个点，Stroke就是`对所有id<Mark Protocol> child执行drawWithContext然后执行strokePath`。这里就是所谓的`组合模式`。














