---
title: c++11特性之std-move的使用和原理    
date: 2019-5-14 11:20:12   
toc: true   
comments: true   
tags:
  - C++11特性
  - 技术   
categories:   
  - C++基础
---

面试经常遇到std::move，同时也确实存在一些知识点需要整理，在这里总结一下。
<!--more-->

### 1、左值和右值
左值与右值的根本区别在于是否允许取地址&运算符获得对应的内存地址。

一般来说，变量可以取地址，所以是左值，但是常量和临时对象等不可以取地址，所以是右值。

**左值的声明符号为&，右值的声明符号为&&。**

### 2、右值引用的作用与移动语义
我们可能在各种场合（初始化,push_back,函数返回等）调用拷贝构造函数将一个临时对象初始化给另一个对象，而这时如果是深拷贝则代价会比较大。

深拷贝对程序的影响比较大，把临时对象的内容直接移动（move）给被赋值的左值对象，效率改善将是显著的。这就产生了移动语义，**右值引用是用来支持转移语义的**。

移动语义可以将资源（堆，系统对象等） 从一个对象转移到另一个对象，这样能够减少不必要的临时对象的创建、拷贝以及销毁，能够大幅度提高 C++ 应用程序的性能。

### 3、移动语义的实现
移动语义意味着两点：
- **原对象不再被使用**，如果对其使用会造成不可预知的后果。
- 所有权转移，资源的所有权被转移给新的对象。

移动语义通过**移动构造函数**和**移动赋值操作符**实现，其与拷贝构造函数类似，区别如下：
- 参数的符号必须为右值引用符号，即为&&。
- 参数不可以是常量，因为函数内需要修改参数的值
- 参数的成员转移后需要修改（如改为nullptr），避免临时对象的析构函数将资源释放掉。

下面来一个实例：
```
#include <iostream>
#include <algorithm>
#include <vector>
#include <memory>

using namespace std;

class Test
{
public:
    Test(){};
    Test(Test &&test) //移动构造函数
    {
        std::cout << "Move Constructor" << std::endl;
        m_p=test.m_p;
        test.m_p = nullptr; //修改参数的资源
    }
    Test &operator=(Test &&test) //移动赋值操作符
    {
        std::cout << "Move Assignment operator" << std::endl;
        if (this != &test)
        {
            m_p = test.m_p;
            test.m_p= nullptr; //修改参数资源
        }
        return *this;
    }
    
private:
    int *m_p;
};

int main()
{
    std::vector<Test> vec;
    vec.push_back(Test()); //移动构造函数
    Test foo = Test(); //注意.....这里是拷贝构造函数...但是为什么？？？
    foo = Test();      //移动赋值操作符
    return 0;
}
```
> 如果没有实现移动构造函数，则默认调用拷贝构造函数，拷贝方式为浅拷贝。但是对于数组等还会有清空处理...

### 4、std::move的实现
std::move除了能实现右值引用，同时也能实现对左值的引用。在左值上使用移动语义。

std::move的实现主要依赖于static<T&& >，但同时也会做一些参数推导的工作。其实现如下：

```
template<typename T>
typename remove_reference<T>::type&& move(T&& t)
{
    return static_cast<typename remove_reference<T>::type &&>(t);
}
```

#### 4.1 对于右值
有如下代码：
```
std::move(string("dengwen"));
```
首先模板类型推导确定T的类型为string，得remove_reference<T>::type为string，故返回值和static的模板参数类型都为string &&;而move的参数就是string &&,于是不需要进行类型转换直接返回。

#### 4.2 对于左值
> 引入一条规则：当将一个左值传递给一个参数是右值引用的函数，且此右值引用指向模板类型参数(T&&)时，编译器推断模板参数类型为实参的左值引用。

有如下代码：
```
string str("dengwen");
std::move(str);
```
此时明显str是一个左值，首先模板类型推导确定T的类型为string &，得remove_reference<T>::type为string。故返回值和static的模板参数类型都为string &&;而move的参数类型为string& &&，**折叠**后为sting &。

所以结果就为将string &通过static_cast<string &&>转为string &&。返回string &&。

#### 4.3 引用折叠
如果间接的创建一个引用的引用，则这些引用就会“折叠”。如：     
- X& &、X& &&、X&& &都折叠成X&
- X&& &&折叠为X&&




### 5、测试实例
下面我们写一个完整的测试程序来测试std::move在各种场景下的运用和结果：

```
#include <deque>
#include <iostream>
#include <algorithm>
#include <vector>
#include <memory>
#include <sstream>
#include <string>
#include <queue>
using namespace std;

//std::move进行右值引用，可以将左值和右值转为右值引用， 这种操作意味着被引用的值将不再被使用，否则会引起“不可预期的结果”。

class Base
{
public:
    Base(int k)
    {
        p=new int(1);
        q=*p=k;
    }

    ~Base()
    {
        delete p;
    }
    void show()
    {
        cout<<"q address: "<<&q<<endl;
        cout<<"p address: "<<p<<endl;
    }
private:
    int q;
    int *p;
};

int main()
{
    cout << endl << "常规变量-----------------------------------------" << endl;
    int k = 6, s = 7;
    cout << k << " " << s << endl;
    k=std::move(s);
    cout << k << " " << s << endl;
    k=8;
    cout<<k<<endl;

    cout << endl << "常规数组（自动清空）-----------------------------------------" << endl;
    vector<int> data1 = {1, 2};
    vector<int> data2 = {1, 3, 4, 5, 4, 3, 5, 2};
    data1 = std::move(data2);
//     data1 = static_cast<vector<int> &&>(data2);
    cout<< "after move:" << endl<< "data1:";
    for (int foo : data1)  cout << foo << " ";

    cout << endl<< "data2:";
    for (int foo : data2)  cout << foo << " ";

    cout<< endl << endl << "指针变量-----------------------------------------" << endl;
    int m = 3, n = 5;
    int *p = &m, *q = &n;
    p = std::move(q);
    cout<<p<<"       "<<*p<<endl;
    cout<<q<<"       " << *q << endl;

    cout << endl << "class 对象-----------------------------------------" << endl;
    Base ba(5);
    Base bb(2);
    bb=std::move(ba);
    ba.show();
    bb.show();

    cout << endl << "string（自动清空） -----------------------------------------" << endl;
    string str=std::move("deng wen");
    string str1("luo chao");
    str1=std::move(str);
    cout<<&str<<"   "<<str<<endl;
    cout<<&str1<<"   "<<str1<<endl;

    cout << endl << "vector（自动清空） -----------------------------------------" << endl;
    vector<int> vec1={1,2,3,4,5};
    vector<int> vec2={9,0,9};
    vec1=std::move(vec2);
    for (int foo : vec1)  cout << foo << "  ";cout<<endl;
    for (int i=0;i<vec1.size();i++)  cout << &vec1[i] << "   ";
    cout<<endl;

    cout << endl << "&&的真正含义--------------------------------------" << endl;
    int ta = 3; // int &&tb=2;//临时对象的引用，即tb存的是临时对象2的地址
    int tb = 2; //生成对象tb,并将2赋值给tb所指的地址中。 感觉两种的结果一样，但是含义不一样
    cout << &ta << "  " << ta << endl;
    cout << &tb << "  " << tb << endl;
    cout << endl;
    int tc = 1;
    ta = tc;
    tb = tc;
    cout << &ta << "  " << ta << endl;
    cout << &tb << "  " << tb << endl;

    return 0;
}
```

