---
layout: post
title: "UITableView优化方法"
date: 2015-06-06 14:21:29 +0800
comments: true
categories: iOS
---
<!--more-->

> 结论：搜集了很多资料，看下来感觉提到的方法都差不多，问题的关键是如何做出普适性强的通用库，或者和现有开发框架相结合，设计优雅的API，让大家开发业务时愿意去用。【黑客与画家】有一章的标题是【设计与研究：研究必须是“新”的，而设计必须是“好”的】，其实很多问题也能归到这两类上，一个问题是偏重设计问题还是研究问题，那关注的点就应该不同。

先看一下UITableView的[官方文档](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UITableView_Class/)，看看有没有什么以前不知道的细节。  

- NSIndexPath
官方文档里：
> The NSIndexPath class represents the path to a specific node in a tree of nested array collections. This path is known as an index path.

UITableView对NSIndexPath做了个类扩展所以能直接取section和row，实际就是长度为2的index path第一个指明在sections里的位置第二个指明在rows里的位置。对于indexPath1.4.2.3如[图](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSIndexPath_Class/Art/indexpath.gif)，以前除了在tableView几乎没用过indexPath。什么时候有用？OC写个b树？    
代码如下：
```objective-c
// This category provides convenience methods to make it easier to use an NSIndexPath to represent a section and row
@interface NSIndexPath (UITableView)

+ (NSIndexPath *)indexPathForRow:(NSInteger)row inSection:(NSInteger)section;

@property(nonatomic,readonly) NSInteger section;
@property(nonatomic,readonly) NSInteger row;

@end
```
另外在使用runtime方法`void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)`和`id objc_getAssociatedObject(id object, const void *key)
`时需要指定一个唯一的key`static char const * const ObjectTagKey = "ObjectTag"`，在[这里](http://stackoverflow.com/questions/16020918/avoid-extra-static-variables-for-associated-objects-keys/16020927#16020927)看到一个可以省去这个key的办法，直接用`@selector(anAssociatedObject)`做这个key。更进一步，可以用`_cmd`，原理是每个方法调用都会带两个隐藏参数，一个self用来取到消息接收者的iVar和iMethods，一个_cmd代表方法自身的selector，具体见官方文档的[消息机制](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtHowMessagingWorks.html)。

- State Preservation  
看到UIDataSourceModelAssociation，以前没留意过。复制给tableView的[restorationIdentifier](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIViewController_Class/index.html#//apple_ref/occ/instp/UIViewController/restorationIdentifier)，并且tableView的dataSource实现UIDataSourceModelAssociation协议，然后实现两个代理方法，就能实现关闭app再打开时完全恢复tableView的状态。

##为什么卡

UITableView继承自UIScrollView，需要根据contentSize、bounds、contentInset、contentOffset等属性共同决定是否可以滑动以及滚动条的长度，而且在加载tableView的时候就要询问所有cell的高度，以便确定UIScrollView的contentSize，iOS8引入了estimatedHeight就是为了减少这个过程的耗费，一开始只是估算一下，等到真正要展现cell的时候才计算cell的真实高度。另外就是cell的重用机制，拿出来旧的cell，往里设置内容时需要处理内容，而滚动时候会一直发生cell重用。

##各种常见优化方法

- 尽可能使用`estimatedHeightForRowAtIndexPath`方法

iOS7设备占比已经非常高，有必要专门去做一下优化，有实验表明使用了下面这个代理方法后帧数提升30%。
```objective-c
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath;

// 把上面方法的调用推迟到cell即将展现出来时
- (CGFloat)tableView:(UITableView *)tableView estimatedHeightForRowAtIndexPath:(NSIndexPath *)indexPath NS_AVAILABLE_IOS(7_0);
```

- 高度缓存

在类方法中先判断item.itemHeight是不是0，如果不是则直接返回，否则进入计算过程，并在计算完成后赋值给itemHeight保存。关于算高有[优化UITableViewCell高度计算的那些事](http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/)。
```objective-c
//Model
@property (nonatomic, assign) CGFloat itemHeight

+ (CGFloat)heightForItem:(AAAModel *)item;
```
另外如果cell是定高的，那么直接用`self.tableview.rowHeight = 64`而不要用代理方法。如果写了这句还写了代理方法，那么会去调代理方法，这句设置失效。类似的有sectionFooterHeight、sectionHeaderHeight。不过话说回来，如果定高cell一般又不会卡。

- Cache Everything. Use Proper Model.

其实跟上一条类似，所有复杂计算都保存结果，保存的地方就是model新增的property，或者干脆不暴露太多需要再加工的数据，暴露能直接展现的。

- cellForRowAtIndexPath做尽可能少的事

把费时操作放到后台，可以用`GCD`，`NSOperationQueue`，另外看到个方法，利用通知机制，把不重要的处理放在idle里执行。
```objective-c
-(void)registerForIdleNotification
{
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(idleNotificationMethod)
                                                 name:@"IdleNotification"
                                               object:nil];

    NSNotification *notification = [NSNotification notificationWithName:@"IdleNotification" object:nil];

    [[NSNotificationQueue defaultQueue] enqueueNotification:notification postingStyle:NSPostWhenIdle];
}
```

##一些好习惯
- reload局部而不是全部
```objective-c
- (void)reloadRowsAtIndexPaths:(NSArray *)indexPaths withRowAnimation:(UITableViewRowAnimation)animation
```




vvebo 缓存力度 专门针对uitableview的缓存框架

国外搜一搜 问问师兄们
用layer
runloop
http://www.atatech.org/articles/27707 圆角优化

注意iOS8中字体变大变小

iOS 7 Programming Pushing the limits  
简单元素自己绘制  按钮图片用customView。图片要decode等等。
可以有个tableView.needAutoPixelAlign = YES  
没像素对齐的话会导致抗锯齿处理，也就是把元素的不同部分画到不同像素并且给以不同的alpha
也可以有个tableView.needRegenerateImage = YES 适合没有交互的也可延伸到简单交互的cell，如点击展开tag，如点击预览评论里的图片，这样就基本没法搞了。要handle很多东西，删除啊、移动啊等等的  
一个有意思的观点，奇数的字体更容易像素对齐，比如13和12。其实只要用奇数字体，然后把中心放在整数像素上，就能像素对齐了。看一下新鹏的。


cellForRow里做尽可能少的操作



hack weixin
25 tips to optimize http://traximus.github.io/blog/2014/01/21/ios-performance-tips-and-tricksks/




像素对齐 图片对齐
cache everything 日期格式化等放在model里
用shadow path


15.Optimize tableview
reuse cell
set subview opaque
avoid gradients, image scale, offscreen drawing
cache height if height is variable
use asynchronously method for cell’ contents   如果滚出就不要计算了；思考一下，整个cellForRow都用异步
use shadowPath to set shadow
reduce te number of subViews 安卓有类似需求，减少xml的层级
do as little work possible in cellForRowAtIndexPath:
use the appropriate data structure
use rowHeight, sectionFooterHeight,sectionHeaderHeight to set constant height instead of delegte  直接设置省的查代理方法





ata 蚂蚁搜索一下  
1、stringWithFormat比起sprintf慢了好多，应该还有类似的api，一些操作可以考虑用c写[ata](http://www.atatech.org/articles/19944)  
2、NSDateFormatter的重用大家都知道，但是是线程不安全的，这里有安全的写法[ata](http://www.atatech.org/articles/17301)  
3、图片剪裁很关键，这里有一些可用代码，也有一个用bezierPath给图片加圆角的方法[ata](http://www.atatech.org/articles/19955)

看了好些东西感觉bat里还是会被业务压到没办法研究很深，微信有专门负责性能优化的部门，很不错，钱包似乎也有。创业公司、个人开发者在这方面倒是研究更多。


问题：方法很多，怎么能做到更高的通用性？