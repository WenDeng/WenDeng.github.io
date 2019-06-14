---
title: 《深度探索c++对象模型》（四）Function语意学
date: 2018-11-19 16:44:12
toc: true
comments: true
tags:
  - C++基础
  - 技术
categories:
  - C++基础
---

Function是c\+\+中的又一大重要部分， 本章的的主题是Function语意学，主要是探究编译器对class中的static member function、nonstatic member function和virtual member function所做的处理，并用实际测试分析其使用对代码效率的影响。同时也会进一步探究“指向member function”的指针和Inline function的原理和效率。
<!--more-->

------------------------------------
### 一、Member function的各种调用方式
#### 1.1 Nonstatic Member Function是怎么被调用的？
C\+\+的设计准则之一就是：nonstatic member function至少必须和一般的nonmember function有相同的效率。     
在c\+\+中，member function会被被编译器转化为nonmember function，然后执行期被调用，转化过程如下：
* 1. 改写函数原型，以安插一个额外的参数(this指针)到member function中，用以提供一个存取管道，使class object得以将此函数调用。
* 2. 将每一个“对nonstatic data member的存取操作”改为经由this指针来存取。
* 3. 将member function重新写成一个外部函数。将函数名称经过“mangling”处理，使它在程序中称为独一无二的词汇。

#### 1.2 编译器为什么要进行名称处理（name mangling）？怎么处理？
继承所带来的重复变量名、函数的重载等都需要编译器能唯一识别，这时候就需要就需要进行名称处理。一般而言，member的名称前面会加上class的名称，形成独一无二的命名，有时候member function的名称也需要加上参数类型等。    

#### 1.3 Virtual Member Functions是如何被调用的？
编译器内部会对virtual member function进行如下转换：
```
ptr->f();   //f()为virtual member function

（*ptr->vptr[1](ptr);//内部转化结果
```
* vptr表示编译器产生的指针，指向virtual table。  
* 1 是virtual table slot的索引值，关联到normalize()函数。   
* 第二个ptr表示this指针。  

#### 1.4 Static Member Functions有什么特性？如何被调用的？
static member functions的主要特性是它没有this指针。以下的次要特性统统根源于其主要特性：
* 它不能够直接存取其class中的nonstatic members
* 它不能够被声明为const、volatile或virtual
* 它不需要经由class object才被调用，虽然大部分时候它是这样被调用的。    

如果取一个static member function的地址，获得的将是其在内存中的位置，也就是其地址。由于static member function没有this指针，所以其地址的类型并不是一个“指向class member of function的指针”，而是一个“nonmember函数指针”。

### 二、Virtual Member function
#### 2.1 什么是多态？
C++中，多态表示以“一个public base class 的指针（或reference)，寻址出一个derived class object”。
> runtime type identification(RTTI)

#### 2.2 为了能方便class指针在执行期找到对应的函数实例，就需要编译器决定是否需要给class添加额外信息，那么，到底何时才需要这份信息？
答案是在必须支持某种形式之“执行期多态”的时候，要鉴定哪些classes展现多态特性，就需要额外的执行期信息。
识别一个class是否支持多态，唯一适当的方法就是看看它是否有任何virtual function。只要class拥有一个virtual function，它就需要这份额外的执行期信息。

#### 2.3 什么样的额外信息是我们需要存储起来的？
在实现上，编译器可以做到在每一个多态对象的class object身上增加两个members:     
* 一个字符串或数字，表示class的类型
* 一个指针，指向表格，表格中带有程序的virtual function的执行期地址。

#### 2.4 执行期如何找到对应的virtual function地址？
执行期要做的，只是在特定的virtual table slot中激活virtual function。这些active virtual function包括：
* 这一class所定义的函数实例。它会改写(overriding)一个可能存在的base class virtual function函数实例。
* 继承自base class的函数实例。这是在derived class决定不改写virtual function时才会出现的情况
* 一个pure\_virtual\_called()函数实例，它既可以扮演pure virtual function的空间保卫者角色，也可以当做执行期异常处理函数(有时候会用到)。  

每一个virtual function都被指派一个固定的索引值，这个索引在整个继承体系中保持与特定的virtual function的关系。执行期通过vptr和对应的slot获得对应的virtual function地址并进行调用。

> 在一个单一继承体系中，virtual function机制的行为十分良好，不但有效率而且很容易塑造出模型来。但是在多重继承和虚拟继承中，对virtual function的支持就没有那么美好了。

