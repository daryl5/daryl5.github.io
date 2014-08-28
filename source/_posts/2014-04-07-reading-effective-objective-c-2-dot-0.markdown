---
layout: post
title: "『Effective Objective-C 2.0』笔记"
date: 2014-04-07 13:22:29 +0800
comments: true
categories: iOS
---
《Effective Objective-C 2.0：52 Specific Ways to Improve Your iOS and OS X Programs》 作者：[Matt Galloway](http://www.galloway.me.uk/)  
<!--more-->
##**1: Accustoming Yourself to Objective-C**
###Item 2: 缩减头文件--Minimize Importing Headers in Headers
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
##**2: Objects, Messaging, and the Runtime**
###Item 6: 理解Properties-Understand Properties【需要再看】
a.自己写了Accessor中的一个，编译器将会synthesize另一个，LazyLoading就是这样的。如果不想让编译器自动synthesize，可以用`@dynamitc`指令，这样编译器不会生成Accessor和对应的实例变量名，而且编译时候编译器会忽略Accessor还没有被定义但是已经在self.property，编译器会认为Accessor在runtime时会有的。  
b.`weak`不会release旧值，也不会retain新值，类似assign，不过这个property所指对象被销毁时，这个property会被nilled out。  
`unsafe_unretained`也和assign类似，只是关于object的assign，unretained对应nonowning，unsafe对应不会被nilled out，这一点和weak不一样。  
`copy`对应mutable copy，如果不想让对象把传进来的值改变了，应该做一下copy，也就是immutable copy。虽然增加了拷贝操作，但是会更安全。  
c.关于自定义getter名在BOOL上的应用，为了加is为什么不直接把property命名成isXxx。
###Item 7: Access Instance Variables Primarily Directly When Accessing Them Internally
**a.总结**

1. 一般来说，要读实例变量的时候就直接访问，要写实例变量的时候就用property。
2. 如果在初始化方法（或dealloc）中改变property的值，那么也是直接访问实例变量。因为如果初始化方法也用property，那么父类初始化时候其实会调用子类的初始化方法，可能会产生问题。另外，有时候是必须在初始化方法中使用setter的，比如当实例变量在父类中声明，那子类只能通过setter来访问。
3. 如果property使用了`Lazy Loading`，那么需要使用getter来获取，不如可能获得空值。

**b.`直接访问`和`使用property`的区别**

1. 直接访问无需`method dispatch`那么肯定更快；
2. 直接使用实例变量会绕过property所设置的内存管理策略；
3. 直接访问实例变量，KVO可能不会触发了；
4. 使用property可以方便debug，知道谁在什么时候访问了某一实例变量。

###Item 8: Understand Object Equality
a.有的类实现了判断相等的方法，如NSString的`isEqualToString:`，这个比`isEqual:`，因为后者还需要查看所比较对象所属的类。  
#看不下去了

###Item 10: 在既有类中适用关联对象存放自定义数据-Use Associated Objects to Attach Custom Data to Existing Classed
a.一般都是通过`继承`来给既有类添加数据，但行不通的时候就要用到`关联对象`了。以UIAlertView为例：

- 一般的实现中buttonIndex和其action的对应，即button的定义和其响应方法的声明不在一个地方，这样可读性不好；
- 如果同一个类里有多个alertView，那么在代理方法中还要判断是哪一个，然后再判断buttonIndex；
```objective-c
static const void *EOCMyAlertViewKey = "EOCMyAlertViewKey";

- (void)askUserAQuestion {
    UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"Question"
                                                    message:@"What do you want to do?"
                                                   delegate:self
                                          cancelButtonTitle:@"Cancel"
                                          otherButtonTitles:@"Continue", nil];
    
    void(^block)(NSInteger) = ^(NSInteger buttonIndex) {
        if (buttonIndex == 0) {
            [self doCancel];
        } else {
            [self doContinue];
        }
    };
    
    objc_setAssociatedObject(alert, EOCMyAlertViewKey, block, OBJC_ASSOCIATION_COPY);
    
    [alert show];
}

// UIAlertViewDelegate protocol method
- (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex {
    void (^block)(NSInteger) = objc_getAssociatedObject(alertView, EOCMyAlertViewKey);
    block(buttonIndex);
}
```
b.总结  
关联对象就是把两个object链接在一起了；关联对象只有在其他方法都不可行的时候才使用，因为很容易造成`Retain Cycle`；上例中还可以通过继承AlertView并附加这个block对象来实现，如果类中alertView要多次用到，更建议继承而不是关联对象。
###Item 14: 理解类的对象- Understand What a Class Object Is
a.OC是`动态类型`的，对象的类型并不是在编译时候进行绑定的，而是在运行时进行查找。
##**4: Protocols and Categories**
###Item 23: 通过委托和数据源协议进行对象间通信
a.有一句话没太懂，这个单词居然查不出来，完了看看Item6，里面有这个单词(尼玛，就是自动设置空的意思，对应weak)：
> A delegate property will always be defined using either the weak attribute to benefit from `autonilling` or unsafe_unretained if autonilling is not required.  

b.在interface中声明遵守协议的话，别的类会知道这一行为，一般把“遵守协议”放在类扩展部分，虽然没太多影响。要知道一个对象是否遵守某个协议可以这样：`[self conformsToProtocol:@protocol(DRDHandWritingProtocol)]`  
c.可以把正在代理的对象也传回给代理，这样方便判断具体是哪个代理对象在调用代理方法。
```objective-c
- (void)networkFetcher:(EOCNetworkFetcher *)fetcher didReceiveData:(NSData *)data {
    if (fetcher == _myFetcherA) {
        //Handle data
    } else if (fetcher == _myFetcherB) {
        //Handle data
    }
}
```
d.如果开发API，用代理可以实现一些属性设置，如`- (BOOL)networkFetcher:(EOCNetworkFetcher *)fetcher shouldFollowRedirectToURL:(NSURL *)url;`，甚至这里可以给fetcher传递一个NSDictionary来进行Fetcher一系列属性的设置。  
e.对`@optional`的代理方法，方法调用者负责通过`[delegate respondsToSelector]`确保程序不会崩溃，unrecognized selector。`@required`的话编译器会确认。如果没设置，默认都是required。  
f.对dataSource类的delegate，在获取每一个小的data piece的时候都查询一次respondsToSelector是低效的，应该将当前对象对协议的遵守情况缓存下来。如下：
```objective-c
//有个DataModel或者ViewController是该对象的代理
//In class extension
@interface EOCNetworkFetcher() {
    struct {
        unsigned int didReceiveData      : 1;
        unsigned int didFailWithError    : 1;
        unsigned int didUpdateProgressTO : 1;
    }_delegateFlags;
}

//In delegate Setter
- (void)setDelegate:(id<EOCNetworkFetcher>)delegate {
    _delegate = delegate;
    _delegateFlags.didReceiveData = [delegate respondsToSelector:@selector(networkFetcher:didReceiveData:)];
    ...
}

//Query if responds to selector (FRENQUENTLY)
if (_delegateFlags.didReceiveData) {
    [_delegate networkFetcher:self didReceiveData:dataReceived];
}
```
g.协议的“继承”
```objective-c
@protocol A
    -(void) methodA;
@end

@protocol B <A>
    -(void) methodB;
@end
```













