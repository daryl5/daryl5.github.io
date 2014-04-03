---
layout: post
title: "UIDocument的使用"
date: 2014-04-01 21:42:32 +0800
comments: true
categories: iOS
---
读[Inkpad](https://github.com/sprang/Inkpad)源码时看到使用了[UIDocument](https://developer.apple.com/library/ios/documentation/uikit/reference/UIDocument_Class/UIDocument/UIDocument.html#//apple_ref/c/tdef/UIDocumentState)，然后搜到了Ray Wenderlich的博文：[iCloud and UIDocument: Beyond the Basics](http://www.raywenderlich.com/12779/icloud-and-uidocument-beyond-the-basics-part-1)，整理了一下大致用法。
<!--more-->
![useuidocument](/blogimage/2014/useuidocument.jpg)
##其他
###1.设备旋转支持
好多大厂的App居然也能支持反过来操作，真是不可理解。
```objective-c
- (BOOL)shouldAutorotateToInterfaceOrientation:(UIInterfaceOrientation)interfaceOrientation
{
    return (interfaceOrientation != UIInterfaceOrientationPortraitUpsideDown);
}
```