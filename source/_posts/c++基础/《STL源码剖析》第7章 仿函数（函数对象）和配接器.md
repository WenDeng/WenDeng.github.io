---
title: 《STL源码剖析》第7章 仿函数（函数对象）和配接器    
date: 2019-5-22 18:50:12    
toc: true   
comments: true   
img:     
tags:
  - STL源码
  - 技术   
categories:   
  - C++基础
---
所谓的仿函数(functor)，是通过重载()运算符模拟函数形为的类。因此，这里需要明确两点：
- 1 仿函数不是函数，它是个类； 
- 2 仿函数重载了()运算符，使得它的对你可以像函数那样子调用。 

《Design Patterns》一书提到23个最普及的设计模式，其中对adapter样式的定义如下：**将一个class的接口转换为另一个class 的接口，使原本因接口不兼容而不能合作的classes，可以一起运作**。

<!--more-->

### 1、仿函数
仿函数是早期的命名，C++标准规定所采用的新名称是函数对象。**函数对象，如其名字一样是指一种具有函数特质的对象**。调用者可以像函数一样的使用这些对象，但必须重载operator()，先产生类对象的一个匿名对象，再调用相应的函数。

例如在很多STL算法中，都可以看到，我们可以将一个方法作为模板内的参数传入到算法实现中，例如sort的时候我们可以根据我们传入的自定义的compare函数来进行比较排序。解决办法是使用函数指针，或者是将这个“操作”设计为一个所谓的仿函数，再用这个仿函数生成一个对象，并用这个对象作为算法的一个参数。

**那为什么STL不使用函数指针而使用仿函数呢？**      
这主要是因为函数指针不能满足STL对抽象性的要求，**无法和STL的其他组件搭配**以产生更加灵活的效果。

### 2、仿函数的可配接性
仿函数灵活性的关键就在于仿函数。
STL仿函数应该有能力被函数适配器修饰，就像积木一样串接，然而，为了拥有配接能力，每个仿函数都必须定义自己的**函数参数类型和返回值类型**，就像迭代器如果要融入整个STL大家庭，也必须按照规定定义自己的5个相应的类型一样。下面主要讲解一元函数和二元函数的基本的相关设计。

#### 2.1 unary_function
用来呈现一元函数的参数类型和返回值类型，使用者实现对应的一元仿函数时只需继承这个类并进行事项即可。
```
Template<class Arg, class Result> //基类
Struct unary_function
{
    Typedef Arg argument_type;
    Typedef Result result_type;
}
 
//自定义的一元仿函数可以继承上类来获得类型定义
Template<class T>
Struct negate:public unary_function<T,T> //取反
{
    T operator()(const T& x)const {return –x;}
};
```

#### 2.2 binary_function
用来呈现二元函数的第一个参数类型、第二个参数类型和返回值类型。
```
Template<class Arg1, class Arg2, class Result>
Struct binary_function
{
    Typedef Arg1 first_argument_type;
    Typedef Arg2 second_argument_type;
    Typedef Result result_type;
};
 

Template<class T>
Struct plus: public binary_function<T,T,T>  //用法示例，加运算
{
    T operator()(const T& x, const T& y)const {return x+y ;}
};
```


### 3、几种仿函数
- 算术类仿函数     
 用于算术运算，包括加法：plus<T>，减法：minus<T>，乘法：multiplies<T>，除法：divides<T>，取模：modulus<T>，取反：negate<T>。

- 关系类仿函数     
用于进行关系运算，包括等于：equal\_to<T>，不等于：not\_equal\_to<T>，大于：greater<T>，大于等于：greater\_equal<T>，小于：less<T>，小于等于：less_equal<T>。

- 逻辑类仿函数     
提供几种逻辑运算，包括逻辑运算and：logical\_and<T>，逻辑运算or：logical\_or<T>，逻辑运算not：logical_not<T>。

- 证同(identity)、选择(select)、投射(project)   
证同用于返回本身；选择用于接受一个pair,返回第一个元素或第二个元素；投射传回第一参数，忽略第二参数或相反。


**来看一下他们的运用实例**：
```
#include<functional>
#include<iostream>
using namespace std;
 
int main()
{
	//算术类仿函数
	plus<int> plusobj;
	minus<int> minusobj;
	multiplies<int> mulobj;
	divides<int> divobj;
	modulus<int> modobj;
	negate<int> negobj;
   
    //除下面的使用方式外，还可以用临时对象调用
	cout << plusobj(3, 5) << endl; //8
	cout << minusobj(3, 5) << endl; //-2
	cout << mulobj(3, 5) << endl; //15
	cout << divobj(3, 5) << endl; //0
	cout << modobj(3, 5) << endl; //3
	cout << negobj(3) << endl; //-3

	//关系类仿函数，不等于：，大于：greater<T>，大于等于：greater_equal<T>，小于：less<T>，小于等于：less_equal<T>。
	equal_to<int> equal_to_obj; 
	not_equal_to<int> not_equal_to_obj;
	greater<int> greater_obj;
	greater_equal<int> greater_equal_obj;
	less<int> less_obj;
	less_equal<int> less_equal_obj;
 
	cout << boolalpha << equal_to_obj(3, 5) << endl; //false
	cout << not_equal_to_obj(3, 5) << endl;//true
	cout << greater_obj(3, 5) << endl;//false
	cout << greater_equal_obj(3, 5) << endl;//false
	cout << less_obj(3, 5) << endl;//true
	cout << less_equal_obj(3, 5) << endl;//true

	//逻辑类仿函数
	logical_and<int> logical_and_obj;
	logical_or<int> logical_or_obj;
	logical_not<int> logical_not_obj;
 
	cout << logical_and_obj(1, 0) << endl; //false
	cout << logical_or_obj(1, 0) << endl;//true
	cout << logical_not_obj(1) << endl;//false

	return 0;
}
```
### 4、配接器
配接器（Adapter）在STL组件的灵活组合运用功能上，**扮演着轴承、转换器的角色**，即将一个class的接口转换为另一个class的接口，使原本因接口不兼容而不能合作的classes，可以一起运作，它事实上**是一种设计模式**。

