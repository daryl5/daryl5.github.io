---
layout: post
title: "《iOS设计模式解析》笔记"
date: 2015-05-21 14:14:16 +0800
comments: true
categories: iOS
---
<!--more-->

[TOC]

##第一章 你好，设计模式
####1.5.4 作为复合设计模式的MVC
MVC本身并不是最基本的设计模式，它包含了很多更加基本的设计模式，在MVC中，基本设计模式相互配合。  
Cocoa及Cocoa Touch的MVC中包含的基本模式有：Composite、Command、Mediator、Strategy、Observer。

- Composite：视图间的相互组合
- Command：Target-Action模式，action代码的执行被延迟到特定事件发生
- Mediator：Controller在Model和View中起的作用
- Strategy：原话是“控制器可以是视图对象的一个策略”，我理解类似代理吧
- Observer：模型对象向它所关注的控制器等对象发出内部状态变化的通知，KVO嘛，应该是模型对象向关注它的控制器等对象发出通知