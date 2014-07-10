---
layout: post
title: "Swift Language Highlights: An Objective-C Developer’s Perspective"
date: 2014-06-06 13:57:06 +0800
comments: true
categories: iOS
---
<!--more-->
[原文](http://www.raywenderlich.com/73997/swift-language-highlights)是RW Tutorials里Matt Galloway写的，跟之前自己写的[Introduction to C++ for iOS Developers](http://daryl5.github.io/blog/2014/04/24/introduction-to-cplusplus-for-ios-developers/)类似，这篇文章是站在Objc开发者的角度来看Swift。这种通过比较两种语言的特性来学习的方法，我认为是最好的。
####1.Types
a.Swift引入了[Type Inference](http://en.wikipedia.org/wiki/Type_inference)，开发者不需要显式指定变量类型，`编译器`会根据变量被赋的值来推断变量的类型，如下例。
```objective-c
// automatically inferred
var name1 = "Matt"
// explicit typing (optional in this case)
var name2:String = "Matt"
```
Swift还引入了[Type Safety](http://en.wikipedia.org/wiki/Type_safety)。在Swift中，大多数情形下编译器明确知道一个对象所属的类型，这使得编译器能够在编译期间做更多的事，比如优化。其他少数情形可以参考C++中的[虚表](http://en.wikipedia.org/wiki/Vtable)适用情形。  
b.Objc是`Dynamic Typing`的，编译期间不会知道变量类型，`运行时`负责处理对象类型分析、消息分派等。这使得我们拥有在运行时给一个类动态添加方法等等黑魔法。
```objective-c
Person *matt = [[Person alloc] initWithName:@"Matt Galloway"];
[matt sayHello];
```
对上面的代码，编译器会去Person类的头文件去查看是否有sayHello方法，但是只会进行最简单的方法存在确认，由于Objc的动态特性，编译器不知道sayHello方法是否会在运行时改变，甚至不知道sayHello到底有没有实现。这种情形需要手动调用`respondsToSelector:`来检查。  
正是因为这种动态类型的特性，编译器能做的优化很少。系统用`objc_msgSend`来处理方法调度，在这个方法中运行时会根据调用方法的selector去寻找其对应实现并调用。这个过程增加了一些开销。  
c.下面分析Swifit对于类型的处理，如下例。
```python
var matt = Person(name:"Matt Galloway")
matt.sayHello()
```
编译器明确知道matt是Person类的实例，并知道sayHello方法具体定义在哪儿。那么在编译时就可以优化使得运行时能直接跳到方法实现的地方，而不需要动态调用。在其他特殊情形下，会利用类似C++[虚表](http://en.wikipedia.org/wiki/Vtable)的机制来实现，这虽然增加了一些开销但是比`Dynamic Dispatch`少了很多。
> 总的来说：  
The compiler is much more helpful in Swift. It will help stop subtle type related bugs from entering your codebase. It will also make your code run faster by enabling smart optimisations.

####2.Generics
Swift引入了[泛型](http://en.wikipedia.org/wiki/Generic_programming)，类似C++中的`模板`。这里有引文：
>  Swift is strict about types, you must declare a function to take parameters of certain types. But sometimes you have some functionality that is the same for multiple different types.

```objective-c
struct IntPair {
    let a: Int!
    let b: Int!

    init(a: Int, b: Int) {
        self.a = a
        self.b = b
    }

    func equal() -> Bool {
        return a == b
    }
}

let intPair = IntPair(a: 5, b: 10)
intPair.a // 5
intPair.b // 10
intPair.equal() // false
```
如果还需要一个FloatPair的话，就需要用到泛型了。
```objective-c
struct Pair<T: Equatable> {
    let a: T!
    let b: T!

    init(a: T, b: T) {
        self.a = a
        self.b = b
    }

    func equal() -> Bool {
        return a == b
    }
}

let pair = Pair(a: 5, b: 10)
pair.a // 5
pair.b // 10
pair.equal() // false

let floatPair = Pair(a: 3.14159, b: 2.0)
floatPair.a // 3.14159
floatPair.b // 2.0
floatPair.equal() // false
```
####3.Containers

