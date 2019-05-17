---
title: 《深度探索c++对象模型》（二）构造函数语意学
date: 2019-05-17 16:44:12
toc: true
comments: true
tags:
  - C++基础
categories:
  - C++基础
---

本章的的主题是构造函数语意学，主要是讨论constructor如何工作，以及它什么时候被合成，同时挖掘编译器对于“对象构造过程”的干涉，以及对于“程序形式”和“程序效率”上的冲击。
<!--more-->

### 前述
> 本章的的主题是构造函数语意学，主要是讨论constructor如何工作，以及它什么时候被合成，同时挖掘编译器对于“对象构造过程”的干涉，以及对于“程序形式”和“程序效率”上的冲击。

------------------------------------
参考书籍及链接：《深度探索c\+\+对象模型》    

------------------------------------
## 一、Default Constructor的构造操作
#### 1. 什么时候才会合成一个default construct呢？  
   答案是当编译器需要的时候，default constructor会被合成出来，只执行编译器所需要的任务。另外要注意程序的需要和编译器的需要之间的区别，如果程序有需要，那是程序员的责任，就需要自己实现constructor。     
   对于class X，如果没有任何user-declared constructor，那么会有一个default constructor被隐式(implicitly)声明出来...一个被隐式声明出来的default constructor将是一个trivial(浅薄而无能，没啥用的)constructor...  
   一个nontrivial default constructor在ARM的术语中就是编译器需要的那种，必要的话由编译器合成出来。下面4小节分别讨论nontrivial default constructor的4种情况

#### 2. 几种对象构建时的区别。
   Global objects的内存保证会在程序启动的时候被清0。Local objects配置于程序的堆栈中，heap objects配置于自由空间，都不一定会被清零，它们的内容将是内存上次被使用的遗迹。
  
#### 3. 第一种情况：“带有Default Constructor”的member class object
如果一个class没有任何constructor，但它内含一个member object，而后者有default constructor，那么这个class的implicit default constructor就是“nontrivial”，编译器为该class合成出一个default constructor。不过这个合成操作只有在constructor真正需要被调用时才会发生。

#### 4. 多成员对象的情况。
编译器的处理是：如果一个class A内含一个或者一个以上member class objects，那么class A的每一个constructor必须调用每一个member classes 的default constructor。编译器会扩张已存在的constructors,在其中安插一些代码，使得user code在被执行之前，先调用必要的default constructors。**调用顺序与member objects在class中的声明次序一致**。

#### 5. 第二种情况：“带有Default constructor”的base class。
如果一个没有任何constructors的class派生自一个“带有default constructor”的base class，那么这个derived class的default constructor会被视为nontrivial，并因此需要被合成出来。对于一个后继派生的class而言，这个合成的constructor和一个“被显式提供的default constructor”并没有差异。
> 注意一点，如果有constructor,但没有default constructor,那就会对每一个constructors进行扩充。如果亦存在Member Class Object，那些default constructor也会在base class constructor都被调用之后调用。

#### 6. 第三种情况：“带有一个Virtual Funtion”的class。
如果class声明(或继承)一个virtual function，编译器也需要合成出default constructor或扩充construtor。下面两个扩张行动会在编译期间发生：
* 一个virtual function table(在cfront中被称为vtbl)会被编译期产生出来，内放class的virtual functions地址。
* 在每一个class object中，一个额外的pointer member(也就是vptr)会被编译期合成出来，内含相关之class vtbl的地址。

> 编译器会为每一个含有virtual function的class objects的vptr进行适当的初始化，以放置适当的virtual table地址。

#### 7. 第四种情况：“带有一个virtual base class”的class。
如果class派生自一个继承串链，其中有一个或更多的virtual base classes编译器也需要合成出default constructor或扩充construtor。其目的在于必须使 virtual base class 在其每一个derived class object中的位置能够在执行期准备妥当。对于class所定义的每一个constructor。编译器都会安插那些“允许每一个virtual base class 的执行期存取操作”的代码。

#### 8. 总结。
除以上四种情况外，在没有声明constructor时就默认其是无用的， 其default constructor也就不会被合成出来的。     
在合成的default constructor中，只有base class subobjects和member class objects会被初始化。所有其他的nonstatic data member ，如整数，整数指针，整数数组等是不会被初始化的，这些初始化操作对程序是必须的，但对编译器则并非需要的。   
C++新手一般有两个误解：
* 任何class 如果没有定义default constructor ，就会被合成出来一个。
* 编译器合成出来的default constructor 会明确设定 class 内每一个data member的默认值。



