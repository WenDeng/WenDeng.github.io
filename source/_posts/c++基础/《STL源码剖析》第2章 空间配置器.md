---
title: 《STL源码剖析》第2章 空间配置器
date: 2019-5-23 22:50:12    
toc: true   
comments: true   
img: https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/1.png?raw=true   
tags:
  - STL源码
  - 技术   
categories:   
  - c++基础
---

由于整个STL的操作对象都放在容器内，而容器需要配置空间以放置数据，学习空间配置器的原理是掌握其它部分知识的基础。

为什么不说allocator是内存配置器而说它是空间配置器呢？这是因为空间配置器的配置不一定是内存，其实也可以是磁盘或者其他辅助存储介质。**我们也可以自己实现空间配置器，只需要自己对应实现相应的标准接口即可**。

参考链接：    
https://blog.csdn.net/xiajun07061225/article/details/8241203    
https://zhuanlan.zhihu.com/p/34725232

<!--more-->



### 1、SGI STL配置器简介

SGI STL的配置器与众不同，也与标准规范不同，**其名称是 alloc 而非 allocator** ,而且不接受任何参数。如果要在程序中明确使用SGI配置器，那么应该这样写：
```
vector<int,std::alloc> iv; //而不是vector<int,std::allocator<int> > iv;
```
标准配置器的名字是allocator，而且可以接受参数。
SGI STL的每一个容器都已经指定了**缺省配置器**：alloc。我们很少需要自己去指定空间配置器。比如vector容器的声明：
```
template <class T, class Alloc = alloc>  // 预设使用alloc为配置器
class vector {
//...
}
```

> 其实SGI也定义了一个符合部分标准，名为allocator的配置器，但是它自己不使用，也不建议我们使用，主要原因是效率不佳。**它只是把C++的操作符::operator new和::operator delete做了一层简单的封装而已**，可以用但是不建议我们使用。



### 2、SGI特殊的空间配置器alloc
我们所习惯的c++内存配置操作和释放操作如下：
```
class Foo { ... };
Foo* pf = new Foo; // 配置内存，然后构造对象
delete pf; // 将对象析构，然后释放内存
```
这其中的new 操作符（new operator）包含两阶段操作：
- （1）调用operator new配置内存       
- （2）调用Foo::Foo( )构造函数构造对象内容。

delete操作符包含两阶段操作：
- （1）调用Foo::~Foo( )析构函数将对象析构。        
- （2）调用operator delete释放内存

SGI也提供了类似的操作函数，**STL allocator 将这两阶段操作区分开来。内存配置操作由 alloc::allocate() 负责，内存释放操作由 alloc::deallocate() 负责；对象构造操作由 ::construct() 负责，对象析构操作由 ::destroy() 负责。**

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/1.png?raw=true)

如上图，STL空间配置器主要分三个文件实现：  
- （1）`<stl_construct.h>`：定义了全局函数construct()和destroy()，负责对象的构造和析构。
- （2）`<stl_alloc.h>`：定义了一、二两级配置器，彼此合作，配置器名为alloc。
- （3）`<stl_uninitialized.h>`：这里定义了一些全局函数，用来初始化填充(fill)或复制大块内存数据，他们也都隶属于STL标准规范,具体实现可能为construct或memmove。

### 3、构造和析构工具：construct()和destroy()
这两个函数的主要版本和功能如下：

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/2.png?raw=true)

#### 3.1 destory()函数
destroy()函数有两个版本，分别如下：
(1)第一个版本较简单，接受一个指针作为参数，直接调用对象的析构函数即可，其源代码：
```
template <class T>
inline void destroy(T* pointer) {
    pointer->~T();	// 调用析构函数
}
```
(2)第二个版本，其参数接受两个迭代器，将两个迭代器所指范围内的所有对象析构掉。而且，**它采用了一种特别的技术：依据元素的型别，判断其是否有trivial destructor（无用的析构函数）进行不同的处理**。value\_type用于获取迭代器所指对象的型别，\_type_traits用于判断类型的析构函数是否有作用。如果对象的析构函数是trivial的,则不需要做析构操作。
SGI的destroy()的实际由\_destroy()调用\_destroy_aux()实现。
```
// 以下是 destroy() 第二版本，接受两个迭代器。它会设法找出元素的数值型別，
// 进而利用 __type_traits<> 求取最适当措施。
template <class ForwardIterator>
inline void destroy(ForwardIterator first, ForwardIterator last) {
  __destroy(first, last, value_type(first));
}

// 判断元素的数值型別（value type）是否有 trivial destructor，分别调用上面的函数进行不同的处理
template <class ForwardIterator, class T>
inline void __destroy(ForwardIterator first, ForwardIterator last, T*) {
  typedef typename __type_traits<T>::has_trivial_destructor trivial_destructor;
  __destroy_aux(first, last, trivial_destructor());
}

// 如果元素的数值型別（value type）有 non-trivial destructor…
template <class ForwardIterator>
inline void
__destroy_aux(ForwardIterator first, ForwardIterator last, __false_type) {
  for ( ; first < last; ++first)
    destroy(&*first);//调用析构函数,destory不支持迭代器，转为第一个版本执行
}
 
// 如果元素的数值型別（value type）有 trivial destructor…
template <class ForwardIterator> 
inline void __destroy_aux(ForwardIterator, ForwardIterator, __true_type) {}//不调用析构函数
 
```

