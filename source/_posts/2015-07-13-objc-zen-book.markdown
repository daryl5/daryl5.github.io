---
layout: post
title: "Objc Zen Book总结"
date: 2015-07-13 22:28:32 +0800
comments: true
categories: iOS
---
<!--more-->
[原文](https://github.com/objc-zen/objc-zen-book), [中译版](https://github.com/oa414/objc-zen-book-cn)

## Yoda表达式

写`if (count == 5)`而不是`if (5 == count)`，因为后者看上去像是“5是count的值”而不是“count的值是5”，“蓝色是天空的颜色”而不是“天空的颜色是蓝色”，显然前者更合理啊。名字源于星球大战，因为Yoda讲话的方式总是倒桩:-p  

## 黄金大道

不要嵌套过多的if，多个return也行，看上去也清晰，比如：
```objective-c
- (void)someMethod {
    if (![someOther boolValue]) {
        return;
    }

    //Do something important
}
```

## if判断如果过于复杂应拆分

三元运算符?:里的表达式如果复杂也应该拆分出来，提升代码可读性。`result = object ? object : [self createObject];`简化成`result = object ? : [self createObject];`
```objective-c
BOOL nameContainsSwift  = [sessionName containsString:@"Swift"];
BOOL isCurrentYear      = [sessionDateCompontents year] == 2014;
BOOL isSwiftSession     = nameContainsSwift && isCurrentYear;

if (isSwiftSession) {
    // Do something very cool
}
```

## <font color="red">错误处理</font>

当方法返回一个错误参数的引用的时候，检查返回值，而不是错误的变量。
```objective-c
NSError *error = nil;
if (![self trySomethingWithError:&error]) {
    // Handle Error
}
```
此外，一些苹果的 API 在成功的情况下会对 error 参数（如果它非 NULL）写入垃圾值（garbage values），所以如果检查 error 的值可能导致错误 （甚至崩溃）。

## switch语句

当在 switch 语句里面使用一个可枚举的变量的时候，default是不必要的。
```objective-c
switch (menuType) {
    case ZOCEnumNone:
        // ...
        break;
    case ZOCEnumValue1:
        // ...
        break;
    case ZOCEnumValue2:
        // ...
        break;
}
```
此外，为了避免使用默认的 case，如果新的值加入到 enum，会马上收到一个 warning 通知`Enumeration value 'ZOCEnumValue3' not handled in switch.`

## 方法命名

使用“and”命名的时候应当更加谨慎，它不应该用作阐明有多个参数。
```objective-c
//pros
- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height;
//cons
- (instancetype)initWithWidth:(CGFloat)width andHeight:(CGFloat)height;
```

## 字面量

尽量用简洁语法，一方面可读，一方面避免要往array里插入nil导致崩溃。  
不过应该避免`NSMutableArray *aMutableArray = [@[] mutableCopy]`这种写法，先创建了一个不可变变量，然后进行了mutableCopy，造成浪费。

## Initializer 和 dealloc

dealloc应该放在最前面，init紧随其后，这两个从字面意思上看也应该放在一起。如果有多个initializer，designated initializer应该放在最前面。  
详细解释`[[NSObject alloc] init]`，从字面上看`分配内存`和`初始化`是两个操作。

- alloc表示对象分配内存，这个过程涉及分配足够的可用内存来保存对象，写入isa指针，初始化 retain 的计数，并且初始化所有实例变量。
- init 是表示初始化对象，这意味着把对象转换到了个可用的状态。这通常是指把可用的值赋给了对象的实例变量。

alloc返回一个没有初始化的对象。每个发送到实例的消息都会被运行时转为`objc_msgSend()`函数的调用，这个函数的第一个参数就是指向alloc返回的、名为self的指针。一般的objc方法也带了两个隐含参数，一个`self`，一个`_cmd`。alloc后，self有值了，这个实例便可执行所有方法。

## Mutable Object

任何可以用可变对象设置值 (eg. NSString,NSArray,NSURLRequest) 的 property 都要用`copy`，以免被其他对象随意赋值。  
另外对于本身就mutable的property，如果需要暴漏给外界就暴漏一个immutable的拷贝，如下：

```objective-c
/* .h */
@property (nonatomic, readonly) NSArray *elements

/* .m */
- (NSArray *)elements {
    return [self.mutableElements copy];
}
```

`readonly`还是要多用。













