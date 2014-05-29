---
layout: post
title: "Introduction to C++ for iOS Developers"
date: 2014-04-24 21:43:30 +0800
comments: true
categories: C/C++
---
摘自[Introduction to C++ for iOS Developers](http://www.raywenderlich.com/62989/introduction-c-ios-developers-part-1)，这篇文章第二部分在[这里](http://www.raywenderlich.com/62990/introduction-c-ios-developers-part-2)。  
写了太久OC再去看C++，最好的方式当然是比较着来学习，所以说这篇博文真的太赞了。先挖坑，明天填。如果要学可以去[Learn C++](http://www.learncpp.com)。
<!--more-->
###1.Getting Started: A Brief History of Languages
C++曾经叫做`"C with Classes"`，OC其实也是在C上做了类这一扩展，两者的区别在于：C++的很多行为发生在编译期而OC更多在运行时期。所以像`Method Swizzling`这种东西C++是不可能有的。C++也没有`Introspection`和`Reflection methods`，不可能知道一个对象属于哪个类。
###2.C++ Classes
OC的类
```objective-c
// MyClass.h
#import <Foundation/Foundation.h>
@interface MyClass : NSObject
@end
 
// MyClass.m
#import “MyClass.h”
@implementation MyClass
@end
```
C++的类
```c++
// MyClass.h
class MyClass {
};
 
// MyClass.cpp
#include “MyClass.h”
/* Nothing else in here */
```
在OC中所有的类都集成自`NSObject`，如果删掉会编译报错`Class 'Test' defined without specifying a base class`。OC给C引入了`#import`来保证一个文件只能被包含一次，C++无对应机制。
###3.Class Member Variables and Functions
1.C++里叫`成员变量&成员函数`，OC里叫`实例变量&实例方法`。C++中没有`method`这个术语。在OC中，method就是通过消息派送来调用的东西，而函数是静态的C方式的函数调用如myFunction(intValue)。  
2.C++中成员变量和方法默认是`Private`的，OC中没有绝对的private。  
3.C++使用类：
```c++
//注意这里没有星号，m是存在栈上的，在Memory Management部分可以看到，OC里所有对象都必须加星号声明，使之存在堆
//存在栈上的所以不用new
MyClass m;
m.x = 10;
m.y = 20;
m.foo();
```
###4.Implementing Class Member Functions
1.`MyClass::foo() {}`这里的`::`就是表示foo函数是作为MyClass的一部分实现的。  
2.在头文件中直接实现成员函数就是`让编译器去尝试内联`。内联就是说当调用该函数时，不会跳到一个新的函数块，而是把整个函数的代码都在编译时候内联在了调用处。
```c++
// MyClass.h
class MyClass {
    int x;
    int y;
    void foo() {
        // Do something
    }
};
```
###5.Namespaces
1.比如`::`就是对命名空间的一种指定方式，用来告诉编译器`应该去哪里找这个foo函数`。  
2.C++命名空间的使用。OC无命名空间，只有设定工程的`类名前缀`。
```c++
namespace MyNamespace {
    class Person { … };
}
 
namespace LibraryNamespace {
    class Person { … };
}

MyNamespace::Person pOne;
LibraryNamespace::Person pTwo;
```
###6.Memory Management
a.栈固定大小；当指定函数运行时，数据将会入栈，函数运行完同样数量的数据将会出栈。因此随着程序运行，对栈容量的需求不会持续增长。  
b.堆也是一块用于运行程序的内存，不过大小不固定；随着程序运行，占用堆将会增多；程序会把在函数外还会使用的东西放在堆上（比如函数调用传指针，用来当做参数的指针在栈里，而指针所指内容在外部需要用到，因此指针内容在堆上）；此外比较大的数据也会在堆上因为栈很小。  

C++和OC对堆栈使用的区别
```c++
int stackInt = 5; //用的栈空间，函数返回时存储数字'5'的内存会被自动释放
//heapInt用堆空间，需要手动释放
//是不是应该说存储5的内存是分配在堆上的而不是heapInt，heapInt和stackInt应该在符号表什么的吧
int *heapInt = malloc(sizeof(int));
*heapInt = 5;
free(heapInt);
```
在OC中尝试分配到栈上`NSString stackString;`将会有编译错误`interface type cannot be statically allocated`。OC需要所有object都在堆上这样RC就能控制所有对象的生存与否。  
C++中对象也可以在栈上，如`MyPerson person`vs`MyPerson *person = new MyPerson()`，参考Implementing Class Member Functions部分和下面这一部分。
###7.C++ new and delete
C++引入new和delete来创建&摧毁堆上的对象
```c++
Person *person = new Person(); //对应OC里的alloc init
delete person; //OC里没有delete，这一操作由运行时在对象的引用计数为0时自动完成

//标量也行
int *x = new int();
*x = 5;
delete x;
```
###8.Accessing Members of Stack and Heap Objects
1.C++访问栈里对象的成员变量或调用成员函数要用点，堆里的用->。
```c++
Person stackPerson;
stackPerson.name = “Bob Smith”; ///< Setting a member variable
stackPerson.doSomething(); ///< Calling a member function
 
Person *heapPerson = new Person();
heapPerson->name = “Bob Smith”; ///< Setting a member variable
heapPerson->doSomething(); ///< Calling a member function
```
OC里面对property的访问用消息发送和点操作符都行（点操作符会被转为消息发送，应该是在编译阶段），对非property的实例变量直接访问或者用->都行（无property则不会响应这一消息所以点操作符不行）。  
2.this和self对应，用来在类成员函数（实例方法）内部访问当前对象。  
3.OC里给nil发送任何消息都ok，C++里对NULL对象去调用函数会崩溃（nullObject->doSomething();）。
###9.References
```c++
class Foo {
  public:
    int x;
};
 
void changeValue(Foo foo) {
    foo.x = 5;
}

Foo foo; //注意这个声明方式
foo.x = 1;
changeValue(foo); //foo.x还是1，因为C++会做一个foo的拷贝作为参数并不是想的
```
上面是`传值`，`传址`和`传引用`都会真正改变foo.x，如下
```c++
//传址
void changeValue(Foo *foo) {
    foo->x = 5;
}

//传引用。对上述例子直接加&就好了
void changeValue(int &x) {
    x = 5;
}
```
###10.Inheritance
1.普通继承
```objective-c
@interface Person : NSObject
@end
 
@interface Employee : Person
@end
```
```c++
class Person {
};

//public说明Person中所有public的成员变量在Employee仍然是public
//这块可以看看文章-http://www.learncpp.com/cpp-tutorial/115-inheritance-and-access-specifiers/
class Employee : public Person {
};
```
2.多继承  
OC也可以实现“多继承”，不过如下play和manage方法都必须实现，对C++只需要在base class中实现就行了，子类自动继承其实现。  
当两个base class都实现了就要消除歧义；OC用协议的话PlayerManager必须自己去实现foo，就没有这个问题。  
C++多继承
```c++
class Player {
    void play();
};
 
class Manager {
    void manage();
};

class PlayerManager : public Player, public Manager {
};

//Player和Manager都有foo函数
PlayerManager p;
p.foo();          ///< Error! Which foo?
p.Player::foo();  ///< Call foo from Player
p.Manager::foo(); ///< Call foo from Manager
```
OC多继承
```objective-c
@protocol Player
- (void)play;
@end
 
@protocol Manager
- (void)manage;
@end
 
@interface Player : NSObject <Player>
@end
 
@interface Manager : NSObject <Manager>
@end
 
@interface PlayerManager : NSObject <Player, Manager>
@end
```
###11.Polymorphism
1.“运行时”的OC和“编译时”的C++的一个区别：
```c++
class Foo {
  public:
    int value() { return 5; }
};
 
class Bar : public Foo {
  public:
    int value() { return 10; }
};

Bar *b = new Bar();
Foo *f = (Foo*)b;
printf(“%i”, f->value());
// Output = 5
```
在OC中，把一个子类转换成它的父类是OK的，因为当给对象发送消息时，运行时会去查询该对象所属的类，并调用该类对应的方法。在上例中，就会识别出f其实是一个指向Bar对象的指针，所以会调用Bar::value()然后输出10。
在C++中，编译器遇到value函数调用，然后发现f是一个指向Foo对象的指针，所以去找Foo::value()。  
2.Static Binding and Dynamic Binding  
静态绑定就是编译器在编译时即把哪个函数会被调用这些信息生成到了可执行文件里，所以运行的时候是不可能动态改变调用哪个方法。
动态绑定就是运行时在运行的时候负责决定哪个方法应该被调用，所以可以在运行的时候动态的给类添加方法或者动态改变方法的实现。  
不过C++也可以实现动态绑定，需要借助`虚函数`。  
3.Virtual Functions and the Virtual Table  
借助虚函数得到“情理之中”的结果，不过增加了查表的开销，如果是静态绑定那么只有函数调用的开销：
```c++
class Foo {
  public:
    virtual int value() { return 5; }
};
 
class Bar : public Foo {
  public:
    virtual int value() { return 10; }
};

Bar *b = new Bar();
Foo *f = (Foo*)b;
printf(“%i”, f->value());
// Output = 10
```
C++中很多设计都很灵活，把一些决定留给开发者去做，所以说其是一个`多范式`的语言；OC强制开发者使用严格的模式，特别是当使用Cocoa框架时。
###12.The Inner Workings of Virtual Functions
如果foo()函数不是虚函数，那么编译器产出的代码将会直接跳到对应类的foo函数处进行函数调用。  
如果是虚函数，那么会去查`v-table`，每一个类有一个虚表。对于上例，虽然强制转换了对象，但对象本身的`内容不变`，所以`虚表也不变`，进而就能找到正确的函数了。
###13.Constructors and Destructors
1.普通构造器：
```c++
class Foo {
  private:
    int x;
 
  public:
    Foo() {
        x = 0;
    }
 
    Foo(int x) {
        this->x = x;
    }
};
```
文艺构造器：
```c++
class Foo {
  private:
    int x;
 
  public:
    Foo() : x(0) {
    }
 
    Foo(int x) : x(x) {
    }
};
```
如果除了初始化对象的内部状态之外还有别的初始化逻辑或者函数调用，那么可以上面两者混用，也就是在文艺构造器中进一步实现一下函数体。
2.当用到`继承`的时候往往需要调用父类的初始化器，如OC中`self = [super init];`，在C++11之前是不能这么做的，不过可以通通去调用父类对应的初始化器：
```c++
class Foo {
  private:
    int x;
 
  public:
    Foo() : x(0) {
    }
 
    Foo(int x) : x(x) {
    }
};
 
class Bar : public Foo {
  private:
    int y;
 
  public:
    Bar() : Foo(), y(0) {
    }
 
    Bar(int x) : Foo(x), y(0) {
    }
 
    Bar(int x, int y) : Foo(x), y(y) {
    }
};
```
3.OC中一般都有指定初始化器，C++中无对应机制。如果两个初始化器有共同的初始化逻辑，那么就将其提出作为函数然后调用，如下：
```c++
class Bar : public Foo {
  private:
    int y;
    void commonInit() {
        // Perform common initialisation
    }
 
  public:
    Bar() : Foo() {
        this->commonInit();
    }
 
    Bar(int y) : Foo(), y(y) {
        this->commonInit();
    }
};
```
C++11之后终于有了类似指定初始化器的设计，如下：
```c++
class Bar : public Foo {
  private:
    int y;
 
  public:
    Bar() : Foo() {
        // Perform common initialisation
    }
 
    //如果想通过加上一个y(y)来同时初始化y，那么在Xcode中会出现错误
    //An initializer for a delegating constructor must appear alone
    Bar(int y) : Bar() {
        this->y = y;
    }
};
```
4.析构器不需要参数，所以只有一个析构函数。跟上面虚函数的例子一样，析构函数也存在调用问题，如下：
```c++
class Foo {
  public:
    ~Foo() {
        printf(“Foo destructor\n”);
    }
};

class Bar : public Foo {
  public:
    ~Bar() {
        printf(“Bar destructor\n”);
    }
};

Bar *b = new Bar();
Foo *f = (Foo*)b;
delete f;
// Output:
// Foo destructor
```
那么跟上面一样，把析构函数做成虚函数即可。但是会发现先调用了子类的析构函数，然后还调用了父类的，这是因为析构函数比较特殊，子类的析构函数运行完了会自动调用父类的，就跟OC在不用ARC时候在`dealloc`函数末尾需要`[super dealloc]`一样。
```c++
class Foo {
  public:
    virtual ~Foo() {
        printf(“Foo destructor\n”);
    }
};
 
class Bar : public Foo {
  public:
    virtual ~Bar() {
        printf(“Bar destructor\n”);
    }
};
 
Bar *b = new Bar();
Foo *f = (Foo*)b;
delete f;
// Output:
// Bar destructor
// Foo destructor
```
那为什么编译器不直接把析构函数作为虚函数来编译？因为虚函数有个查虚表的操作，每次析构对象都会调用，所以只要不继承就用普通析构函数就行。
###14.Operator overloading
1.一般的运算符操作其实就是函数调用，这是对标量进行运算，要实现对对象的运算，就要借助运算符重载。
```c++
int x = 5;
int y = x + 5; //可能内部实现是int y = add(x, 5);
```
2.运算符重载示例：
```c++
class DoubleInt {
private:
    int x;
    int y;
    
public:
    DoubleInt(int x, int y) : x(x), y(y) {}
    
    //operator+函数，必须有operator关键字，然后加一个运算符
    DoubleInt operator+(const DoubleInt &rhs) {
        return DoubleInt(x + rhs.x, y + rhs.y);
    }
    
    //跟int型相加
    DoubleInt operator+(const int &rhs) {
        return DoubleInt(x + rhs, y + rhs);
    }
};

DoubleInt a(1, 2);
DoubleInt b(3, 4);
DoubleInt c = a + b; //DoubleInt(4, 6)

DoubleInt a(1, 2);
DoubleInt b = a + 10; //DoubleInt(11, 12)
```
###15.Templates
使用模板有一点需要注意：如果多个类用到同一模板，那么一定要把`模板的实现`放在头文件。因为当编译器看到一个模板函数调用时会为调用类型编译出一个版本来，也就是说编译器需要知道模板函数的实现。因为编译器是一次只能处理一个编译单元, 也就是一次处理一个cpp文件,所以实例化时需要看到该模板的完整定义。这不同于普通的函数, 在使用普通的函数时，编译时只需看到该函数的声明即可编译, 而在链接时由链接器来确定该函数的实体。

扩展：C++标准中提到，一个编译单元`translation unit`是指一个.cpp文件以及它所include的所有.h文件，.h文件里的代码将会被扩展到包含它的.cpp文件里，然后编译器编译该.cpp文件为一个.obj文件，后者拥有PE[PortableExecutable,即Windows可执行文件]文件格式，并且本身包含的就已经是二进制码，但是，不一定能够执行，因为并不保证其中一定有main函数。当编译器将一个工程里的所有.cpp文件以分离的方式编译完毕后，再由连接器(linker)进行连接成为一个.exe文件。
```c++
template <typename T>
void swap(T &a, T &b) {
    T temp = a;
    a = b;
    b = temp;
}

int ix = 1, iy = 2;
swap(ix, iy);
```
###16.Template Classes
原本实现：
```c++
class IntTriplet {
  private:
    int a, b, c;
 
  public:
    IntTriplet(int a, int b, int c) : a(a), b(b), c(c) {}
 
    int getA() { return a; }
    int getB() { return b; }
    int getC() { return c; }
};
```
改为模板：
```c++
template <typename T>
class Triplet {
  private:
    T a, b, c;
 
  public:
    Triplet(T a, T b, T c) : a(a), b(b), c(c) {}
 
    T getA() { return a; }
    T getB() { return b; }
    T getC() { return c; }
};
```
模板类的使用。要注意这里尖括号里给出了类型，而调用模板函数的话直接`swap(ix, iy);`，这是因为模板函数的参数使得编译器能推断出需要编译哪个类型的版本。而对于模板类，需要明确告诉编译器要让其编译的类型。
```c++
Triplet<int> intTriplet(1, 2, 3);
Triplet<float> floatTriplet(3.141, 2.901, 10.5);
Triplet<Person> personTriplet(Person(“Matt”), Person(“Ray”), Person(“Bob”));
```
除了三个同类型成员变量，也可以支持三个不同类型的
```c++
template <typename TA, typename TB, typename TC>
class Triplet {
  private:
    TA a;
    TB b;
    TC c;
 
  public:
    Triplet(TA a, TB b, TC c) : a(a), b(b), c(c) {}
 
    TA getA() { return a; }
    TB getB() { return b; }
    TC getC() { return c; }
};

//使用
Triplet<int, float, Person> mixedTriplet(1, 3.141, Person(“Matt”));
```
###17.Standard Template Library (STL)
1.Containers  
C++里所有容器都是`mutable`的，没有OC里的`immutable`。  
2.Vector相当于数组:
```c++
#include <vector>
 
std::vector<int> v; //内部可能用模板实现
v.push_back(1);

int first = v[1];
int outOfBounds = v.at(100); //会检查是否越界，越界了产生异常
```
List是双向链表：
```c++
#include <list>
 
std::list<int> l;
l.push_back(1);

std::list<int>::iterator i;
for (i = l.begin(); i != l.end(); i++) {
    int thisInt = *i; //这里的i++和*i其实都是运算符重载
    // Do something with thisInt
}
```
此外还有对应OC中set的std::set，对应dictionary的std::map，还有std::pair等等。  
3.Shared Pointers  
C++11才引入的，实现`引用计数`。因为共享指针本身是存在栈上的，所以在离开一个作用域后会被销毁，这正是ARC所做的。引文：  
> Since shared pointers are stack objects themselves, they will be destroyed when they go out of scope. Therefore they will behave in much the same way that pointers to objects do under ARC in Objective-C.

示例：
```c++
std::shared_ptr<int> p1(new int(1)); ///< Use count = 1
 
if (doSomething) {
    std::shared_ptr<int> p2 = p1; ///< Use count = 2;
    // Do something with p2
}
 
// p2 has gone out of scope and destroyed, so use count = 1
 
p1.reset();
 
// p1 reset, so use count = 0
// The underlying int* is deleted
```
为什么说会超出作用域？因为当以传值形式传递函数参数时（应该不限于传参数，使用成员变量等也是在用参数，也会将其拷贝到栈），会拷贝一份给函数。所以当传递共享指针时，一个新的共享指针会产生并传给函数，这个新的拷贝就会增加共享指针的`Use Count`，而且在函数结束时会减1。
###18.Objective-C++
把实现文件从.m改为.mm就是告诉编译器，在这个文件里要用Objective-C++。注意OC++和OC的类不可能相互继承。
```objective-c
// Forward declare so that everything works below
@class ObjcClass;
class CppClass;
 
// C++ class with an Objective-C member variable
class CppClass {
  public:
    ObjcClass *objcClass;
};
 
// Objective-C class with a C++ object as a property
@interface ObjcClass : NSObject
//必须用assign，因为不能retain或者release一个不是OC object的对象，assign只是保证内存管理语义正确并让property生成accessor
@property (nonatomic, assign) std::shared_ptr<CppClass> cppClass;
@end
 
@implementation ObjcClass
@end
 
// Using the two classes above
std::shared_ptr<CppClass> cppClass(new CppClass());
ObjcClass *objcClass = [[ObjcClass alloc] init];
 
cppClass->objcClass = objcClass;
objcClass.cppClass = cppClass;
```
OC所有的对象都要求在堆上，而C++的可以在堆可以在栈。一个栈对象作为一个OC类的property就有点奇怪，但实际上还是在堆上的，因为这个类的对象本身肯定在堆上。
为了支持这种`栈对象`，编译器给alloc和dealloc添加了construct和destruct `"stack" C++ objects`的方法，`.cxx_construct & .cxx_destruct`。引文没太懂：
> Note: ARC actually piggybacks on .cxx_destruct; it now creates one of these for all Objective-C classes to write all the automatic cleanup code.

另外还有一点需要注意，如下，如果另外一个类需要使用MyClass，那么要import MyClass.h，也就是引入了一个使用了C++的文件，因此引用了MyClass的文件也需要被当做OC++来编译，哪怕它本身并不是用C++。
```objective-c
// MyClass.h
#import <Foundation/Foundation.h>
#include <list>
 
@interface MyClass : NSObject
 
@property (nonatomic, assign) std::list<int> listOfIntegers;
 
@end
 
// MyClass.mm
#import “MyClass.h”
 
@implementation MyClass
// …
@end
```
解决这个问题的办法是：在公开的头文件里尽量少的使用C++的东西，可以把私有property或者实例变量放在实现文件并在其中使用C++。以免所有import该类头文件的都被强制当做OC++来编译。