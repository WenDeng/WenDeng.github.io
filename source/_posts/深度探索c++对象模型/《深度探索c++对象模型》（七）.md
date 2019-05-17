---
title: 《深度探索c++对象模型》（七）站在对象模型的顶端
date: 2018-11-22 16:44:12
toc: true
comments: true
tags:
  - C++基础
categories:
  - C++基础
---

本章的的主题是站在对象模型的尖端，套路哦三个著名的c\+\+语言扩充性质，它们都会影响c\+\+对象，他们分别是exception handling（EH）、template support、runtime type identification(RTTI)。
<!--more-->

------------------------------------
## 一、Template
这一节的焦点放在template的语意上面，我们将讨论templates在编译系统中“何时”，“为什么”以及“如何”发挥其功能。下面是有关template的三个主要讨论方向：
* template的声明，基本上来说就是当你声明一个template class、template class member function等等，会发生什么事情。
* 如何"实例化"class object、inline nonmember以及member template functions，这些是"每一个编译单元都会拥有的一份实体"的东西。
* 如何“实例化”出nonmember、member templates functions以及static template class members，这些都是"每一个可执行文件中只需要一份实体"的东西，这也就是一般而言template所带来的问题。

#### 1.template的“实例化”行为
对于如下template class:
```
template<class Type>
class Point
{
public:
    enum Status { unallocated, normalized };
    
    Point(Type x = 0.0, Type y = 0.0, Type z = 0.0);
    ~Point();
    
    void *operator new(size_t );
    void operator delete(void *, size_t );
    //...
private:
    static Point<Type> *freeList;
    static int chunkSize;
    Type _x, _y, _z;
};
```
编译器对于template class会根据type的不同而产生不同的实例class。
* enum Status、freeList、chunkSize以及object都必须进行实例化，如**Point<double>::freeList;**,而不能是**Point::freeList;**`。
* 定义一个指针，指向特定的实例。例如**Point<float> *ptr=0;**因为一个指向class object的指针，本身并不是一个class object，编译器不需要知道与该class有关的任何member的数据或object的布局数据。所以不需要实例化。
* 定义一个reference,例如**Point<float> &refer=0;**就需要产生一个Point的float实例。


#### 2.member function需要实例化吗？
member functions(至少对于那些未被使用过的)不应该被“实体”化，只有在member functions被使用的时候，C++ Standard才要求它们被“实例化”。当前的编译器并不精 确遵循这项要求，之所以由使用者来主导“具现”规则，有两个主要原因：
* 空间和时间效率的考虑。如果class中有100个member functions，但你的程序只针对某个类型使用其中两个，针对另一个类型使用其中5个，那么其他193个函数都“具现”将花费大量的时间和空间。
* 尚未实现的功能，并不是一个template实例化的所有类型就一定能够支持一组member functions所需要的所有运算符。如果只“具现”那些真正用到的memeber functions，template就能够支持那些原本可能会造成编译时期错误的类型(types)。

#### 3.template的错误报告。
目前的编译器，面对一个template声明，在它被一组实际参数实例化之前，只能施行以有限的错误检查。template中那些与语法无关的错误，程序员可能认为十分明显，编译器却让它通过了，只有在特定实例被定义之后，才能发出抱怨。这是目前实现技术上的一个大问题。

#### 4.Template中的名称决议法。
Template有两种语境，一种是C++ Standard所谓的"Scope of the template definition"，也就是“定义出template”的程序。另一种是C++ Standard所谓的"scope of the template instantiation"，也就是说“具现出template”的程序。        
Template之中，对于一个nonmember name的决议结果，是根据这个name的使用是否与“用以实例化该template的参数类型”有关而设定的。如果其使用互不相关，那么就以“scope of the template declaration”来决定name。如果其使用互有关联，那么就以“scope of template instantiation”来决定name。例如：
```
//scope of the template definition
extern double foo(double); 

template<class type>
class ScopeRules{
public:
    void invariant() { _member = foo(val); }
    type type_dependent() {return foo(_member);}
    //...
private:
    int _val;
    type _member;
};