STL 主要提供如下三种配接器：
- （1）改变仿函数（functors）接口，称之为function adapter
- （2）改变容器（containers）接口，称之位container adapter
- （3）改变迭代器（iterators）接口者，称之为iterator adapter

#### 4.1 容器配接器container adapter
STL 提供的两个容器queue和stack，其实都不过是一种配接器，是对deque （双端队列）接口的修饰而成就自己的容器风貌。queue和stack底层都是由deque构成的，它们封住所有deque对外接口，只开发符合对应原则的几个函数，故它们是适配器，是一个作用于容器之上的适配器。

如果按照该标准衡量其他容器的话，序列式容器的 set 和 map 其实是对其内部所维护的RB-tree接口的改造。


#### 4.2 迭代器配接器iterator adapter
STL提供了许多应用于迭代器身上的配接器，包括**insert iterators，reverse iterators，iostream iterators**。

insert iterators可以将一般迭代器的赋值操作转变为插入操作。此迭代器包括专门从尾端插入操作back\_insert\_iterator，专门从头端插入操作front\_insert\_iterator，以及可从任何位置执行插入操作的insert\_iterator。因iterator adapters使用接口不是十分直观，STL提供三个相应的函数back\_inserter()、front\_inserter()、inserter()，从而提高使用时的便利性。

reverse iterators可以将一般迭代器的行进方向逆转，使原本应该前进的operator++变成了后退操作，使原本应该后退的operator–变成了前进操作。此操作用在“从尾端开始进行”的算法上，有很大的方便性。

iostream iterators可以将迭代器绑定到某个iostream对象身上。绑定到istream对象身上，为istream\_iterator，拥有输入功能；绑定到ostream对象身上，成为ostream_iterator，拥有输出功能。此迭代器用在屏幕输出上，非常方便。


#### 4.3 functor配接器functor adapter
functor adapters是所有配接器中数量最庞大的一个族群，其配接灵活度是后两者不能及的，可以配接、配接、再配接。其中配接操作包括系结（bind）、否定（negate）、组合（compose）、以及对一般函数或成员函数的修饰（使其成为一个仿函数）。它的价值在于，通过它们之间的绑定、组合、修饰能力，几乎可以无限制地创造出各种可能的表达式（expression），搭配STL算法一起演出。

由于仿函数就是“将function call操作符重载”的一种class，而任何算法接受一个仿函数时，总是在其演算过程中调用该仿函数的operator()，这使得不具备仿函数之形、却有真函数之实的“一般函数”和“成员函数（member functions）感到为难。**如果”一般函数“和“成员函数”不能纳入复用的体系中，则STL的规划将崩落了一角**。为此，STL提供了为数众多的配接器，使“一般函数”和“成员函数”得以无缝地与其他配接器或算法结合起来。

所有期望获取配接能力的组件，本身都必须是可配接的，即一元仿函数必须继承自unary\_function，二元仿函数必须继承自binary\_function，成员函数必须以mem\_fun处理过，一般函数必须以ptr\_fun处理过。

**一个未经ptr_fun处理过的一般函数，虽然也可以函数指针的形式传给STL算法使用，却无法拥有任何配接能力**。

```
// 以下配接器其实就是把一个一元函数指针包起来；
// 当仿函数被使用时，就调用该函数指针
template <class _Arg, class _Result>
class pointer_to_unary_function : public unary_function<_Arg, _Result> 
{
protected:
  _Result (*_M_ptr)(_Arg);     // 内部成员，一个函数指针
public:
  pointer_to_unary_function() {}
  // 以下constructor将函数指针记录于内部成员之中
  explicit pointer_to_unary_function(_Result (*__x)(_Arg)) : _M_ptr(__x) {}
  // 通过函数指针执行函数
  _Result operator()(_Arg __x) const { return _M_ptr(__x); }
};

// 辅助函数，使我们能够方便运用pointer_to_unary_function
template <class _Arg, class _Result>
inline pointer_to_unary_function<_Arg, _Result> ptr_fun(_Result (*__x)(_Arg))
{
  return pointer_to_unary_function<_Arg, _Result>(__x);
}

```

### 5、小结
本节主要介绍仿函数和配接器这两种思想，二者的主要的目的都是粘合STL的另外三大组件容器、迭代器和算法，理解其意，思而用之，也必能有所收益。