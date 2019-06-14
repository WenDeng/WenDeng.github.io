---
title: 《STL源码剖析》第6章 算法     
date: 2019-5-20 14:32:12    
toc: true   
comments: true   
img: https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/23.png?raw=true    
tags:
  - STL源码
  - 技术   
categories:   
  - C++基础
---
STL算法部分主要由头文件<algorithm>,<numeric>,<functional>组成。要使用 STL中的算法函数必须包含头文件<algorithm>，对于数值算法须包含<numeric>，<functional>中则定义了一些模板类，用来声明函数对象。

<!--more-->

> 大部分内容摘自：https://www.cnblogs.com/linuxAndMcu/p/10264339.html

### 1、常见的算法种类
（1）质变与非质变算法：
- 质变算法-会改变操作对象的值。所有的STL算法都作用在[first,last)所标示的区间上，在运算过程中改变区间元素值。例如：copy,swap,replace,fill,remove,permulation,partition,sort等。
- 非质变算法-不改变操作对象之值。例如：find,search,count,equal,max,min等。

（2）STL中算法大致分为四类：
- 非可变序列算法：指不直接修改其所操作的容器内容的算法。
- 可变序列算法：指可以修改它们所操作的容器内容的算法。
- 排序算法：包括对序列进行排序和合并的算法、搜索算法以及有序序列上的集合操作。
- 数值算法：对容器内容进行数值计算。

细致分类可分为13类，由于算法过多，所以不一一做介绍，只选取几个最常用的算法介绍。

### 2、SGI STL中的常见算法
#### 2.1 查找算法
查找算法共13个，包含在<algorithm>头文件中，用来提供元素排序策略，这里只列出一部分算法：
- adjacent_find: 在iterator对标识元素范围内，查找一对相邻重复元素，找到则返回指向这对元素的第一个元素的ForwardIterator。否则返回last。重载版本使用输入的二元操作符代替相等的判断。
- count: 利用等于操作符，把标志范围内的元素与输入值比较，返回相等元素个数。
- count_if: 利用输入的操作符，对标志范围内的元素进行操作，返回结果为true的个数。
- binary_search: 在有序序列中查找value，找到返回true。重载的版本实用指定的比较函数对象或函数指针来判断相等。
- equal_range: 功能类似equal，返回一对iterator，第一个表示lower_bound，第二个表示upper_bound。
- find: 利用底层元素的等于操作符，对指定范围内的元素与输入值进行比较。当匹配时，结束搜索，返回指向该元素的Iterator。
- find_if: 使用输入的函数代替等于操作符执行find。
- search: 给出两个范围，返回一个ForwardIterator，查找成功指向第一个范围内第一次出现子序列(第二个范围)的位置，查找失败指向last1。重载版本使用自定义的比较操作。
- search_n: 在指定范围内查找val出现n次的子序列。重载版本使用自定义的比较操作。

