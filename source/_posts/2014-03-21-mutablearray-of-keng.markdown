---
layout: post
title: "所有遇到过的坑们"
date: 2014-03-21 12:22:22 +0800
comments: true
categories: iOS 
---
###1.CGBitmapContextCreate: unsupported parameter combination
创建位图上下文时出现的错误：  
> CGBitmapContextCreate: unsupported parameter combination: 8 integer bits/component; 32 bits/pixel; 3-component colorspace; kCGImageAlphaNoneSkipFirst; XXX bytes/row.

代码如下：
```objective-c
contextToDraw = CGBitmapContextCreate(NULL,
                                      charUnit.width * scale,
                                      charUnit.height * scale,
                                      8,
                                      0,
                                      colorSpace,
                                      kCGBitmapByteOrderDefault);
```
<!--more-->
在iOS7之前的代码中，最后一个CGBitmapInfo是kCGImageAlphaPremultipliedFirst，在iOS7下用这个会有警告  
> Implicit conversion from enumeration type 'enum CGImageAlphaInfo' to different enumeration type 'CGBitmapInfo' (aka 'enum CGBitmapInfo')

解决办法是改为`kCGBitmapByteOrderDefault | kCGImageAlphaPremultipliedFirst`。这是因为iOS7中最后一个属性有变化，由`CGImageAlphaInfo`变为`CGBitmapInfo`。[这里](http://www.cnblogs.com/wuxiufang/p/3397070.html)说得很详细。
###2.创建支持Retina的CGBitmapContext
原理是创建两倍长两倍宽的位图，同时ScaleCTM使系统绘图时进行坐标转换。代码如下，摘自[这里](http://stackoverflow.com/questions/10867767/how-to-create-a-cgbitmapcontext-which-works-for-retina-display-and-not-wasting-s)：
> A key factor is that, CGContextScaleCTM(context, scaleFactor, scaleFactor); is used to adjust the coordinate system, so that any drawing by Core Graphics, such as CGContextMoveToPoint, etc, will automatically work, no matter it is standard resolution or the Retina resolution.

```objective-c
//The sample is to create a 768 x 768 point region.  
//On The New iPad, it will be 1536 x 1536 pixel.  
//On iPad 2, it is 768 x 768 pixel.
float scaleFactor = [[UIScreen mainScreen] scale];

CGSize size = CGSizeMake(768, 768);

CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();

CGContextRef context = CGBitmapContextCreate(NULL, 
                           size.width * scaleFactor, size.height * scaleFactor, 
                           8, size.width * scaleFactor * 4, colorSpace, 
                           kCGImageAlphaPremultipliedFirst);

CGContextScaleCTM(context, scaleFactor, scaleFactor);
```
###4.使用CGBitmapContext并行绘图时报错
用如下代码进行简单的并行绘制每个字符，结果出现`CGContextAddLineToPoint: no current point`的错误，就是说在`CGContextAddLineToPoint`之前并没有一个`CGContextMoveToPoint`的操作。  
想了一下下面代码有问题，并行绘图的代码里会有这两个操作，待绘制字符多的时候并行程度高，就可能在绘制单个字符的代码中一个的`CGContextStrokePath`刚执行完，**Stroking the path resets the context's current path to empty, so there is no currentpoint after you call CGContextStrokePath().** context中move到的point已经释放，而另一个正好要执行`CGContextAddLineToPoint`，于是就`no current point`了。下面的引文来自[这里](http://lists.apple.com/archives/quartz-dev/2011/Feb/msg00030.html)

> Stroking the path resets the context's current path to empty, so there is no currentpoint after you call CGContextStrokePath().ClosePath() isn't there to balance BeginPath() — it modifies the current path by adding a line segment from the current point to the beginning of the current subpath (in such a way that there's a linejoin at that point rather than a pair of line ends).  

并行绘制最终没想到一个好的并行办法，因为context必须用同一个。最后的解决办法是，能保留上次绘制的context就保留，在其基础上接着绘制- -！
```objective-c
//Draw characters concurrently
NSIndexSet *indexSetToDraw = [NSIndexSet indexSetWithIndexesInRange:NSMakeRange(startIndex, characterNum)];
[self.dataUnitArray enumerateObjectsAtIndexes:indexSetToDraw
                                      options:NSEnumerationConcurrent
                                   usingBlock:^(DRDCharUnit *charUnitToDraw, NSUInteger idx, BOOL *stop) {
                                       currentCharUnit = charUnitToDraw;
                                       originPoint = dataUnitUpperLeftPoints[idx - startIndex];
                                       [self drawCharacter:currentCharUnit withOrigin:originPoint inContext:bitmapContext];
                                   }];

```
[Trouble drawing CoreGraphics lines in drawRect() method](http://stackoverflow.com/questions/9150665/trouble-drawing-coregraphics-lines-in-drawrect-method) 里的回答:You have to place CGContextStrokePath(context); outside the for loop. Otherwise it will create a fresh path on every run through the loop and that fails.
这个人的问题在于for循环最后都strokePath了，而for循环开头都没有moveToPoint，于是错误发生。  
[CGContextAddLineToPoint: no current point](http://stackoverflow.com/questions/9799682/cgcontextaddlinetopoint-no-current-point) 还有一个回答：You must first create a path with CGContextBeginPath before you can start adding points and lines to it. 然后有一个针对这个答案的评论：Not true. A CGContext always has a current path (possibly an empty one) which you can add elements to. You need CGContextBeginPath only when you want to discard the current path and start with a new empty path.
###5.CGBitmapContext画透明图的实现
之前是先用一个特定的颜色填充，然后画内容，画完之后再把之前那个颜色mask掉。
```objective-c
//Use a "strange or unique" color fill the context add mask it after drawing
UIColor *fillColor = [UIColor colorWithRed:220.0/255.0 green:198.0/255.0 blue:225.0/255.0 alpha:1.0];
CGContextSetFillColorWithColor(self.bitmapContext, fillColor.CGColor);
CGContextFillRect(self.bitmapContext, CGRectMake(0, 0, realDisplayWidth, realDisplayHeight));

//Mask
CGImageRef pageImageRef = CGBitmapContextCreateImage(bitmapContext);
const float colorMasking[8] = {220, 220, 198, 198, 225, 225, 0, 255};
CGImageRef pageImageMask = CGImageCreateWithMaskingColors(pageImageRef, colorMasking);
UIImage *pageImage = [UIImage imageWithCGImage:pageImageMask];
```
这样的实现非常2，内存开销增加了很大。用`CGContextClearRect(bitmapContext, CGRectMake(0, 0, realDisplayWidth, realDisplayHeight));`就好了，函数说明就是`Paints a transparent rectangle`。实现新功能时候先查查比较好的实现方式或者查文档很重要。
###6.toolbar死活不显示
navigationController的setToolbarHidden属性没设置默认是TRUE。
###7.Decode Mutable的东西时要注意mutableCopy
> Decoding an archive gives you immutable objects regardless of whether they were mutable or immutable when you encoded them.

需要`[self setMyArray:[[decoder decodeObjectForKey:@"myArray"] mutableCopy];`  
还要注意的就是object要copy不然会autorelease `You must also copy or retain the string object, otherwise it will be autoreleased:`。链接在[NSKeyedArchiver and NSKeyedUnarchiver with NSMutableArray](http://stackoverflow.com/questions/10391803/nskeyedarchiver-and-nskeyedunarchiver-with-nsmutablearray/10392017#10392017)
###8.程序突然运行不起来
先记着后面写，可能是因为最开始建工程时候包含storyboard，中途删掉了storyboard改了Main Interface和AppDelegate和plist，但是可能设备里还是有storyboard的。这样删了程序重新跑的时候就错了。很悲剧的时，设置了异常断点，而问题出在main.m里，这个都没报错的，去掉了才说找不到storyboard。  
试了一下只是简单的在viewDidLoad里改背景颜色，感觉还是怪怪的。直接删了storyboard不改工程配置也能运行，完了删了app重新运行也行，甚至删掉makeKeyAndVisible都能，但是这时候再把Main Interface里的Main删掉，就是黑屏了。
###9.DRDPoint和DRDInkPoint的encode decode