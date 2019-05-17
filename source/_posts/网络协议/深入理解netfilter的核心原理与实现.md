---
title: 深入理解netfilter的核心原理与实现
date: 2019-04-26 18:44:12
toc: true
comments: true
tags:
  - Iptabls
  - Netfilter
categories:
  - 网络协议
---

本文旨在一探Iptables和Netfilter的关系，了解网络包经过网络协议栈的过程，从而对linux的防火墙机制有更深入的认识。
<!--more-->

### 1、Iptables和Netfilter的关系
iptables是用户用来管理和配置防火墙规则的一种策略，但是实际解析规则并按照规则实施产生作用的是Netfilter。

iptables与协议栈内有包过滤功能的hook交互来完成工作，这些内核hook构成了netfilter框架。每个进入网络系统的包（接收和发送）在经过协议栈的时候都会触发这些hook，程序可以通过**注册hook函数**的方式在一些关键路径上处理网络流量。iptables相关的内核模块在这些hook注册了处理函数，因此可以通过iptables规则来使得网络流量符合防火墙规则。

### 2、Netfilter Hooks
netfilter提供了5个关于IPv4的hook点，数据包经过协议栈时会触发内核模块注册在这里的处理函数。触发哪个hook取决于包的方向（接收还是接收）、包的目的地址、以及包在上一个hook点是被丢弃还accept等等。

下面几个hook是内核协议栈已经定义好的：
* NF_IP_PRE_ROUTING: 接收到的包进入协议栈立即触发此个hook（刚刚进行完版本号，校验和等检测），在进行任何路由判断之前
* NF_IP_LOCAL_IN: 接收到的包经过路由判断，如果目的是本机，将触发此hook
* NF_IP_FORWARD: 接收到的包经过路由判断，如果目的是其他机器，将触发此hook
* NF_IP_LOCAL_OUT：本机产生的准备发送的包，在进入协议栈后立即触发此hook
* NF_IP_POST_ROUTING: 本机产生的准备发送的包或者转发的包，在经过路由的判断之后，将触发此hook

注册处理函数时必须提供优先级，以便hook触发能按照优先级高低调用处理函数，这使得多个模块可以在同一个hook点注册，并且有确定的处理顺序，内核模块会依次被调用，每次返回一个结果给netfilter框架，提示该对这包做什么操作。

### 3、Hooks和Iptables table and chain的关系
Iptable使用table来组织规则，分为以下5类table：
* Filter Table：是最常用的table之一，用于判断是否允许一个包通过。
* NAT Table: 用于实现网络地址转换规则。当包进入协议栈的时候，这些规则决定是否以及如何修改包的源/目的地址，以改变包被 路由时的行为。nat table通常用于将包路由到无法直接访问的网络。
* Mangle Table: 用于修改包的IP头。如可以修改包的TTL，增加或减少包可以经过的跳数。还可以对包打只在内核内有效的“标记”，后续的table或工具处理的时候可以用到这些标记。标记不会修改包本身，只是在包的内核表示上做标记。
* Raw Table：其功能非常有限，其唯一目的就是提供一个让包绕过连接跟踪的框架。
* Security Table：作用是给包打上SELinux标记，以此影响SELinux 或其他可以解读 SELinux 安全上下文的系统处理包的行为。这些标记可以基于单个包，也可以基于连接。

在每个table内部，规则被进一步组织成chain，内置的chain是由内置的hook触发的。chain基本上能决定规则何时被匹配。内置的chain名字和netfilter hook名字是一一对应的：
- PREROUTING: 由 NF_IP_PRE_ROUTING hook触发 ——————> raw,mangle,nat(目的)
- INPUT: 由 NF_IP_LOCAL_IN hook触发 ——————> mangle,filter,security,nat(源)
- FORWARD: 由 NF_IP_FORWARD hook触发 ——————> mangle,filter,security
- OUTPUT: 由 NF_IP_LOCAL_OUT hook触发 ——————> raw,mangle,nat,filter,security,nat(源)
- POSTROUTING: 由 NF_IP_POST_ROUTING hook触发 ——————> mangle，nat(源)

