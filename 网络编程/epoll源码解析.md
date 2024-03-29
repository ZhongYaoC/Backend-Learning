#### EPOLL

epoll是Linux独有的io复用函数，其内部使用了红黑树提高效率（用在何处？？）





##### 相关函数

###### create函数

首先，epoll不需要像select一样，每次都传入文件描述符集合或事件集，epoll将这些事件放入内核的一个事件表中，对外需要创建一个额外的文件描述符，用以唯一的标识内核中的事件表，此描述符使用`epoll_create`创建。

```c
#include <sys/epoll.h>
int epoll_create( int size );
// size不起作用，只是给内核一个提示--“该事件表有多大”
// 返回的描述符将作为其他epoll系统调用的第一个参数，以指定要访问内核事件表
```



###### 操作函数

提供函数以操作内核的事件表

（即不再需要自己设置readset、writeset、errorset，而是通过函数向内核设置这些集合）

```c
#include <sys.epoll.h>
int epoll_ctl( int epfd, int op, int fd, struct epoll_event* event );
// 成功返回0，失败返回-1，并设置errno

// op的操作类型分为：
// EPOLL_CTL_ADD 往事件表中注册fd上的事件
// EPOLL_CTL_MOD 修改fd上的注册事件
// EPOLL_CTL_DEL 删除fd上的注册事件
```

```C
// event参数指定事件

struct epoll_event
{
  __uint32_t events; // epoll事件
  epoll_data_t data; // 用户数据
};

// 其中events成员描述事件，事件类型主要有EPOLLIN（数据可读）、EPOLLOUT（数据可写）、EPOLLERR（错误）、EPOLLET（边缘触发）、EPOLLONESHOT


// epoll_data成员用以存储用户数据，是个联合体
// 使用最多的是fd，指定事件所从属的目标文件描述符
// 而使用ptr可以指定与fd相关的用户数据，但是使用了ptr就不能使用fd，所以一般在ptr所指的用户数据中包含fd
typedef union epoll_data
{
 void* ptr;
 int fd;
 uint32_t u32;
 uint64_t u64;
}epoll_data_t;
```



###### 主要调用接口

```C
#include <sys/epoll.h>
int epoll_wait( int epfd, struct epoll_event* events, int maxevents, int timeout );
// 成功返回就绪的文件描述符个数，失败返回-1，并设置errno
// maxevents指定最多监听多少事件，必须大于0
// 如果检测到事件，会从epfd指定的内核事件表中将所有就绪事件复制到events指向的数组中，此参数仅用于输出，无需输入（浓浓的posix返回风格，返回值无任何实质意义，具体返回在参数中）
// 这也是和select最大的区别，从而无需对所有fd遍历检查是否就绪，直接遍历就绪事件即可
```



##### 水平触发 Level Trigger & 边缘触发 Edge Trigger

LT是epoll的默认工作模式，相当于一个效率更高的poll，当往内核事件表中注册一个文件描述符上的`EPOLLET`事件，epoll将会以ET模式操作该文件描述符，ET模式是epoll的高效工作模式。

（每个描述符可以设置各自的触发模式，模式不是全局的）

 区别：LT模式下的文件描述符，当epoll_wait检测到其上有事件发生并将此事件通知应用程序后，应用程序可以不立即处理该事件，或者可以不一次处理完全（如recv时没有一次接收到所有的报文），那么当应用程序下一次调用epoll_wait时，epoll_wait还会再次向应用程序通知此事件，知道此事件处理完毕。

> 如recv未能一次接收到所有报文，那么只要该fd的内核读缓冲区还有未读出的数据，就会再次触发就绪

而ET模式下的文件描述符，当epoll_wait检测到其上有事件发生并将此事件通知应用程序后，应用程序必须立即处理此事件，因为后续的epoll_wait不会再向应用程序通知此事件。

这意味着ET模式下同一个epoll事件的重复触发次数将会降低，从而更高效。

所以LT模式下，recv不需要置于while循环下，因为只要没处理完总会触发；而ET下就需要while以recv完全，否则下一次不会触发



##### EPOLLONESHOT

注册了该事件的文件描述符，操作系统最多触发一次其上注册的可读、可写或异常事件，且只触发一次，除非使用epoll_ctl重置该文件描述符上注册的EPOLLONESHOT事件。

所以，如果使用多线程处理就绪事件，那么当一个线程处理时，又有新的事件到达，不会触发，也就不会有别的线程处理，保证了只有一个线程处理；但是如果线程处理完之后，不重置EPOLLONESHOT，那么此文件描述符将不会触发新的就绪事件，所以为了后续处理，线程处理完毕之后，应立即重置EPOLLONESHOT事件。



##### 三组IO复用函数的比较

| 系统调用       | selcet                                   | poll                                    | epoll                                    |
| ---------- | ---------------------------------------- | --------------------------------------- | ---------------------------------------- |
| 事件集合       | 提供三个文件描述符集合fd_set，分别对应可读、可写、异常事件，也仅支持此三类事件；因为内核会更新集合，下次调用前必须重置三个集合 |                                         | 内核维护事件表，通过接口控制fd的对应事件；每次调用无需重置感兴趣的事件集合；epoll wait会返回就绪事件集合 |
| 返回事件       | 返回所有事件集合（就绪和未就绪）                         | 返回所有事件集合（就绪和未就绪）                        | 只返回就绪事件集合                                |
| 最大支持文件描述符数 |                                          | 65535                                   | 65535（系统允许的最大文件描述符数量）                    |
| 工作模式       | LT                                       | LT                                      | 支持ET                                     |
| 实现原理       | 轮询方式检测就绪事件（每次调用都扫描所有的注册文件描述符集合，然后返回），复杂度O(n) | 轮询方式检测就绪事件（每次调用都扫描所有的注册文件描述符集合），复杂度O(n) | 回调方式，内核检测到fd就绪，触发回调，回调将此fd上的对应事件插入内核就绪事件队列。然后内核在合适的时机将就绪事件队列拷贝至用户空间。因此无需轮询整个事件集合，时间复杂度O(1) |

不过当活动连接数量比较多时，epoll不一定比select、poll高效，因为回调将会被频繁调用，所以epoll适合连接数量多，但是活动连接数量较少的情况





