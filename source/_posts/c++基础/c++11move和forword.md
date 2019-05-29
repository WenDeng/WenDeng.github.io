---
title: c++11特性之move和forward应用与区别  
date: 2019-5-14 14:20:12   
toc: true   
comments: true   
tags:
  - C++11特性
  - 技术   
categories:   
  - C++基础
---

move和forward有他们各自的应用场景，学习并合理区别他们的用法很重要。
最近刚好把它们俩梳理了一遍, 来写写move和forward为什么会出现, 他们能解决什么痛点.
<!--more-->

### 1、特性背景
c++中传值默认是copy，而copy开销很大， 比如下面例子, 他们都要经历至少一次复制操作:

```
func("some temporary string"); //初始化string, 传入函数, 可能会导致string的复制
v.push_back(X()); //初始化了一个临时X, 然后被复制进了vector
a = b + c; //b+c是一个临时值, 然后被赋值给了a
x++; //x++操作也有临时变量的产生
a = b + c + d; //c+d一个临时变量, b+(c+d)另一个临时变量
```
这些临时变量在C++11里被定义为**右值**, 因为没有对应的变量名存他们，同时有对应变量名的被称为**左值**。

### 2、案例
以上copy操作有没有必要呢? 有些地方可不可以省略呢? 我们来看看下面一个案例, 之后我们会用到它来推导出为什么我们需要move和forward:

```
class A{...};

void A::set(const string & var1, const string & var2){
  m_var1 = var1;  //copy
  m_var2 = var2;  //copy
}
```
下面这个写法是没法避免copy的, 因为怎么着都得把外部初始的string传进set函数, 再复制给成员变量:

```
A a1;
string var1("string1");
string var2("string2");
a1.set(var1, var2); // OK to copy
```

但下面这个呢? 临时生成了2个string, 传进set函数里, 复制给成员变量, 然后这两个临时string再被回收，本质上来说，和上一个是一样的，是不是有点多余?

```
A a1;
a1.set("temporary str1","temporary str2"); //temporary, unnecessary copy
```

上面复制的行为, 在底层的操作很可能是这样的:
- (1)临时变量的内容先被复制一遍
- (2)被复制的内容覆盖到成员变量指向的内存
- (3)临时变量用完了再被回收

这里能不能优化一下呢? 临时变量反正都要被回收, 如果能直接把临时变量的内容, 和成员变量内容交换一下, 就能避免复制了? 如下:
- (1)成员变量内部的指针指向"temporary str1"所在的内存
- (2)临时变量内部的指针指向成员变量以前所指向的内存
- (3)最后临时变量指向的那块内存再被回收

上面这个操作避免了一次copy的发生, 其实它就是所谓的**move语义**.

### 3、传变量和临时变量
再回到我们的例子，没法避免copy操作的时候, 还是要用const T&把**变量**传进set函数里, 现在T &叫左值引用了, 如下:
```
void set(const string & var1, const string & var2){
  m_var1 = var1;  //copy
  m_var2 = var2;  //copy
}

A a1;
string var1("string1");
string var2("string2");
a1.set(var1, var2); // OK to copy
```
传临时变量的时候, 可以传T &&, 叫右值引用, 它能接收临时变量（右值）,之后再调用std::move就避免copy了。

```
void set(string && var1, string && var2){
  //avoid unnecessary copy!
  m_var1 = std::move(var1);  
  m_var2 = std::move(var2);
}
A a1;
//temporary, move! no copy!
a1.set("temporary str1","temporary str2");
```

### 4、引入forward
现在终于能处理临时变量了, 但如果按上面那样写, 处理临时变量用右值引用string &&, 处理普通变量用const引用const string &...这代码量有点大呀? 每次都至少要写两遍,怎么避免重复呢？那就是forward。

```
template<typename T1, typename T2>
void set(T1 && var1, T2 && var2){
  m_var1 = std::forward<T1>(var1);
  m_var2 = std::forward<T2>(var2);
}
```
perfect forward (完美转发)能将上述两者结合在一起：
- 如果外面传来了右值临时变量, 它就转发右值并且启用move语义.
- 如果外面传来了左值, 它就转发左值并且启用copy. 然后它也还能保留const.

### 5、有了forward为什么还要用move?
技术上来说, forward确实可以替代所有的move，但还有一些问题:

首先, forward常用于template函数中, 使用的时候必须要多带一个template参数T:forward<T>, 代码略复杂;还有, 明确只需要move的情况而用forward, 代码意图不清晰, 其他人看着理解起来比较费劲.

更技术上来说, 他们都可以被static\_cast替代. 为什么不用static_cast呢? 也就是为了读着方便易懂.

### 6、总结
可以这么说，move属于强转，forward对于左值还是会转换成左值，对于右值转换成右值。一般在模板元编程里面，对于forward需求比较多，因为可以处理各种不同场景。而一般的代码里面，由于可以确认传入的是左值还是右值，所以一般直接就调用std::move了。

