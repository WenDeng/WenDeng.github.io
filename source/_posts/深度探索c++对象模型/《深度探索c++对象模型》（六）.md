---
title: 《深度探索c++对象模型》（六）执行期语意学
date: 2018-11-21 16:44:12
toc: true
comments: true
tags:
  - C++基础
  - 技术
categories:
  - C++基础
---

本章的的主题是执行器语意学，查看执期的某些对象模型行为。包括零时性对象的生命及其死亡，以及对new运算符和delete预算符的支持。
<!--more-->

------------------------------------
## 0、基础
实际上，一个简单的操作，其背后所隐藏的意义都要经过编译器进行适当的解析，编译器所作的工作可能会超出我们的想象许多，编译器所做的这些填充操作最后都会在执行期一一执行，本章就是要看执行期所发生的转换。

## 一、对象的构造和析构
#### 1. 关于构造和析构
构造通常在object被定义之后，而destructor要放在每一个离开点之前。一般而言我们会把object尽可能放置在使用它的那个程序区段附近，这么做可以节省非必要的对象产生操作和摧毁操作。
> 我们很多时候习惯把变量和object的定义放在函数的开始部分，这使得部分不必要的destructor不得不被调用，增加了部分开销。

#### 2. 全局对象
C++ 保证，一定会在main()函数中第一次用到global object之前，把它构造出来，而在main() 函数结束之前把global object摧毁掉。global object如果有constructor和destructor的话， 我们说它需要静态的初始化操作和内存释放操作。   
C++程序中所有的global objects都被放置在程序的data segment中。如果显式指定给它一个值， 此object 将以该值为初值。否则object配置到的内存内容为0。虽然class object在编译时期可以 被放置于data segment中并且内容为0，但constructor一直要到程序启动(startup)时才会实施。  

#### 3. 局部静态对象
关于局部静态对象，注意两点：    
（1）即使其所在的函数被调用多次，对应的constructor也只能执行一次。    
（2）即使其所在的函数被调用多次，对应的deconstructor也只能执行一次。    
为了能只执行一次对应的constructor，编译器引入一个临时性变量用于进行判断，初始时临时性变量为false，当local static object被构建好后，临时性变量变为true。而destructor则根据该 临时性是否为true决定是否析构local static object。

#### 4. 对象数组
C\+\+编译器之一cfront提供一个被命名为ve\_new()函数，产生出以class objects构造而 成的数组。在vec\_new()中，constructor施行于elem_count个元素之上;在vec_delete()中，destructor被施行于elem\_count个元素身上。
```
Point knots[10] = {Point(),Point(1.0, 1.0, 0.5),-1.0};
```
对于上述这种明显获得初值的元素，vec\_new()不再有必要。对于那些尚未被初始化的元素，vec_new()的施行方式就像面对“由class elements组成的数组，而该数组没有explicit initialization list”一样。类似下面这样：
```
//明确初始化前3个元素
Point::Point(&knots[0]);
Point::Point(&knots[1], 1.0, 1.0, 0.5);
Point::Point(&knots[2], -1.0, 0.0, 0.0);
//以vec_new初始化后7个元素
vec_new(&knots + 3, sizeof(Point), 7, &Point::Point, 0);
```

## 二、new和delete运算符
#### 1.new和delete对内置类型的处理。
```
int *pi = new int(5);
```
对于如上的语句，实际分为如下两个步骤完成：     
（1）通过适当的new运算符配置所需的内存。    
（2）给配置得来的对象设立初值。   
```
//实际执行过程
int *pi;
if(pi = _new(sizeof(int)))  *pi = 5; //成功了才初始化

//delete运算符的情况类似
if(pi != 0)   _delete(pi);
```
#### 2.construct如何配置一个class object？
以constructor来配置一个class object，处理类似如下：
```
Point3d *origin = new Point3d;
//被转换为：
Point3d *origin;
if(origin  = _new(sizeof(Point3d)))  origin = Point3d::Point3d(origin);

//对于delete origin，转换结果类似于：
if(origin != 0)   
{
   Point3d::~Point3d(origin);
   _delete(origin);
}
```
new运算符实际上总是以标准的C malloc()完成，虽然并没有规定一定得这么做不可。相同情况，delete运算符总是以标准的C free()完成。

#### 3.针对数组的new语意
* 对于像**int *p_array = new int[5];**这样的语句，vec\_new()不会真正被调用，因为它 的主要功能是把default constructor施行于class objects所组成的数组的每一个元素身上。
* 对于**simple_aggr *p_aggr = new simple_aggr[5];**,vec\_new()也不会被调用，因为simple\_aggr并没有定义一个constructor或destructor，所以配置数组以及清除p\_aggr数组的操作，只是单纯地获得内存和释放内存而已，这些操作由new和delete运算符来完成就绰绰有余了。

然而如果class定义有一个default constructo，某些版本的vec_new()就会被调用，配置并构 造class objects所组成的数组，如第一节中所示那样。  
* 寻找数组维度，对于delete运算符的效率带来极大的冲击，所以才导致这样的妥协：只有在中括号出现时，编译器才寻找数组的维度，否则它便假设只有单独一个objects要被删除。

#### 4.Placement Operator new的语意
有一个预先定义好的重载的(overloaded) new运算符，称为placement operator new，它需要第二个参数，类型为void*，调用方式如下：
```
Point2w *ptw = new(arena) Point2w;
```
其中arena指向内存中的一个区块，用以放置新产生出来的Point2w object。这个预先定义好的placement operator new的实现方法简直是出乎意料的平凡，它只要将“获得的指针”(上例为arena)所指的地址传回。

**用这个的意义是什么呢？**


## 三、临时性对象
#### 1.编译器什么时候产生临时性对象和摧毁临时性对象？
是否会导致一个临时性对象，视编译器的进取性以及程序上下语境而定。C\+\+ Standard允许编译器对于临时性对象的产生有完全的自由度。    
c\+\+标准指出，临时性对象的被摧毁，应该是对完整表达式求值过程中的最后一个步骤，该完整表达式造成临时性对象的产生。

#### 2.什么是完整表达式？
非正式地说，完整表达式是被涵括的表达式中最外围的那个。
```
((objA > 1024) && (objB > 1024) ? objA + objB : foo(objA, objB));
```
对于上述表达式，一共有五个子算式，内带一个"? : 完整表达式"中。任何一个子表达式所产生的任何一个临时对象，都应该在完整表达式被求值完成后，才可以毁去。

#### 3.关于临时性对象生命规则的的两个例外。
临时性对象的生命规则有两个例外：
* 第一个例外发生在表达式被用来初始化一个object时，C++ Standard要求说：凡含有表 达式执行结果的临时性对象，应该存留到object的初始化操作完成为止。
* 临时性对象的生命规则的第二个例外是"当一个临时性对象被一个reference绑定"时。如果一个临时性对象被绑定于一个reference，对象将残留，直到被初始化之reference的生命结束，或直到临时对象的生命范畴(scope)结束——视哪一种情况先到达而定。