```
#include "stdafx.h"
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>  

using namespace std;

int main(int argc, char* argv[])
{
    int iarr[] = { 0, 1, 2, 3, 4, 5, 6, 6, 6, 7, 8 };
    vector<int> iv(iarr, iarr + sizeof(iarr) / sizeof(int));

    /*** adjacent_find: 在iterator对标识元素范围内，查找一对相邻重复元素 ***/
    //原型： _FwdIt adjacent_find(_FwdIt _First, _FwdIt _Last)
    cout << "adjacent_find: ";
    cout << *adjacent_find(iv.begin(), iv.end()) << endl;

    /*** count: 利用等于操作符，把标志范围内的元素与输入值比较，返回相等元素个数。 ***/
    //原型： count(_InIt _First, _InIt _Last, const _Ty& _Val)
    cout << "count(==7): ";
    cout << count(iv.begin(), iv.end(), 6) << endl;//统计6的个数

    /*** count_if: 利用输入的操作符，对标志范围内的元素进行操作，返回结果为true的个数。 ***/
    //原型： count_if(_InIt _First, _InIt _Last, _Pr _Pred)
    //统计小于7的元素的个数 :9个
    cout << "count_if(<7): ";
    cout << count_if(iv.begin(), iv.end(), bind2nd(less<int>(), 7)) << endl;

    /*** binary_search: 在有序序列中查找value，找到返回true。 ***/
    //原型： bool binary_search(_FwdIt _First, _FwdIt _Last, const _Ty& _Val)
    cout << "binary_search: ";
    cout << binary_search(iv.begin(), iv.end(), 4) << endl; //找到返回true

    /*** equal_range: 功能类似equal，返回一对iterator，第一个表示lower_bound，第二个表示upper_bound。 ***/
    //原型： equal_range(_FwdIt _First, _FwdIt _Last, const _Ty& _Val)
    pair<vector<int>::iterator, vector<int>::iterator> pairIte;  
    pairIte = equal_range(iv.begin(), iv.end(), 3);
    cout << "pairIte.first:" << *(pairIte.first) << endl;//lowerbound 3   
    cout << "pairIte.second:" << *(pairIte.second) << endl; //upperbound 4

    /*** find: 利用底层元素的等于操作符，对指定范围内的元素与输入值进行比较。 ***/
    //原型： _InIt find(_InIt _First, _InIt _Last, const _Ty& _Val)
    cout << "find: ";
    cout << *find(iv.begin(), iv.end(), 4) << endl; //返回元素为4的元素的下标位置

    /*** find_if: 使用输入的函数代替等于操作符执行find。 ***/
    //原型： _InIt find_if(_InIt _First, _InIt _Last, _Pr _Pred)
    cout << "find_if: " << *find_if(iv.begin(), iv.end(), bind2nd(greater<int>(), 2)) << endl; //返回大于2的第一个元素的位置：3 

    /*** search: 给出两个范围，返回一个ForwardIterator，查找成功指向第一个范围内第一次出现子序列的位置。 ***/
    //原型： _FwdIt1 search(_FwdIt1 _First1, _FwdIt1 _Last1, _FwdIt2 _First2, _FwdIt2 _Last2)
    //在iv中查找 子序列 2 3 第一次出现的位置的元素   
    int iarr3[3] = { 2, 3 };
    vector<int> iv3(iarr3, iarr3 + 2);
    cout << "search: " << *search(iv.begin(), iv.end(), iv3.begin(), iv3.end()) << endl;

    /*** search_n: 在指定范围内查找val出现n次的子序列。 ***/
    //原型： _FwdIt1 search_n(_FwdIt1 _First1, _FwdIt1 _Last1, _Diff2 _Count, const _Ty& _Val)
    //在iv中查找 2个6 出现的第一个位置的元素   
    cout << "search_n: " << *search_n(iv.begin(), iv.end(), 2, 6) << endl;  

    return 0;
}

```

#### 2.2 排序和通用算法
排序算法共14个，包含在<algorithm>头文件中，用来判断容器中是否包含某个值，这里只列出一部分算法：
- merge: 合并两个有序序列，存放到另一个序列。重载版本使用自定义的比较。
- random_shuffle: 对指定范围内的元素随机调整次序。重载版本输入一个随机数产生操作。
- nth_element: 将范围内的序列重新排序，使所有小于第n个元素的元素都出现在它前面，而大于它的都出现在后面。重载版本使用自定义的比较操作。
- reverse: 将指定范围内元素重新反序排序。
- sort: 以升序重新排列指定范围内的元素。重载版本使用自定义的比较操作。
- stable_sort: 与sort类似，不过保留相等元素之间的顺序关系。

