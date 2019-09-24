---
title: epoll的一些细节和注意事项
date: 2017-08-07 12:13:53
categories: 技术开发
tags: [网络编程]
---
`epoll_event结构`

``` cpp
struct epoll_event
{
  uint32_t events;  /* Epoll events */
  epoll_data_t data;    /* User data variable */
} __attribute__ ((__packed__));

typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;
```

`epoll_data是一个联合体`,有的网络库使用了fd字段，比如redis，有的使用了u32，比如libiop，个人认为在epoll_wait之后内核会自动移动
`epoll_event队列`的内容。因为`epoll_wait`返回就绪的文件描述符数量，之后我们采用循环从0到n,`从epoll_event队列中取出对应第i个
epoll_event结构，通过epoll_data中的fd或者u32回调找到用户自己封装的事件回调单元，调用对应的回调函数`。
<!--more-->
举一个例子假设epoll_event队列中有1000个文件描述符，第一次调用
epoll_wait返回5，那么表示队列前五个元素就绪了，如果不处理第三个就绪事件，其他的都处理。第二次调用epoll_wait会返回8,那么这8个epoll_event也是按顺序排列的。
所以认为epoll_wait这个函数做了内部的优化排序，返回给用户按顺序拍好的内存

看下epoll_wait 函数参数

``` cpp
#include <sys/epoll.h>

int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
int epoll_pwait(int epfd, struct epoll_event *events,
               int maxevents, int timeout,
               const sigset_t *sigmask);
```
`第二个参数表示epoll_event那个队列的首地址，第三个参数表示epoll_event队列的大小`。

我查看了下`manpage`对于epoll的说明

The epoll_wait() system call waits for events on the epoll(7) instance referred to by the file descriptor epfd. The memory area pointed to by events will contain the events that will be available for the caller. Up to maxevents are returned by epoll_wait(). The maxevents argument must be greater than zero.The timeout argument specifies the minimum number of milliseconds that epoll_wait() will block
大体意思是`epoll_wait 是系统等待处理  epfd所指向的 epoll实例。指向epoll_events的内存是可以被用户访问的`。

`epoll_wait最多返回maxevents大小`。而且maxevents必须大于0。`timeout 这个参数表示epoll_wait阻塞的最小毫秒数`。
![1.png](1.png)

The data of each returned structure will contain the same data the user set with an epoll_ctl(2) (EPOLL_CTL_ADD,EPOLL_CTL_MOD) while the events member will contain the returned event bit field.
`可以通过EPOLL_CTL_ADD或者EPOLL_CTL_MOD更改event的事件属性。`比如epoll_ctl(state->epfd,EPOLL_CTL_DEL,fd,&ee);
下面是对返回值的说明

When successful, epoll_wait() returns the number of file descriptors ready for the requested I/O, or zero if no file descriptor became ready during the requested timeout milliseconds. When an error occurs, epoll_wait() returns -1 and errno is set appropriately
当成功时epoll_wait返回就绪的I/O描述符个数，0表示超时，-1表示出错。

查看man手册给我们的一个例子

``` cpp
#define MAX_EVENTS 10
           struct epoll_event ev, events[MAX_EVENTS];
           int listen_sock, conn_sock, nfds, epollfd;

           /* Code to set up listening socket, 'listen_sock',
              (socket(), bind(), listen()) omitted */

           epollfd = epoll_create1(0);
           if (epollfd == -1) {
               perror("epoll_create1");
               exit(EXIT_FAILURE);
           }

           ev.events = EPOLLIN;
           ev.data.fd = listen_sock;
           if (epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {
               perror("epoll_ctl: listen_sock");
               exit(EXIT_FAILURE);
           }

           for (;;) {
               nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
               if (nfds == -1) {
                   perror("epoll_wait");
                   exit(EXIT_FAILURE);
               }

               for (n = 0; n < nfds; ++n) {
                   if (events[n].data.fd == listen_sock) {
                       conn_sock = accept(listen_sock,
                                          (struct sockaddr *) &addr, &addrlen);
                       if (conn_sock == -1) {
                           perror("accept");
                           exit(EXIT_FAILURE);
                       }
                       setnonblocking(conn_sock);
                       ev.events = EPOLLIN | EPOLLET;
                       ev.data.fd = conn_sock;
                       if (epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock,
                                   &ev) == -1) {
                           perror("epoll_ctl: conn_sock");
                           exit(EXIT_FAILURE);
                       }
                   } else {
                       do_use_fd(events[n].data.fd);
                   }
               }
           }
```
`同样是采用epoll_wait后返回的大小进行轮询，每次epoll_events回传，保证就绪的事件排在前边`，这样比select效率高，`因为select存在set传入和传出，epoll不需要传入，只需要用户开辟一段连续空间，并且将首地址告诉epoll_wait即可。并且需要轮询所有文件描述符，而不是只针对就绪的轮询`。

对于网络库，常常会封装一个供用户使用的EventBase 结构，例如libiop中

``` cpp
struct tag_iop_t
{
    int id;                    /*对应的id*/
    io_handle_t handle;        /*关联的句柄*/
    int iop_type;            /*对象类型：0：free,1:io,2:timer*/
    int prev;                /*上一个对象*/
    int next;                /*下一个对象*/
    unsigned int events;                /*关注的事件*/
    int timeout;            /*超时值*/
    iop_event_cb evcb;        /*事件回调*/
    void *arg;                /*用户指定的参数,由用户负责释放资源*/
    void *sys_arg;            /*系统指定的参数，系统自动释放资源*/
    /*以下字段对定时器无用*/
    dbuf_t *sbuf;        /*发送缓存区*/
    dbuf_t *rbuf;        /*接收缓存区*/
    iop_time_t last_dispatch_time;    /*上次调度的时间*/
};
```
例如redis中

``` cpp
typedef struct aeFileEvent {
     int mask; /* one of AE_(READABLE|WRITABLE) */ //文件事件类型 读/写
     aeFileProc *rfileProc;
     aeFileProc *wfileProc;
     void *clientData;
 } aeFileEvent;
```

我们大都会开辟一段内存空间, 例如

`tag_iop_t * ioplist = malloc(sizeof(struct tag_iop_t) * size );`

`当调用epoll_wait成功之后会返回n个就绪事件,从0到n遍历,对于第i个epoll_events[i],怎么找到应用层的数据节点tag_iop_t呢？`

`常用的做法就是将节点的标号存在epoll_data 的 u32字段，或者 将关联的socketfd存在epoll_data的 fd 字段`

通过如下伪代码可以实现类似的调用

``` cpp
int socketFd = epoll_events[i].epoll_data.fd;
ioplist[socketFd].callBack;
//或者根据节点回调

int node = epoll_events[i].epoll_data.u32;
ioplist[node].callBack;

//至于采用何种方式回调
//关键在于我们之前如何绑定iop的

//用socket下标绑定回调函数
ioplist[fd].callBack = ...;

//用node 序号绑定
//node是用户自己管理的从0到maxnum的数字
ioplist[node].fd = fd;
ioplist[node].callBack = ...;
```
这些都是读过一些网络库自己的理解，如果有什么好的建议请大家告诉我，一起进步吧。