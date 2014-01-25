# Muduo源码阅读（3）：Muduo的回调函数机制
回调指将一段可执行的代码作为变量传给另外一部分代码，以供同步或异步调用[1]。在Reactor模式中，在事件到来时调用相应的处理函数就是一种异步回调的过程。回调函数的实现可以有各种各样的方式，我们最熟悉的就是C中的函数指针。考虑一个非常简单的例子，我们要实现对两个数的操作，这个操作可以是加减乘除等。首先声明一个函数指针：
```c
typedef int (operFunc*)(int a, int b);
```
这个类型的函数指针可以作为参数传给用于实现两个数操作的函数，这个函数仅仅是简单的调用传入的函数指针：
```c
int operation(int a, int b, operFunc func)
{
    return func(a, b);
}
```
现在我们就可以定义各种回调函数了：
```c
int add(int a, int b) 
{
    return a + b;
}

int minus(int a, int b)
{
    return a - b;
}

int multi(int a, int b)
{
    return a * b;
}

int divide(int a, int b)
{
    return a / b;
}
```
这样我们在调用operation()函数时可以根据传入回调函数的不同，实现不同的功能：
```
int main(void)
{
    int a = 4;
    int b = 2;
    // 加
     std::cout << operation(a, b, add) << std::endl;
    // 减
     std::cout << operation(a, b, minus) << std::endl;
    // 乘
     std::cout << operation(a, b, multi) << std::endl;
    // 除
     std::cout << operation(a, b, divide) << std::endl;
}
```
这当然是非常简单的一种回调函数使用。Muduo由于使用了Reactor模式，读、写、客户端连接到达服务器都是一种事件，当事件到达后调用我们事先注册的回调函数完成处理，这是一种异步回调的过程。

回调函数的实现可以有各种各样的方式。C/C++可使用函数指针；Python和JavaScript中函数可作为对象传递；函数式语言中有first-class functions；C++的STL通过函数对象与各种各样的算法结合等。而Muduo则是使用了boost库中的boost::function和boost::bind实现回调函数，这里可以参考作者陈硕的博文[2]。

boost::function类似于函数指针的封装。boost::function类型的变量可以保存一个可以调用的函数指针，它与函数指针的区别可以见官方文档[3]。我们之所以要使用boost::function，主要是由于它可以与boost::bind结合起来使用实现一些函数指针无法完成的功能。比如：
* 绑定特定变量到函数上，实现某些函数式编程语言的功能，甚至是真正的闭包。


以下代码来自[4]，其中同样的一个函数由于绑定了不同的变量而导致函数签名发生了变化，包括参数的个数以及顺序。

```cpp
#include "boost/function.hpp" 
#include "boost/bind.hpp" 
 
#include <string> 
#include <iostream> 
 
namespace 
{ 
  void function(int number, float floatation, std::string string) 
  { 
    std::cout << "Int: \"" << number << "\" Float: \"" << floatation 
              << "\" String: \"" << string << "\"" << std::endl; 
  } 
 
} // namespace 
 
int main(int c, char** argv) 
{ 
  // declare function pointer variables 
  boost::function<void(std::string, float, int)> shuffleFunction; 
  boost::function<void(void)> voidFunction; 
  boost::function<void(float)> reducedFunction; 
 
  // bind the methods 
  shuffleFunction = boost::bind(::function, _3, _2, _1); 
  voidFunction = boost::bind(::function, 5, 5.f, "five"); 
  reducedFunction = boost::bind(::function, 13, _1, "empty"); 
 
  // call the bound functions 
  shuffleFunction("String", 0.f, 0); 
  voidFunction(); 
  reducedFunction(13.f); 
 
} // main 
```
* 保存类成员函数的指针

在上文中我们举了对两个数完成任意操作的回调函数例子，这个回调函数能够成员函数吗？考虑下面类A的定义：
```cpp
class A
{
public:
    int add(int a, int b) {
        std::cout << "A::add()" << endl;
        return a + b;
    }
}
```
成员函数指针变量的定义必须显式指出其所属的类：
```cpp
typedef int (A::*operFunc)(int, int);
int main(void)
{
    operFunc oper = &A::add;
    A ca;
    int a = 2;
    int b = 3;
    int res = (ca.*oper)(a, b);
    std::cout << res << std::endl;
}
```
成员函数具有隐含的this指针，我们在调用成员函数时必须通过类的实例。所以成员函数指针变量不仅在定义时有特殊的语法，更重要的是在实际调用回调函数必须用(ca.*oper)(a, b);这样的形式。很明显这样的语法比较复杂，而且由于与非成员函数的声明不同，无法传入同一个函数。
那么有没有一种定义“函数指针”变量的方法，使调用函数既能接收成员函数又能接收非成员函数呢？上文中提到boost::bind可以将变量绑定到boost::function的参数上，所以结合bind和function机制，完全可以满足这样的需求。现在我们用bind、function重写上面对两个数完成任意操作的例子。
```
typedef boost::function<int(int, int)> operFunc;
int main(void)
{
    int a = 4;
    int b = 2;
    operFunc oper;
    A ca;
    oper = boost::bind(&A::add, &ca, _1, _2);
    // 加
    std::cout << operation(a, b, oper) << std::endl;
    // 减
     std::cout << operation(a, b, minus) << std::endl;
    // 乘
     std::cout << operation(a, b, multi) << std::endl;
    // 除
     std::cout << operation(a, b, divide) << std::endl;
}
```
我们仅仅是将回调函数的定义类型从函数指针替换为了boost::function，再通过bind将成员函数A::add()隐含的对象指针this绑定到了ca对象上，就可以完成和上文一样的功能。

boost::function和boost::bind还有针对重载函数、模板函数的用法，在Muduo中都没有涉及，这里就不展开说了，下面的系列文章写得很清楚，大家可以自行看一下：
1. [How To Use Boost.Function](http://www.radmangames.com/tutorials/programming/how-to-use-boost-function)
2. [How To Use Boost.Bind](http://www.radmangames.com/tutorials/programming/how-to-use-boost-bind)
3. [How To Use Function Pointers in C++](http://www.radmangames.com/programming/how-to-use-function-pointers-in-cplusplus)

这里再多扯两句，boost的function和bind结合起来使用实际上是实现了[闭包](http://www.ibm.com/developerworks/cn/linux/l-cn-closure/)的功能。所谓闭包，简单来说就是函数+引用环境，或者说是附有数据的行为。我们知道，在事件到来时调用回调函数处理，我们还需要知道其它的一些信息，在网络编程中可能是连接当前的状态等。`FIXME:纯用C函数指针的处理方式一般是将这些引用环境信息作为参数传入函数，这样我们必须`在Muduo中，一个回调函数通常是一个类的成员函数，所以这个回调函数同时也保存了某个类实例的信息，包括这个类的成员变量、成员函数等，我们在实际调用函数的时候就可以使用这些信息和状态，这样的处理方式令程序的编写显得非常的清晰和方便。

[1] http://en.wikipedia.org/wiki/Callback_(computer_programming)

[2] http://blog.csdn.net/solstice/article/details/3066268

[3] http://www.boost.org/doc/libs/1_55_0b1/doc/html/function/misc.html#idp99672040

[4] http://www.radmangames.com/tutorials/programming/how-to-use-boost-bind

