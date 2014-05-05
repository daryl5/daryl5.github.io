---
layout: post
title: "所有遇到过的坑们"
date: 2014-03-21 12:22:22 +0800
comments: true
categories: iOS 
---
<!--more-->
###12.DRDPoint和DRDInkPoint的encode decode
###13.深拷贝和浅拷贝
工程中用如下代码触发KVO。考虑到要在dataUnitArray变化时进行绘图，而且要给UIDocument实现undo/redo功能，所以没有直接`[self.dataUnitArray addObject:object]`，而是做一个临时变量然后直接赋值来是dataUnitArray变化进而触发KVO，同时，dataUnitArray的setter也是注册undo/redo行为最方便的地方。一开始有问题，写入字符时候点undo老是没反应，而且tempArray和dataUnitArray长度相等，很简单，一开始没有`mutableCopy`，直接指针赋值给了tempArray接着给tempArray添加也就是直接给dataUnitArray添加。  
另外，通过临时变量再赋值触发KVO的方式不好；赋值整个dataUnitArray更不好。后面重构一下。
```objective-c
NSMutableArray *tempArray = [self.dataUnitArray mutableCopy];
[tempArray addObjectsFromArray:charUnitArray];
self.dataUnitArray = tempArray;
```
###12.自定义Accessor
自己实现了setter和getter后，synthesize就不会起作用了，也就是说`没有_propertyName`这个实例变量，这种情形下如果在Accessor中还需要直接访问实例变量，那就要手动的添加`@sythesize propertyName = _propertyName`。
###11.连按删除按钮时候下标越界
在`- (void)deleteDataUnit`中：
```objective-c
NSMutableArray *tempArray = self.dataUnitArray;

//删除
[tempArray removeObjectAtIndex:cursorIndex - 1];

//触发KVO去绘制
self.dataUnitArray = tempArray;
```
然后在`- (UIImage *)generatePageImageWithPageIndex:(int)pageIndex`中就越界了。  
  
**原因**是tempArray和dataUnitArray指的同一块内存，比如原本长度12，删除一个变为11，在后台线程中绘制这个11长度的数组的时候，又按了一下删除，这时候已经触发了KVO，又多了一个绘制线程，之前那个绘制线程在绘制11长度，但是现在的长度是10，所以按的快的时候就会越界，按的慢等每一次绘制结束就没事。
  
**解决办法**是给tempArray赋值的时候进行一下mutable copy，这样每一个绘制线程就用的不同的拷贝，不会导致越界。
  
**更好的解决办法**：类似的情形下，数据改变了就应该停止之前的绘制线程，直接进入新的绘制。打印了一下当前线程，按的快了最多同时开了6个线程在绘制。  
  
