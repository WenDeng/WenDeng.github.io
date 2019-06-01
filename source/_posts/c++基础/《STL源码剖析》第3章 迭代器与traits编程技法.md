---
title: 《STL源码剖析》第3章 迭代器与traits编程技法    
date: 2019-5-30 22:50:12    
toc: true   
comments: true   
img: https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/7.png?raw=true   
tags:
  - STL源码
  - 技术   
categories:   
  - C++基础
---
本节主要介绍迭代器和traits编程技法。

迭代器是一种模板class，迭代器在STL中得到了广泛的应用，通过迭代器，容器和算法可以有机的粘合在一起，只要对算法给予不同的迭代器，就可以对不同容器进行相同的操作。此外，迭代器本身也是一种设计模式，其设计思想也值得我们仔细体会。

traits编程技法主要是通过模板的类型推导和class内嵌类型定义从而获取并使用迭代器的所指对象的真正类型，比如作为返回值等。

<!--more-->


### 1、迭代器
什么是迭代器呢？迭代器有什么用呢？

**迭代器模式**的定义：提供一种方法，在不需要暴露某个容器的内部表现形式情况下，使之能依次访问该容器中的各个元素，能针对多种容器提供统一的接口。

下面以算法find为例，展示容器、算法和迭代器之间的合作：
```
template<typename InputIterator, typename T>
InputIterator find(InputIterator first, InputIterator last, const T &value) 
{   
    //迭代器对容器进行了屏蔽，不需要知道容器是什么。
    while (first != last && *frist != value)  ++first;
    return first;
}
```
只要给予不同的迭代器，比如vector<int>::iterator、list<int>::iterator，find()就能对不同的容器进行查找，而无需针对某个容器来设计多个版本。这样看来，迭代器似乎依附在容器之下，**有没有独立而范用的迭代器**？这个问题先留着，后面自有答案。



#### 1.1 迭代器是一种smart pointer
实际上**迭代器是一种智能指针，是一种行为类似指针的对象**，它内部封装了一个原始指针，并重载了operator*() 和operator->()等操作。

迭代器在这里使用的好处主要有：
- (1) 不用担心内存泄漏（类似智能指针，析构函数释放内存）；
- (2) 对于list，取下一个元素不是通过自增而是通过next指针来取，这样子暴露了太多东西，使用智能指针可以对自增进行重载，从而**提供统一接口**。
```
//对于数组的实现
template<typename T>
Iterator& operator++() //对于算法来说，就是++n操作，不关心内部实现
{ 
    ++m_ptr; 
    retrun *this;
}

//对于链表的实现
template<typename T>
Iterator& operator++()
{
    m_ptr = m_ptr->next(); //next()用于获取链表的下一个节点 
    return *this;
}
```


#### 1.2 设计一个简单迭代器
知道迭代器的主要功能后，**下面我们来设计一个独立的list迭代器**。首先先设计一个list：
```
//List节点的结构
template<typename T>
class ListItem //链表节点
{
public:
    ListItem() { m_pNext = 0;}
    ListItem(T v, ListItem *p = 0) { m_value = v; m_pNext = p;}
    T value() const  { return m_value;} //取值
    ListItem* next() const { return m_pNext;}  //取下一个指针

private:
    T m_value;  //存储的数据
    ListItem* m_pNext;  //指向下一个ListItem的指针
};
```

```
//List表的结构
template<typename T>
class List 
{
public:
    void  Push(T value) //从链表尾部插入元素
    {
       m_pTail = new ListItem<T>(value);
       m_pTail = m_pTail->next();
    }

    //返回链表头部指针
    ListItem<T>* begin() const { return m_pHead;}

    //返回链表尾部指针
    ListItem<T>* end() const { return m_pTail;}

    ...//其它成员函数

private：
    ListItem<T> *m_pHead;    //指向链表头部的指针
    ListItem<T> *m_pTail;    //指向链表尾部的指针
    long m_nSize;    //链表长度
};
```

