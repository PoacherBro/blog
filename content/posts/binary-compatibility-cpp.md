---
title: "DLL的二进制兼容"
date: 2018-06-08T10:44:47+08:00
draft: false
---

## 详解  

### 什么是二进制兼容？  
所谓二进制兼容就是在做版本升级（也可能是Bug fix）库文件的时候，不必要重新编译使用这个库的可执行文件或使用这个库的其他库文件，同时能保证程序功能不被破坏。  
当然，这只是一个现象级描述，其实在一些简单的例子里，假设我们导出一个C++类，在调用时，第三方仍然不需要重新编译可以运行。如下面例子：  

* **FastString.dll** - *FastString.h文件*  

```cpp
//导出类
class __declspec(dllexport) FastString
{
public:
    FastString();
    ~FastString();
    
    size_t length() { return 0; }

private:
    unsigned char *m_bytes;
}
```

* **test.exe** - *main.cpp文件*  

```cpp
int main()
{
    FastString fStr;
    size_t len = fStr.length();
    
    printf("fast string length %d\n", len);
    _getch();
    return 0;
}
```

如果我们给导出类加上一个虚函数
```cpp
virtual boole isEmpty();  // 位于 length 方法之前
```
重新编译FastString.dll，然后直接运行test.exe，发现仍然能打印出`fast string length 0`，并且没有运行错误。  
所以按照上面所说，FastString.dll是二进制兼容的。然而不是的！因为它增加了一个虚函数，导致FastString实例增加了一个虚函数表（是一个`void **`指针），那为什么运行的时候没有错误呢？参考这个问题：[SO- why new virtual function will not break binary compatibility per phenomenon?](https://stackoverflow.com/questions/49317106/why-new-virtual-function-will-not-break-binary-compatibility-per-phenomenon)  

所以严格来讲，二进制兼容是保证在版本升级的情况下，对象实例的内存布局没有发生变化。  

### 为什么需要二进制兼容？  

打个比方，如果库A升级没有做到二进制兼容，那么所有依赖它的程序（或库）都需要要重新编译才能应用A库的新版本，否则会出现各种未知异常，其直接现象就是程序莫名其妙的挂掉。  

譬如像Qt这种使用率很广的程序库，如果每次版本升级都需要第三方使用者重新编译源程序，我想肯定是很多人不愿意的。  

### 哪些常见做法会破坏二进制兼容？  

1. 给函数增加参数，现有的可执行文件无法传这个额外的参数。  
2. 增加虚函数，会造成虚函数表vtbl里的排列变化。（不要考虑“只在末尾增加”这种取巧行为，因为你的class可能已被继承）  
3. 增加默认模板类型参数  
    例如：`template <typename T> class Grid {}` 变更为 `template <typename t, typenameContainer=vector> class Grid{}`。  
4. 改变`enum`的值。把`enum Color { Red = 3};`改为`Red = 4`，这会造成错位。当然，由于`enum`自动排列取值，添加`enum`项也是不安全的，除非是在末尾添加。  

### 哪些做法多半不会破坏二进制兼容？  
1. 增加新的class  
2. 增加非虚成员函数  

> 关于更多的 **Do's and Don'ts**，可以阅读KDE的两篇wiki：[Policies/Binary Compatibility Issues With C++](https://community.kde.org/Policies/Binary_Compatibility_Issues_With_C%2B%2B#The_Do.27s_and_Don.27ts)， [Policies/Binary Compatibility Examples](https://community.kde.org/Policies/Binary_Compatibility_Examples)

## 如何实现二进制兼容？  

### COM理论  
COM (Component object model) 组件对象模型是微软提出的一个伟大想法，它其实是一个规范，并且是二进制规范，也就是说只要遵循这个规范，任何语言、任何平台都可以相互调用相应组件。  

COM涉及到几个概念：  
1. *class ID*，可以是*CLSID - class的GUID* 或者 *IID - interface的GUID*。COM通过这个ID来保证跨语言，因为基本上所有语言都可以处理GUID字符串；另外COM开发者可以通过GUID来获取到准确的对象结构。  
2. *coclass - **c**omponent **o**bject **class***，简单来说就是COM组件提供给使用者的接口类，这些类其实都是都继承 `IUnkown`接口的抽象类，里面都是纯虚函数。这个`IUnknown`包含三个方法：  
    - `AddRef` - 增加对象引用计数  
    - `Release` - 减少引用计数，如果计数为0，则销毁  
    - `QueryInterface` - 根据GUID来查到对象  

> COM组件还涉及到注册表，它可以注册到操作系统的注册表中，这样就算当前这个组件DLL物理位置与运行文件不在同一个目录，也可以加载并获取DLL的导出对象或者函数。更多了解可以看 [CodeProject - Introduction to COM - What It Is and How to Use It](https://www.codeproject.com/Articles/633/Introduction-to-COM-What-It-Is-and-How-to-Use-It)。  

**那为什么可以说COM能保证二进制兼容呢？**  

其实通过上面两个概念可以有点思绪，所谓二进制兼容对于C++ 来说就是要保证第三方使用DLL提供的接口对象时，保证内存布局不会改变，或者说不会影响。对于C++来说，对象内存布局的主要包括：  
1. 变量  
2. 虚函数 - 每个实例都会有一个虚函数列表（包括基类的）  

对于COM实现来说，因为是通过GUID来获取对象，并且这些对象都是由接口来提供的实例化（抽象类不能创建实例，这些实例都是继承的子类实现），就像 **caller ----> coclass (*interface*) --*create*--> instance** 这样调用。  
由于 instance 是在COM组件类（DLL）实例化以及释放，所以其内存布局对于 caller 来说是没有影响的。  

### D指针设计模式  

D指针模式其实和上面COM的方式有点类似，但是它没有COM那么复杂。我们用一个例子来说明为什么D指针模式能做到二进制兼容。  

假设你的class `Foo` 里定义了一个前置声明类FooPrivate  
```cpp
class FooPrivate;
```
并且把D指针放在private区  
```cpp
private:
    FooPrivate *d;
```
`FooPrivate` 类可以完全在class实现的地方定义（一般是 *.cpp），例如：  
```cpp
class FooPrivate {
public:
    FooPrivate() : m1(0), m2(0) {}
    int m1;
    int m2;
    QString s;
}
```
然后你所要做的就是在Foo的构造函数或者 `init` 方法里创建 private 数据  
```cpp
d = new FooPrivate();
```
并且在析构函数里 delete 掉  
```cpp
delete d;
```
当然，在很多时候，我们可能不想让D指针被修改，或者被复制导致我们失去了它的控制权，最后导致内存泄漏。所以很多时候我们会把D指针声明为 `const`，即  
```cpp
private:
    FooPrivate* const d;
```
这样就可以允许第三方去修改D指针指向的内容，但是不能修改这个指针的指向目标。  

当这样实现后，我们所有的数据操作都是通过class Foo 的成员方法来做，例如：  
```
QString Foo::string() const
{
    return d->s;
}

void Foo::setString( const QString& s )
{
    d->s = s;
}
```

从上面可以看到，D指针的实现形式其实也是把数据区域隐藏，只通过方法的调用形式来操作。这样当我们需要修改 Foo 成员变量，对于第三方来说是没有影响的，因为这个成员变量是在 FooPrivate 实例里。


## 引用  
1. [KDE - Policies/Binary Compatibility Issues With C++](https://community.kde.org/Policies/Binary_Compatibility_Issues_With_C%2B%2B)  
2. [stackoverflow - When do we break binary compatibility](https://stackoverflow.com/questions/37149479/when-do-we-break-binary-compatibility)  
3. [Wikipedia - Application binary interface (ABI)](https://en.wikipedia.org/wiki/Application_binary_interface)  
4. [stackoverflow - What is an application binary interface](https://stackoverflow.com/questions/2171177/what-is-an-application-binary-interface-abi)  
5. [HowTo: Export C++ classes from a DLL](https://www.codeproject.com/Articles/28969/HowTo-Export-C-classes-from-a-DLL)  

**和COM相关：**  

1. [MSDN - COM Objects and Interfaces](https://msdn.microsoft.com/en-us/library/ms690343\(v=vs.85\).aspx)  
2. [SO - COM(C++) programming tutorials?\[closed\]](https://stackoverflow.com/questions/2938435/comc-programming-tutorials)  
3. [博客园 - COM 入门(1)](http://www.cnblogs.com/zxjay/archive/2010/08/28/1811163.html)  