```
#include "stdafx.h"
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional> //定义了greater<int>()

using namespace std;

//要注意的技巧
template <class T>
struct display
{
    void operator()(const T&x) const
    {
        cout << x << " ";
    }
};

//如果想从大到小排序，可以采用先排序后反转的方式，也可以采用下面方法:
//自定义从大到小的比较器，用来改变排序方式
bool Comp(const int& a, const int& b) {
    return a > b;
}

int main(int argc, char* argv[])
{
    int iarr1[] = { 0, 1, 2, 3, 4, 5, 6, 6, 6, 7, 8 };
    vector<int> iv1(iarr1, iarr1 + sizeof(iarr1) / sizeof(int));
    vector<int> iv2(iarr1 + 4, iarr1 + 8); //4 5 6 6
    vector<int> iv3(15);

    /*** merge: 合并两个有序序列，存放到另一个序列 ***/
    //iv1和iv2合并到iv3中（合并后会自动排序）
    merge(iv1.begin(), iv1.end(), iv2.begin(), iv2.end(), iv3.begin());
    cout << "merge合并后: ";
    for_each(iv3.begin(), iv3.end(), display<int>());
    cout << endl;

    /*** random_shuffle: 对指定范围内的元素随机调整次序。 ***/
    int iarr2[] = { 0, 1, 2, 3, 4, 5, 6, 7, 8 };
    vector<int> iv4(iarr2, iarr2 + sizeof(iarr2) / sizeof(int));
    //打乱顺序  
    random_shuffle(iv4.begin(), iv4.end());
    cout << "random_shuffle打乱后: ";
    for_each(iv4.begin(), iv4.end(), display<int>());
    cout << endl;

    /*** nth_element: 将范围内的序列重新排序。 ***/
    //将小于iv.begin+5的放到左边   
    nth_element(iv4.begin(), iv4.begin() + 5, iv4.end());
    cout << "nth_element重新排序后: ";
    for_each(iv4.begin(), iv4.end(), display<int>());
    cout << endl;

    /*** reverse: 将指定范围内元素重新反序排序。 ***/
    reverse(iv4.begin(), iv4.begin());
    cout << "reverse翻转后: ";
    for_each(iv4.begin(), iv4.end(), display<int>());
    cout << endl;

    /*** sort: 以升序重新排列指定范围内的元素。 ***/
    //sort(iv4.begin(), iv4.end(), Comp); //也可以使用自定义Comp()函数
    sort(iv4.begin(), iv4.end(), greater<int>());
    cout << "sort排序（倒序）: ";
    for_each(iv4.begin(), iv4.end(), display<int>());
    cout << endl;

    /*** stable_sort: 与sort类似，不过保留相等元素之间的顺序关系。 ***/
    int iarr3[] = { 0, 1, 2, 3, 3, 4, 4, 5, 6 };
    vector<int> iv5(iarr3, iarr3 + sizeof(iarr3) / sizeof(int));
    stable_sort(iv5.begin(), iv5.end(), greater<int>());
    cout << "stable_sort排序（倒序）: ";
    for_each(iv5.begin(), iv5.end(), display<int>());
    cout << endl;

    return 0;
}
```
#### 2.3 删除和替换算法
删除和替换算法共15个，包含在<numeric>头文件中，这里只列出一部分算法：
- copy: 复制序列。
- copy_backward: 与copy相同，不过元素是以相反顺序被拷贝。
- remove: 删除指定范围内所有等于指定元素的元素。注意，该函数不是真正删除函数。内置函数不适合使用remove和remove_if函数。
- remove_copy: 将所有不匹配元素复制到一个制定容器，返回OutputIterator指向被拷贝的末元素的下一个位置。
- remove_if: 删除指定范围内输入操作结果为true的所有元素。
- remove_copy_if: 将所有不匹配元素拷贝到一个指定容器。