#### 2.5 多重继承下virtual function编译器需要做什么？
当把一个从heap中配置而得的Derived对象的地址，指定给一个Base2指针时，编译器需要如下处理：
```
Base2 *pbase2 = new Derived;

//编译器会做的处理
Derived *tmp = new Derived;
Base2 *pbase2 = tmp ? tmp + sizeof(Base1) : 0;//转移以支持第二个base class
```
当要删除pbase2所指的对象时，指针必须被再一次调整，以求再一次指向Derived对象的起始处(推测它还指向Derived对象)。然而上述的offset加法却不能够在编译时期直接设定，因为pbase2所指的真正对象只有在执行期才能确定。

#### 2.6 多重继承下virtual function带来的负担是什么？
在多重继承之下，一个derived class内含n-1个额外的virtual tables，n表示其上一层base classes的个数(因此，单一继承将不会有额外的virtual tables)。     
针对每一个virtual tables，Derived对象中有对应的vptr。vptrs将在constructor(s)中被设定初值。

#### 2.7 Thunk技术是什么？用来做什么？ 
offset的大小，以及把offset加到this指针上头的那一小段**程序代码**，必须经由编译器在某个地方插入。较有效率的解决办法是利用所谓的thunk。所谓thunk是以小段assembly代码，用来：
* (1) 以适当的offset值调整this指针
* (2) 跳到virtual function去。
Thunk技术允许virtual table slot继续内含一个简单的指针，因此多重继承不需要任何空间上的额外负担。Slots中的地址可以直接指向virtual function，也可以指向一个相关的thunk(如果需要调整this指针的话)。

#### 2.8 哪些情况，第二或后继的base class会影响对virtual functions的支持？
有以下三种情况，第二或后继的base class会影响对virtual functions的支持。
* 第一种情况是，通过一个"指向第二个base class"的指针，调用derived class virtual function。例如：
```
Base2 *ptr = new Derived;

//调用Derived::~Derived
//ptr指向Derived对象中的Base2 subobject；
//为了能够正确执行，ptr必须调整指向Derived对象的起始处。
delete ptr;
```
* 第二种情况是第一种情况的变化，通过一个“指向derived class”的指针，调用第二个base class中一个继承而来的virtual function。例如：
```
Derived *pder = new Derived;

//调用Base2::mumble()
//在此情况下，derived class指针必须再次调整，以指向第二个base subobject。
pder->mumble();
```
* 第三种情况发生于一个语言扩充性质之下：允许一个virtual function的返回值类型有所变化，可能是base type，也可能是publicly derived type。
```
Base2 *pb = new Derived;

//调用Derived * Derived::clone()
//当进行pb1->clone()时，pb1会被调整指向Derived对象的起始地址
//于是clone()的Derived版会被调用；
//它会传回一个指针，指向一个新的Derived对象，该对象的地址在被指定给pb2之前
//必须先经过调整，以指向Base2 subobject。
Base2 *pb2 = pb->clone();
```

#### 2.9 虚拟继承下virtual functions呢？
当一个virtual base class从另一个virtual base class派生而来，并且两者都支持virtual functions和nonstatic data members时，编译器对于virtual base class的支持简直就像进了迷宫一样。**不要在一个virtual base class中声明nonstatic data members**，否则你将距离复杂的深渊越来越近。

### 三、函数的效率
nonmemeber、static member或nonstatic member函数都被转换为完全相同形式，所以三者效率完全相同。      
导入virtual function之后，class constructor将获得参数以设定virtual table指针。所以每多一层继承，就会多增加一个额外的vptr设定。
> constructor的额外操作在多次调用的情况下可能会拖低效率，减少常用函数中的局部对象可以在一定程度上提高效率。

## 四、指向Member Function的指针
#### 4.1 指向nonstatic member function的指针是如何工作的？
取一个nonstatic data member的地址，如果该函数是nonvirtual，得到的结果是它在内存中真正的地址。然而这个值也是不完全的。它也需要被绑定于某个class object的地址上，才能够通过它调用该函数。所有的nonstatic member functions都需要对象的地址(以参数this指出)。
```
double  ( Point::*pmf)(); //member function的指针名
pmf = &Point::y; //获得对应的member function地址
(origin.*coord)(); //调用方式,origin是一个object,指针(ptr->*corrd)();
(coord)(&origin); //编译器内部转化
```

#### 4.2 指向nonstatic member function的指针会带来负担吗？
看情况，如果并不用于virtual function、多重继承、virtual base class等情况的话，并不会比使用一个“nonmember function指针”的成本高。
但上述三种情况对于“member function指针”的类型以及调用都太过于复杂。