#### 3.2 construct()函数
construct()函数使用了new placement操作符，其也叫new placement运算子，作用是在指针所指的空间上初始化一个对象值，实现代码如下：

```
template <class T1, class T2>
inline void construct(T1* p, const T2& value) {
  new (p) T1(value); 	// 定位new操作符placement new; 在指针p所指处构造对象
}
```


### 4、空间的配置和释放,std::alloc
对象构造前的空间分配和析构后的空间释放，定义在头文件<stl_alloc.h>中，设计思想主要考虑以下几点：
- 向system heap要求空间。
- 考虑多线程状态。
- 考虑内存不足时的应变措施。
- 考虑过多“小额区块”可能造成的内存碎片问题。

**本节不考虑多线程状态的处理**，SGI绕过new()和free(),直接以malloc()和free()完成内存的配置和释放。考虑到**小型区块**可能造成的内存破碎问题，**SGI设计了双层级配置器，第一级配置器直接使用malloc()和free()，第二级则视情况采用不同的策略：当配置区块超过128bytes时，便调用第一级配置器，小于128bytes时则采用复杂的内存池memory pool整理方式，而不再调用一级配置器**。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/3.png?raw=true)

整个设计究竟是只开放第一级配置器还是同时开放第二级配置器取决于宏\_\_USE_MALLOC是否被定义。

```
# ifdef __USE_MALLOC 
... 
typedef __malloc_alloc_template<0> malloc_alloc; //令 alloc为第一级配置器
typedef malloc_alloc alloc; //alloc为默认空间配置器
# else 
... 
//令 alloc 为第二级配置器 
typedef __default_alloc_template<__NODE_ALLOCATOR_THREADS, 0> alloc; 
#endif /* ! __USE_MALLOC */ 
```
无论alloc被定义为第一级或者是第二级配置器，SGI还为它包装一个接口如下，使配置器的接口能够符合STL规格：
```
template<class T, class Alloc>
class simple_alloc {
public:
    static T* allocate(size_t n)
                { return 0 == n? 0 : (T*) Alloc::allocate(n * sizeof (T)); }
    static T* allocate(void)
                { return (T*) Alloc::allocate(sizeof (T)); }
    static void deallocate(T *p, size_t n)
                { if (0 != n) Alloc::deallocate(p, n * sizeof (T)); }
    static void deallocate(T *p)
                { Alloc::deallocate(p, sizeof (T)); }
};
```
SGI STL容器全部是使用这个simple_alloc接口，比如vector:

```
template<class T,class Alloc=alloc> //默认使用alloc为配置器
class vector{
protected:
    typedef simple_alloc<value_type,Alloc> data_allocator;//配置器对象
    void deallocate(){
        if(...) deta_allocator::deallocate(start,end_of_storage-start);
    }
}
```


### 5、双层级配置器机制
如上节所述，SGI设计了双层级配置器，第一级配置器直接使用malloc()和free()，第二级则视情况采用不同的策略：当配置区块超过128bytes时，便调用第一级配置器，小于128bytes时则采用复杂的内存池memory pool整理方式，而不再调用一级配置器。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/4.png?raw=true)

#### 5.1 第一级配置器__malloc_alloc_template
第一级配置器直接使用malloc(),free(),realloc()等C函数执行实际的内存配置、释放、重配置操作，并实现出**类似C++ new handler机制**（意思是可以要求系统在配置需求无法被满足时，调用一个你所指定的函数，这里之所以说类似是因为SGI没有new operator，不能直接使用对应的handler）。

它有独特的**out-of-memory**(内存不足)内存处理机制：      
SGI第一级配置器的allocate()和reallocate()在调用malloc()和realloc()不成功后，在抛出std::bad\_alloc异常之前，改调用oom\_malloc(),oom\_realloc()，这两个函数两种都有内循环，不断调用“内存不足处理例程”以期望获得足够内存满足分配请求。如果没有设定“内存不足处理例程”，则还是会抛出std::bad\_alloc异常。


