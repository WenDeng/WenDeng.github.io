---
title: 《深度探索c++对象模型》（三）Data语意学
date: 2019-05-16 16:44:12
toc: true
comments: true
tags:
  - C++基础
categories:
  - C++基础
---

本章的的主题是Data语意学，主要是探究编译器对class中的Data member的绑定、布局和存储等操作，最后探究Data member存取和多种继承方式之间的效率关系，以及指向Data member的指针的效率问题。
<!--more-->

### 前述
> 本章的的主题是Data语意学，主要是探究编译器对class中的Data member的绑定、布局和存储等操作，最后探究Data member存取和多种继承方式之间的效率关系，以及指向Data member的指针的效率问题。

------------------------------------
参考书籍及链接：《深度探索c\+\+对象模型》    

------------------------------------
## 0、本章基础
#### 1. 空类对象的大小是多少？
```
class X { };//空类
```
对于空类，它有一个隐藏的1byte大小，那个被编译器安插进去的一个char,这使得这一class的两个objects得以在内存中配置独一无二的地址。

#### 2. class object的size会受到哪些因素的影响？
会影响class object的size的因素有如下三个，编译器：
* 1. 语言本身所造成的额外负担：当语言支持virtual base classes时，就会导致一些额外负担。需要一个指针，它或者指向virtual base class subobject,或者指向一个相关的表格，表格用于存储subobject地址或偏移值。
* 2. 编译器对于特殊情况所提供的优化处理：Virtual base class subobject的1 byte大小也会出现在derived class上。
* 3. Alignment（边界对齐）的限制：在大部分的机器上，聚合的结构体大小会受到alignment的限制，使他们能够更有效率地在内存中被存取。比如32机器字上就是4的整数倍。

#### 3. 各种类型data member的存放。
nonstatic直接放在class object之中。static data member放置在程序的一个global data segment中，不会影响个别class object的大小。无论class产生多少个object,甚至是0个，其static data members永远也只存在一份实例。**但是一个template classs的static data members的行为稍有不同。**


## 一、Data member的绑定
#### 1. member function取用的是global还是local data member?
当member funtion取用Data时，优先考虑member data,人们称这种情况为“member rewriting rule”，意思是对于member functions本身的分析，会直到整个class的声明都出现了才开始。在一个inline member function躯体之内的一个data member绑定操作，会在整个class声明之后才发生。
> 以前人们提倡两种程序设计风格，即将所有的data members放在class声明起始处，或者把所有的inline  function都放在class声明之外。就是为解决绑定问题，但这种情况在c++ 2.0之后已经解决了。 

#### 2. member function的argument list的情况又是怎么样的呢?
与取用data member不同的是，argument list中的名称还是会在它们第一次 遭遇时被适当地决议（resolved）完成。
```
typedef int length;

class Point3d{
public:
    void mumble(length val) { _val=val;} //length被决议为global
    length mumble() {return val;}
    // ...
private:
    typedef float length;//这样的声明将使先前的参考操作不合法
    length _val;
    // ...
};
```
> 虽然编译器能处理，但还是提倡一种防御性程序风格：即总是把“nested（嵌套的） type声明”放在class的起始处。    



## 二、Data member的布局
#### 1. Data  member是怎样被放置的？
关于data member的布局，记住以下三点：
* nonstatic data members在class object中的排列顺序和其被声明的顺序一样，任何中间介入的static data members都不会被放进对象布局之中。       
* C++ standard允许编译器将多个access sections(也就是private、public、protected等区段)之中的data members整体自由排列，不必在乎他们的出现在class中的声明顺序（连续的两个privata也算两个section）。 
* 编译器还可能会合成一些内部使用的data members，以支持整个对象模型，vptr就是这样的东西，当前所有的编译器都把它安插在每一个“内含virtual function之class”的object内。



## 三、Data member的存取
#### 1. 经由一个class object和一个指针存取data member，有重大差异吗？
答案是显然的，这跟data member的类型和class的继承等都有关系，分如下两种情况讨论：
* **data member 为 static**     
  static data members会被编译器提出于class之外，并被视为一个global变量(但只在class生命范围内可见)。每一个static data member只有一个实例，存放在程序的data segment之中，通过一个指针和通过一个对象来存取data member都是一样的。  
  
  若取一个static data member的地址，会得到一个指向其数据类型的指针，而不是一个指向其class member的指针，因为static member并不内含在一个class object之中。   
  
  如果有两个classes，每一个都声明了一个同名的static member，编译器就会暗中对每一个static data member编码(对于这种手法有个很美的名称：name-mangling)，以获得一个独一无的程序识别代码。