## 二、Copy Constructor的构造操作
#### 1. 哪些情况需要有copy constructor？
有三种情况，会以一个object的内容作为另一class object的初值，即需要有 copy constructor。
* 1. 把一个object直接赋值给另一个object进行初值。
* 2. 当object被当做参数交给某个函数
* 3. 当函数返回一个class object。

> 一个class object可用两种方式复制得到，一种是被初始化，另一种是赋值。从概念上看，这两种操作分别是以copy constructor和copy assignment operator完成的。      
> Default constructors和copy constructor在**必要的时候**才由编译器 产生，这里的“必要”意指当class不展现bitwise copy sematics时。

#### 2. Default Memberwise Initialization
当class object以“**相同**的另一个object作为初值是，其内部是以所谓的default memberwise initialization方式完成的。也就是把每一个内建的或派生的data member（例如一个数组或指针）的值，从某个object拷贝一份到另一个object上，但不拷贝其具体内容。例如只拷贝指针地址，不拷贝一份新的指针指向的对象，这也就是**浅拷贝**，不过它并不会拷贝其中member class object，而是以递归的方式实行memberwise initialization。

#### 3. 递归的memberwise initialization是如何实现的呢？
答案就是Bitwise Copy Semantics和default copy constructor。如果class展现了Bitwise Copy Semantics，则使用bitwise copy（bitwise copy semantics编译器生成的伪代码是memcpy函数），否则编译器会生成default copy constructor。

#### 4. Memberwise copy(深拷贝)与Bitwise copy(浅拷贝)的区别
Memberwise copy: 在初始化一个对象期间,基类的构造函数被调用,成员变量被调用,如果它们有构造函数的时候,它们的构造函数被调用,这个过程是一个递归的过程。
Bitwise copy: 原内存拷贝。例子,给定一个对象object,它的类型是class Base。对象object占用10字节的内存,地址从0x0到0x9.如果还有一个对象objectTwo,类型也是class Base。那么执行objectTwo = object;如果使用Bitwise拷贝语义,那么将会拷贝从0x0到0x9的数据到objectTwo的内存地址，也就是说Bitwise是字节到字节的拷贝。

对于默认的拷贝构造函数不会使用深拷贝,它只是使用浅拷贝。这意味着类的所有的成员是一层深度的拷贝而已。如果你的类或结构体成员中只是包含基本的数据类型例如int, float, char,那么Memberwise copy与Bitwise copy基本是相同的。但如果类中有指针存在,那么你可能会遇到问题。    
例如下面的例子:
```
class A
{
   int m;
   double d;
   char *Str;
};

如果你创建两个这样的类对象,class A  a, b;并且你给a赋值,      
a.m = 6;   
a.d = 10.123;   
a.Str = new char[10];   
astrcpy(a.Str, "test");//这里是浅拷贝   
如果执行b = a;那么会把对象a的每一个成员的值赋值给b的每个成员。   
b.m = a.m;    
b.d = a.d;   
b.Str = a.Str;//现在对象a和b的成员Str都执向相同的内存,删除任一个内存都会析放另一个对象的内存。   
```
所以你需要深拷贝,它不是拷贝的内存地址而是拷贝内存地址的内容。一个默认的拷贝构造函数经常执行浅拷贝,只有拥有自己的拷贝函数才可以实现深拷贝。

#### 5. 什么时候一个class不展现出“bitwise copy semantics”呢？
有四种情况：
* 1. 当class内含有一个member class object，而这个member class内有一个默认的copy构造函数(不论是class设计者明确声明，或者被编译器合成)
* 2. 当class继承自一个base class，而base class有copy构造函数(不论显式声明或是被编译器合成]
* 3. 当一个类声明了一个或多个virtual 函数
* 4. 当class派生自一个继承串链，其中一个或者多个virtual base class


#### 6. 重新设定Virtual Table的指针（virtual funtion的情况）
当编译器导入一个vptr到class之中时，该class就不再展现bitwise semantics了。编译器需要合成出一个copy constructor，以求将vptr适当地初始化。          
当一个base class object以其derived class的object内容做初始化操作时，其vptr复制操作也必须要保证安全（非pointer和reference)。也就是说，合成出来的基类构造函数会显式设定object的vptr指向基类对应的virtual table，而不是直接将右手边的class object中将其vptr现值拷贝过来。

