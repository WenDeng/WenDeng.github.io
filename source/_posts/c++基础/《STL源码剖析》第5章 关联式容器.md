---
title: 《STL源码剖析》第5章 关联式容器     
date: 2019-5-18 20:15:12    
toc: true   
comments: true   
img: https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/18.png?raw=true   
tags:
  - STL源码
  - 技术   
categories:   
  - C++基础
---
**所谓关联式容器，观念上类似于关联式数据库，每个元素都有一个键值key和一个实值value**。当向容器插入元素时，容器根据其key值将实值放到适当的位置。关联式容器没有头尾的概念，所以也不会有push\_front()，push_back()等操作函数。

STL关联容器分为set（集合）和map（映射表）两大类，及其衍生体multiset和multimap。这些容器的底层机制均以RB-tree（红黑树）实现。RB-tree也是一个独立容器，但并不开放使用。

SGI STL还提供一个不在标准规格的关联式容器 hash\_table（散列表），以及以 hash\_table 为底层机制而完成的 hash\_set散列集合、hash\_map散列映射表、hash\_multiset散列多键集合、hash_multimap散列多键映射表。

本章我们将主要学习各种容器对应的底层实现原理和应该注意的地方。
<!--more-->

### 1、树的基本的概念
一个关于树的比较重要且容易模糊的概念是：
- 节点路径长度（也叫深度）：根节点到当前节点所经过的边数和。
- 节点的高度：某节点至其最远叶子节点的路径长度的值。

二叉搜索树在一些情况下不能很好地保持平衡性，所以引入了AVL树（带额外平衡条件的二叉搜索树），其保证任意节点的左右两颗子树的高度差不超过1，这就要求每插入一个节点
时都需要进行调整以保证平衡性。调整分为四种情况对应两种调整方式：单旋转和双旋转。

AVL的不足之处在于过分追求平衡，从而导致插入效率变低，在不大影响查找效率的基础上同时满足大概的平衡就好了，于是人们引入了RB-Tree。

### 2、RB-tree
RB-tree（红黑树）是一种被广泛使用的平衡二搜索树，其通过一些着色法则确保没有一条路径会比其它路径长两倍，从而达到接近平衡目的。RB-tree必须满足以下规则：
- （1）每个节点不是红色就是黑色；
- （2）根节点为黑色；
- （3）如果节点为红，其子节点必须为黑；
- （4）任一节点至NULL（树尾端）的任何路径，所含之黑节点数必须相同。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/19.png?raw=true)

根据规则（4），新增节点必须为红，根据规则（3），新增节点之父节点必须为黑。当新节点根据二叉搜索树的规则到达其插入点，却未能符合上述条件时，就必须调整颜色并旋转树形。

#### 2.1 RB-tree效率所在
红黑树之所以为红黑树的原因：红黑颜色用来检测树的平衡性，达到AVL树的平衡要求，降低了对旋转的要求，从而提高了统计性能。红黑树相对AVL树能够给我们一个比较便宜的解决方案。红黑树的算法时间复杂度和AVL相同，但统计性能比AVL树更好。

RB-tree不仅在树性的平衡上表现不错，在效率表现和实现复杂度上也保持相当的平衡，所以运用甚广。主要用于存储有序的数据，它的时间复杂度为O(logn)效率非常之高，Java集合中的TreeSet和TreeMap，C++的STL中的set，map以及Linux虚拟内存的管理，都通过红黑树去实现。

#### 2.2 RB-tree插入节点的调整
假设新节点为X，其父节点为P，祖父节点为G，伯父节点（父节点的兄弟节点）为S，曾祖父节点为GG。当向RB插入一个节点时，主要讨论四种情况：
- （1）状况1：S为黑且X为外侧插入。对此情况，先对P,G做一次单选转，再更改P,G颜色，即可重新满足红黑树的规则3。
- （2）状况2：S为黑且X为内测插入。对此情况，必须现对P,X做一次单选转并更改G,X颜色，再将结果对G做一次单选转，即可再次满足红黑树规则3。
- （3）状况3：S为红且X为外侧插入。对此情况，现对P和G做一次单选转，并改变X的颜色。此时如果GG为黑，一切搞定，但如果GG为红，则问题比较大，见状况4。
- （4）状况4：S为红且X为外侧插入。对此情况，先对P和G做一次单选转，并改便X的颜色。此时如果GG也为红。害的持续网上做，直到不再有父子连续为红的情况。

红黑树删除基本思想是：删除后，用其子树替换，这部分与二叉搜索树的删除的思想本质一样，但是红黑树删除后，可能会破坏红黑树的性质，此时就需要进行树的调整操作即可。

