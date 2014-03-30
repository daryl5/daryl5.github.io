---
layout: post
title: "读Inkpad源代码笔记"
date: 2014-03-30 19:55:11 +0800
comments: true
categories: iOS 
---
[Inkpad](https://github.com/sprang/Inkpad)是一个开源的基于OpenGLES的画图App，这篇博文记录阅读其源代码时的收获。
###1. 关于`CGRectGetWidth`
以前总是写成`self.frame.size.width`。此外还有CGRectGetMidX。写了好多累赘代码。
```
CGPointMake(CGRectGetWidth(frame) / 2, CGRectGetHeight(frame) / 2);
```
###2. 应用的**`帮助`**, **`关于`**等信息可以做成html页面
建立本地文件夹，在其中放置html、图片、css等，然后直接`self.view = webview;`即可。
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
###3.
