---
title: 第9章 虚拟内存之Linux内存系统
date: 2019-5-22 24:32:12
toc: true
comments: true
img: https://github.com/WenDeng/Picture_markdown/blob/master/picture/25.png?raw=true
tags:
  - 操作系统
  - 技术
categories:
  - 操作系统
---

按照书籍实例，用一个实际系统的案例研究来总结虚拟内存的讨论，这是一个运行于Linux的Inter Core i7。需要注意的是，虽然我们说64位系统，而且处理器体系也允许64位的虚拟地址空间，但是**实际上，Core i7现在只是支持48位（256TB）虚拟地址空间和52位（4PB）物理地址空间，兼容支持32位（4GB）地址空间**。

<!--more-->

### 1、Core i7 内存系统结构
Core i7的内存系统的结构如下：

![](https://github.com/WenDeng/Picture_markdown/blob/master/picture/21.png?raw=true)

由图可知：
* 处理器包括4个核、所有核共享L3高速缓存和DDR3内存控制器。
* 每个核包含一个层次结构的TLB、一个层次结构的指令高速缓存，以及一组快速的点到点链路，这种QuickPath技术用于与其他核和外部I/O桥直接通信。
* TLB是虚拟寻址的，四路组相联。L1、L2和L3都是物理寻址，分别位8、8和16路组相联。
* Linux使用的是4KB的页。

### 2、Core i7 地址翻译过程
翻译过程如图9-22：

![](https://github.com/WenDeng/Picture_markdown/blob/master/picture/22.png?raw=true)


由图可知Core i7的地址翻译采用了TLB、高速缓存、多级页表等机制，需要注意的是：
* Core i7采用四级页表层次结构。同时每个进程都有它自己的页表层次结构。
* 进程运行时，虽然允许页表换进换出，但是与分配了的页相关联的页表都是驻留在内存中的。
* **CR3控制器**指向第一级页表的起始地址，**CR3的值是每个进程上下文的一部分，每次上下文切换时，CR3的值都会被恢复**。

### 3、第四级页表条目
图9-24给出了第四级页表中条目的格式：

![](https://github.com/WenDeng/Picture_markdown/blob/master/picture/23.png?raw=true)

注意一下几点：
* PTE（page table entry）有三个权限位，控制对页的访问，分别是R/W控制读写，U/S是否能在用户模式中访问，XD（禁止执行），禁止从某些内存页取指令，防止缓冲区溢出攻击。
* MMU翻译虚拟地址时，还会更新另外两个内核缺页处理程序会用到的位。A位称为引用位，内核用这个引用位来实现它的页替换算法。**D位（修改位/脏位），告诉内核在复制替换页之前是否必须写回牺牲页**。

图9-25给出Core i7如何使用四级页表来将虚拟地址翻译成物理地址的。36位VPN被划分为四个9位的片，每个片被用作到一个页表的偏移量，CR3寄存器（控制寄存器3）包含L1页表的物理地址。VPN1提供到一个L1 PTE的偏移量，这个PTE包含L2页表的基地址。VPN2提供一个到L2 PTE的偏移量，以此类推。

![](https://github.com/WenDeng/Picture_markdown/blob/master/picture/24.png?raw=true)

**优化地址翻译**   
当CPU需要翻译个虚拟地址时，它就发送一VPN到MMU,发送VPO到高速L1缓存。当MMU向TLB请求一个页表条目时，L1高速缓存正忙着利用VPO位查找对应的组，并读取这个组里的8个标记和相应的数据字。然后等MMU拿到PPN后直接就可以和8个标记进行匹配，决定是否取出其中的值。

### 4、Linux虚拟内存系统
本节主要是了解一个实际的操作系统如何组织虚拟内存和处理缺页。

一个Linux进程的虚拟内存如图9-26。
![](https://github.com/WenDeng/Picture_markdown/blob/master/picture/25.png?raw=true)