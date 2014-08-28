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
####1.2.2 ARC进行内存管理的思考方式
- 自己生成的对象，自己持有（alloc/new/copy/mutableCopy了就也retain了）
- 非自己生成的对象，自己也能持有（retain）
- 自己不再需要持有的对象时释放（release）
- 非自己持有的对象无法释放（没retain/alloc/new/copy/mutableCopy就不能release）

1.OC的内存管理方法实际上不在OC语言里，而是包含在Cocoa框架中的。（OC是C的超集，Cocoa是对OC加了内存管理、多媒体、网络通讯等等的类库形成的用来给OS X开发应用的框架，Cocoa Touch是对Cocoa进行移动适配形成的另一套OC的类库）  
2.Cocoa框架中Foundation框架类库中的NSObject类担负内存管理的职责，内存管理调用的alloc、retain等方法指代的就是NSObject类中的alloc类方法、retain实例方法等。（所有对象继承自NSObject，alloc、retain等内存管理方法在NSObject中实现，这也就是为什么初始化对象时候要先[super init]以及dealloc最末要[super dealloc]）  
3.`[NSObject new`和`[[NSObject alloc] init]`是完全等价的。  
4.类似`id obj = [NSMutableArray array]`这样的生成的类对象并不会被obj所持有，也就是说这行代码里不包含`[obj retain]`(有`[obj autorelease]`)，总是这么写不过都是在ARC下，没意识到这个问题。如果是手动管理，加retain就好。
####1.2.3 alloc/retain/release/dealloc实现
1.NSZone是为了防止内存碎片化而引入的结构。如果分配几块内存依次是小大小大小大，这样释放所有的“小”之后就形成了碎片，所以要有一大一小两种`区域`，小的去小的大的去大的。  
**现在的运行时的内存管理已经很高效，使用NSZone反而可能会降低效率或使源代码复杂，所以运行时都会忽略NSZone**。  
2.GNUstep是Cocoa框架的互换框架，两者实现不一定相同不过行为是一样的，这一节主要通过读GNUstep中`NSObject.m`中alloc、retain等方法的源代码来深入了解内存管理。  
3.GNUstep使用`内存块头部管理引用计数`，苹果使用`引用计数表`管理引用计数。
####1.2.5 autorelease
1.和C语言的`局部变量(Automatic Variable)`类似，如在函数内定义的int，除了函数就废掉了。autorelease的作用范围是NSAutoreleasePool，在`[pool drain]`时每个调用过autorelease的对象都会调用release。
2.只要不drain pool，生成的对象就不会被释放，内存就越来越少，所以在for循环里如果有很多autorelease对象那就定义一个NSAutoreleasePool来降低内存峰值。
3.Cocoa框架也有很多类方法会返回autorelease的对象。如`[NSMutableArray arrayWithCapacity:1]`等效于`[[[NSMutableArray alloc] initWithCapacity:1] autorelease]`。
###1.2.6 autorelease的实现
1.本质就是`调用NSAutoreleasePool的addObject方法`。  
2.NSAutoreleasePool的管理应该用的是栈，一个pool是就是栈中的一个项目。
##1.3 ARC规则
####1.3.3 所有权修饰符
1.ARC有效时，一共有4种修饰符：  

a. __strong
b. __weak
c. _\_unsafe__unretained
d. __autoreleasing

2.所有id和对象都必须加上所有权修饰符，默认是__strong，也就是说:
```objective-c
//这两个等效
id obj = [[NSObject alloc] init];
id __strong obj = [[NSObject alloc] init];
```
3.所谓强/弱引用实际就是`指针对一块内存（或者说一个对象）的引用`。  
强引用就是，别人可以release，但是运行时不能销毁这个对象，因为我还强引用着呢。其实就是给retainCounter加了1，所以对象不会被销毁，除非自己再去调用release。  
弱引用就是我在用但是我没给引用计数加1，别人release了我也就用不了了。  
4.简单总结  
`ARC实现方式`  
GNUstep中每个对象头部都存储着自身的引用计数；  
Apple用runtime统一存储所有对象的引用计数，用的是哈希表，以对象地址作为key，引用计数值和真正的内存地址作为值。有一个好处，即使因为故障导致对象所在内存被破坏，但只要引用计数表还在，就能确认内存块位置（这里真的没太懂，可以参考下面\_\_weak的部分，存储指针的地址）。  
`__autoreleasing`  
用该关键词修饰的变量会被注册到autoreleasepool；  
当方法名不是alloc/new/copy/mutablecopy时候，如果要返回object，这个object会被注册到autoreleasepool；  
访问\__weak修饰的变量时，该变量会被注册到autoreleasepool，以确保不会因为原有对象被释放而非法访问，因为只要pool还在，该对象都是有效的。  
`__weak`  
也称智能指针，因为当对象被release时候指针会被置nil。  
实现原理是：本质也是`哈希表`，把对象地址作为key，然后把指向对象的指针的地址（可能有多个指针都指着）作为value。这样当release对象的时候就在weak表中找到所有指向该对象的指针的地址，进而将所有指针都设置为nil。然后还会在weak表中删除该键值对。
