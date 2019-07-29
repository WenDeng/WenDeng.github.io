---
title: IO多路复用与epoll原理探究     
date: 2019-6-09 16:10:30    
toc: true   
comments: true   
img: https://github.com/WenDeng/Picture_markdown/blob/master/picture/%E6%9C%8D%E5%8A%A1%E7%AB%AF%E7%BC%96%E7%A8%8B/1.png?raw=true     
tags:
  - IO multiplexing
  - epoll原理
  - 技术   
categories:   
  - 服务端编程
---
本文在引入五种IO模型的基础上，介绍常见的三种IO multiplexing技术select、poll和epoll的用法，并总结分析三者的应用场景和优缺点0，最后详细介绍epoll高效的原因。

<!--more-->
### 1、五种IO模型
服务器端编程经常需要构造高性能的IO模型，常见的IO模型有四种：
- （1）同步阻塞IO（Blocking IO）：即传统的IO模型。即用户需要等待read将socket中的数据读取到buffer后，才继续处理接收的数据。整个IO请求的过程中，用户线程是被阻塞的，这导致用户在发起IO请求时，不能做任何事情，对CPU的资源利用率不够。
- （2）同步非阻塞IO（Non-blocking IO）：默认创建的socket都是阻塞的，非阻塞IO要求socket被设置为NONBLOCK。虽然用户线程每次发起IO请求后可以立即返回，但是为了等到数据，仍需要不断地轮询、重复请求，消耗了大量的CPU的资源。一般很少直接使用这种模型，而是在其他IO模型中使用非阻塞IO这一特性。
- （3）IO多路复用（IO Multiplexing）：即经典的Reactor设计模式，主要作用是可以避免同步非阻塞IO模型中轮询等待的问题，可达到在同一个线程内同时处理多个IO请求的目的。。
- （4）信号驱动 I/O（ signal driven IO）：信号驱动IO在实际中并不常用。
- （5）异步IO（Asynchronous IO）：即经典的Proactor设计模式，也称为异步非阻塞IO。

需要记住的是上述**前四种IO都是同步IO**。对于一次read IO访问，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。

**同步和异步的主要区别在于谁负责拷贝数据**：同步方式下由用户线程将数据拷贝到用户空间；异步方式下由内核负责将数据拷贝到用户空间，拷贝完成后会通知用户线程或者调用用户线程注册的回调函数进行后续处理。

**阻塞和非阻塞的主要区别在于是否需要等待完成**：阻塞和非阻塞主要是针对同步方式，阻塞是指IO操作需要彻底完成后才返回到用户空间；而非阻塞是指IO操作被调用后立即返回给用户一个状态值，无需等到IO操作彻底完成。

### 2、IO multiplexing技术
I/O多路复用（multiplexing）的本质是通过一种机制（系统内核缓冲I/O数据），让单个进程可以监视多个文件描述符，一旦某个描述符就绪（读就绪或写就绪），就通知程序进行相应的读写操作。

select、poll和epoll都是Linux API提供的IO复用方式。注意**select，poll，epoll本质上都是同步I/O**，因为这些就绪的IO上的数据都由用户线程进行拷贝。

下面主要讲select、poll、epoll的用法。
#### 2.1 select函数
select系统调用函数介绍：
```
int select(int maxfd,fd_set *readset,fd_set *writeset,fd_set *exceptset,const struct timeval *timeout);
//maxfd表示需要监听的套接字最大描述符+1  
//fd_set是一个文件描述符集合。三个参数分别表示监听读、写和异常文件描述符集合，可以设为空指针。
//timeout设定等待时间，timeval结构用于指定这段时间的秒数和微秒数，可以精确到微秒。
//返回值：有就绪描述符就返回其数目，若超时则为0，若出错则为-1

FD_ZERO(fd_set *fdset) //将指定的文件描述符集清空，必须进行初始化。
FD_SET(fd_set *fdset) //用于在文件描述符集合中增加一个新的文件描述符。
FD_CLR(fd_set *fdset) //用于在文件描述符集合中删除一个文件描述符。
FD_ISSET(int fd,fd_set *fdset) //用于测试指定的文件描述符是否在该集合中。
```

**select运行机制**  
select()的机制中提供一种fd\_set的数据结构，实际上是一个long类型的数组，每一位代表一个对应的文件描述符，通过宏进行添加和删除。当调用select()时，由内核根据IO状态修改fd_set的内容，由此来返回那些就绪IO。

相比同步阻塞模型，select怎加了添加监听文件描述符以及调用select函数的额外操作，还有返回后的遍历代价。其最大的优势是用户可以在一个线程内同时处理多个socket的IO请求。

