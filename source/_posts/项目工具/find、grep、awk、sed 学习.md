---
title: find、grep、awk、sed命令学习
date: 2018-06-02 14:44:12
toc: true
comments: true
tags:
  - 技术
categories:
  - 项目工具
---

本文主要是总结了一下常见的文本操作命令，并进行了实践。
<!--more-->

### 1、find命令
find主要用于查找文件。

```
find /home/ -name "text.txt" //寻找名为text.txt的文件，会输出所有文件及其路径名。
find /home/ -name "*.txt"   //查找home目录下所有的.txt 字符串结尾的文件。
find /home/ -name "text.txt" | more //当显示的内容太多时，可以加上管道符more 进行分页查看 ，space就是翻页，enter就是下一行。

因为显示出来的既会包含文件，也会包含目录，所以要区分，通过-type

find /home/ -name "*.txt" -type d  //这个显示出来的都是目录directory
find /home/ -name "*.txt" -type f  //这个显示出来的都是文件file
find . "*.txt" -type f  //用.来表示相对路径，在当前目录下寻找。
find . "*.txt" -type f -mtime +30  //30天之前修改的文件
find . "*.txt" -type f -mtime -1 //1天内修改的文件
find . "*.txt" -type f -mtime -1 |xargs rm -rf {} \ //将find出来的内容替换到rm -rf后的大括号，通过xargs管道符进行连接。 这样的话就把find出来的文件全部删除了。
find . "*.txt" -type f -mtime -1 -exec cp -r {} \tmp\ \  // 把find出来的文件拷贝到\tmp\目录下。cp的-r：若给出的源文件是一个目录文件，此时将复制该目录下所有的子目录和文件。

```
> xargs:  只支持rm       
> exec:  支持 cp,mv,chmod,chown