```
#include "stdafx.h"
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional> //定义了greater<int>()

using namespace std;

template <class T>
struct display
{
    void operator()(const T&x) const
    {
        cout << x << " ";
    }
};

int main(int argc, char* argv[])
{
    int iarr1[] = { 0, 1, 2, 3, 4, 5, 6, 7, 8 };
    vector<int> iv1(iarr1, iarr1 + sizeof(iarr1) / sizeof(int));
    vector<int> iv2(9);

    /*** copy: 复制序列 ***/
    // 原型： _OutIt copy(_InIt _First, _InIt _Last,_OutIt _Dest)
    copy(iv1.begin(), iv1.end(), iv2.begin());
    cout << "copy(iv2): ";
    for_each(iv2.begin(), iv2.end(), display<int>());
    cout << endl;

    /*** copy_backward: 与copy相同，不过元素是以相反顺序被拷贝。 ***/
    // 原型： _BidIt2 copy_backward(_BidIt1 _First, _BidIt1 _Last,_BidIt2 _Dest)
    copy_backward(iv1.begin(), iv1.end(), iv2.rend());
    cout << "copy_backward(iv2): ";
    for_each(iv2.begin(), iv2.end(), display<int>());
    cout << endl;

    /*** remove: 删除指定范围内所有等于指定元素的元素。 ***/
    // 原型： _FwdIt remove(_FwdIt _First, _FwdIt _Last, const _Ty& _Val)
    remove(iv1.begin(), iv1.end(), 5); //删除元素5
    cout << "remove(iv1): ";
    for_each(iv1.begin(), iv1.end(), display<int>());
    cout << endl;

    /*** remove_copy: 将所有不匹配元素复制到一个制定容器，返回OutputIterator指向被拷贝的末元素的下一个位置。 ***/
    // 原型：  _OutIt remove_copy(_InIt _First, _InIt _Last,_OutIt _Dest, const _Ty& _Val)
    vector<int> iv3(8);
    remove_copy(iv1.begin(), iv1.end(), iv3.begin(), 4); //去除4 然后将一个容器的元素复制到另一个容器
    cout << "remove_copy(iv3): ";
    for_each(iv3.begin(), iv3.end(), display<int>());
    cout << endl;

    /*** remove_if: 删除指定范围内输入操作结果为true的所有元素。 ***/
    // 原型： _FwdIt remove_if(_FwdIt _First, _FwdIt _Last, _Pr _Pred)
    remove_if(iv3.begin(), iv3.end(), bind2nd(less<int>(), 6)); // 将小于6的元素 "删除"
    cout << "remove_if(iv3): ";
    for_each(iv3.begin(), iv3.end(), display<int>());
    cout << endl;

    /*** remove_copy_if: 将所有不匹配元素拷贝到一个指定容器。 ***/
    //原型： _OutIt remove_copy_if(_InIt _First, _InIt _Last,_OutIt _Dest, _Pr _Pred)
    // 将iv1中小于6的元素 "删除"后，剩下的元素再复制给iv3
    remove_copy_if(iv1.begin(), iv1.end(), iv2.begin(), bind2nd(less<int>(), 4));
    cout << "remove_if(iv2): ";
    for_each(iv2.begin(), iv2.end(), display<int>());
    cout << endl;

    return 0;
}
```

- replace: 将指定范围内所有等于vold的元素都用vnew代替。
- replace_copy: 与replace类似，不过将结果写入另一个容器。
- replace_if: 将指定范围内所有操作结果为true的元素用新值代替。
- replace_copy_if: 与replace_if，不过将结果写入另一个容器。
- swap: 交换存储在两个对象中的值。
- swap_range: 将指定范围内的元素与另一个序列元素值进行交换。
- unique: 清除序列中重复元素，和remove类似，它也不能真正删除元素。重载版本使用自定义比较操作。
- unique_copy: 与unique类似，不过把结果输出到另一个容器。

