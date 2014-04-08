---
layout: post
title: "读Inkpad源代码"
date: 2014-03-30 19:55:11 +0800
comments: true
categories: iOS 
---
[Inkpad](https://github.com/sprang/Inkpad)是一个开源的基于OpenGLES的画矢量图的App，这篇博文记录阅读其源代码时的收获。
###0. 内联函数的使用

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