#### 2.3 RB-tree节点设计和迭代器
RB-tree有红黑二色，并且拥有左右子节点，很容易勾勒出其结构风貌。实现上，为了有更大的弹性，节点分为两层。同时由于RB-tree的各种操作时常需要上溯其父节点，所以特别在数据结构中安排了一个parent指针。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/20.png?raw=true)

SGI将RB-tree迭代器实现分为两层。上图是两层节点结构和双层迭代器结构间的关系，其中_ rb\_tree\_node继承自rb\_tree\_node\_base，rb\_tree\_iterator继承自\_rb\_tree\_base_iterator。

#### 2.4 RB-tree的元素插入接口
RB-tree提供两种插入操作：insert\_unique()和insert_equal()，前者标识被插入节点的键值（key）在整棵树中必须独一无二,如果整棵树中已存在相同的键值，插入操作就不会真正进行;后者标识被插入节点的键值在整棵树中可以重复，因此，无论如何插入都会成功。

当然插入元素后是需要对RB-tree进行调整的，这里不进行讲解了。

### 3、set和multiset
set的所有特性可以归纳为以下几点： 
- （1）所有元素都会根据元素的键值自动被排序。 
- （2）set是集合，它的元素的键值就是实值，实值就是键值，不允许两个元素有相同的值。
- （3）**不可以通过set的iterator来改变元素的值，因为set的元素值就是键值，改变键值会违反元素排列的规则**。 
- （4）在客户端对set进行插入或删除操作后，之前的迭代器依然有效。当然，被删除的元素的迭代器是个例外。 
- （5）它的底层机制是RB-tree，几乎所有的操作都只是转调用RB-tree的操作行为而已。
- （6）set提供的算法包括交集、并集、差集、对称差集等。

multiset和set几乎一样，唯一的区别是，multiset允许键值重复。因此set使用底层RB-tree的insert\_unique()实现插入，而multiset插入采用的是RB-tree的insert\_equal()而非insert_unique()。 

### 4、map和multimap
map的特性可以归纳为以下几条： 
- （1）所有元素都会根据元素的键值自动被排序。 
- （2）map的所有元素都是pair，第一个值是键值，第二个是实值。 
- （3）map不允许两个元素拥有相同的键值。 
- （4）可以通过map的迭代器来改变元素的实值，但不可以改变键值，那样会违反元素的排列规则。 
- （5）在客户端对map进行插入或删除操作后，之前的迭代器依然有效。当然，被删除的元素的迭代器是个例外。 
- （6）它的底层机制是RB-tree。几乎所有的操作都只是转调用RB-tree的操作行为而已。

multimap和map几乎一样，唯一的区别是，multimap允许键值重复。因此map使用底层RB-tree的insert\_unique()实现插入，而multimap插入采用的是RB-tree的insert\_equal()而非insert_unique()。


### 5、hashtable
二叉搜索树具有对数平均时间的表现，但这样的表现依赖于一个假设：**输入的数据有足够的随机性**。本节要结束一种名为hash table(散列表)的数据结构，这种结构使得插入、删除、搜寻等操作上都具有“常数平均时间”的表现，而且这种表现以统计为基础，不依赖于输入元素的随机性。

#### 5.1 hashtable的散列函数和碰撞冲突问题
hashtable可以提供对任意有名项的存取和删除操作，这种结构的用意在于提供常数时间的的基本操作，而不依赖于插入元素的随机性，是以统计为基础的。

为了将特定键值key输入转为hash table的索引，就需要**散列函数hash function**，其主要负责将某一元素映射为一个”大小可接受之索引”。使用hash function带来的问题：可能有不同元素映射到相同的位置，即具有相同索引，这便是**碰撞或冲突问题**。

解决碰撞问题的方法常见的有线性探测、二次探测、开链等。**stl hashtable采用的hash方式是开链法**。
- （1）线性探测：当hash function计算出某个元素的插入位置，而该位置空间不再可用时，就循序往下一一寻找，直到找到一个可用空间为止。线性探测会造成**主集团问题**：平均插入成本的成长幅度，远高于**负载系数**的成长幅度。
- （2）二次探测：主要用来解决主集团问题。解决碰撞的方程式为F(i) = i^2。如果hash function计算出新元素的位置为H,而该位置实际上已被使用，那么就依次尝试H+1^2,H+2^2,H+3^2,H+4^2,....,H+i^2，而不像线性探测尝试的是H+1,H+2,H+3,H+4,....,H+i。**二次探测可以消除主集团，却可能造成次集团**：两个元素经hash function计算出来的位置若相同，则插入时所探测的位置也相同，形成某种浪费。消除次集团的方法如复式散列。
- （3）开链：这种做法是在每一个表格元素中维护一个list。hash function为选择某一个list，然后我们在那个list身上执行元素的插入、搜寻、删除等操作。若list够短，速度还是够快。使用开链法，表格的负载系数将大于1。