* **data member 为 nonstatic**    
  Nonstatic data members直接存放在每一个class object之中。只有经过class object才能存取它们（implicit 存取如this指针）。欲对一个nonstatic data member进行存取操作，编译器需要把class object的起始地址加上data member的偏移位置(offset)。     

  每一个nonstatic data member的偏移位置(offset)在编译时期即可获知，甚至如果member属于一个base class subobject(派生自单一或多重继承串链)也是一样的。因此，存取一个nonstatic data member，其效率和存取一个C struct member或一个nonderived class的member是一样的。
  
  但是如果该data member是一个virtual base class 的member,那么通过**指针**的存取速度会稍慢一点。（指针的真正class type 只有在执行器才真正确定）。  
  
  
## 四、“继承”与Data Member
> C++ standard未强制指定derived class members和base class members的排列顺序，理论上编译器可以自由安排之。在大部分编译器上头，base class members总是先出现，但属于virtual base class的除外。“继承”会对Data Member的布局有什么影响？接下来分四种情况进行讨论。

#### 1. 第一种情况：只要继承不要多态。
这种情况不会存储时间上的额外负担，由于base class和derived class的objects都是从相同的地址开始，其差异只在于derived object 比较大，用以容纳自建的nonstatic data members，把一个derived class object指定给base class 的指针或引用，并不需要编译器去调停或修改地址，可以提供了最佳执行效率。

#### 2. 第二种情况：加上多态。
加上virtual function接口后，弹性增加了，但也同时增加了空间和存取时间上的额外负担，如何取舍，视多态程序所带来的利益。可能带来的额外负担如下：
* 导入一个和virtual table ，用来存储它所声明的每一个virtual functions的地址。再加上一两个slots(type_info)。
* 在每一个class object中导入一个vptr,提供执行期的链接，使每一个object能够找到相应的virtual table。
* 加强constructor，使它能够为vptr设定初始值，让它指向class所对应的virtual table。
* 加强destructor，使它能够消抹“指向class 相关virtual table”的vptr。

#### 3. 第三种情况：多重继承。
对于单一继承，如果没有virtual function，那么编译器就不需要做其他工作;但如果base class没有virtual function而derived class有，并且vptr放在object首部，那么当把一个derived object转换为其base object时，就需要编译器对vptr进行调整。在既是多重继承又是虚拟继承的情况下，编译器的需要做的会更多。    
对一个多重派生对象，将其地址指定给“最左端(也就是第一个)base class的指针”，情况将和单一继承时相同，因为二者都指向相同的起始地址。至于第二个或后继的base class的地址指定操作，则需要将地址修改为：加上(或减去)介于中间的base class subobjects大小。比较需要注意的是，如果在取drived class object的地址时进行偏移计算时，若其为指针，就需要判断其是否为0，若为0则基类object的地址也应为0。**当然，这些都是编译器的工作，我们需要了解，但不需要自己去实现。**

> 如果要存取第二个(或后继)base class中的一个data member会是怎样的情况？需要付出额外的成本吗？ 不，members的位置在编译期就固定了，因此，存取members只是一个简单的offset运算，就像单一继承一样简单，不管是经由一个指针，一个reference或是一个object来存取。

#### 4. 第四种情况：虚拟继承。
虚拟继承的出现是为了避免多个相同base class subobject的出现，将其只保留一份，从而减少空间浪费。      
class如果含有一个或多个virtual base class subobjects将被分割为两部分：一个不变区域和一个共享区域。不变区域中的数据，总是能有固定的offset，这部分可以被直接存取，至于共享部分，所表现的就是virtual base class subobject ，这个部分数据，其位置因为每次派生操作而有变化，所以只能间接存取。
> 一般而言，virtual base class最有效的一种运用形式就是：一个抽象的virtual base class，没有任何data members。

#### 5、对象成员的效率
程序员如果只关心起程序效率，应该实际测试，不能光凭推论、常识判断或假设。
参考书籍作者所做的测试表明，虚拟继承所造成确实会严重影响data member的存取效率。

## 五、指向Data members的指针(Pointer to Data Members)
#### 1. 如果获取Data member的偏移值？偏移值应该为多少？
通过如（&Point3d::z）这样的操作可以获得data member的偏移值。实际测试表明所获得的offset比预想大1，这是为什么？实际上这样做的目的是为了区分一个“没有指向任何data member”的指针，和一个指向“第一个data member”的指针的情况。比如：
```
float Point3d::*p1 = 0;//“没有指向任何data member”的指针
float Point3d::*p2 = &Point3d::x;//指向“第一个data member”的指针

if(p1 == p2) //如何区分?
{
    cout << "p1 & p2 contain the same value --" ;
    cout << " they must address the same member!" << endl;
}
```
因此，不论编译器或使用者都必须记住，在真正使用该值以指出一个member之前，请先减掉1。

#### 2.“指向Member的指针”对数据的存取有什么影响？
无继承时，指向member的指针对数据的存取操作，首先需要计算offset-1,其次具体的object需要用offset计算地址，会极大地降低效率，但目前的一些编译器提供了对应的优化，可以使其像直接通过对象取值一下快速。     
有继承时，data member是直接放在class object中的，理论上不会影响代码的效率，但继承的使用会妨碍优化的效果。