```
#include "stdafx.h"
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional> //定义了greater<int>()

using namespace std;

template <class T>
struct display
{
    void operator()(const T&x) const
    {
        cout << x << " ";
    }
};

int main(int argc, char* argv[])
{
    int iarr[] = { 8, 10, 7, 8, 6, 6, 7, 8, 6, 7, 8 };
    vector<int> iv(iarr, iarr + sizeof(iarr) / sizeof(int));

    /*** replace: 将指定范围内所有等于vold的元素都用vnew代替。 ***/
    // 原型： void replace(_FwdIt _First, _FwdIt _Last, const _Ty& _Oldval, const _Ty& _Newval)
    //将容器中6 替换为 3   
    replace(iv.begin(), iv.end(), 6, 3);
    cout << "replace(iv): ";
    for_each(iv.begin(), iv.end(), display<int>()); //由于_X是static 所以接着 增长
    cout << endl; //iv:8 10 7 8 3 3 7 8 3 7 8   

    /*** replace_copy: 与replace类似，不过将结果写入另一个容器。 ***/
    // 原型： _OutIt replace_copy(_InIt _First, _InIt _Last, _OutIt _Dest, const _Ty& _Oldval, const _Ty& _Newval)
    vector<int> iv2(12);
    //将容器中3 替换为 5，并将结果写入另一个容器。  
    replace_copy(iv.begin(), iv.end(), iv2.begin(), 3, 5);
    cout << "replace_copy(iv2): ";
    for_each(iv2.begin(), iv2.end(), display<int>());  
    cout << endl; //iv2:8 10 7 8 5 5 7 8 5 7 8 0（最后y一个残留元素）   

    /*** replace_if: 将指定范围内所有操作结果为true的元素用新值代替。 ***/
    // 原型： void replace_if(_FwdIt _First, _FwdIt _Last, _Pr _Pred, const _Ty& _Val)
    //将容器中小于 5 替换为 2   
    replace_if(iv.begin(), iv.end(), bind2nd(less<int>(), 5), 2);
    cout << "replace_copy(iv): ";
    for_each(iv.begin(), iv.end(), display<int>());   
    cout << endl; //iv:8 10 7 8 2 5 7 8 2 7 8   

    /*** replace_copy_if: 与replace_if，不过将结果写入另一个容器。 ***/
    // 原型： _OutIt replace_copy_if(_InIt _First, _InIt _Last, _OutIt _Dest, _Pr _Pred, const _Ty& _Val)
    //将容器中小于 5 替换为 2，并将结果写入另一个容器。  
    replace_copy_if(iv.begin(), iv.end(), iv2.begin(), bind2nd(equal_to<int>(), 8), 9);
    cout << "replace_copy_if(iv2): ";
    for_each(iv2.begin(), iv2.end(), display<int>()); 
    cout << endl; //iv2:9 10 7 8 2 5 7 9 2 7 8 0(最后一个残留元素)

    int iarr3[] = { 0, 1, 2, 3, 4, 5, 6, 7, 8, };
    vector<int> iv3(iarr3, iarr3 + sizeof(iarr3) / sizeof(int));
    int iarr4[] = { 8, 10, 7, 8, 6, 6, 7, 8, 6, };
    vector<int> iv4(iarr4, iarr4 + sizeof(iarr4) / sizeof(int));

    /*** swap: 交换存储在两个对象中的值。 ***/
    // 原型： _OutIt replace_copy_if(_InIt _First, _InIt _Last, _OutIt _Dest, _Pr _Pred, const _Ty& _Val)
    //将两个容器中的第一个元素交换  
    swap(*iv3.begin(), *iv4.begin());
    cout << "swap(iv3): ";
    for_each(iv3.begin(), iv3.end(), display<int>());  
    cout << endl;

    /*** swap_range: 将指定范围内的元素与另一个序列元素值进行交换。 ***/
    // 原型： _FwdIt2 swap_ranges(_FwdIt1 _First1, _FwdIt1 _Last1, _FwdIt2 _Dest)
    //将两个容器中的全部元素进行交换  
    swap_ranges(iv4.begin(), iv4.end(), iv3.begin());
    cout << "swap_range(iv3): ";
    for_each(iv3.begin(), iv3.end(), display<int>());
    cout << endl;

    /*** unique: 清除序列中相邻的重复元素，和remove类似，它也不能真正删除元素。 ***/
    // 原型： _FwdIt unique(_FwdIt _First, _FwdIt _Last, _Pr _Pred) 
    unique(iv3.begin(), iv3.end());
    cout << "unique(iv3): ";
    for_each(iv3.begin(), iv3.end(), display<int>());
    cout << endl;

    /*** unique_copy: 与unique类似，不过把结果输出到另一个容器。 ***/
    // 原型： _OutIt unique_copy(_InIt _First, _InIt _Last, _OutIt _Dest, _Pr _Pred)
    unique_copy(iv3.begin(), iv3.end(), iv4.begin());
    cout << "unique_copy(iv4): ";
    for_each(iv4.begin(), iv4.end(), display<int>());
    cout << endl;

    return 0;
}

```