chain使管理员可以控制在包的传输路径上哪个点应用策略。因为每个table有多个chain，因此一个 table可以在处理过程中的多个地方施加影响。特定类型的规则只在协议栈的特定点有意义，因此并不是每个table都会在内核的每个hook注册chain。可以看出raw table只有两个链prerouting和output，分别在对应的hook点发挥作用。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/1.jpg?raw=true)

### 4、从IP协议栈入手
要想理解Netfilter的工作原理，必须从对Linux IP报文处理流程的分析开始，Netfilter正是将自己紧密地构建在这一流程之中的。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/2.jpg?raw=true)

#### 4.1 接收中断
如果网卡收到一个和自己MAC地址匹配或链路层广播的以太网帧，它就会产生一个中断。此网卡的驱动程序会处理此中断做入下处理：
- 从DMA/PIO或其他地方得到分组数据，写到内存里去；
- 接着，会分配一个新的套接字缓冲区skb，并调用与协议无关的、网络设备均支持的通用网络接收处理函数netif\_rx(skb)。netif_rx()函数让内核准备进一步处理skb。
- 然后，skb会进入到达队列以便CPU处理（对于多核CPU而言，每个CPU维护一个队列）。如果FIFO队列已满，就会丢弃此分组。在skb排队后，调用\_\_cpu\_raise\_softirq()标记NET\_RX\_SOFTIRQ 软中断，等待 CPU 执行。
- 至此， netif_rx() 函数调用结束，返回调用者状况信息（成功还是失败等）。此时，中断上下文进程完成任务，数据分组继续被上层协议栈处理。

流程：网卡收到一帧————>引发中断————>cpu调用相应的中断处理函数（指向此网卡驱动中的相应的处理函数）（把此packet读到ram中）————>呼叫netif\_rx函数来打上timestamp，并把此skb放入到cpu设置的队列中————>标记软中断（\_\_cpu\_raise_softirq）————>中断完成。 

#### 4.2 softirq
内核2.4以后，整个协议栈不再使用bottom half，而是被软中断softirq取代。软中断 softirq优势明显，可以同时在多个CPU上执行；而bottom half一次只能在一个CPU上执行，即在多个CPU执行时严格保持串行。

整个softirq机制的设计与实现中自始自终都贯彻了一个思想：“谁触发，谁执行 ”，也即触发软中断的那个CPU负责执行它所触发的软中断，而且每个CPU都由它自己的软中断触发与控制机制。这个设计思想也使得softirq机制充分利用了SMP系统的性能和特点。

#### 4.3 NET_RX_SOFTIRQ 网络接收软中断
这一阶段会根据协议的不同来处理数据分组。 CPU开始处理软中断do\_softirq()，接着 net\_rx\_action() 处理前面标记的NET\_RX\_SOFTIRQ ，把出对列的skb送入相应列表处理（根据协议不同到不同的列表）。比如，IP分组交给 ip\_rcv()处理， ARP分组交给arp\_rcv()处理等。



#### 4.4 处理IPv4分组
下面讲讲数据包到达网络层后所做的处理，整理流程如下图，从图中可以看到netfilter起作用的5个hooks。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/3.jpg?raw=true)

##### 4.4.1 上述处理的详细过程如下：
- ip\_rcv()函数验证IP分组，比如目的地址是否本机地址，校验和是否正确等。若正确，**则交给netfilter的NF\_IP\_PRE_ROUTING钩子**,否则丢弃。
- 到了ip\_rcv\_finish()函数，数据包就要根据skb结构的目的或路由信息各奔东西了。**ip\_local\_deliver**()处理到本机的数据分组、**ip\_forward**()处理需要转发的数据分组、**ip\_mr\_input**()转发组播数据包。如果是转发的数据包，还需要找出出口设备和下一跳。ip\_rcv\_finish()函数最后执行dst_input()，决定数据包的下一步的处理。

##### 4.4.2 转发数据包
转发数据包的主要流程如下：
- 处理IP头选项。如果需要的话，会记录本地IP地址和时间戳；
- 确认分组可以被转发；
- 将TTL减一，如果TTL为0 ，则丢弃分组；
- 根据 MTU 大小和路由信息，对数据分组进行分片，如果需要的话；
- 将数据分组送往外出设备。