**select机制的缺点：**    
- （1）每次调用select，都需要把fd\_set集合从用户态拷贝到内核态，如果fd_set集合很大时，那这个开销也很大，比如百万连接却只有少数活跃连接时这样做就太没有效率。
- （2）每次调用select都需要在内核遍历传递进来的所有fd\_set，如果fd_set集合很大时，那这个开销也很大。
- （3）为了减少数据拷贝带来的性能损坏，内核对被监控的fd_set集合大小做了限制，一般为1024，如果想要修改会比较麻烦，可能还需要编译内核。
- （4）每次调用select之前都需要遍历设置监听集合，重复工作。

#### 2.2 poll函数
poll的机制与select类似，与select在本质上没有多大差别，管理多个描述符也是进行轮询，根据描述符的状态进行处理，但是**poll没有最大文件描述符数量的限制**。也就是说，poll只解决了上面的问题3，并没有解决问题1，2的性能开销问题。
```
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
//nfds记录数组fds中描述符的总数量
//返回值表示fds集合中就绪描述符数量，返回0表示超时，返回-1表示出错；

typedef struct pollfd {
        int fd;                         // 需要被检测或选择的文件描述符
        short events;                   // 对文件描述符fd上感兴趣的事件
        short revents;                  // 文件描述符fd上当前实际发生的事件
} pollfd_t;
```

#### 2.3 epoll函数
epoll在Linux2.6内核正式提出，是基于事件驱动的I/O方式，相对于select来说，**epoll没有描述符个数限制**，使用一个文件描述符管理多个描述符，将用户关心的文件描述符的事件存放到内核的一个事件表中，这样在**用户空间和内核空间只需copy一次**。
```
struct epoll_event {
    __uint32_t events;  /* Epoll events */
    epoll_data_t data;  /* User data variable */
};

typedef union epoll_data {
    void *ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;


int epoll_create(int size);
//epoll_create 函数创建一个epoll句柄，参数size表明内核要监听的描述符数量，失败时返回-1。

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
//epoll_ctl函数用于注册要监听的事件类型，带四个参数。
//epfd 表示epoll句柄；op 表示fd操作类型，包括EPOLL_CTL_ADD、EPOLL_CTL_MOD、EPOLL_CTL_DEL三种；
//fd 是要监听的描述符；event 表示要监听的事件

int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
// epoll_wait函数用于等待就绪事件，成功时返回就绪的事件数目，调用失败时返回 -1，等待超时返回 0。
// epfd是epoll句柄；events表示从内核得到的就绪事件集合；
// maxevents告诉内核events的大小；timeout表示等待的超时事件。
```
可以看出上述epoll机制的好处在于：分清了频繁调用和不频繁调用的操作。例如，epoll\_ctrl是不太频繁调用的，而epoll\_wait是非常频繁调用的。这时，epoll_wait却几乎没有入参，这比select的效率高出一大截，而且，它也不会随着并发连接的增加使得入参越发多起来，导致内核执行效率下降。


### 3、epoll的LE和ET
epoll除了提供select/poll那种IO事件的水平触发（Level Triggered）外，还提供了边缘触发（Edge Triggered），这就使得用户空间程序可以通过记录IO状态，从而减少epoll\_wait的调用，提高应用程序效率。

- **水平触发（LT）**：默认工作模式，表示当epoll\_wait检测到某描述符事件就绪并通知应用程序时，应用程序可以不立即处理该事件；下次调用epoll_wait时，会再次通知此事件。
- **边缘触发（ET）**： 当epoll\_wait检测到事件就绪并通知应用程序时，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次通知此事件。边缘触发只在状态由未就绪变为就绪时只通知一次。

ET和LE容易走向两个极端，LT会在你不能处理读或写时不断epoll_wait返回告诉你可读可写从而浪费不必要的时间；而ET则可能会在你想读或想写时由于错过第一次时机从而获取不到对应的响应。

总而言之，一般应用场景上两者的性能不会有什么大的差距，**ET的优点在于epoll\_wait的调用次数会减少一些**，某些场景下连接在不必要唤醒时不会被唤醒（此唤醒指epoll_wait返回）。但实际上，这不单纯是一个网络问题，而跟应用场景相关，虽然大部分开源框架都是基于ET写的，但框架追求的是纯技术问题，力求尽善尽美，与实际应用还是有区别的。

> 小结：LT模式和ET模式各有优缺点，无所谓孰优孰劣。使用 LT 模式，我们可以自由决定每次收取多少字节（对于普通 socket）或何时接收连接（对于侦听 socket），但是可能会导致多次触发；使用ET模式，我们必须每次都要将数据收完（对于普通socket）或必须理解调用accept接收连接（对于侦听socket），其优点是触发次数少。

