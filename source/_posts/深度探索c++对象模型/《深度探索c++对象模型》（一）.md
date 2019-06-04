---
title: 《深度探索c++对象模型》（一）关于对象
date: 2018-11-16 16:44:12
toc: true
comments: true
tags:
  - C++基础
  - 技术
categories:
  - C++基础
---

看完《深度探索c\++对象模型》，心中对c\++编译器在编译期间所做的处理有了更深入的认识，我想，除了对编译器本身有深入认识的作者之外，应该很少有人对c\++的对象模型有这么深的认识。能接触了这本书，是我们的幸运，是作者让我们有机会能一窥其貌，感谢作者。

其实第一遍读这本书，我的收获还不算多，这可能是我对c++的使用还不够多的缘故，但通过这本书，我以后使用c\++的时候，就会心里有更多的底气，也会有更多需要注意的地方，在经过更多的实践之后，我一定还会回来拜读这本书的。

现在，我想就本书所学到的的知识做一些总结。   
<!--more-->

------------------------------------
### 一、关于对象
#### 1.1 C++在加入封装后(只含有数据成员和普通成员函数）的布局成本增加了多少？  
答案是并没有增加布局成本。就像C struct一样，memeber functions虽然含在class的声明之内，却不出现在object中。每一个non-inline member function只会诞生一个函数实体。至于每一个“拥有零个或一个定义的” inline function则会在其每一个使用者(模块)身上产生一个函数实体。

#### 1.2 C++在布局以及存取时间上主要的额外负担是由virtual引起的，包括： 
* virtual funciton机制，用以支持一个有效率的“执行期绑定”
* virtual base class，用以实现“多次出现在继承体系中的base class，有一个单一而被共享的实体”



### 二、C++ 对象模式(The C++ Object Model)
#### 2.1 在C++中，有两种class data members：static 和 nonstatic，以及三种class member functions：static、nonstatic和virtual。

#### 2.2 C++对象模型中，nonstatic data members被配置于每一个class object之内。
static data members则被存放在所有的class object之外。static和nonstatic function members也被放在所有的class object之外。virtual function则以两个步骤支持之：
* 1. 每个class产生出一堆指向virtual functions的指针，放在表格之中。这个表格被称为virtual table(vtbl)
* 2. 每一个class object被安插一个指针，指向相关的virtual table。通常这个指针被称为vptr。vptr的设定和重置都由每一个class的constructor、destructor和copy assignment运算符自动完成。每一个class所关联的type_info object(用以支持runtime type identification, RTTI)也经由virtual table被指出来，通常放在表格的第一个slot处。  
 
这个模型的主要优点在于它的空间和存取时间的效率。  
主要缺点是：如果应用程序代码未曾改变，但所用到的class objects的nonstatic data members有所修改(有可能是增加、移除或更改)，那么应用程序代码同样得重新编译。

#### 2.3 继承关系可以指定为虚拟(virtual，也就是共享的意思)：
在虚拟继承的情况下，base class不管在继承链中被派生(derived)多少次，永远只会存在一个实例(称为subobject)。



### 三、关键词带来的差异
#### 3.1 什么时候一个人应该在c++程序中以struct取代class?
答案之一是当他让人感觉比较好的时候。单独来看，关键词本身并不提供任何差异，c\+\+编译器对二者都提供了相同支持，我们可以认为支持struct只是为了方便将c程序迁移到c\+\+中。

#### 3.2 那为什么我们要引入class关键词？
这是因为引入的不只是class这个关键词，更多的是它所支持的封装和继承的哲学。

#### 3.3 怎么在c++中用好struct？
将struct和class组合起来，组合，而非继承，才是把c和c++结合在一起的唯一可行的方法。另外，当你要传递“一个复杂的class object的全部或部分”到某个c函数去时，struct声明可以将数据封装起来，并保证拥有与c兼容的空间布局。