list有了，现在我们需要设计一个迭代器来访问list的值和next,来看一下怎么实现：
```
template<typename T>
class ListIterator  //mylist迭代器
{
public:
    ListIterator(T *p = 0) : m_ptr(p){} //构造函数

    T& operator*() const { return *m_ptr;}  //取值，即dereference
    T* operator->() const { return m_ptr;} //成员访问，即member access

    //前置++操作
    ListIterator& operator++() 
    { 
        m_ptr = m_ptr->next(); //暴露了ListItem的东西
        return *this;
    }

    //后置++操作
    ListIterator operator++(int)
    {
        ListIterator temp = *this;
        ++*this;
        return temp;
    }

    //判断两个ListIter是否指向相同的地址
    bool opeartor==(const ListIterator &arg) const { return arg.m_ptr == m_ptr;}

    //判断两个ListIter是否指向不同的地址
    bool operator!=(const ListIterator &arg) const { return arg.m_ptr != m_ptr;}

private:
    T* m_ptr;
};
```
好了，现在list和其对应的迭代器都有了，那我们该怎么将ListIter将List和find()粘合起来呢？来看一下：
```
int main(int argc, const  char *argv[])
{
    List<int> mylist;
    for (int i = 0; i < 5; ++i) mylist.push(i);

    //这里暴露了ListItem，为了获取对应的指针
    ListIterator<ListItem<int> > begin(mylist.begin());
    ListIterator<ListItem<int> > end(mylist.end());
    ListIterator<ListItem<int> > iter;

    iter = find(begin, end, 3);//从链表中查找3
    if (iter != end)  cout<<"found"<<endl;
    else  cout<<"not found"<<endl;
}
```

需要注意的是，算法find是通过*first != value用来判断元素是否符合要求，而上面测试代码中，first的类型为ListItem<int >，而value的类型为int，两者之间并没有可用的operator!=函数，因此，需要另外声明一个全局的operator!=重载函数，代码如下：
```
template<typename T>
bool operator!=(const ListItem<T> &item, T n)
{
    return item.Value() != n;
}
```
#### 1.3 迭代器小结
从上面的代码我们可以看到，为了实现迭代器ListIter，我们在很多地方暴露了容器List的内部实现ListItem，这违背一开始说的**迭代器模式中不暴露某个容器的内部表现形式情况下，使之能依次访问该容器中的各个元素的定义**。

可见，**独立的迭代器并不能满足我们的要求，所以STL将迭代器的实现交给了容器，每种容器都会以嵌套的方式在内部定义专属的迭代器**。各种迭代器的接口相同，内部实现却不相同，这也直接体现了泛型编程的概念。（泛型编程的思想主要是定义相同的接口，而内部实现各不相同，从而实现多态）  

总而言之，**迭代器依附于具体的容器，即不同的容器有不同的迭代器实现**。     对于泛型算法find，只要给它传入不同的迭代器，就可以对不同的容器进行查找操作。迭代器的穿针引线，有效地实现了算法对不同容器的访问。

### 2、traits编程技法
其实上一章我们就涉及过trait编程技法，就是在配置器的destroy()函数中，我们需要用traits来判断某类型的构造和析构函数是否是trival(无用的)，从而决定有没有必要调用析构函数，从而减少一些不必要的操作。

本节我们来了解其实现原理。

#### 2.1 迭代器型别value type
迭代器所指对象的型别，称为该迭代器的value type，比如int*的value type为int。那我们该怎么来获取这个型别呢？
RTTI性质中的typeid()只能获得型别的名称，但不能用来声明变量。要想获得迭代器型别，**参数推导机制**是一个不错的方法。

在下面这个函数里面，如果我们想要得到迭代器的value type，以下做法是否可行？看看下面这种情况：
```
template<typename Iteratorator>
void func(Iteratorator iter) {
    *Iteratorator var;//对迭代器取值，得其值类型？
}
```
事实上，以上代码是编译失败的，C++并不提供这个支持。

#### 2.2 参数推导机制+内嵌型别机制
既然上述方式不能行，那我们换一种思路，引入中间层试一下，来看代码实现:
```
template<typename Iterator, typename T> //T推导为int型
void func_impl(Iterator iter, T t) 
{
    T temp;//这里就解决了问题
    //这里做原本func()的工作
}

template<typename Iterator> //Iterator推导为int*类型
void func(Iterator iter) 
{
    func_impl(iter, *iter);//这里通过传递*iter，让func_impl去推导
}

int main(int argc, const  char *argv[])
{
    int i;
    func(&i); //这里传入的是一个迭代器（原生指针也是一种迭代器）
}
```

上面做法通过多层迭代很巧妙地导出了T，但是却很有局限性，比如，我的func()希望返回迭代器的value type类型返回值，那么上面的做法就无能为力了。**这种template参数推导的只是参数，无法推导返回值**。