//scope of the template instantiation
extern int foo(int);
//...
ScopeRultes<int> sr0;
```
* 对于**sr0.invariant();**由于被用来实例化这个template的真正类型，对于 \_val的类型并没有影响。所以选中**extern double foo(double);**
* 对于**sr0.type_dependent();**\_member与template参数有关,所以选中的foo()跟参数有关,所以选中**extern int foo(int);**。

#### 5.Member function的实例化行为。
对于template的支持，最困难的莫过于template function的实例化，目前的编译器提供了 两个策略：一个是编译时期策略，程序代码必须在program text file中备妥可用；另一个是链接时期策略，程序代码必须在meta-compliation工具可以导引编译器的实例化行为(instantiation)。       
下面是编译器设计者必须回答的三个主要问题：
* （1）编译器如何找出函数的定义？     
答案之一是包含template program text file，就好像它是个header文件一样，Borland编译器就是遵循这个策略。另一种方法是要求一个文件命名规则，例如，我们可以要求，在Point.h文件中发现的函数声明，其template program text一定要放置于文件Point.c或者Point.cpp中，以此类推。cfront就是遵循这个策略。Edison Desigin Group编译器对此两种策略都支持。
* （2）编译器如何能够只实例化出程序中用到的member functions?     
解决办法之一就是，根本忽略这项要求，把一个已经具现出来的class的所有member functions都产生出来。Borland就是这么做的——虽然它也提供#pragmas让你压制(或具现出)特定实体。另一种策略就是仿真链接操作，检测看看哪一个函数真正需要，然后只为它(们)产生实体。cfront就是这么做的，Edison Design Group编译器对此两种策略都支持。
* （3）编译器如何阻止member definitions在多个.o文件中都被实例化呢?       
解决办法之一是产生多个实体，然后从链接器中提供支持，只留下其中一个实体，其余都忽略。另外一个办法就是由使用者来导引“仿真链接阶段”的实例化策略，决定哪些实体(instances)才是所需求的。

实际上，template instantiation似乎拒绝全面自动化，甚至居然没意见工作都对了，产生出来的object files的重新编译成本仍然可能很高。**以手动方式先在个别的object module中完成预先实例化操作，虽然沉闷，却是唯一有效率的方法。**

## 二、异常处理
#### 1.编译对异常处理的支持
欲支持exception handling，编译器的主要工作就是找出catch子句，以处理被丢出来的exception。这多少需要追踪程序堆栈中的每一个函数当前作用区域(包括追踪函数中的local class objects当时的情况)。同时，编译器必须提供某种查询exception objects的方法，以知道其实际类型(这直接导致某种形式的执行期识别，也就是RTTI)。最后，还需要某种机制用以管理被丢出的object，包括它的产生、储存、可能的解构(如果有相关的destructor)、清理(clean up)以及一般存取，也可能有一个以上的objects同时起作用。      

一般而言，exception handling机制需要与编译器所产生的数据结构以及执行期的一个exception library紧密合作，在程序大小和执行速度之间，编译器必须有所抉择：
* 为了维持执行速度，编译器可以在编译时期建立起用于支持的数据结构，这会使程序大小膨胀，但编译器可以几乎忽略这些结构，直到有个exception被丢出来。
* 为了维持程序大小，编译器可以在执行期建立起用于支持的数据结构。这会影响程序的执行速度，但意味着编译器只有在必要的时候才建立那些数据结构(并且可以抛弃之)。

#### 2.Exception Handling 快速检阅
C++的exception handing由三个主要的语汇组件构成：
* 一个throw子句。它在程序某处发出一个exception。被抛出去的expection可以是內建类型，也可以是使用者自定类型。
* 一个或多个catch子句。每一个catch子句都是一个exception handler。它用来表示说，这个子句准备处理某种类型的exception，并且在封闭的大括号区段中提供实际的处理程序
* 一个try区段。它被围绕以一系列的叙述句(statements)，这些叙述句可能会引发catch子句起作用  

当一个exception被丢出去时，控制权会从函数调用中被释放出来，并寻找一个吻合的catch子句。如果都没有吻合者，那么默认的处理例程terminate()会被调用。当控制权被抛弃后，堆栈中的每一个函数调用也就被推离(popped up)，这个程序称为unwinding the stack。**在每一个函数被推离堆栈之前，函数的local class objects的destructor会被调用。**

#### 3.对Exception Handling的支持
当一个exception发生时，编译系统必须完成以下事情：    
（1）检验发生throw操作的函数；   
（2）决定throw操场是否发生在try区段中；   
（3）若是，编译系统必须把exception type拿来和每一个catch子句比较；   
（4）如果比较吻合，流程控制应该交到catch子句手中；    
（5）如果throw的发生并不在try区段中，并没有一个catch子句吻合，那么系统必须(a)摧毁所有active local objects，(b)从堆栈中将当前的函数"unwind"掉，(c)进行到程序堆栈中的下一个函数中去，然后重复上述步骤2~5


#### 4.当一个实际对象在程序执行时被丢出，会发生什么事？
当一个exception被丢出时，exception object会被产生出来并通常放置在相同形式的exception数据堆栈中，从throw端传染给catch子句的是exception object的地址、类型描述器(或是一个函数指针，该函数会传回与该exception type有关的类型描述器对象)，以及可能会有的exception object描述器(如果有人定义它的话)。


## 三、执行器类型识别（RTTI）
RTTI是用于支持EH而获得的副产品，主要目的是处理和识别throw的object类型。
#### 1.Type-Safe Downcast(保证安全的向下转型操作)
一个type\-safe downcast(保证安全地向下转换操作)必须在执行期对指针有所查询，看看它是否指向它所展现(表达)之object的真正类型。因此，欲支持type-safe downcast在object空间和执行时间上都需要一些额外的负担：
* 需要额外的空间以存储类型信息(type information)，通常是一个指针，指向某个类型信息节点
* 需要额外的时间以决定执行期的类型(runtime type)，因为，正如其名所示，这需要再执行期才能决定。

c\+\+的RTTI机制提供了一个安全的downcast设备,但只对那些展现“多态”的类型有效。c\+\+中，一个具备多态性质的class，正式内含着继承而来的virtual function。

#### 2.Type-Safe Dynamic cast(保证安全的动态转型)
dynamic\_cast运算符可以在执行期决定真正的类型。如果downcast是安全的，这个运算符会传回被适当转换过的指针。如果downcast不是安全地，这个运算符会传回0.

#### 3.References并不是Pointers
程序中对一个class指针类型施以dynamic_cast运算符，会获得true或false：
* 如果传回真正的地址，表示这个object的动态类型被确认了，一些与类型相关的操作现在可以施行于其上。
* 如果传回0，表示没有指向任何object，意味应该以另一种逻辑施行于这个动态类型未确定的object身上。

dynamic_cast运算符也适用于reference身上。然而对于一个non-type-safe cast，其结果不会与施行于指针的情况相同。为什么？      
一个reference不可以像指针那样"把自己设为0就代表了"no object"；若将一个reference 设为0，会引起一个临时性对象(拥有被参考到的类型)被产生出来，该临时对象的初值为0，这个reference然后被设定成为该临时性对象的一个别名。   
因此当dynamic_cast运算符施行于一个reference时，不能够提供对等于指针情况下的那一组true/false。取而代之的是，会发生下列事情：
* 如果reference真正参考到适当的derived class(包括下一层或下下一层或下下下一层或...)，downcast会被执行而程序可以继续执行。
* 如果reference并不真正是某一种derived class，那么，由于不能传回0，遂丢出一个bad_cast exception.

#### 4.Typeid运算符
typeid运算符传回一个const reference，类型为type_info。
type_info object由什么组成？ C++ Standard中对type_info的定义如下：
```
class type_info{
public:
    virtual ~type_info();
    bool operator==(const type_info& ) const;
    bool operator!=(const type_info& ) const;
    bool before(const type_info&) const;
    bool char* name() const;  //传回class原始名称
private:
    //prevent memberwise init and copy
    type_info(const type_info& );
    type_info& operator=(const type_info& );
    //data members
};
```
编译器必须提供的最小量信息是class的真实名称、以及在type_info objects之间的某些排序算法(这就是before()函数目的)、以及某些形式的描述器，用以表现explicit class type和这个class的任何subtype。

## 四、效率有了，弹性呢？
创痛的c\+\+对对象模型提供有效率的执行期支持。这份效率，再加上与c之间的兼容性，造成了C\+\+的广泛被接受。然而，在某些领域方面，像是动态共享库、共享内存以及分布式对象方面，这个对象模型的弹性还是不够。
