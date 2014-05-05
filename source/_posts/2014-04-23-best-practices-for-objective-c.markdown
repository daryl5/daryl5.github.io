---
layout: post
title: "Best Practices & Tips"
date: 2014-04-23 19:19:22 +0800
comments: true
categories: iOS
---
<!--more-->
###4.用block实现代理
关于block的大致介绍，[Objective-C中的Block](http://onevcat.com/2011/11/objective-c%E4%B8%AD%E7%9A%84block/)这篇博文写的挺好。其中提到的一点需要注意，如下：
```objective-c
int price = 1.99;

float (^finalPrice)(int) = ^(int quantity) {
    //Price is accessible here
    return quantiy * price;
}

int orderQuantity = 10;
//Ordering 10 units, final price is: $19.90
NSLog(@"Ordering %d units, final price is: $%2.2f", orderQuantity, finalPrice(orderQuantity));

price = .99;
//Ordering 10 units, final price is: $19.90
NSLog(@"Ordering %d units, final price is: $%2.2f", orderQuantity, finalPrice(orderQuantity));
```
> 可以理解为在block内的price是readonly的，只在定义block时能够被赋值（补充说明，实际上是因为price是value type，block内的price是在申明block时复制了一份到block内，block外面的price无论怎么变化都和block内的price无关了。如果是reference type的话，外部的变化实际上是会影响block内的）。

解决办法有两种：  
1.将局部变量声明为__block，表示外部变化将会在block内进行同样操作;  
2.使用实例变量。关于这点有引文：
> 这个比较没什么好说的，实例内的变量横行于整个实例内..可谓霸道无敌…=_=
block外的对象和基本数据一样，也可以作为block的参数。而让人开心的是，block将自动retain传递进来的参数，而不需担心在block执行之前局部对象变量已经被释放的问题。这里就不深究这个问题了，只要严格遵循Apple的thread safe来写，block的内存管理并不存在问题。（更新，ARC的引入再次简化了这个问题，完全不用担心内存管理的问题了）

这里有个问题就是：因为block会自动retain传进来的参数，所以会出现[capturing self strongly in this block is likely to lead to a retain cycle](http://stackoverflow.com/questions/14556605/capturing-self-strongly-in-this-block-is-likely-to-lead-to-a-retain-cycle)的问题，即就是传进去`self.xxx`之后，会retain self，即构成了循环引用。解决办法是定义一个`__weak`的self然后在block中使用。  
  
**回到正题，block代替protocol**
```objective-c
//In DRDHandWriting.h
typedef void(^didFinishWriting)(NSMutableArray *);
@property (copy, nonatomic) didFinishWriting didFinishWritingBlock;//注意这里是copy

//In DRDHandWriting.m
//Submit charUnitArray to Document
self.didFinishWritingBlock(segmentedCharUnitArray);

//In DRDManuscriptController::loadHandWritingView
handWritingView = [[DRDHandWriting alloc] initWithFrame:handWritingRect];
__weak DRDManuscriptController *weakSelf = self;
//这里的参数只是被代理对象需要返回给代理的数据，具体就是手写视图需要返回切分好的字符给controller
//另一种情形就是多个AlertView的代理，这里就可以传回当前alertView
[handWritingView setDidFinishWritingBlock:^(NSMutableArray *segmentedArray) {
    [weakSelf.document addOrInsertCharUnitArray:segmentedArray];
}];
```
关于上例中block的copy，有[官方文档](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/WorkingwithBlocks/WorkingwithBlocks.html)：
> Note: You should specify copy as the property attribute, because a block needs to be copied to keep track of its captured state outside of the original scope. This isn’t something you need to worry about when using Automatic Reference Counting, as it will happen automatically, but it’s best practice for the property attribute to show the resultant behavior. For more information, see Blocks Programming Topics.

block在创建的时候是分配在栈上的，如果`retain`（栈上的东西不能retain），在调用block的时候程序就会崩溃，实际上ARC下测试了一下retain，只是代码编辑阶段即给出警告`Retain'ed block property does not copy the block - use copy attribute instead`并且程序正常运行。而如果是`copy`，将会把这个block拷贝到堆里，同时retain这个block所引用的所有东西。block在堆上的时候进行copy就没事，不会再拷贝一遍，只是相当于retain。这段摘自[Copy vs retain for Objective-C blocks](http://blog.refractalize.org/post/10476042560/copy-vs-retain-for-objective-c-blocks)。

**结论：**  
自己实验了一下上面那个改变price输出总价不变的例子，block在创建的时候是分配在栈上的。  
1.如果block里用到了局部变量，那么就会拷贝一份：比如`int a = 5`就会拷贝一个int变量其值为5；如果是`int *ptr = 5`就会拷贝ptr这个指针，这样改变*ptr的值，block里也会跟着变。  
2.如果用到了实例变量，那么不管是value type还是指针型的，都不会出现外部更新而block内部没跟着变的情况，猜想是因为block引用实例变量其实就是对self指针的弱引用，用到实例变量时候用指针去读的。  
3.上面`capturing self strongly in this block is likely to lead to a retain cycle`这一问题，是因为用到了实例变量。两种情况：如果是property那么隐含`self.propertyXXX`；如果是非property的实例变量，隐含`self->iVar`，都有self。因为是用block实现代理，那么需要传block给被代理对象，被代理对象copy block，同时运行时会copy block到堆上并retain block里所有的东西，这就相当于被代理对象里声明了一个`@property (strong, nonatomic) id<XXXProtocol> delegate`，所以就造成了循环引用。上面那个SO的链接讲得很清楚。  
4.最后，什么时候用block什么时候用delegation，[Blocks or Delegation](http://stablekernel.com/blog/blocks-or-delegation/)讲了，实在不想看了，先马克- -！
###3.Category也能Conform to a protocol
###2.对集合中每个对象发送消息
```objective-c
// remove all subviews
[[self.scrollView subviews] makeObjectsPerformSelector:@selector(removeFromSuperview)];
```
###1.[The Code Commandments: Best Practices for Objective-C Coding ](http://ironwolf.dangerousgames.com/blog/archives/913)
**a.在实例变量前加`@private`**[放在类扩展就好]

- 表明该变量与API无关
- 默认是`@protected`即就是子类可以访问，如果子类不需要那就不要暴露，面向对象的`信息隐藏`，（其实最好的应该是子类和其他类用到的在.h，私有变量和方法在类扩展）

**b.为每一个`data member`创建`@property`，在.m中都用`self.name`来访问**[这个有问题]

- property会把访问权限加入，如readonly
- property会把内存管理加入，strong&weak


**c.`实例变量`就是只给当前类及其子类使用，`@property`就是外部类也能用**  
之前看到在头文件里interface部分定义实例变量，同时又用@property重新写一遍。对于想要公开给这个类及其子类以外的类的实例变量，那么就为其写一个对应的property。  
**d.关于`readonly`**
就是说这个property在外界不能修改，如`self.document.displayHeight`会产生`assign to readonly property`。在类的实现文件里可以通过实例变量来修改，如`_displayHeight = 600`，此外，在实现文件还可以通过`self->_displayHeight = 600`来修改，这里之所以用->大概是因为类本身拥有这个实例变量就意味着类的实例有一个指向这个实例变量的指针，而property只是用于公开给外部，在这里没property什么事。
另外还可以在类扩展里告诉编译器我还需要一个setter，但只是在实现文件里：
```objective-c
//公有可读，私有可写
@interface YourClass ()
@property (nonatomic, copy) NSString* eventDomain;
@end
```
这个问题在[Readonly Properties in Objective-C?](http://stackoverflow.com/questions/4586516/readonly-properties-in-objective-c)