#### 7. 如何处理virtual base class subobject的情况？
virtual base class的存在需要特别处理。一个class object如果以另一个object作为初值，而后者有一个virtual base class subobject，那么也会使“bitwise copy semantics”失效。       
这时需要合成一个copy constructor,从而安插一些代码以设定virtualbase class pointer/offset的初值，对每一个members执行必要的memberwise初始化操作，以及执行其他的内存相关工作。



## 三、程序转化语意学(Program Transformation Semantics)
#### 1. class object的显式初始化操作。
初始化object时，必要的程序转化有以下两个阶段：
* 重写每一个定义，其中的初始化操作会被剥除，在c\+\+中，“定义”指占用内存的行为。
* class的copy constructor调用操作会被安插进去。

#### 2. 参数的初始化所做的程序转换。
C++ Standard说，把一个class object当做参数传给一个函数(或是作为一个函数的返回值)，相当于以下形式的初始化操作:
```
X xx = arg;//其中xx代表形式参数(或返回值)而arg代表真正的参数值

//因此，若已知如下函数：
void foo(X xo);
 
//转换的结果为：
X xx;
//xo以memberwise的方式将xx当作初值...
foo(xx);
```
有一种策略是导入所谓的临时性object，并调用copy constructor将它初始化，然后将此临时性object交给函数，临时性object会在函数结束处被析构。

#### 3. 返回值的初始化所做的程序转换。
函数bar()的返回值为一个对象，那该怎么把局部对象xx拷贝过来？ Stroustrup在cfront中的解决办法是一个双阶段的转化：
* 1. 首先加上一个额外参数，其类型是class object的一个reference，这个参数将被用来放置被“拷贝建构”而得的返回值。
* 2. 在return指令之前安插一个copy constructor调用操作，以便将欲传回之object的内容当做上述新增参数的初值。函数也对应变为void类型。

#### 4. 在编译器层面所做的优化。
编译器会以result参数取代name return val。这样的编译器优化操作，有时被称为Named Return Value(NRV)优化。NRV优化如今被视为是标准C++编译器的一个义不容辞的优化操作。**NRV需要一定的条件，即对应的类要有copy constructor**。     
一般而言，面对“以一个class object作为另一个class object的初值”的情形，语言允许编译器有大量的自由发挥空间。其优点当然是导致机器码产生时有明显的效率提升。缺点则是你不能安全地规划你的copy constructor的副作用，必须视其执行而定。
> NRV与返回值初始化的区别在于：NRV中不产生local object，直接以\_result带入其中进行各种处理，减少调用copy constructor。而返回值初始化则是在最后用copy constructor将local object的值拷贝给\_result, 中间不处理\_result。一个是优化，一个是程序转换。

#### 5. 那Copy Constructor要还是不要？
copy constructor的应用，迫使编译器多多少少对你的程序代码做部分优化。尤其当一个函数以传值(by value)的方式传回一个class object，而该class有一个copy constructor(不论是明确定义出来的，或是合成的)时。这将导致深奥的程序转化——不论在函数的定义或使用上，此外编译器也将copy constructor的调用操作优化，以一个额外的第一参数(数值被直接存放在其中)取代NRV。  
* 如果编译器能自动为你实施了最好的行为,那就没有必要实现一个自己的copy constructor。
* 如果class需要大量的memberwise初始化操作，例如以传值的方式传回object，此时提供一个explicit inline copy constructor就是非常合理的（在有NRV的前提下）。


## 四、成员们的初始化队伍(Memeber Initialization List)
#### 1. 在下列情况下，为了让你的程序能够顺利编译，你必须使用member initialization list:
* 当初始化一个reference member时
* 当初始化一个const member时
* 当调用一个base class的constructor，而它拥有一组参数时
* 当调用一个member class的constructor，而它拥有一组参数时

#### 2.member initialization list中到底会发生什么事情？
编译器会一一操作initialization list，以适当顺序在constructor之内安插初始化操作，并且在任何explicit user code之前。     
initialization list中的项目顺序是由class中的members声明顺序决定的，不是由initialization list中的排列顺序决定的。