#### 5.2 STI STL的hashtable的数据结构
SGI STL中hash table使用的是开链法进行的冲突处理，其结构如图所示： 

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/21.png?raw=true)

**bucket所维护的linked list不采用STL的list或者slist，而是自行维护hash table node**。而至于**buckets聚合体，则使用vector来完成**，以便有动态扩容能力。

STL中的hash迭代器，是一种forward迭代器，只能++。有指向当前节点的指针和指向对应的vector的指针，没有后退操作，也就是没有所谓的逆向迭代器。有过next到list的尾端，就跳至下一个bucket。

设计思想：hashtable以质数来设计表格大小，预先计算好了28个质数，以备随时访问，大约都是两倍的关系递增，同时提供一个函数，查询28个质数中“最接近某数且大于某数”的质数作为vector的长度，如果需要重新分配，则分配下一个质数长度的vector。
**stl hash table扩张表格的触发条件是：当元素的数目大于或等于表格的大小**。（这个条件应该是为了保证常数操作时间，在统计基础上得出的）。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/22.png?raw=true)

insert分为insert\_unique和insert\_equal操作，前者保证插入的数不能有重复，后者可以插入键值相同的数。可以先用unique之后再用equal。insert\_unique：先调用resize函数，看是否需要增大vector，然后插入，vector的索引通过取余得到。

resize：如果已有元素的个数大于vector的size，需要根据得到的最新质数，分配新的空间，将在旧空间的元素，重新计算hash，复制到新的空间，最后旧空间与新空间swap一下即可。insert_equal:也是先调用resize，遍历找到和他相同的节点，在该节点的前面插入。

**hashtable有一些无法处理的型别，比如string，double，float。除非用户为那些型别写了相应的hash function**。


### 6、hash_set和hash_multiset
**hash\_set是以hashtable为底层机制**。因hash\_set所供应的操作接口，hashtable都提供了，所以几乎所有的hash_set操作行为，都只是转调用hashtable的操作行为。

set是为了能够快速搜寻元素。其底层是Rb-tree有自动排序功能，但hashtable没有自动排序功能，故hash\_set没有自动排序功能。hash\_set和set一样，元素的键值就是实值，实值就是键值。hash_set和set使用方式基本相同。

**hash\_multiset和multiset完全相同**，唯一差别是底层实现机制不同，hash\_multiset的底层实现机制是hashtable，multiset的底层实现机制是Rb-tree。**hash\_multiset和hash\_set实现上的唯一差别是**，hash\_set的插入操作采用hashtable中的insert\_unique()，而hash\_multiset的插入操作采用hashtable中的insert\_euqal()。hash\_multiset和hash_set使用方式基本相同。

### 7、hash_map和hash_multimap
hash\_map的底层实现机制也是hashtable。故hash\_map所供应的操作接口，hashtable都提供了，所以几乎所有的hash_map操作行为，都只是转调用hashtable的操作行为而已。

map能够根据键值快速搜索元素，其底层实现机制是Rb-tree，Rb-tree具有自动排序功能，故map具有自动排序功能，但hashtable没有自动排序功能，故hash\_map没有自动排序功能。hash\_map和map都有相同的特性，即每一个元素都同时拥有一个实值（value）和一个键值（key）。hash_map和map使用方式大体相同。

**hash\_multimap的特征与multimap完全相同**，唯一差别为它的底层实现机制是hashtable，故hash\_multimap的元素并不会被自动排序**。hash\_multimap和hash\_map实现上的唯一差别是**，hash\_multimap的插入操作使用底层机制hashtable中的insert\_equal()，而hahs\_map使用的是底层机制hashtable中的insert\_unique()。hash\_multimap使用方式与hash_map完全相同。

### 8、总结
在实际使用过程中，到底选择这几种容器中的哪一个？通常应该根据遵循以下原则： 
- （1）如果需要高效的随机存取，不在乎插入和删除的效率，使用vector； 
- （2）如果需要大量的插入和删除元素，不关心随机存取的效率，使用list； 
- （3）如果需要随机存取，并且关心两端数据的插入和删除效率，使用deque； 
- （4）如果打算存储数据字典，并且要求方便地根据key找到value，一对一的情况使用map，一对多的情况使用multimap； 
- （5）如果打算查找一个元素是否存在于某集合中，唯一存在的情况使用set，不唯一存在的情况使用multiset。