#### 4.3 虚拟机制能在使用“指向member function的指针”的情况下运行吗？如果能，又是怎样实现的？
对一个nonstatic member function取其地址，将获得该函数在内存中的地址。然而面对一个virtual function，其地址在编译时期是未知的，取其地址所能获得的只是其在virtual table中的索引值。
```
&Point::x();  //x()为非虚函数，得其内存地址 
&Point::z();  //z()为虚函数，得其索引值

(*ptr->vptr[(int)pmf])(ptr);//pmf指向virtual函数时的调用方式
```
为了使pmf能支持上述两种情况，编译器必须定义函数指针使它能够(1)含有两种数值,(2)更重要的是其数值可以被区别代表内存地址还是virtual table中的索引值。

#### 4.4 在多重继承下，指向Member Functions的指针如何工作？
为了让指向member functions的指针也能够支持多重继承和虚拟继承，Stroustrup设计了下面一个结构体：
```
struct _mptr
{
    int delta; //delta字段表示this指针的offset值
    int index; //virtual table索引,不用时设为-1
    union{
        protofunc faddr; //nonvirtual member function地址
        int v_offset; //v_offset字段放的是一个virtual base class的vptr位置。
    };
};

(ptr->*pmf)();//原始调用
// 编译器转换
(pmf.index < 0) ? ( *pmf.faddr )( ptr) : (* ptr->vptr[pmf.index](ptr));
```
Microsoft就供应了三种风味，以减少不必要的字段：
* 1. 一个单一继承实例(其中带有vcall thunk地址或是faddr)
* 2. 一个多重继承实例(其中带有faddr和delta、vcall thunk地址)
* 3. 一个虚拟继承实例(其中带有四个members)

### 五、Inline Functions
#### 5.1 Inline Function有什么优点？
为了处理类内部数据，有时候会用friend function进行操作。然而如果我们将这些函数声明为inline，我们就可以保持直接存取members 的那种高效率，同时也能兼顾函数的封装性，此外，也不用再用friend。

#### 5.2 Inline Function什么时候被展开？
编译器会决定是否将Inline Functiong按照一个expression进行展开。处理一个inline函数，有两个阶段：
* 1. 分析函数定义，以决定函数的“intrinsic inline ability”。“intrinsic” (本质的，固有的)一词在这里意指“与编译器相关”，如果函数因其复杂度，或因其建构问题，被判断不可成为inline，它会被转为一个static函数，并在“被编译模块”内产生对应的函数语义。
* 2. 真正的inline函数扩展操作是在调用的那一点上。这会带来参数的求值操作(evaluation)以及临时性对象的管理。
同样在扩展点上，编译器将决定这个调用是否“不可为inline”。


#### 5.3 Inline Function如何处理形式参数？
扩展Inline function时，每一个形式参数都会被对应的实际参数取代。如果实际参数是一个常量表达式，我们可以在替换之前先完成其求值操作；后继的inline替换，就可以把常量直接“绑”上去。如果既不是常量表达式，也不是带有副作用的表达式，那么就直接替换之。
例如：
```
inline int bar(){
    int minval;
    int val1 = 1024;
    int val2 = 2048;
    
    minval = min(val1, val2);  /*(1)*/ 
    minval = min(1024, 2048);  /*(2)*/
    minval = min(foo(), bar() + 1); /*(3)*/
    return minval;
}
```
(1) 处形参无副作用，直接展开：
``` 
minval = val1 < val2 ? val1 : val2; 
```     
(2) 处那一行直接拥抱常量：
``` 
minval = 1024;  
```      
(3) 处那一行则引发参数的副作用，它需要导入一个临时对象，以避免重复求值:
```
int t1;
int t2;
minval = (t1 = foo()), (t2 = bar() + 1),t1 < t2 ? t1 : t2;
```
#### 5.4 Inline Function如何处理局部变量？
一般而言，inline函数中的每一个局部变量都必须被放在函数调用的一个封闭区段中，拥有一个独一无二的名称。
如果inline函数以单一表达式扩展多次，则每次扩展都需要自己的一组局部变量。如果inline函数以分离的多个式子被扩展多次，那么只需一组局部变量，就可以重复使用(译注：因为它们被放在一个封闭区段中，有自己的scope)

```
minval=min(val1,val2)+min(foo(),foo()+1);//这就是单一表达式，进行两次扩展，多出两组变量
```

#### 5.5 Inline Function的缺点。
一个inline函数如果被调用太多次，会产生大量的扩展码，使程序大小暴涨。参数带有副作用或者以一个单一表达式做多重调用、或者其本身有多个局部变量，都会产生大量局部变量，当然，编译器有可能帮你处理，也可能不会。     

**对于既要安全又要效率的程序，inline函数提供了一个强有力的工具。然而，与non-inline函数比起来，他们需要更加小心地处理。**