#### 5.2 第二级配置器__default_alloc_template
相比第一级配置器，第二级配置器多了一些机制，**避免小额区块造成内存的碎片**。小区块带来的不仅仅是碎片的问题，配置时的额外负担也是一个大问题。因为区块越小，额外负担所占的比例就越大，愈显得浪费（额外负担是指动态分配内存块的时候，位于其头部的额外信息）。

第二级配置器的思想主要包括以下两点：
- 如果要分配的区块大于128bytes，则移交给第一级配置器处理。
- 如果要分配的区块小于128bytes，则以内存池管理（memory pool），又称之次层配置）：每次配置一大块内存，并维护对应的自由链表（free-list）。下次若有相同大小的内存需求，则直接从free-list中取。如果有小额区块被释放，则由配置器回收到free-list中。


##### 5.2.1 free_list的实现     
在第二级配置器中，**小额区块内存需求大小都被上调至8的倍数**，比如需要分配的大小是30bytes，就自动调整为32bytes。系统中总共维护16个free-lists，各自管理大小为8,16，...，128bytes的小额区块。

为了维护链表，需要额外的指针，为了避免造成另外一种额外的负担，这里采用了一种技术：用union表示链表节点结构：

```
  union obj { //实现头节点和每个链表节点的复用
        union obj * free_list_link; //指向下一个节点
        char client_data[1]; //值表示所对应的每个链中的块的大小
  };
  
```
##### 5.2.2 __default_alloc_template的allocate()
身为一个配置器，\_\_default\_alloc_template拥有配置器的标准接口函数allocate(),此函数首先判断区块大小，如果要分配的区块大于128bytes，就调用第一级配置器。小于128bytes就检查对应free-list，如果free-list之内有可用的区块就就直接从free-list中取用，负责调用refill()为free list重新填充空间,refill()函数将下节介绍。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/6.png?raw=true)


##### 5.2.2 __default_alloc_template的deallocate()
身为一个配置器，\_\_default\_alloc_template也拥有析构接口deallocate()，函数判断如果需要回收的区块大于128bytes，则调用第一级配置器的deallocate()。
如否则找到对应的free -list，将区块回收，**注意是将区块放入free-list的头部**。


### 6、refill()函数的处理机制
**前面提到，如果free-list中没有可用的区块，就通过refill为对应的free-list 重新填充空间。此时refill进行了怎样的操作呢**？

默认操作是通过**chunk_alloc**从内存池中取得20个新的节点添加到free-list链表中，而如果内存池中的内存不够用，这时候能分多少就分多少节点，返回的相应的节点数。再万一内存池**一个节点都提供不了**，就给内存池新增空间，如果失败，再抛出bad_alloc异常。

#### 6.1 chunk_alloc——从内存池中取空间供free list使用

通过**chunk_alloc()**，从内存池中分配空间给free-list。**chunk_alloc()**的流程总结如下：
- (1) 内存池剩余空间完全满足20个区块的需求量，则直接取出对应大小的空间（其中1个返回，剩下部分放入free-list）。
- (2) 内存池剩余空间不能完全满足20个区块的需求量，但是足够供应一个及一个以上的区块，则取出能够满足条件的区块个数的空间。
- (3) 内存池剩余空间不能满足一个区块的大小，则:
- - 首先判断内存池中是否有残余零头内存空间，如果有则进行回收，将其编入free list。然后向heap申请空间，补充内存池。
- heap空间满足，空间分配成功(分配的个数为需求的2倍再加一个附加量，然后一半留给内存池，剩下的除去请求的的那部分后放入free-list)。
- heap空间不足，malloc()调用失败。则
搜寻适当的free list（适当的是指：**尚有未用区块，并且区块足够大**），调整以进行释放，将其编入内存池。然后递归调用chunk_alloc函数从内存池取空间供free list。
搜寻free list释放空间也未能解决问题，这时候调用第一级配置器，利用out-of-memory机制尝试解决内存不足问题。如果可以就成功，否则抛出bad_alloc异常。


**如果有需求的话，内存池中会不断的通过malloc申请新的内存，最后内存池所拥有的内存也会越来越大，当然最后进程结束的时候，这些内存都会由操作系统收回。**


### 7、内存基本处理工具
处理construct()和destroy(),另外还有三个函数作用于未初始化空间上，分别为uninitialized\_copy()、uninitialized\_fill()、uninitialized\_copy_n()，此处不再介绍。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/5.png?raw=true)

### 8、小结
学完空间配置器的相关知识，有必要总结一下：
- constructor: 由new placement操作符实现。
- destroy: 进行特化，利用value\_type和type_trait<T>确定析构函数是否有效，从而决定是否利用指针调用析构函数。
- alloc:双层配置器，第一级配置器采用malloc+类似new handle实现，第二级配置器采用free-list处理小额区块+块为8的整数倍+memory pool+一级配置器实现。
- dealloc：大于128bytes用free()+否则将空间放到free-list的首部。 