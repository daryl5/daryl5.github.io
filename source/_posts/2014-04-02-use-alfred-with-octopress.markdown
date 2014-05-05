---
layout: post
title: "Octopress & Alfred Workflow"
date: 2014-04-02 15:54:07 +0800
comments: true
categories: Mac
---
**[Alfred](http://www.alfredapp.com/)真是神器！**自己做了两个workflow来简化Octopress发表博文和修改博文的流程，实现的很简单。
<!--more-->
`发博文`用的是：Templates-Essentials-Keyword to Terminal Command。新建并用xcode打开空的博文后终端会运行`rake preview`，这样就直接进入编辑并预览状态。命令是`pb（post blog）`  
![pb](/blogimage/2014/pb.png)

`修改博文`用的是：Templates-Files and Apps-File filter from keyword and open。根据关键词显示相关文件，回车用xcode打开，打开终端运行`rake preview`。命令是`mb（modify blog）`

![mb](/blogimage/2014/mb.png)<br>
##下载
可以戳[这里](/blogfile/2014/归档.zip)下载然后根据自己的需求修改。

