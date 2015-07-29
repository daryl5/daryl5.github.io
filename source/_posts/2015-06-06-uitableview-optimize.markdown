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

在类方法中先判断item.itemHeight是不是0，如果不是则直接返回，否则进入计算过程，并在计算完成后赋值给itemHeight保存。关于算高有[优化UITableViewCell高度计算的那些事](http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/)。另外这里有关于RunLoop的使用可以看看。

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

- reload局部而不是全部

```objective-c
- (void)reloadRowsAtIndexPaths:(NSArray *)indexPaths withRowAnimation:(UITableViewRowAnimation)animation
```

- 像素对齐

不对齐会导致抗锯齿处理，就是把一个像素的东西画在相邻的两个像素上，并且设置不同的亮度。在iOS7 Programming Pushing the Limits看到一个小tip，可以用奇数字体，一般比偶数字体更容易像素对齐。

另外对齐中心：

```objective-c
- (void)setAlignedCenter:(CGPoint)center {
    self.center = center;
    self.frame = CGRectIntegral(self.frame);
}
```

- image大小和imageView保持一致

这样可以防止系统进行图片解码浪费性能。对图片数据进行decode。在子线程中设置image的大小后，在imageview中使用缩放后的image。原因：由于UIImage的imageWithData函数是每次画图的时候才将Data解压成ARGB的图像，所以在每次画图的时候，会有一个解压操作，UIImage初始化后仅仅是把图片加载到内存中，而实际的解码和重采样是在图片需要显示时才进行。
另外`Debug-Color misaligned images`里可以看到没对齐的imageView。

```objective-c
//图片重采样，在子线程中进行
CGSize itemSize = CGSizeMake(width, height);//实际要缩放的大小
UIGraphicsBeginImageContext(itemSize);
CGRect imageRect = CGRectMake(0.0, 0.0, itemSize.width, itemSize.height);
[image drawInRect:imageRect];
UIImage newImage = UIGraphicsGetImageFromCurrentImageContext(); //重采样后的图片
UIGraphicsEndImageContext();
```

- 简单控件用Core Graphics绘制

设置圆角、简单控件、设置阴影等都用Core Graphics。

- C Functions

自定义含表情显示的Label。服务端返回的是类似/:026、/:-W之类的伪符号，所以循环把这些伪符号替换成一段类似html的代码，并且每次循环都用到[NSString stringWithFormat:]方法构建了这段类似html的代码。

```objective-c
sprintf(emojiKey, EMOJI_DIRECTORY, [value UTF8String],pointSize,pointSize);
NSString *imageValue = [NSString stringWithCString:emojiKey encoding:NSASCIIStringEncoding];
s = [s stringByReplacingOccurrencesOfString:key withString:imageValue];
```

- 方法指针缓存

如果一个方法在一个循环次数非常多的循环中使用，在进入循环前使用methodForSelector获取该方法的IMP，在循环体中直接调用该IMP。

- 按需加载

如果滚出就不加载内容了，不过体验不好。

- AsyncDisplayKit

待续

avoid gradients, image scale, offscreen drawing


2、NSDateFormatter的重用大家都知道，但是是线程不安全的，这里有安全的写法[ata](http://www.atatech.org/articles/17301)  
```objective-c
- (NSDateFormatter *)getDateFormatter {
    NSMutableDictionary *threadDict = [[NSThread currentThread] threadDictionary];
    NSDateFormatter *dateFormatter = threadDict[@"reuse_dateFormatter"];
    if (!dateFormatter) {
        @synchronized(self) {
            dateFormatter = [[NSDateFormatter alloc] init];
            [dateFormatter setDateFormat:@"yyyy-MM-dd a HH:mm:ss EEEE"];
            threadDict[@"reuse_dateFormatter"] = dateFormatter;
        }
    }

    return dateFormatter;
}
```

3、减少subviews数量，定制复杂cell使用drawRect。尽量使用drawRect而不是layoutSubView。

1、不在viewWillApear中进行费时操作

