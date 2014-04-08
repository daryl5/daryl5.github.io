---
layout: post
title: "『Effective Objective-C 2.0』笔记"
date: 2014-04-07 13:22:29 +0800
comments: true
categories: iOS
---
《Effective Objective-C 2.0：52 Specific Ways to Improve Your iOS and OS X Programs》 作者：[Matt Galloway](http://www.galloway.me.uk/)  
##**1: Accustoming Yourself to Objective-C**
###Item 2: 缩减头文件--Minimize Importing Headers in Headers</font>
a.如下引文，试了一下互相import，没有报错- -！
> The use of #import rather than #include doesn't end in an infinite loop but does mean that one of ther classes won't compile correctly.

b.协议相关  
一些代理相关的协议，可以把协议头文件在.m文件里再引入，把代理`id<XXXProtocol> delegate`定义放在类扩展里。
也可以考虑单独创建一个头文件，里面只放协议定义。
<!--more-->
###Item 3: 使用字面量语法而不是其等价方法--Prefer Literal Syntax over the Equivalent Methods
a.字面量使用
    NSString *str           = @"Test String";
    NSNumber *boolValue     = @YES;
    NSNumber *charValue     = @'a';
    NSNumber *expressionVal = @(x * y);//int x & float y
    
    NSArray *animals        = @[@"cat", @"dog"];
    NSString *dog           = animails[1];//下标直接访问
    
    NSDictionary *person    = @{@"firstName" : @"Matt", @"age" : @28};
    NSString *firstName     = person[@"firstName"];//key值直接访问value
    
    //Assign value or replace
    mutableArray[1] = @"dog";
    mutableDictionary[@"lastName"] = @"Galloway";
b.好处  
如果NSArray用3个object来初始化，第二个是nil，那么arrayWithObjects将会返回一个只包含object1的array，而字面量初始化会报错。**“报错”比“少了个数”更安全。**
对于NSDictonary，初始化会在遇到nil时候停止，dictionaryWithObjectsAndKeys可能会少了某一个value；字面量更安全。  
c.缺点  
字面量语法只能用于Foundation框架的一些类，对于自定义的类没法这样定义（显然）。  
对于mutable对象的定义，需要进行mutableCopy。增加了额外的方法调用和对象创建  
    NSMutableArray *mutable = [@[@1, @2, @3] mutableCopy];
###Item 4: 使用类型化常量而不是预处理指令#define--Prefer Typed Constants to Preprocessor #define
a.#define的缺点  
预处理指令`#define ANIMATION_DURATION 0.3`只是在预处理阶段进行`替换`，它没有包含类型信息。（如果定义在头文件，所有包含该头文件的文件也都会进行替换。）  
b.类型化常量的使用（static const）  
用上编译器让常量有类型信息：`static const NSTimerInterval kAnimationDuration = 0.3;`
常量命名规范：
> The usual convertion for constants is to prefix with lettr k for constants that are local to a translation unit(implementation file). For constants that are exposed outside of a class, it is usual to prefix with the class name.  
在.m中定义，对于这个translation unit是局部变量的话用k作前缀；如果要暴露给外界，用类名开头，如EOCViewClassAnimationDuration。

`static`关键词说明这个变量对于当前translation unit是局部的，如果没定义为static，编译器将会为其创建一个external symbol，如果另一translation unit声明了一个同名变量，链接器将会报错；`const`关键词说明不能被修改。
> A translation unit is the input the compiler receives to generate one object file. In OC, this usually means that there is one translation unit per class: every .m file.

SO的一个问题：[How different static variable declarations in Objective-C?](http://stackoverflow.com/questions/11382502/how-different-static-variable-declarations-in-objective-c)  
c.类型化常量的使用（extern）  
跟`NSNotification`相关，通知名应该这样处理：
    // In .h
    // extern--"Trust me, there's a variable called EOCStringConstant declared in another file"
    // EOC是类名
    extern NSString *const EOCStringConstant;
    
    // In .m
    //反着读：EOCStringConstantis a const pointer point to NSString
    NSString *const EOCStringConstant = @"NotificationName";
    //如果不是对象那这里是const NSTimeInterval EOCAnimated...
d.总结  
类型化常量有类型- -！
类型化常量借助编译器来确保常量的一致性（const），预处理指令#define可以在别的地方重定义。  
.m的局部变量如动画时间等用k开头放在.m文件里用static；通知名等用类名+通知名，.m里声明+定义，.h里extern，如UIApplication类里声明+定义的UIApplicationDidEnterBackGroundNotifation。

















