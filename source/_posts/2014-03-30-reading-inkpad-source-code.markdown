---
layout: post
title: "读Inkpad源代码"
date: 2014-03-30 19:55:11 +0800
comments: true
categories: iOS 
---
[Inkpad](https://github.com/sprang/Inkpad)是一个开源的基于OpenGLES的画图App，这篇博文记录阅读其源代码时的收获。
###0. 内联函数的使用

###1. 一些简洁的写法
- `CGRectGetWidth`，以前总是写成`self.frame.size.width`。此外还有`CGRectGetMidX`，直接得到矩形中心点的X坐标。

        CGPointMake(CGRectGetWidth(frame) / 2, CGRectGetHeight(frame) / 2);

- `[NSDictionary objectForKey]`，可以直接`dictionary[key]`。
- `BOOL isolate`转NSString，`@(isolate)`。NSArray的类似写法：

        NSArray *items = @[actionItem_, gearItem_, albumItem_, zoomToFitItem_];

###2. Best Practice
应用的**`帮助`**, **`关于`**等信息可以做成html页面。建立本地文件夹，在其中放置html、图片、css等，然后直接`self.view = webview;`即可。
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
