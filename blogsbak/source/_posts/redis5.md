---
title: 对于redis底层框架的理解（五）
date: 2017-08-07 12:25:11
categories: 技术开发
tags: [网络编程]
---
之前总结了redis的通讯流程，基本框架，epoll的封装等等，这次介绍下redis对于select模型的封装

``` cpp
//select 模型
typedef struct aeApiState {
    //读文件描述符集合，写文件描述符集合
    fd_set rfds, wfds;
    /* We need to have a copy of the fd sets as it's not safe to reuse
     * FD sets after select(). */
     //读写集合的副本
    fd_set _rfds, _wfds;
} aeApiState;
```
`_rfds和_wfds是读写结合的副本，因为select调用后会将读写集合中未就绪的文件描述符清除，所以每次用_rfds和_wfds传入，就不用担心原读写集合描述符被清除`。
<!--more-->
## 封装的基于select的初始化函数

``` cpp
static int aeApiCreate(aeEventLoop *eventLoop) {
    //开辟aeApiState空间
    aeApiState *state = zmalloc(sizeof(aeApiState));
    
    if (!state) return -1;
    //读写集合清零
    FD_ZERO(&state->rfds);
    FD_ZERO(&state->wfds);
    eventLoop->apidata = state;
    return 0;
}
```
`函数将读写集合清零，并且将state回传给eventloop的apidata部分`。

## 内存回收功能

``` cpp
//释放空间
static void aeApiFree(aeEventLoop *eventLoop) {
    zfree(eventLoop->apidata);
}
```
## 封装的添加和删除事件

``` cpp
//select 添加事件
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;

    if (mask & AE_READABLE) FD_SET(fd,&state->rfds);
    if (mask & AE_WRITABLE) FD_SET(fd,&state->wfds);
    return 0;
}

//select 删除事件
static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;

    if (mask & AE_READABLE) FD_CLR(fd,&state->rfds);
    if (mask & AE_WRITABLE) FD_CLR(fd,&state->wfds);
}
```

`添加事件函数将文件描述根据mask是读事件还是写事件放入不同的set`

`删除事件根据文件描述符mask是读事件还是写事件从不同的set中清除`

 

## 下面是核心功能，事件派发

``` cpp
//select 触发事件
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, j, numevents = 0;
    //将select读集合的数据拷贝到_rfds
    memcpy(&state->_rfds,&state->rfds,sizeof(fd_set));
       //将select写集合数据拷贝到_wfds
    memcpy(&state->_wfds,&state->wfds,sizeof(fd_set));

    //从读和写的copy集合里选出就绪的文件描述符
     retval = select(eventLoop->maxfd+1,
                &state->_rfds,&state->_wfds,NULL,tvp);

    //大于零表示有就绪的文件描述符
    
    if (retval > 0) {
        //select的弊端所在，每次都要将所有的文件描述符轮询一遍
        for (j = 0; j <= eventLoop->maxfd; j++) {
            
            int mask = 0;
            aeFileEvent *fe = &eventLoop->events[j];

            if (fe->mask == AE_NONE) continue;
            //aeFileEvent 事件可读
            if (fe->mask & AE_READABLE && FD_ISSET(j,&state->_rfds))
                mask |= AE_READABLE;

            //aeFileEvent 事件可写
            if (fe->mask & AE_WRITABLE && FD_ISSET(j,&state->_wfds))
                mask |= AE_WRITABLE;
            eventLoop->fired[numevents].fd = j;
            eventLoop->fired[numevents].mask = mask;
            numevents++;
        }
    }
    return numevents;
}
```
先将读写集合中的内容copy的_rfds和_wfds中，分别传入select函数中，

这样select后返回的_rfds中只有就绪的读socket,_wfds中只有就绪的写socket,通过FD_ISSET判断读写事件之后放到eventloop的fire队列里。

基本的封装就是这个样子，select模型相对容易理解