上述方法还是不行，但如果在参数推导机制上加上内嵌型别(typedef)呢？来看一下实现：
```
template<typename T>
class Iterator
{
public:
    typedef T value_type; //内嵌类型声明
    Iterator(T *p = 0) : m_ptr(p) {}
    T& operator*() const { return *m_ptr;}
    //...

private:
    T *m_ptr;
};

template<typename Iterator>
//以迭代器所指对象的类型作为返回类型
//注意typename是必须的，它告诉编译器这是一个类型
typename Iterator::value_type func(Iterator iter) 
{
    return *iter;
}

int main(int argc, const  char *argv[]) 
{
    Iterator<int> iter(new int(10));
    cout<<func(iter)<<endl;  //输出：10
}
```

上面的解决方案近乎完美了，但其实有一个隐晦的陷阱：**实际上并不是所有的迭代器都是class type，原生指针也是一种迭代器,由于原生指针不是class type，所以没法为它定义内嵌型别**。要解决这个问题，Partial specialization（模板偏特化）就出场了。

#### 2.4 Partial specialization（模板偏特化）
所谓偏特化是指如果一个class template拥有一个以上的template参数，我们可以针对其中某个（或多个，但不是全部）template参数进行特化，比如下面这个例子：
```
template <typename T>
class C {...}; //此泛化版本的T可以是任何类型

template <typename T>
class C<T*> {...}; //特化版本，仅仅适用于T为“原生指针”的情况，是泛化版本的限制版
```
所谓特化，就是特殊情况特殊处理，第一个类为泛化版本，T可以是任意类型，第二个类为特化版本，是第一个类的特殊情况，只针对原生指针。

#### 2.5 原生指针怎么办？——特性“萃取”traits
还记得前面的参数推导机制+内嵌型别机制获取型别有什么问题吗？问题就是原生指针虽然是迭代器但不是class，无法定义内嵌型别，而偏特化似乎可以解决这个问题。

有了上面的认识，我们再看看STL是如何应用的,STL定义了下面的类模板，它专门用来“萃取”迭代器的特性，而value type正是迭代器的特性之一：
```
template<typename Iterator>
struct iterator_traits //类型萃取机
{
    typedef typename Iterator::value_type value_type;
};
```

我们看看加入萃取机前后的变化：
```
template<typename Iterator> //萃取前
typename Iterator::value_type  func(Iterator iter)
{
    return *iter;
}

//通过iterator_traits作用后的版本
template<typename Iterator>  //萃取后
typename iterator_traits<Iterator>::value_type  func(Iterator iter)
{ 
    return *iter;
}
```

看到这里也许你会晕了，iterator\_traits::value\_type跟Iterator::value\_type完全是同一个东西，为什么还要增加iterator\_traits这一层封装，是不是多此一举？

回想萃取之前的版本有什么缺陷：不支持原生指针。而通过萃取机的封装，**我们可以通过类模板的特化来支持原生指针的版本**！如此一来，无论是智能指针，还是原生指针，iterator\_traits::value\_type都能起作用，这就解决了前面的问题。
```
//iterator_traits的偏特化版本，针对迭代器是原生指针的情况
template<typename T>
struct iterator_traits<T*>
{
    typedef T value_type;
};
```

#### 2.6 偏特化处理更多的情况——const指针
通过偏特化添加一层中间转换的traits模板class，能实现对原生指针和迭代器的支持，但是这里还有一个特殊情况，对于指向常数对象的指针又该怎么处理呢？比如下面的例子：
```
iterator_traits<const int*>::value_type  //获得的value_type是const int，并不是int
```
这里会调用的还是上述原生指针对应的偏特化，获得的value taype是const int，我们知道，const变量只能初始化，而不能赋值（这两个概念必须区分清楚）。这将带来问题：
```
template<typename Iterator>
typename iterator_traits<Iterator>::value_type  func(Iterator iter) 
{ 
    typename iterator_traits<Iterator>::value_type tmp;//注意不是typedef哦..
    tmp = *iter; //ok?
}

int val = 8;
const int *p = &val;
func(p); //这时函数里对tmp的赋值都将是不允许的
```
可以看到，我们的本意是获取int，而事实上获取到的是const int，这将造成误会，那该怎么办呢？答案还是偏特化，来看实现：
```
template<typename T>
struct iterator_traits<const T*> //特化const指针
{
    typedef T value_type; //得到T而不是const T
}
```
#### 2.7 traits编程技法总结
通过本节的学习，我们知道traits编程技法就是增加一层中间的模板class，以解决获取迭代器的型别中的原生指针问题，其核心知识点在于**模板参数推导机制+内嵌类型定义机制**，为了能处理原生指针这种特殊的迭代器，在traits机这个模板class上引入了**偏特化机制**。traits就像一台“特性萃取机”，把迭代器放进去，就能榨取出迭代器的特性。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/7.png?raw=true)

