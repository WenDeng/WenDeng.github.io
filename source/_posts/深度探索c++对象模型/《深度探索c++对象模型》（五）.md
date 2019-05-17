---
title: 《深度探索c++对象模型》（五）构造、析构、拷贝语意学
date: 2019-05-20 16:44:12
toc: true
comments: true
tags:
  - C++基础
categories:
  - C++基础
---

本章的的主题是构造、析构、拷贝语意学。主要是讨论如何支持class模型，探讨object的整个生命周期。
<!--more-->


### 前述
> 本章的的主题是构造、析构、拷贝语意学。主要是讨论如何支持class模型，探讨object的整个生命周期。

------------------------------------
参考书籍及链接：《深度探索c\+\+对象模型》    

------------------------------------
## 0、基础
#### 1. class data member应该何时被初始化？
一般而言，class的data member应该被初始化，并且只在constructor中或是在class的其他member functions中指定初值。其他任何操作都将破坏封装性质，使class的维护和修改更加困难。

#### 2. 关于纯虚函数的几点认识。
* c++中可以定义和调用(invoke)一个pure virtual function：不过它只能被**静态地调用**(用类名调用)，不能经由虚拟机制调用。
* class设计者如果声明就一定要定义pure virtual destructor，因为每一个 derived class destructor会被编译器加以扩展，以静态调用的方式调用其“每一个virtual base class”以及“上一层base class”的destructor。因此，只要缺乏 任何一个base class destructor的定义，就会导致链接失败。**最好的方式就是不要把virtual destructor声明为pure。**

#### 3. 关于虚拟机制的几点认识。
* 类中设计虚函数时应先考虑清楚，不会被derived class改写的函数最好被设计 为virtual function。总靠编译器进行优化并不是好的设计理念。
* 决定一个virtual function是否为const需要先想清楚，不必要的地方别用。

## 一、“无继承”情况下的对象构造
#### 1.对象的生命周期。
一个object的生命，是该object的一个执行期属性。local object的生命对应其所 在的scope。global object的生命和整个程序的生命相同。heap object的生命从 它被new运算符配置出来开始，到它被delete运算符摧毁为止。

#### 2.Plain OI' Data 和其相关处理
形如下列的结构，被C\+\+标准称为Plain OI' Data。
```
typedef struct{
    float x, y, z;
}Point;
```
* 如果以C++ 来编译这段码，理论上编译器会为Point声明一个trivial default constructor、一个trivial destructor、一个trivial copy constructor，以及一个trivial copy assignment operator。但实际上，编译器会分析这个声明，并为它贴上Plain of Data标签。
* 对于``` Point global; ```理论上,constructor在程序起始处被调用而destructor 在程序的exit()处被调用。然而，事实上那些tirvial members要不是没被定义， 就是没被调用，程序的行为一如它在C中的表现一样。此外，C++ 的所有全局对象都被当作“初始化过的数据”来对待。
* 对于``` Point *heap = new Point; ```会被转换为对new运算符的调用。但并没有default constructor施行与new运算符所传回的Point object身上。
* ```*heap = local; ```理论上，这样的指定操作会触发trivial copy assignment operator进行拷贝搬运操作。然而实际上此object是一个Plain old data，所以赋值操作(assignment)将只是像C那样的纯粹位搬移操作。
* ```delete heap; ```会被转换为对delete运算符的调用,观念上，这样的操作会触发Point的trivial destructor。但是一如我们所见，destructor要不是没有被产生就是没有被调用。

#### 3.抽象数据类型(Abstract Data Type)和其相关处理
以下是Point的第二次声明，在public接口之下多了private数据，提供完整的封装性，但是没有提供virtual function:
```
class Point{
public:
    Point(float x = 0.0, float y = 0.0, float z = 0.0): _x(x), _y(y),_z(y) { }
    //no copy constructor, copy operator or destructor defined
private:
    float _x, _y, _z;
};
```
* 对于Point，我们不需要定义一个copy constructor或copy assignment operator，因为默认的位拷贝已经足够，也不需要destructor,因为默认的内存管理方法也已经足够，如果我们不自己定义，编译器也因为判断不会用到而不会产生的函数。
* 对于```Point global;  ```default constructor作用于其上。由于global被定义在全局范畴中，其初始化操作将延迟到程序激活时才开始，扩展调用default constructor。如果要将class中的所有成员都设定常 量初值，那么给予一个explicit initialization list会比较有效率些 。
* 对于``` Point *heap = new Point; ```会被转换为对new运算符的调用。然后调用default Point Constructor并自行扩展。
* ```*heap = local; ```理论上，这样的指定操作会触发trivial copy assignment operator进行拷贝搬运操作。然而并没有，只进行简单的位拷贝操作。
* ```delete heap; ```，由于没有destrucor,同样不会被调用。