### 四、对象的差异
#### 4.1 C++程序设计模型直接支持三种程序设计典范（programming paradigms）：
* 程序模型：数据和函数分开。 
* 抽象数据类型模型：数据和函数一起封装以来提供。
* 面向对象模型：可通过一个抽象的base class封装起来，用以提供共同接口，需要付出的就是额外的间接性。
> 虽然你可以直接或间接处理继承体系中的一个base class object,但只有通过pointer或reference的间接处理，才支持OO程序设计所需的多态性质。**c\+\+通过class的pointers和reference来支持多态，这种程序设计风格就称为面向对象**

```
Liberary_materials thing1;//基类
Book book;//派生类
thing1=book;
thing1.check_in();//这种情况下，调用的是基类的check_in()
```

```
Liberary_materials &thing2=book
thing2.check_in();//这种情况下调用的才是book的check_in()
```
#### 4.2 C++以下列方法支持多态：
* 1. 经由一组隐式的转化操作。例如把一个derived class指针转化为一个指向其public base type的指针
```
    shape *ps=new circle();
```
* 2. 经由virtual function机制
```
    ps->rotate();
```
* 3. 经由dynamic_cast和typeid运算符
```
    if(circle *pc=dynamic_cast<circle *>(ps))...
```
> **多态的主要用途是经由一个共同的接口来影响类型的封装**，这个接口通常被定义在一个抽象的base class中。这个共享接口是以virtual function机制引发的，它可以在执行期根据object的真正类型解析出到底是哪一个函数实体被调用。


#### 4.3 需要多少内存才能表现一个class object?
* 其nonstatic data members的总和大小
* 加上任何由于aliginment的需求而填补上去的空间(可能存在于members之间，也可能存在于集合体边界),aliginement就是将数值调整到某数的倍数，如在32位的计算机上为4。
* 加上为了支持virtual而由内部产生的任何额外负担

#### 4.4 一个指针(引用)，不管它指向哪一种数据结构，指针本身所需的内存大小是固定的(一个机器字)。
例如：一个指向ZooAnimal的指针是如何地与一个指向整数得指针或一个指向template Array的指针有所不同的呢？
```
ZooAnimal *px;
int *pi;
Array<string> *pta;
```
以内存需求的观点来说，没有什么不同！它们三个都需要足够的内存来放置一个机器地址(通常是个word)。**“指向不同类型的各指针”间的差异，既不在其指针表示法不同，也不在其内容(代表一个地址)不同，而是在其所寻址出来的object类型不同**，也就是说，“指针类型”会教导编译器如何解释某个特定地址中的内存内容及其大小。

#### 4.5 转型(cast)其实是一种编译器指令。
大部分情况下它并不改变一个指针所含的真正地址，它只影响“被指出之内存大大小和其内容”的**解释方式**。
> 如一个类型为void *的指针只能够持有一个地址，但不能 通过它操作所指object。

#### 4.6 一个基类指针和其派生类指针有什么不同？（单一一层继承，且其都指向派生类对象）
二者都指向基类对象的第一个byte,其间的差别是，派生类指针涵盖的地址包含整个派生类对象，而一个基类指针所涵盖的地址只包含派生类对象的基类子对象部分。
> 但基类指针可以通过virtual机制访问派生类对象的函数。

#### 4.7 当一个base class object被直接初始化为(或被指定为)一个derived class object时。 
derived object就会被切割(sliced)以塞入较小的base type内存中，derived type将没有留下任何蛛丝马迹。多态于是不再呈现，而一个严格的编译器可以在编译器解析一个“通过此object而触发的virtual function调用操作”，因而回避virtual机制。如果virtual function被定义为inline，则更有效率上的大收获。

#### 4.8 C++也支持具体的ADT程序风格，如今被称为object-based(OB)。
一个OB设计可能比一个对等的OO设计速度更快而且空间更紧凑。速度快是因为所有的函数调用操作都在编译时期解析完成，对象构建起来时不需要设置virtual机制。空间紧凑是因为每一个class object不需要负担传统上为了支持virtual机制儿需要的额外负荷。不过，**OB设计比较没有弹性。**     
在弹性（OO）和（OB）之间常常存在着取舍。一个人能够有效选择其一之前，必须先清楚了解两者的行为和应用领域的需求。