#### 2.4 排列组合算法
排列组合算法共2个，包含在<algorithm>头文件中，用来提供计算给定集合按一定顺序的所有可能排列组合，这里全部列出：
- next_permutation: 取出当前范围内的排列，并重新排序为下一个字典序排列。重载版本使用自定义的比较操作。
- prev_permutation: 取出指定范围内的序列并将它重新排序为上一个字典序排列。如果不存在上一个序列则返回false。重载版本使用自定义的比较操作。

```
#include "stdafx.h"
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

template <class T>
struct display
{
    void operator()(const T&x) const
    {
        cout << x << " ";
    }
};

int main(int argc, char* argv[])
{
    int iarr[] = { 12, 17, 20, 22, 23, 30, 33, 40 };
    vector<int> iv(iarr, iarr + sizeof(iarr) / sizeof(int));

    /*** next_permutation: 取出当前范围内的排列，并重新排序为下一个字典序排列。***/
    // 原型： bool next_permutation(_BidIt _First, _BidIt _Last)
    //生成下一个排列组合（字典序）   
    next_permutation(iv.begin(), iv.end());
    for_each(iv.begin(), iv.end(), display<int>());
    cout << endl;

    /*** prev_permutation: 取出指定范围内的序列并将它重新排序为上一个字典序排列。 ***/
    // 原型： bool prev_permutation(_BidIt _First, _BidIt _Last)
    prev_permutation(iv.begin(), iv.end());
    for_each(iv.begin(), iv.end(), display<int>());
    cout << endl;

    return 0;
}
```

#### 2.5 数值算法
数值算法共4个，包含在<numeric>头文件中，分别是：
- accumulate: iterator对标识的序列段元素之和，加到一个由val指定的初始值上。重载版本不再做加法，而是传进来的二元操作符被应用到元素上。
- partial_sum: 创建一个新序列，其中每个元素值代表指定范围内该位置前所有元素之和。重载版本使用自定义操作代替加法。
- inner_product: 对两个序列做内积(对应元素相乘，再求和)并将内积加到一个输入的初始值上。重载版本使用用户定义的操作。
- adjacent_difference: 创建一个新序列，新序列中每个新值代表当前元素与上一个元素的差。重载版本用指定二元操作计算相邻元素的差。

```
#include "stdafx.h"
#include <iostream>
#include <vector>
#include <numeric> //数值算法
#include <iterator> //定义了ostream_iterator

using namespace std;

int main(int argc, char* argv[])
{
    int arr[] = { 1, 2, 3, 4, 5 };
    vector<int> vec(arr, arr + 5);
    vector<int> vec2(arr, arr + 5);

    // accumulate: iterator对标识的序列段元素之和，加到一个由val指定的初始值上。
    int temp;
    int val = 0;
    temp = accumulate(vec.begin(), vec.end(), val);
    cout << "accumulate(val = 0): " << temp << endl;
    val = 1;
    temp = accumulate(vec.begin(), vec.end(), val);
    cout << "accumulate(val = 1): " << temp << endl;

    //inner_product: 对两个序列做内积(对应元素相乘，再求和)并将内积加到一个输入的初始值上。
    //这里是：1*1 + 2*2 + 3*3 + 4*4 + 5*5
    val = 0;
    temp = inner_product(vec.begin(), vec.end(), vec2.begin(), val);
    cout << "inner_product(val = 0): " << temp << endl;

    //partial_sum: 创建一个新序列，其中每个元素值代表指定范围内该位置前所有元素之和。
    //第一次，1   第二次，1+2  第三次，1+2+3  第四次，1+2+3+4
    ostream_iterator<int> oit(cout, " "); //迭代器绑定到cout上作为输出使用
    cout << "ostream_iterator: ";
    partial_sum(vec.begin(), vec.end(), oit);//依次输出前n个数的和
    cout << endl;
    //第一次，1   第二次，1-2  第三次，1-2-3  第四次，1-2-3-4
    cout << "ostream_iterator(minus): ";
    partial_sum(vec.begin(), vec.end(), oit, minus<int>());//依次输出第一个数减去（除第一个数外到当前数的和）
    cout << endl;

    //adjacent_difference: 创建一个新序列，新序列中每个新值代表当前元素与上一个元素的差。
    //第一次，1-0   第二次，2-1  第三次，3-2  第四次，4-3
    cout << "adjacent_difference: ";
    adjacent_difference(vec.begin(), vec.end(), oit); //输出相邻元素差值 后面-前面
    cout << endl;
    //第一次，1+0   第二次，2+1  第三次，3+2  第四次，4+3
    cout << "adjacent_difference(plus): ";
    adjacent_difference(vec.begin(), vec.end(), oit, plus<int>()); //输出相邻元素差值 后面-前面 
    cout << endl;

    return 0;
}
```