> 应用场景：
> * (1) 读频次少，每次数据很多：如果LT和ET模式下的缓冲区足够大，那么两种模式没有区别。但是如果缓冲区比较小，那么很明显应该用LT模式，而且也方便控制读入数据的量或者甚至推迟读数据。
> * (2) 写数据少，但频次多：对于写，LT为了避免频繁触发epoll_wait,每次写开始和写完后向epoll注册和注销事件，如果频次多，那就不太好，即使频次少，多次调用epoll_ctrl也会带来开销。与之对应的是，ET模式在写数据情况下表现很好。



### 4、epoll的高效原理与内核管理机制
通过前面两节的知识，我们知道epoll高效的一个原因在于ET机制的引入减少epoll_wait的调用此时，而poll相对select优势在于不用重复遍历设置监听文件描述符集合，而epoll相对poll和select的优势在于不用来回在内核和用户空间copy监听集合，能快速返回活跃IO集合。

说到epoll都夸赞它的效率和并发量，那么它好在哪里呢？   
**epoll的核心数据结构在于红黑树+双向链表**，首先调用epoll\_create时内核帮我们在epoll文件系统里建了个file结点；除此之外在内核cache里建立红黑树用于存储以后epoll\_ctl传来的socket，当有新的socket连接来时，先遍历红黑书中有没有这个socket存在，如果有就立即返回，没有就插入红黑数，然后给内核中断处理程序注册一个钩子函数，每当有事件发生时就通过钩子函数把这些文件描述符放到用来存储就绪事件的链表中。**epoll_wait并不监听文件句柄，而是等待就绪链表不空or收到信号or超时这三种条件后返回**。

epoll_wait返回时，会将就绪链表上的事件摘除，在LT模式下，这些就绪socke事件会再次被放回到刚刚清空的准备就绪链表，保证所有的事件都得到正确的处理。如果到timeout时间后链表中没有数据也立刻返回。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/%E6%9C%8D%E5%8A%A1%E7%AB%AF%E7%BC%96%E7%A8%8B/1.png?raw=true)

对于epoll，需要建立文件系统，包括红黑树和链表代价会比较高，同时回调机制也会在fd活跃数目较多的情况下被反复调用，效率反而不高。所以：**当监测的fd数目较小，或者fd数目多且各个fd都比较活跃，建议使用select或者poll；当监测的fd数目非常大，成千上万，且单位时间只有其中的一部分fd处于就绪状态，这个时候使用epoll能够明显提升性能**，比如ngix web服务器就是使用epoll实现的。


### 5、select、poll和epoll的应用场景
我们知道epoll的优势非常明显，几乎没有描述符数量的限制，并发支持完美，不会随着socket的增加而降低效率，也不用在内核空间和用户空间之间做无效的copy操作。但是是不是所有的场景都适合epoll呢？

 > 一个游戏服务器，tcpserver负责接收客户端的连接，dbserver负责处理数据信息，一个webserver负责处理服务器的web请求，gameserver负责游戏的逻辑处理，所有这些服务都和另外一个gateserver相连，gateserver负责服务器间的通信和转发（进程间通信），只要游戏服务器在服务状态，这些连接几乎不会断开（异常情况可能会断开），并且这些连接数量一般不会很多。
 
 > 这种情况，gateserver是选择select还是epoll呢？很明显是select，因为每时每刻这些连接的socket都有事件发生（比如：服务期间的心跳信息，还有大型网络游戏的同步信息（一般每秒在20-30次）），最重要的是，这种场景下，并发量也不会很大。如果此时用epoll，为此所建立的文件系统，红黑书和链表对于此来说就是杀鸡用牛刀，效率反而不高。
 
 > 但是这里的tcpserver负责大量的客户端的连接，毫无疑问epoll是首选，它接受大量的客户端连接，收到客户端的消息之后把消息转发发给select网络模型的gateserver，gateserver再转发给gameserver进行逻辑处理，最后返回给客户端就over了。
 
因此在如果在并发量低，socket都比较活跃的情况下，select就不见得比epoll慢了。



### 6、总结
总结上述的知识可以看出，epoll建立红黑树和链表、调用回调函数都需要开销，适用于高并发而活跃连接较少的情况。select和poll的代价在于用户空间与内核态的数据拷贝和遍历处理，适用于连接量较少但其中大多数都比较活跃的情况。

![image](https://github.com/WenDeng/Picture_markdown/blob/master/picture/%E6%9C%8D%E5%8A%A1%E7%AB%AF%E7%BC%96%E7%A8%8B/2.png?raw=true)