#### 4.在上述情况中加入虚函数又将怎么处理？
将人虚函数之后，class object除了多负担一个vptr之外，也引发编译器对Point class产生膨胀作用。例如：
```
class Point{
public:
    Point(float x = 0.0, float y = 0.0): _x(x), _y(y) { }
    //no destructor, copy constructor or copy operator
    virtual float z();
protected:
    float _x, _y;
};
```
* 首先constructor将需要附加一些代码用于将vptr初始化。这些代码位于base class构造函数和用户代码之间。
```
Point* Point::Point(Point *this, float x, float y): _x(x), _y(y)
{
    this->_vptr_Point = _vtbl_Point; //设定object的virtual table pointer
    this->_x = x; //扩展member initialization list
    this->_y = y;

    return this; //传回this对象
}
```
* 其次需要合成一个copy constructor和一个copy assignment operator，因为直接bitwise操作对于vptr可能是非法的。
```
//copy constructor的内部合成
inline Point* Point::Point(Point* this, const Point& rhs)
{
    this->_vptr_Point = _vtbl_Point;//设定object的vptr
    //将rhs坐标中的位连续拷贝到this对象
    //或是经由member assignment提供一个member...
    return this;
}
```
* 一般而言，如果你的设计之中有许多函数都需要以传值方式传回一个local class object，此时提供一个copy constructor就比较合理，它的出现会触发NRV优化。NRV 优化后就不再需要调用copy constructor，因为运算结果已经被直接置于“将被传回 的object”体内了。(有它->NRV->不用它？？？？)

## 二、继承体系下的对象构造
#### 1. 编译器会对constructor做什么？
像这样```T object ```定义一个对象时,会调用constructor,其内部做的工作包括：
* （1）记录在member initialization list中的data members初始化操作会被放进constructor的函数本身，并以members的声明顺序为顺序。
* （2）如果有一个member并没有出现在member initialization list中，但它有一个default constructor，那么该default constructor必须被调用。
* （3）在那之前，如果class object有virtual functions, 它们必须被设定初值，指向适当的virtual tables.
* （4）在那之前，所有上一层的base class constructors必须被调用，以base class生声明顺序为顺序(与member initialization list中的顺序没有关联)：
* * 如果base class被列于member initialization list中，那么任何明确指定的参数都应该被传递进去。、
* * 如果base class没有被列于member initialization list中，而它有default constructor(或default memberwise copy constructor),那么就调用之。
* * 如果base class是多重继承下的第二或后继的base class，那么this指针必须有所调整。
* （5）在那之前，所有virtual base class constructors必须被调用，从左到右，从最深到最浅
* * 如果class被列于member initialization list中，那么如果有任何显式指定的参数，都应该传递过去。若没有列于list之中，而class有一个default constructor，亦应该调用之
* * 此外，class中的每一个virtual base class subobject的偏移位置(offset)必须在执行期可被存取
* * 如果class object是最底层(most-derived)的class，其constructors可能被调用，某些用以支持这一行为的机制必须被放进来。

#### 2. 一个实例说明编译器在对象构造的过程中所做的操作。
有一个基类和其对应的派生类如下：
```
class Point
{
  public:
    Point(float x = 0.0, float y = 0.0);
    Point(const Point&);     //copy constructor
    Point& operator=(const Point&);   //copy assignment operator
    virtual ~Point();       //virtual destructor
    virtual float z() { return 0.0; }
  protected:
    float _x, _y; 
};
```
```
class Line
{
    Point _begin, _end;
  public:
    Line(float = 0.0, float = 0.0, float = 0.0, float = 0.0);
    Line(const Point&, const Point&);
    draw();
    //...
};
```
* （1）对于 ``` Line::Line(const Point& begin, const Point& end): _end(end), _begin(begin) {} ```,它会被编译器扩充并转换为：
```
Line* Line::Line(Line *this, const Point& begin, const Point& end){
    this->_begin.Point::Point(begin);
    this->_end.Point::Point(end);
    return this;
}
```
* （2）对于``` Line a;```implicit Line destructor会被合成出来(如果Line派生自Point,那么合成出来的destructor将会是virtual。然而由于Line只是内带Point objects而非继承自Point，所以被合成出来的destructor只是nontrivial而已)。在其中，它的member class objects的destructor会被调用(与其构造的相反顺序):
```
inline Line::~Line(Line *this){
    this->_end.Point::~Point();
    this->_begin.Point::~Point();
}
```

* (3) 对于``` Line b=a;```implicit Line copy constructor会被合成出来，成为一个inline public member; 
* (4) 对于``` a=b;``` 同样，implicit assignment operator会被合成出来，成为一个inline public member;