```
find . "*.txt" -type f -mtime -1 -size +1k   //修改时间在一天内，同时大小大于1k的才会显示出来。千字节要小写k，兆要大写M。
find . "*.txt" -type f -mtime -1 -size +1k -perm 755 //加上权限为755的限制
find . -type f "*.log" -mtime +30 -exec rm -rf {} \  //删除服务器30天之前的日志
find . -type f -exec chmod -R 644 {} \  //在当前目录下把所有文件的权限改为644
```
关于文件目录权限的基础知识普及：
![](https://pic4.zhimg.com/v2-8c1176b648cab2dc138e19aea200cc0b_b.jpg)

### 2、grep命令
grep本意为过滤查询的意思,主要用于查找文件里的内容。

```
grep "root" /etc/passwd   //查询passwd文件里带有root字符串的行
grep --color "root" /etc/passwd    //把显示出来的root用颜色标记。
grep -n --color "root" /etc/passwd  //再加上行号显示。
grep -n --color "^root" /etc/passwd //找以root开头的行，就是在root前面加个尖括号。
grep -n --color "root$" /etc/passwd  //以root结尾的那一行
grep "#" /nginx.conf.default //只显示带有#号的行。
grep -v "#" /nginx.conf.default //把不包含#号的行显示出来,加了个-v
grep -v "#" /nginx.conf.default | grep -v "^$" //先去掉注释行，然后去掉空行
```
>注意参数： -i 是忽略大小写； -n是输出行号； -v是反向选择

下面要匹配一个IP地址： 主要是grep匹配正则表达式。
```
grep --color "[0-9][0-9]" text.txt  //匹配连续的数字，并加颜色显示
egrep --color "[0-9]{1}" text.txt //匹配一个数字。
egrep --color "[0-9]{1，3}\." text.txt //匹配1到3个连续数字加上 "." ,这里注意加上转义符 \ 。如果没有转义符，一个“.”代表任意字符。
egrep --color "[0-9]{1，3}\.[0-9]{1，3}\.[0-9]{1，3}\.[0-9]{1，3}" text.txt //这样就匹配了IP的4部分了，但是后面如果有多的数字的话也会匹配进来
egrep --color "[0-9]{1，3}\.[0-9]{1，3}\.[0-9]{1，3}\.[0-9]{1，3}$" text.txt //后面加上通配符$，表示以什么结尾，这样就会把后面有多余的排除掉。
egrep --color "([0-9]{1，3}\.){3}[0-9]{1，3}$" text.txt //这样加个小括号括起来就可以省略掉很多字符串了，大括号表示匹配的次数。
```
![](https://pic3.zhimg.com/v2-af23da0b2d45588fcdeaf3a5c101e096_b.jpg)


### 3、awk命令
awk主要用于数据统计，日志分析：
```
awk '{print $1}' /etc/passwd | more   //把文件的第一列打出来   
```
这里要**理解一列的意思，列是以空格作为分隔，没有空格，即使换行也是一列**。比如： 
```
netstat -ant 查看tcp连接
```
![](https://pic4.zhimg.com/v2-714df6e7c83da86974972091da38875f_b.jpg)

```
netstat -ant | awk '{print $6}' //即把第六列输出
```
结果如下：

![](https://pic1.zhimg.com/v2-43d2e5c2b2489352ddfcc2a030acfd2c_b.jpg)

大家注意列的概念。再比如说： 如下，想把每列的开始的用户名取出来。
![](https://pic3.zhimg.com/v2-bb85b4426ad5a73550ba866c8c7bcdaa_b.png)

这里采取的措施是把用户名后面的冒号去掉，就变成了列分隔了。如下：
```
awk -F: '{print $1}' /etc/passwd | head -5  //-F:即-F把后面的:换成" "。head -5取前5行
aawk -F: '{print $1":"$NF}' /etc/passwd | head -5   //$NF则是最后一列。即只输出第一列和最后一列。
```
![](https://pic4.zhimg.com/v2-25fda2d24e29813460d23c07d18d2ce3_b.jpg)
![](https://pic3.zhimg.com/v2-cc3a71211037db4f15814045f9a5dede_b.png)

**双引号在awk里面是添加的意思**，这样就在第一列和最后一列中间加上了冒号。这里可以添加任意字符。

现在进行一个简单的实例：比如ifconfig得到网卡信息如下：
![](https://pic2.zhimg.com/v2-26fa17f1404ccb8c4229c581781dde7d_b.jpg)

**然后想要把ip地址192.168.1.244输出并且变为 192-169-1-244**

第一步：

```
ifconfig | grep "inet"
```
![](https://pic2.zhimg.com/v2-e5569174feb22b11b35c4fa89fca2fe5_b.png)

然后把第一个和第三个去掉 
```
ifconfig | grep "inet"| grep -v "127" |grep -v "0.0.0.0"
```
![](https://pic3.zhimg.com/v2-5c140f1052fa2322ffc76ff525d6b0a6_b.png)

然后这里要区分出有几列，很明显，有4列。然后取出第二列。
```
ifconfig | grep "inet"| grep -v "127" |grep -v "0.0.0.0" | awk '{print $2}'
```
![](https://pic1.zhimg.com/v2-6e67847e1412b653c4dc3f8270a0bda0_b.png)

然后转换格式：

```
ifconfig | grep "inet"| grep -v "127" |grep -v "0.0.0.0" | awk '{print $2}' | awk -F. '{print $1"-"$2"-"$3"-"$4}'
```
![](https://pic4.zhimg.com/v2-3d8f932a65e3be34b16b26e3f4f21243_b.png)

把前面输出的东西当作主机名：也就是   hostname  \`名称\`

```
hostname  `  ifconfig | grep "inet"| grep -v "127" |grep -v "0.0.0.0" | awk '{print $2}' | awk -F. '{print $1"-"$2"-"$3"-"$4}'  `
```
注意，这里有个**反引号在外面包裹**，这个符号在esc下面的地方，我的是和 ~ 在一起的。



### 4、sed命令
sed主要用于获取文件内容并修改显示，先是初始文件：

![](https://pic2.zhimg.com/v2-70aec7aaa45221cca4974bdc0e05fa9d_b.jpg)

然后是我们的操作： 

```
sed 's/jackhe/xiaohong/' a.txt  //单引号后面的s表示字符串替换，即把jackhe替换为xiaohong.
```
最开始的  这里注意是有3个斜杠，而且它只是把文件的输出进行替换，并不进行文件的修改。

![](https://pic4.zhimg.com/v2-0058420a1b0ce8dbd237a791f68916e3_b.jpg)

但是上面的只会把第一个出现的修改，而如果需要修改后面的则需要如下： 
```
sed 's/jackhe/xiaohong/2' a.txt  //这是修改第二个。
sed 's/jackhe/xiaohong/g' a.txt  //g是全部替换
sed 's#jackhe#xiaohong#g' a.txt  //把斜杠改为井号也是可以的，这只是一个格式。
sed 's#jackhe#xiao///.hong#g' a.txt //也是可以的，这样#里面的都会变成字符处理。如果不是#号包裹，而是/，那么里面的/就需要进行转义，即 \/\/\/了

sed -i 's#jackhe#xiaohong#g' a.txt //如果要**真正的修改文件内容，那么就要加上个-i选项**。
```
**如果sed里面有变量，则必须要用双引号了**，不能使用单引号。