如果由于某种原因，数据分组不能被转发，那么就回应 ICMP 消息来说明不能转发的原因。在对转发的分组进行各种检查无误后，执行 ip\_forward\_finish ，准备发送。然后执行dst\_output(skb) 。无论是转发的分组，还是本地产生的分组，都要经过dst\_output(skb) 到达目的主机。 IP 头在此时已经完成就绪。dst\_output(skb) 函数要执行虚函数 output（单播的话为ip\_output ，多播为ip\_mc\_output）。最后，ip\_finish_output 进入邻居子系统。

##### 4.4.3 数据包本地处理
数据包交给netfilter的IP\_LOCAL\_INPUT钩子,作相应处理，然后交给上层比如TCP进行下一步处理。TCP的处理过程如下：

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/5.jpg?raw=true)

ip\_queue\_xmit检查socket结构体中是否含有路由信息，如果没有则执行 ip\_route\_output_flow查找，并存储到sk数据结构中。如果找不到，则丢弃数据包。

数据最终到达驱动层，然后网卡再将数据发送出去。

### 5、Netfilter hook深入
Netfilter的主要工作其实将iptable对应的规则转换成对应nf_hoo_ops变量，然后进行注册从而发挥作用，接下来我们看一下具体过程。

#### 5.1 注册和注销Netfilter hook
注册一个hook函数是围绕nf\_hook\_ops数据结构的一个非常简单的操作，nf\_hook_ops数据结构在linux/netfilter.h中定义，该数据结构的定义如下：
```
struct nf_hook_ops {                    
        struct list_head list;  
        /* User fills in from here down. */
        nf_hookfn *hook;
        struct module *owner;   
        u_int8_t pf;    
        unsigned int hooknum;
        /* Hooks are ordered in ascending priority. */
        int priority;
};
```
- 该数据结构中的list成员用于维护Netfilter hook的列表，并且不是用户在注册hook时需要关心的重点。
- hook成员是一个指向nf\_hookfn类型的函数的指针，该函数是这个hook被调用时执行的函数。nf\_hookfn同样在linux/netfilter.h中定义。
- pf这个成员用于指定协议族。有效的协议族在linux/socket.h中列出，但对于IPv4我们希望使用协议族PF\_INET。
- hooknum这个成员用于指定安装的这个函数对应的具体的hook类型，其值为NF_IP_PRE_ROUTING等。
- priority这个成员用于指定在执行的顺序中，这个hook函数应当在被放在什么地方。对于IPv4，可用的值在linux/netfilter_ipv4.h的 nf_ip_hook_priorities 枚举中定义。出于示范的目的，在后面的模块中我们将使用NF_IP_PRI_FIRST。