#### 3. 虚拟继承：constructor怎么处理virtual base class的构造？
试想下面三种类派生情况：
```
class Vertex : virtual public Point{ ... }
class Vertex3d : public Point3d, public Vertex{ ... }
class PVertex : public Vertex3d { ... }
```
Vertex的constructor必须调用Point的constructor。然而当Point3d和Vertex同为Vertetx3d的subobjects时，它们对Point constructor的调用操作一定不可以发生，取而代之的是，作为一个最底层的class，Vertex3d有责任将Point初始化，而更往后(往下)继承，则由PVertex来负责完成“被共享之Point subobject”的构造。     
对于Vertex3d，当调用Point3d和Vertex的constructor时，可以通过如下扩展，把\_most\_derived参数设为flase从而不调用Point的构造函数。
```
//在virtual base class情况下的constructor扩充内容
Point3d* Point3d::Point3d(Point3d* this, bool _most_derived, float x, float y, float z)
{
    if(_most_derived != false) this->Point::Point(x, y);
        
    this->_vptr_Point3d = _vtbl_Point3d;
    this->vptr_Point3d_Point = _vpbl_Point3d_Point;
    this->_z = rhs._z;
    return this;
}
```
> “virtual base class constructors的被调用”有着明确的定义：只有当一个完整的class object被定义出来时，它才会被调用；如果object只是某个完整object的subject，它就不会被调用。

#### 4. vptr初始化语意学：什么时候设置vptr合适？
constructor的执行算法通常如下：
* (1) 在derived class constructor中，“所有virtual base classes”及“上一层base class”的constructors会被调用
* (2) 上述完成之后，对象的vptrs被初始化，指向相关的virtual tables
* (3) 如果有member initialization list的话，将在constructor体内扩展开来。这必须在vptr被设定之后才做，以免有一个virtual member function被调用。
* (4) 最后，执行程序员所提供的代码。      

## 三、对象复制语意学(Object Copy Semantics)
#### 1. 怎样显式地拒绝将一个class object指定给另一个class object？
如果想要禁止将一个class object指定给另一个class object，那么只要将copy assignment operator声明为private,并且不提供其定义即可。

#### 2. 关于copy assignment operator。
对于编译器来说，class如果有了bitwise copy语意，implicit copy assignment copy就会被视为无用的，从而也不会被合并出来。     
一个class对于默认的copy assignment operator，在以下情况，不会表现出bitwise copy语意：
* （1）当class内含一个member object，而其class有一个copy assignment operator时
* （2）当一个class的base class有一个copy assignment operator时
* （3）当一个class声明了任何virtual functions(我们一定不要拷贝右端class object的vptr地址，因为它可能是一个derived class object)时
* （4）当class继承自一个virtual base class(不论base class有没有copy operator)时
> copy assignment operator需要考虑的是需不需要被合成？什么时候被合成？当多重继承遇到virtual base class共享时，如何避免中间base class对最上层base class的subobject的多重拷贝？          
**书籍作者的建议是不允许virtual base class的拷贝操作，尽量不要在任何virtual base class中声明数据。**


## 四、析构语义学(Semantics of Destruction)
#### 1. 什么时候需要合成destructor?
如果class没有定义destructor，那么只有在class内含的member object或base class拥有destructor的情况下，编译器才会自动合成一个出来。否则，destructor被视为不需要，也就不需被合成。
> 事实上，我们应该拒绝那种被我们称为“对称策略”的奇怪想法：“你已经定义了一个constructor,所以你应该提供一个destructor也是天经地义的事”。我们应该因为“需要”而非“感觉”来提供destructor,更不要因为你不确定是否需要一个destructor，于是就提供它。（取自作者原话）

#### 2. 如果没有destructor,编译会在需要时自动合成，那如果有destructor,编译器又是怎么进行扩展的呢?
一个由程序员定义的destructor被扩展的方式类似constructors被扩展的方式，但顺序相反：
* （1） destructor的函数本体现在被执行，也就是说vptr会在程序员的代码执行前被重设(reset)
* （2）如果object内含一个vptr，那么首先重设(reset)相关的virtual table
* （3）如果class拥有member class objects。而后者拥有destructors，那么它们会以其声明的顺序的相反顺序被调用
* （4）如果有任何直接的(上一层)nonvirtual base classes拥有destructors，它们会以其声明顺序的相反顺序被调用
* （5）如果有任何virtual base classes拥有destructor，而目前讨论的这个class是最尾端(most-derived)的class，那么它们会以其原来的构造顺序的相反顺序被调用。

就像constructor一样，目前对于destructor的一种最佳实现策略就是维护两份destructor实体：
* 一个complete object实例，总是设定好vptr(s)，并调用virtual base class destructors。
* 一个base class subobject实例；除非在destructor函数中调用一个virtual function，否则它绝不会调用virtual base class destructors并设定vptr。

一个object的生命结束于其destructor开始执行之时。由于每一个base class constructor都轮番被调用，所以derived object实际上变成了一个完整的object。例如一个PVertex对象归还其内存空间之前，会依次变成一个Vertex3d对象、一个Vertex对象、一个Point3d对象，最后成为一个Point对象。当我们在destructor中调用member functiions时，对象的蜕变会因为vptr的重新设定而受到影响。
