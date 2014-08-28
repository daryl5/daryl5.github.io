---
layout: post
title: "读开源项目总结"
date: 2014-03-30 19:55:11 +0800
comments: true
categories: iOS 
---
<!--more-->
#[懒人笔记](https://github.com/liaojinxing/Voice2Note)是一个很适合新手学习的项目，调用了讯飞语音识别库、友盟App统计和微信的分享库。功能很简单，实现的也很干净明晰。
###1.隐藏键盘
写新笔记界面有一个UITextField和一个UITextView，为了区分让哪一个resignFirstResponder，原作者添加了一个BOOL类型的_isEditingTitle，实际上用下面的代码即可保证界面上全部可能用到键盘的控件全部都resign了。  
```objective-c
[[UIApplication sharedApplication] sendAction:@selector(resignFirstResponder) to:nil from:nil forEvent:nil];
```
其原理是`响应者链`，具体可以看[The Amazing Responder Chain](http://www.cocoanetics.com/2012/09/the-amazing-responder-chain/)。
###2.根据cell的文本内容来获取cell的高度
```objective-c
+ (CGFloat)heightWithString:(NSString *)string width:(CGFloat)width
{
    NSDictionary *attributes = @{ NSFontAttributeName: [UIFont boldSystemFontOfSize:17] };
    CGSize size = [string boundingRectWithSize:CGSizeMake(width, kMaxTitleHeight)
                        options:NSStringDrawingUsesLineFragmentOrigin
                        attributes:attributes
                        context:nil].size;
    return ceilf(size.height);
}
```
#[DDHTimerControl](https://github.com/dasdom/DDHTimerControl)是一个倒计时控件，做的很好看。
###1.关于loadView和viewDidLoad
[这里](http://www.dreamingwish.com/frontui/article/default/correct-online-information-error-loadview-viewdidload-viewdidunload.html)说的很清楚。  

> loadView方法的默认实现是这样：先寻找有关可用的nib文件的信息，根据这个信息来加载nib文件，如果没有有关nib文件的信息，默认实现会创建一个空白的UIView对象，然后让这个对象成为controller的主view。   
所以，重载这个函数时，你也应该这么做。并把子类的view赋给view属性(property)（你create的view必须是唯一的实例，并且不被其他任何controller共享），而且你重载的这个函数不应该调用super。
如果你要进行进一步初始化你的views，你应该在viewDidLoad函数中去做。在iOS 3.0以及更高版本中，你应该重载viewDidUnload函数来释放任何对view的引用或者它里面的内容（子view等等）。

###2.一个关于Setter的好的习惯
自己重写了一个property的setter，然后别的地方有调用setXxx，那么最好在头文件中也声明一下setter，虽然不声明也没错，但是嘛。。。
###3.instancetype
[这里](http://blog.eddie.com.tw/2013/12/16/id-and-instancetype/)说的很清楚。
> 其實 instancetype 就只是個關鍵字(keyword)，它告訴編譯器回傳型態，讓編譯器可以在編譯階段就有足夠的資訊可以來判斷你寫的程式碼是不是有問題。只能用作返回值，id却可以随便用。

具体例子：
```objective-c
@interface Animal : NSObject
+ (id) createAnimal;
@end

@implementation Animal
+ (id) createAnimal
{
return [[self alloc] init];
}
@end

@interface Fox : Animal
- (void) say;
@end

@implementation Fox
- (void) say
{
NSLog(@"what does the fox say!?");
}
@end

//编译能过，因为createAnimal回传类型是id，编译器不能在编译阶段infer出它的真实类型，
//所以只好先放过，然后在运行时候就会unrecognized selector sent to instance。
//可以通过respondToSelector来检查，更简单的方法是createAnimal返回instancetype
[[Animal createAnimal] say];
```
###4.在类扩展里的`实例变量`和`property`
以往总是会写花括号然后写实例变量，用property一样实现“私有”，其好处有以下：

1. 可以重写getter，实现lazyloading
2. property对于实例变量可以设定权限

#[Inkpad](https://github.com/sprang/Inkpad)是一个开源的基于OpenGLES的画矢量图的App。
###0. 内联函数的使用
内联函数只是给编译器的提示，最终能不能内联还要看编译器。  
####(1)相同点
`static inline double radians (double degrees) { return degrees * M_PI/180; }`；都是在与处理阶段对代码块进行替换。  
####(2)不同点
内联函数是值传递，宏定义是简单替换。内联函数有类型检查，此外`省去了很多函数调用汇编代码如：call和ret等`。  
`#define MAX(a, b) a>b?a:b`如果`MAX( num1, num2 )`就没问题，但是如果是`MAX( 17+32, 25+21)`，展开后是`17+32>25+21?17+32:25+21`。甚至`#define A 2+3`，然后`c = 4 * A`都会成为`4 * 2 + 3`。比较安全的写法是`#define MAX( (a), (b) ) (a)>(b)?(a)b)`，但是这样还是有问题，`MAX(i++,j++)`完了每个都会加2了。
宏定义执行快是因为没有`函数调用`的开销，但是如果宏用的多文件就会变很大，执行文件太大可能导致执行时换页频繁（略夸张）。
###1. 一些简洁的写法
- `CGRectGetWidth`，以前总是写成`self.frame.size.width`。此外还有`CGRectGetMidX`，直接得到矩形中心点的X坐标。

        CGPointMake(CGRectGetWidth(frame) / 2, CGRectGetHeight(frame) / 2);

- `[NSDictionary objectForKey]`，可以直接`dictionary[key]`。
- `BOOL isolate`转NSString，`@(isolate)`。NSArray的类似写法：

        NSArray *items = @[actionItem_, gearItem_, albumItem_, zoomToFitItem_];

###2. Best Practice
####(1) “帮助”、“关于”视图
应用的**`帮助`**, **`关于`**等信息可以做成html页面。建立本地文件夹，在其中放置html、图片、css等，然后直接`self.view = webview;`即可。
<!--more-->
```objective-c
- (NSURL *) helpURL
{
    NSString *resource = NSLocalizedString(@"index", @"Name of Help html file");
    NSString *path = [[NSBundle mainBundle] pathForResource:resource ofType:@"html" inDirectory:@"Help"];
    return [NSURL fileURLWithPath:path isDirectory:NO];
}
- (void)loadView
{
    UIWebView *webView = [[UIWebView alloc] initWithFrame:[[UIScreen mainScreen] bounds]];
    webView.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    self.view = webView;
    [webView loadRequest:[NSURLRequest requestWithURL:[self helpURL]]];
}
```
####(2) 应用“设置”功能
以“显示网格”这一设置项为例，通过`UILable  UISwitch`来实现。其实可以全部通过NSUserDefaults实现，但inkpad中有个NSMutableDictionary的settings_作为”中间变量“:

- 可能是考虑到在初始化时一次读取NSUserDefaults而不是需要属性就去查NSUserDefaults更好;
- 还有个原因就是每个drawing都会把settings_当做Document的一部分进行存储，NSMutableDictionary才能去存储。

在`WDDrawingController`中有`WDPropertyManager`。除了下面的示例代码，很多属性的管理都在这里，**区别是**settings_只管理`设置`视图里能改变的属性，并且其值会被存储到document里，跟单独drawing相关；~~而propertyManager管理`应用状态`，比如不选路径的时候“路径”的popover tableView里就全部不可用，如添加锚点、合并路径。~~
```objective-c
// constantly updating the user defaults kills responsiveness after the keyboard has been made visible
// so use this temporary dictionary to avoid hitting the defaults all the time  
NSMutableDictionary *defaults_ = [[NSMutableDictionary alloc] init];//in init

//- (void) didEnterBackground:(NSNotification *)aNotification
- (void) updateUserDefaults
{
    for (NSString *key in [defaults_ allKeys]) {
        [[NSUserDefaults standardUserDefaults] setObject:defaults_[key] forKey:key];
    }
}
```
相关代码如下：
```objective-c
//属性设置-在“设置”界面控件响应方法中
//WDSettingsController
- (void) takeShowGridFrom:(id)sender
{
    UISwitch    *mySwitch = (UISwitch *)sender;
    
    [[NSUserDefaults standardUserDefaults] setBool:mySwitch.isOn forKey:WDShowGrid];
    [[NSUserDefaults standardUserDefaults] synchronize];
    
    drawing_.showGrid = mySwitch.isOn;//Setter方法
}

//属性管理的中间类-在Accessor中设置和应用，发送通知最终应用设置
//WDDrawing:NSObject
- (void) setShowGrid:(BOOL)showGrid
{
    settings_[WDShowGrid] = @(showGrid);//settings_是NSMutableDictionary，WDShowGrid是NSString “WDShowGrid”
    [[NSNotificationCenter defaultCenter] postNotificationName:WDDrawingChangedNotification object:self];
    
    // this isn't an undoable action so it does not dirty the document
    [self.document markChanged];
}

- (BOOL) showGrid
{
    return [settings_[WDShowGrid] boolValue];
}

//属性改变的即时应用-接收通知并通过setNeedsDisplay处发drawRect重绘
//WDCanvas
- (void) invalidateFromNotification:(NSNotification *)aNotification
{
    NSValue     *rectValue = [aNotification userInfo][@"rect"];
    NSArray     *rects = [aNotification userInfo][@"rects"];
    CGRect      dirtyRect;
    float       fudge = (-1.0f) / viewScale_;
    
    if (rectValue) {
        //...
    } else if (rects) {
        //...
    } else {//没有任何userInfo，对应showGrid这一属性的改变
        [self setNeedsDisplay];
    }
}

- (void)drawRect:(CGRect)rect
{
    //...
    if (drawing_.showGrid && !drawingIsolatedLayer) {
        [self drawGrid:ctx];
    }
    //...
}

//属性的保存-主要在WDSettingsController，设置并同步
借助NSUserDefaults

//属性的加载-各种init方法中从NSUserDefaults初始化settings_
//WDDrawing
- (id) initWithSize:(CGSize)size andUnits:(NSString *)units
{
    // each drawing saves its own settings, but when a user alters them they become the default settings for new documents
    // since this is a new document, look up the values in the defaults...
    NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
    NSArray *keyArray = @[WDShowGrid, WDSnapToGrid, WDSnapToPoints, WDSnapToEdges, WDDynamicGuides, WDRulersVisible];
    for (NSString *key in keyArray) {
        settings_[key] = @([defaults boolForKey:key]);
    }
}
```
###3. `NSNotification`的名字应该在哪定义
在`WDToolManager.m`中，`NSString *WDActiveToolDidChange = @"WDActiveToolDidChange";`，然后在头文件中`extern NSString *WDActiveToolDidChange;`。一般来说关心这个通知（当前工具改变）的类都引用了这个类的头文件，这样比放在一个`ConstantsDefine.h`里要好。  
###4.枚举定义中为什么要用左移
这里解释的很详细[Why use the Bitwise-Shift operator for values in a C enum definition?](http://stackoverflow.com/questions/3999922/why-use-the-bitwise-shift-operator-for-values-in-a-c-enum-definition)。如果用
    typedef enum { WDToolDefault, WDToolShiftKey, WDToolOptionKey }  
那么`WDToolDefault | WDToolShiftKey`就是`1 | 2`会得到3！！用左移就不会有这个问题，就是***可以在一个变量中支持多个枚举值***（将其相加或者取或），判断的时候取与。
    typedef enum {
        WDToolDefault           = 0,
        WDToolShiftKey          = 1 << 0,
        WDToolOptionKey         = 1 << 1,
        WDToolControlKey        = 1 << 2,
        WDToolSecondaryTouch    = 1 << 3
    } WDToolFlags;  
###5. 子类不能重写touchesBegan但要在其发生同时完成处理
`FingerTackerView`重写`touchesBegan`等等方法，然后在其内部进行一些操作后调用`[self methodForSubclassOverwritten]`，这个方法可以置空。`HandWritingView`，`GestureEditorView`等继承自`FingerTrackerView`的类里重写`[self methodForSubclassOverwritten]`，这样就能在采集点的同时，对点的处理根据当前选择功能（即所处的视图）来由对应的子类来处理。子类不会出现`touchesBegan`等，但是可以通过重写方法来实现手势运动时处理。