注册一个Netfilter hook需要调用nf_register_hook()函数，以及用到一个nf_hook_ops数据结构。nf_register_hook()函数以一个nf_hook_ops数据结构的地址作为参数并且返回一个整型的值。以下提供的是一个示例代码，该示例代码简单的注册了一个丢弃所有到达的数据包的函数。该代码同时展示了Netfilter的返回值如何被解析。
```
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/version.h>
#include <linux/skbuff.h>
#include <linux/netfilter.h>
#include <linux/netfilter_ipv4.h>
 
MODULE_LICENSE("GPL");
MODULE_AUTHOR("xsc");
 
static struct nf_hook_ops nfho;
 
unsigned int hook_func(unsigned int hooknum,
                       struct sk_buff *skb,
                       const struct net_device *in,
                       const struct net_device *out,
                       int (*okfn)(struct sk_buff *))
{
	return NF_DROP;//丢弃所有数据包
}
 
static int kexec_test_init(void)
{
    printk("kexec test start ...\n");
 
    nfho.hook = hook_func;
    nfho.owner = NULL;
    nfho.pf = PF_INET;
    nfho.hooknum = NF_INET_LOCAL_OUT;
    nfho.priority = NF_IP_PRI_FIRST;
    
    nf_register_hook(&nfho);// 注册一个钩子函数
    return 0;
}
 
static void kexec_test_exit(void)
{
    printk("kexec test exit ...\n");
    nf_unregister_hook(&nfho); //注销钩子函数
}
 
module_init(kexec_test_init); //初始化
module_exit(kexec_test_exit); //退出处理
```
#### 5.2 hook函数实现
hook函数原型在linux/netfilter.h中给出，如下：
```
typedef unsigned int nf_hookfn(unsigned int hooknum,
                               struct sk_buff *skb,
                               const struct net_device *in,
                               const struct net_device *out,
                               int (*okfn)(struct sk_buff *));
```
- skb之后的两个参数是指向net_device数据结构的指针，net_device数据结构被Linux内核用于描述所有类型的网络接口。这两个参数中的第一个in，用于描述数据包到达的接口，毫无疑问，参数out用于描述数据包离开的接口。必须明白，在通常情况下，这两个参数中将只有一个被提供。例如：参数in只用于NF_IP_PRE_ROUTING和NF_IP_LOCAL_IN hook，参数out只用于NF_IP_LOCAL_OUT和NF_IP_POST_ROUTING hook。
- sk_buff数据结构中最有用的部分可能就是那三个描述传输层包头（例如：UDP, TCP, ICMP, SPX）、网络层包头（例如：IPv4/6, IPX, RAW）以及链路层包头（例如：以太网或者RAW）的联合(union)了。这三个联合的名字分别是h、nh以及mac。这些联合包含了几个结构，依赖于具体的数据包中使用的协议。
- 传递给hook函数的最后一个参数是一个命名为okfn函数指针，该函数以一个sk_buff数据结构作为它唯一的参数，并且返回一个整型的值。

#### 5.3 Netfilter报过滤技术实现
介绍几种过滤技术的实现：
- **基于接口进行过滤**:使用相应的net\_device数据结构的name这个成员，你就可以根据数据包的源接口和目的接口来选择是否丢弃它。如果想丢弃所有到达接口eth0的数据包，你需要做的仅仅是将in->name 的值与"eth0"做比较，如果名字匹配，那么hook函数简单的返回NF_DROP即可，数据包会被自动销毁。
- **基于地址进行过滤**:基于数据包的源或目的IP地址进行过滤也同样可以实现， 获取一个数据包的IP头通过使用sk\_buff数据结构中的网络层包头来完成。这个头位于一个联合中，可以通过sk_buff->nh.iph这样的方式来访问。如果数据包的源地址与我们设定的丢弃数据包的地址匹配，那么该数据包将被丢弃。
- **基于TCP端口进行过滤**:获取一个TCP头的指针是一件简单的事情,而可以分配一个tcphdr数据结构(在linux/tcp.h中定义)的指针，并将它指向我们的数据包中IP头之后的数据。如下代码：

```
static int check_tcp_packet(struct sk_buff *skb)
{
	struct sk_buff *sk = skb_copy(skb, 1);  
    struct tcphdr *tcph = NULL;  
    const struct iphdr *iph = NULL;  
    struct iphdr *ip;  
    __be16 dport;
  
    if (!skb)  
	return NF_ACCEPT;  
    ip = ip_hdr(sk);                                               
    iph = ip_hdr(skb);  
 
    if(ip->protocol == IPPROTO_TCP)       // TCP 协议
    {           
        tcph = (void *) iph + iph->ihl * 4;  // TCP 包头  
        dport = tcph->dest;                  // 目标端口  
        if(ntohs(dport) == 25 )
        {  
            return NF_DROP;  
        }
        else
        {  
            return NF_ACCEPT;  
        }  
    }  
    return NF_ACCEPT;  		
}

```


### 6、下一步延伸
更多更深的内容需要进一步学习linux内核，这里就不再细述了，关于Netfilter的hook攻击技术以及libpcap的通信隐藏等都挺有意思的，有时间不妨深入去实践一下。


### 参考链接
1.https://arthurchiao.github.io/blog/deep-dive-into-iptables-and-netfilter-arch-zh/    
2.https://www.ibm.com/developerworks/cn/linux/l-ntflt/index.html    
3.https://blog.csdn.net/cheng_fangang/article/details/8966242   
4.https://blog.csdn.net/XscKernel/article/details/8186679    