### 3、迭代器的型别和种类
#### 3.1 迭代器的型别
常见迭代器相应型别有5种：
- （1）value\_type：迭代器所指对象的类型，原生指针也是一种迭代器，对于原生指针int*，int即为指针所指对象的类型，也就是所谓的value\_type。
- （2）difference\_type用来表示两个迭代器之间的距离，对于原生指针，STL以C++内建的ptrdiff\_t作为原生指针的difference_type。
- （3）reference\_type是指迭代器所指对象的类型的引用，reference\_type一般用在迭代器的*运算符重载上，如果value\_type是T，那么对应的reference\_type就是T&；如果value\_type是const T，那么对应的reference\_type就是const T&。
- （4）pointer_type就是相应的指针类型，对于指针来说，最常用的功能就是operator*和operator->两个运算符。
- （5）iterator\_category的作用是标识迭代器的移动特性和可以对迭代器执行的操作，从iterator\_category上，可将迭代器分为Input Iterator、Output Iterator、Forward Iterator、Bidirectional Iterator、Random Access Iterator五类，这样分可以尽可能地提高效率。
```
template<typename Category,
         typename T,
         typename Distance = ptrdiff_t,
         typename Pointer = T*,
         typename Reference = T&>
struct iterator //迭代器的定义
{
    typedef Category iterator_category;
    typedef T value_type;
    typedef Distance difference_type;
    typedef Pointer pointer;
    typedef Reference reference;
};
```
iterator class不包含任何成员变量，只有类型的定义，因此不会增加额外的负担。由于后面三个类型都有默认值，在继承它的时候，只需要提供前两个参数就可以了。**这个类主要是用来继承的，在实现具体的迭代器时，可以继承上面的类，这样子就不会漏掉上面的5个型别了**。

对应的迭代器萃取机设计如下：
```
tempalte<typename I>
struct iterator_traits  //特性萃取机，萃取迭代器特性
{
    typedef typename I::iterator_category iterator_category;
    typedef typename I::value_type value_type;
    typedef typeanme I:difference_type difference_type;
    typedef typename I::pointer pointer;
    typedef typename I::reference reference;
};

//需要对型别为指针和const指针设计特化版本
```


#### 3.2 迭代器的分类
迭代器型别iterator\_category对应迭代器类别，这个类别会限制迭代器的操作和移动特性，
**除了原生指针以外**，迭代器被分为五类：
- **(1) Input Iterator**： 此迭代器不允许修改所指的对象，即是只读的。支持==、!=、++、*、->等操作。
- **(2) Output Iterator**：允许算法在这种迭代器所形成的区间上进行只写操作。支持++、*等操作。
- **(3) Forward Iterator**：允许算法在这种迭代器所形成的区间上进行读写操作，但只能单向移动，每次只能移动一步。支持Input Iterator和Output Iterator的所有操作。
- **(4) Bidirectional Iterator**：允许算法在这种迭代器所形成的区间上进行读写操作，可双向移动，每次只能移动一步。支持Forward Iterator的所有操作，并另外支持–操作。
- **(5) Random Access Iterator**：包含指针的所有操作，可进行随机访问，随意移动指定的步数。支持前面四种Iterator的所有操作，并另外支持it + n、it - n、it += n、 it -= n、it1 - it2和it\[n\]等操作。

**为什么我们要对迭代器进行分类呢**?      
设计算法时，如果可能，我们尽量针对上面某种迭代器提供一个明确定义，并针对更强化的某种迭代器提供另一种定义，这样才能在不同情况下提供最大效率。比如，有个算法可接受Forward Iterator，但是你传入一个Random Access Iterator，虽然可用（Random Access Iterator也是一种Forward Iterator），但是不一定是最佳的，因为Random Access Iterator可能更加臃肿，效率不一定高。

对于一个算法，它该调用哪个类型的迭代器，我们可以简单的在内部使用if…else在执行时选择，但是这样却降低了效率，如果能在编译时选择就再好不过了。STL使用了重载函数机制达成了这个目标，此处就不再深入讨论了。
