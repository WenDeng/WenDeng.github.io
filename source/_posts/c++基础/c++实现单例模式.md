---
title: c++实现单例模式
date: 2018-12-11 15:21:12
toc: true
comments: true
tags:
  - 设计模式
  - 技术
categories:
  - 设计模式
---

在所有的设计模式中，单例模式是唯一一个能用较短代码实现的模式，所以其常成为面试的一个考点。《剑指offer》中的第二题中有讲到如何用c#实现单例模式的类，但实际上，我们用的比较多的是c\+\+，c\+\+也肯定会有类似的应用场景，那c\+\+中又该怎样实现呢？
<!--more-->

### 1、引入
加入有这样一个场景：设计一个类，我们只生成该类的一个实例，该怎么实现呢？


### 2、设计思想：
* 单例模式的class主要起工具作用，用于提供全局性的操作。其实用全局或者静态变量也能实现类似的功能，但这样的封装性不好。普通的class在不断new的过程中也会显得比较臃肿。
* 为了只生成一个实例，我们可以将构造函数声明为private级别，从而阻止用户产生多个实例。同时我们提供一个public static的方法，通过该方法获得这个类唯一 的实例化对象。这就是单例模式基本的一个思想。
* 常见的单例模式分为两种：   
（1） 饿汉式：即类产生的时候就创建好实例对象，这是一种空间换时间的方式  
（2） 懒汉式：即在需要的时候，才创建对象，这是一种时间换空间的方式

### 3、代码实现：
#### 3.1 饿汉式实现：
``` 
#include<iostream>
using namespace std;
class Singleton
{
  private:
  	static Singleton instance; // 单例对象在这里！
  	
  private:
	Singleton(){ cout << "单例对象创建！" << endl; };
	Singleton(const Singleton &);
	Singleton& operator=(const Singleton &);
	~Singleton(){ cout << "单例对象销毁！" << endl; };
 
  public:
	static Singleton* getInstance()
	{		
		return &instance;//第一次调用时才会调用构建函数
	}
};

Singleton Singleton::instance;//直接生成一个全局性的对象
int main()
{
	Singleton *ct1 = Singleton::getInstance();
	Singleton *ct2 = Singleton::getInstance();
	Singleton *ct3 = Singleton::getInstance();
 
	return 0;
}
```
#### 3.2 懒汉式实现：
饱汉式的单例用法是线程安全的，不需要考虑线程同步，但是懒汉式的情况就不一样了。懒汉式单例模式是在第一次调用getInstance()的时候，才创建实例对象。我们可以直接把对象定义为static，然后放在getInstance()中。第一次进入该函数，就创建实例对象，然后一直到程序结束，释放该对象代码如下：
```
class Singleton
{
  private:
	Singleton(){ cout << "单例对象创建！" << endl; };
	Singleton(const Singleton &);
	Singleton& operator=(const Singleton &);
	~Singleton(){ cout << "单例对象销毁！" << endl; };
 
  public:
	static Singleton * getInstance()
	{	
		static Singleton instance;//第一次用到时才创建对象
		return instance;
	}
};
```
如果我想把对象放在堆上呢？可以这么实现：
```
class Singleton
{
  private:
    static Singleton *instance;
  
	Singleton(){ cout << "单例对象创建！" << endl; };
	Singleton(const Singleton &);
	Singleton& operator=(const Singleton &);
	~Singleton(){ cout << "单例对象销毁！" << endl; };
 
  public:
	static Singleton * getInstance()
	{	
		if (nullptr == instance)
		{
			instance = new Singleton();
		}
		return instance;
	}

private:
	// 定义一个内部类
	class Nested{
	public:
		Nested(){};
		~Nested()
		{   	
		    // 定义一个内部类的静态对象
	        // 当该对象销毁时，顺带就释放instance指向的堆区资源
			if (nullptr != instance)
			{
				delete instance;
				instance = nullptr;
			}
		}
	};

	static Nested foo;//在用户程序中需要使用该object才会触发创建
};
```

#### 3.3 线程安全吗？
上面这种设计方式在单线程环境下是安全的，没问题，但是如果是多线程呢？在if (nullptr == instance)处会由于线程多个线程可能都得到instance==nullptr,就会创建多个对象，明显不符合要求，为了做到线程安全，需要做双重锁校验DLC，代码实现如下：
```
class Singleton
{
  private:
    static Singleton *instance;
  
	Singleton(){ cout << "单例对象创建！" << endl; };
	Singleton(const Singleton &);
	Singleton& operator=(const Singleton &);
	~Singleton(){ cout << "单例对象销毁！" << endl; };
 
  public:
	static Singleton * getInstance()
	{	
	    lock();//确保线程安全
		if (nullptr == instance)
		{
			instance = new Singleton();
		}
		unlock();
		return instance;
	}

private:
	// 定义一个内部类
	class Nested{
	public:
		Nested(){};
		~Nested()
		{   	
		    // 定义一个内部类的静态对象
	        // 当该对象销毁时，顺带就释放instance指向的堆区资源
	        lock();//确保线程安全
			if (nullptr != instance)
			{
				delete instance;
				instance = nullptr;
			}
			unlock();
		}
	};

	static Nested foo;//在用户程序中需要使用该object才会触发创建
};
```
