---
layout: post
title: "『Objective-C高级编程』笔记"
date: 2014-05-10 19:20:32 +0800
comments: true
categories: iOS
---
<!--more-->
#第一章 ARC
##1.2 内存管理
###1.2.2 内存管理的思考方式
- 自己生成的对象，自己持有（alloc/new/copy/mutableCopy了就也retain了）
- 非自己生成的对象，自己也能持有（retain）
- 不再需要自己持有的对象时释放（release）
- 非自己持有的对象无法释放（没retain/alloc/new/copy/mutableCopy就不能release）

1.OC的内存管理方法实际上不在OC语言里，而是包含在Cocoa框架中的。（OC是C的超集，Cocoa是对OC加了内存管理、多媒体、网络通讯等等的类库形成的用来给OS X开发应用的框架，Cocoa Touch是对Cocoa进行移动适配形成的另一套OC的类库）  
2.Cocoa框架中Foundation框架类库中的NSObject类担负内存管理的职责，内存管理调用的alloc、retain等方法指代的就是NSObject类中的alloc类方法、retain实例方法等。（所有对象继承自NSObject，alloc、retain等内存管理方法在NSObject中实现，这也就是为什么初始化对象时候要先[super init]以及dealloc最末要[super dealloc]）  
3.`[NSObject new`和`[[NSObject alloc] init]`是完全等价的。  
4.类似`id obj = [NSMutableArray array]`这样的生成的类对象并不会被obj所持有，也就是说这行代码里不包含`[obj retain]`，总是这么写不过都是在ARC下，没意识到这个问题。如果是手动管理，加retain就好。
###1.2.3 alloc/retain/release/dealloc实现