[[NSOperation cancelAllOperations]; does not stop the operation](http://stackoverflow.com/questions/12660706/nsoperation-cancelalloperations-does-not-stop-the-operation)，这个链接里面提到了利用`NSOperationQueue`的`cancelAllOperations`来实现“有新数据则停止处理旧数据，从而直接转入对新数据的处理”，实际使用中的确起到了这样的效果，但是跟帖子中例子不同，我的函数里有很多`对象`，而一个RunLoop就会开一个autoreleasepool，在runloop里我没有去手动释放内存而直接返回，然后内存占用飙升了。懒得再去改进了，只求用户不要太变态。
###10.iOS7中自定义导航栏、状态栏的样式
设置状态栏：
```objective-c
//In AppDelegate
//Config StatusBar appearance
[[UIApplication sharedApplication] setStatusBarStyle:UIStatusBarStyleLightContent];

//Config UINavigationController appearance
[[UINavigationBar appearance] setBarTintColor:UIColorFromRGB(0x3395E3)]; // 51, 149, 227
[[UINavigationBar appearance] setTitleTextAttributes:[NSDictionary dictionaryWithObjectsAndKeys:
                                                      [UIColor colorWithWhite:0.9 alpha:1.0],
                                                      NSForegroundColorAttributeName,
                                                      [UIFont systemFontOfSize:20.0],
                                                      NSFontAttributeName, nil]];
[[UINavigationBar appearance] setTintColor:[UIColor colorWithWhite:0.93 alpha:1.0]];

//In ViewController--进入全屏模式，因为主体背景是白色，在AppDelegate设置状态栏是LightContent，全屏了就很瞎眼
[[UIApplication sharedApplication] setStatusBarStyle:UIStatusBarStyleDefault];

//Leave full screen
[[UIApplication sharedApplication] setStatusBarStyle:UIStatusBarStyleLightContent];
```
[Customizing Navigation Bar and Status Bar in iOS 7](http://www.appcoda.com/customize-navigation-status-bar-ios-7/)，这个不错。不过里面对单独viewController的statusbar设置`UIStatusBarStyle`没有成功，没搞懂。[How to change Status Bar text color in iOS 7.](http://stackoverflow.com/questions/17678881/how-to-change-status-bar-text-color-in-ios-7/17768797#17768797)和这个[preferredStatusBarStyle isn't called](http://stackoverflow.com/questions/19022210/preferredstatusbarstyle-isnt-called/19032879#19032879)可以看看。发现我很蛋疼，老是关注这些意义不大的东西。
###9.UIButton设置tintColor
`tintColor`属性并不是所有button都有的，只有`[UIButton buttonWithType:UIButtonTypeSystem]`才能设置。如果是init或者initWithFrame，设置了不会显示。
###8.在主线程更新UI-闪烁光标的实现
在KVO里要调用drawCursorWithRect画光标，而且这个函数里会启动timer来实现光标闪烁，如果直接调用会不闪烁，然后过一会儿闪一次。记着所有跟UI相关的东西都放在主线程执行。这里还有就是NSValue的`valueWithCGRect`和`CGRectValue`。还有就是`runloop`相关的东西，后面看看。
```objective-c
if ([keyPath isEqual:@"cursorStartPoint"]) {
    CGPoint cursorStartPoint = self.document.cursorStartPoint;
    CGRect cursorRect = CGRectMake(cursorStartPoint.x, cursorStartPoint.y + kCursorOffsetY, kCursorWidth, kCursorHeight);
    NSValue *cursorRectValue = [NSValue valueWithCGRect:cursorRect];
    [self performSelectorOnMainThread:@selector(drawCursorWithRect:) withObject:cursorRectValue waitUntilDone:NO];
}
```
另外一个问题是，当有新的手写字符时光标要移动，直接改变cursorLayer的frame属性，会有移动过程的动画，看[CALayer animates with frame change?](http://stackoverflow.com/questions/12103142/calayer-animates-with-frame-change)。去掉这个动画的方法是：
```objective-c
[CATransaction begin];
//或者[CATransaction setAnimationDuration:0];
//或者[CATransaction setDisableActions:YES];
[CATransaction setValue:(id)kCFBooleanTrue forKey:kCATransactionDisableActions];
[myView.myCALayer.frame = (CGRect){ { 10, 10 }, { 100, 100 } ];
[CATransaction commit];
```
###7.程序突然运行不起来
先记着后面写，可能是因为最开始建工程时候包含storyboard，中途删掉了storyboard改了Main Interface和AppDelegate和plist，但是可能设备里还是有storyboard的。这样删了程序重新跑的时候就错了。很悲剧的时，设置了异常断点，而问题出在main.m里，这个都没报错的，去掉了才说找不到storyboard。  
试了一下只是简单的在viewDidLoad里改背景颜色，感觉还是怪怪的。直接删了storyboard不改工程配置也能运行，完了删了app重新运行也行，甚至删掉makeKeyAndVisible都能，但是这时候再把Main Interface里的Main删掉，就是黑屏了。
###6.Decode Mutable的东西时要注意mutableCopy
> Decoding an archive gives you immutable objects regardless of whether they were mutable or immutable when you encoded them.

需要`[self setMyArray:[[decoder decodeObjectForKey:@"myArray"] mutableCopy];`  
还要注意的就是object要copy不然会autorelease `You must also copy or retain the string object, otherwise it will be autoreleased:`。链接在[NSKeyedArchiver and NSKeyedUnarchiver with NSMutableArray](http://stackoverflow.com/questions/10391803/nskeyedarchiver-and-nskeyedunarchiver-with-nsmutablearray/10392017#10392017)
###5.toolbar死活不显示
navigationController的setToolbarHidden属性没设置默认是TRUE。
###4.CGBitmapContext画透明图的实现
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
同时要注意，要创建透明位图就要在初始化位图上下文时指定CGBitmapInfo为`kCGBitmapByteOrderDefault | kCGImageAlphaPremultipliedLast`，也可以换成`kCGImageAlphaPremultipliedFirst`，last就是RGBA反之是ARGB，预乘alpha就代表图像是有alpha通道的，不设置这个clearRect后会是黑色。假如一个像素(A，R，G，B)的四个分量都用规一化的数值表示，(A，R，G，B)为(1，1，0，0)时显示红色。当像素为 (0.5,1,0,0)时，预乘的结果就变成(0.5,0.5,0,0)，这表示原来该像素显示的红色的强度为1，而现在显示的红色的强度降了一半。  
另，`颜色深度`就是颜色位数，比如RGB一个分量用8位，总共24位，每个像素可以用2的24次方就是16777216中颜色表示，也就是几年前卖手机时候说的1600w色- -！
###3.使用CGBitmapContext并行绘图时报错
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
在iOS7之前的代码中，最后一个CGBitmapInfo是kCGImageAlphaPremultipliedFirst，在iOS7下用这个会有警告  
> Implicit conversion from enumeration type 'enum CGImageAlphaInfo' to different enumeration type 'CGBitmapInfo' (aka 'enum CGBitmapInfo')

解决办法是改为`kCGBitmapByteOrderDefault | kCGImageAlphaPremultipliedFirst`。这是因为iOS7中最后一个属性有变化，由`CGImageAlphaInfo`变为`CGBitmapInfo`。[这里](http://www.cnblogs.com/wuxiufang/p/3397070.html)说的很详细。