---
layout: post
title: "UIDocument的使用"
date: 2014-04-01 21:42:32 +0800
comments: true
categories: iOS
---
读[Inkpad](https://github.com/sprang/Inkpad)源码时看到使用了[UIDocument](https://developer.apple.com/library/ios/documentation/uikit/reference/UIDocument_Class/UIDocument/UIDocument.html#//apple_ref/c/tdef/UIDocumentState)，然后搜到了Ray Wenderlich的博文：[iCloud and UIDocument: Beyond the Basics](http://www.raywenderlich.com/12779/icloud-and-uidocument-beyond-the-basics-part-1)，整理了一下大致用法。
<!--more-->
###1. 继承UIDocument并重写方法
需要重写`loadFromContents:ofType:error`方法来读，重写`contentsForType:error`方法来写。
###2. 输入输出格式
UIDocument支持两种类，`NSData`适用于文档只是一个单一文档，`NSFileWrapper`相当于文件夹，适用于文档包含多个想要单独加载的文件，比如矢量图绘制App要显示一个缩略图，这个需要单独提前加载。FileWrapper内部是Key-Value形式的存储，文件名是key，文件存在本地的archive内容（或是archive文件名）是value，所以会有：
<!--more-->
    //从文件名（如xxxx.title，不是xxxx.note）得到FileWrapper
    NSFileWrapper *fileWrapper = [self.fileWrapper.fileWrappers objectForKey:preferredFilename];
    
    //从FileWrapper得到文件内容
    NSData *data = [fileWrapper regularFileContents];
    NSKeyedUnarchiver *unarchiver = [[NSKeyedUnarchiver alloc] initForReadingWithData:data];
###3. iCloud
Apple规定，应用的所有documents要么在沙盒，要么在iCloud本地文件夹，用户不能选择某一文件来让其单独的存储在iCloud。

>All documents of an application are stored either in the local sandbox or in an iCloud container directory. A user should not be able to select individual documents for storage in iCloud.
<!--more-->
###4. 关于undo/redo
UIDocument有undo/redo支持，访问器是注册undo动作最合适的地方，所以document类里，对需要添加undo/redo功能的数据要重写访问器。对不需要undo/redo功能的数据，直接给document用property添加一个model就好。
###5. 整体的实现结构
以记事本为例，title和content两个model，其中都只包含一个NSString。主视图直接在title，detailViewController再加载content。
>```TitleModel```和```ContentModel```都实现`NSCoding`协议方法，这里可以考虑把`VersionNumber`encode进去以便后期支持文件格式扩展。
>**NoteDocument.h**
>>`@property (nonatomic, strong) TitleModel *titleModel;
//这里对ContentModel中的内容实现访问器，原因第4条有，方便undo/redo`
`- (NSString *)contentString;`  
`- (void)setContentString:(NSString *)str; `

>**NoteDocument.m**  
>>
- 类扩展里定义`contentModel`和`fileWrapper`(读文件时候用)，`titleModel`直接放在头文件是因为It’s OK if the user accesses the metadata directly though, as it’s not something the app will modify. Instead, the metadata will be automatically updated when the user sets the photo.
- 重写`loadFromContents`和`contentsForType`，loadFromContents里面self.wrapper = (NSFileWrapper *)contents;给wrapper赋值，contentsForType里面把两个model encode进wrapper
- 重写`titleModel`和`contentModel`两个getter实现Lazy Loading，需要的时候再decode；如果文件不存在则把model初始化为nil
- 实现ContentModel的content的Accessor来实现undo/redo，在Setter里面执行如下代码块：
```objective-c
if ([self.contentModel.content isEqual:str]) {
    return;
}
    
NSString *oldContent = self.contentModel.content;
self.contentModel.content = str;
    
[self.undoManager setActionName:@"Content Change"];
[self.undoManager registerUndoWithTarget:self selector:@selector(setContentString:) object:oldContent];
```

##其他
###1. 设备旋转支持
好多大厂的App居然也能支持反过来操作，真是不可理解。代码摘自引用博文实现的工程。
```objective-c
- (BOOL)shouldAutorotateToInterfaceOrientation:(UIInterfaceOrientation)interfaceOrientation
{
    return (interfaceOrientation != UIInterfaceOrientationPortraitUpsideDown);
}
```
###2. Undo/Redo
[这里](http://blog.163.com/chenchen..1986/blog/static/760631462013222314817/)的例子简单有效。
###3. 自己画的大致思路，实际实现没这么做。
![useuidocument](/blogimage/2014/useuidocument.jpg)