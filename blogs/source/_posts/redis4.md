---
title: 对于redis框架的理解（四）
date: 2017-08-07 16:16:07
categories: 技术开发
tags: [网络编程]
---
上一篇讲述了`eventloop`的结构和创建，添加文件事件删除文件事件，派发等等。

而eventloop主要就是调用不同`网络模型`完成事件监听和派发的。

这一篇主要讲述`epoll网络模型`，redis是如何封装和调用的

下面是`epoll_event`的结构

``` cpp
/*
    epoll_event 结构

    struct epoll_event
    {
        uint32_t events; //epoll_event 要注册的事件类型
        epoll_data_t data; //User data  //联合体用于存储用户要保存的数据
    }

    typedef union epoll_data
    {
        void * ptr;
        uint32_t u32;
        uint64_t u64;
        int fd;  //一般存储accept后生成的socketfd
        
    }epoll_data_t 

*/
```
<!--more-->
`Ae_epoll.c`文件中回传的数据结构

``` cpp
#include <sys/epoll.h>
//该结构用于回传eventLoop->apidata
typedef struct aeApiState {
    int epfd; //管理epoll事件表的句柄
    struct epoll_event *events; //epoll events的队列
} aeApiState;
``` 

`Ae_epoll.c中创建epoll句柄`

``` cpp
//epoll 创建epfd过程
static int aeApiCreate(aeEventLoop *eventLoop) {

    //开辟存储不同网络模型的数据块
    aeApiState *state = zmalloc(sizeof(aeApiState));

    if (!state) return -1;
    //开辟epoll_event * size 大小的空间，这段空间是连续的
    state->events = zmalloc(sizeof(struct epoll_event)*eventLoop->setsize);
    //开辟失败
    if (!state->events) {
        zfree(state);
        return -1;
    }
    //创建epfd，最多关注1024个文件描述符
    state->epfd = epoll_create(1024); /* 1024 is just a hint for the kernel */
    if (state->epfd == -1) {
        zfree(state->events);
        zfree(state);
        return -1;
    }
    
    // eventLoop->apidata数据回传
    eventLoop->apidata = state;
    return 0;
}
```
 
`Ae_epoll.c重新设置events队列大小`

``` cpp
//重新设置aeApiState大小
static int aeApiResize(aeEventLoop *eventLoop, int setsize) {
    aeApiState *state = eventLoop->apidata;

    state->events = zrealloc(state->events, sizeof(struct epoll_event)*setsize);
    return 0;
}
```
 

`Ae_epoll.c中释放内存和回收`

``` cpp
//释放aeApiState和 events 的内存
static void aeApiFree(aeEventLoop *eventLoop) {
    aeApiState *state = eventLoop->apidata;
    //关闭文件描述符
    close(state->epfd);
    //释放events的内存
    zfree(state->events);
    //释放aeApiState 的内存
    zfree(state);
}
```
 

`Ae_epoll.c添加读写事件或者更改读写事件的函数`

``` cpp
//epoll 注册事件，读或者写
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    //aeEventLopp 的数据域
    aeApiState *state = eventLoop->apidata;
    //epoll_event 事件
    struct epoll_event ee;
   
     //aeEventLoop 中注册的文件事件队列标志位如果不是AE_NONE，那么更改，否则添加
    int op = eventLoop->events[fd].mask == AE_NONE ?
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    //events读写事件清零
    ee.events = 0;
    //aeEventLoop 中注册的文件事件标志位进行融合
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    //如果是读事件，那么将epoll_event 注册读事件
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    //如果是写事件，那么将epoll_event 注册写事件
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.u64 = 0; /* avoid valgrind warning */
    //epoll_event 文件描述符
    ee.data.fd = fd;
    //将epoll事件注册到epoll的事件表里
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
    return 0;
}
```
 

`Ae_epoll.c中删除读写事件的函数`

 

``` cpp
static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int delmask) {
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee;
    //这是创建的逆过程
    //按位去反，按位&,即去掉相应的标志位
    int mask = eventLoop->events[fd].mask & (~delmask);

    ee.events = 0;
    //判断此时文件事件是读
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    //判断此时文件事件是写
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    
    ee.data.u64 = 0; /* avoid valgrind warning */
    ee.data.fd = fd;
    
    if (mask != AE_NONE) {
        //更改epoll_event的事件类型
        epoll_ctl(state->epfd,EPOLL_CTL_MOD,fd,&ee);
    } else {
        /* Note, Kernel < 2.6.9 requires a non null event pointer even for
         * EPOLL_CTL_DEL. */
         //删除epoll_event 事件
        epoll_ctl(state->epfd,EPOLL_CTL_DEL,fd,&ee);
    }
}
```
 

`事件派发函数`

``` cpp
//epoll 事件派发
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    //epoll wait 返回就绪状态的文件描述符，后面的结构体如果为空，那么说明阻塞，不为空表示等待多少秒后返回
    //下面是man手册的解释
    //Specifying a timeout of -1 makesepoll_wait(2) wait indefinitely, while specifying 
    //a timeout equal to zero makesepoll_wait(2) to return immediately 
    //even if no events are available (return code equal to zero)
    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    if (retval > 0) {
        int j;

        numevents = retval;
        //轮询处理已经就绪的文件描述符
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            //指针+j，表示每次便宜地址为j*epoll_event个字节
            struct epoll_event *e = state->events+j;

            //可读事件
            if (e->events & EPOLLIN) mask |= AE_READABLE;
            //可写事件
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            //处理错误发送给客户端
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            //对端正常关闭（程序里close()，shell下kill或ctr+c），
            //触发EPOLLIN和EPOLLRDHUP，但是不触发EPOLLERR和EPOLLHUP。
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;
            //添加到aeApiState 的就绪事件队列里
            eventLoop->fired[j].fd = e->data.fd;
            //就绪时间状态
            eventLoop->fired[j].mask = mask;
        }
    }
    return numevents;
}
```
``` cpp
//网络模型名字

static char *aeApiName(void) {
    return "epoll";
}
```

以上是封装的epoll结构和解释