### 3、几个算法的具体实现原理
#### 3.1 next_permutation()的实现
该算法用于求区间序列的**下一个**排列组合，在当前序列中，从尾端往前寻找两个相邻元素，前一个记为\*i，后一个记为\*ii，并且满足\*i < \*ii。然后再从尾端寻找另一个元素\*j，如果满足\*i < *j，即将第i个元素与第j个元素对调，并将第ii个元素之后（包括ii）的所有元素颠倒排序，即求出下一个序列了。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/STL%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/23.png?raw=true)

代码实现：

```
template<class BidirectionalIterator>  
bool next_permutation(BidirectionalIterator first, BidirectionalIterator last)
{  
    if(first == last)  return false; //空序列  
  
    BidirectionalIterator i = first;  
    ++i;  
    if(i == last)  return false;  //一个元素，没有下一个序列了  
      
    i = last;  
    --i;  
  
    for(;;) 
    {  
        BidirectionalIterator ii = i;  //不断往前移动
        --i;  
        
        if(*i < *ii) //满足条件
        {  
            BidirectionalIterator j = last;  
            while(!(*i < *--j)); //找到*i<*j的情况  
            iter_swap(i, j);  //进行交换
            reverse(ii, last);  
            return true;  
        }  
          
        if(i == first) 
        {  
            reverse(first, last);  //全逆向，即为最小字典序列，如cba变为abc  
            return false;  
        }  
    }  
}  
```
#### 3.2 pre_permutation()的实现原理
该算法用于求区间序列的**上一个**排列组合，在当前序列中，从尾端往前寻找两个相邻元素，前一个记为\*i，后一个记为\*ii，直到满足\*i > \*ii。然后再从尾端寻找另一个元素\*j，如果满足\*i > *j，即将第i个元素与第j个元素对调，并将第ii个元素之后（包括ii）的所有元素颠倒排序，即求出上一个序列了。

#### 3.3 sort()的实现原理
STL中的sort并非只是普通的快速排序，**除了对普通的快速排序进行优化，它还结合了插入排序和堆排序**。根据不同的数量级别以及不同情况，能自动选用合适的排序方法。当数据量较大时采用快速排序，分段递归。一旦分段后的数据量小于某个阀值，为避免递归调用带来过大的额外负荷，便会改用插入排序。而如果递归层次过深，有出现最坏情况的倾

Quick Sort中任何一个元素都可以被选做枢轴，比如随机选取，固定选取。但是划分的两端越均匀，执行效率越高；如果其中一段长度为0，那就出现了最坏情况。最理想的选取方法是选取头、尾和中间三个元素中大小处于中间的那个元素作为枢轴。这样可以避免出现最坏情况（**三点中值法**）。

#### 3.4 partition()的实现原理
对[first, last)元素进行处理，使得满足p的元素移到[first, last)前部，不满足的移到后部，返回第一个不满足p元素所在的迭代器，如果都满足的话返回last。一元谓词可以用lambda表达式等。
```
template< class ForwardIt, class UnaryPredicate >
ForwardIt partition( ForwardIt first, ForwardIt last, UnaryPredicate p );
```
