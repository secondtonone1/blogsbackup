---
title: 对于redis框架的理解(三)
date: 2017-08-07 16:41:36
categories: 技术开发
tags: [网络编程]
---
上一篇讲完了`initServer`的大体流程，其中`aeCreateEventLoop（）`,这个函数没有详细说明，我们在这一篇里讲述`Ae.h和Ae.c`, 这里面的api阐述了如何创建

`eventLoop`和添加文件读写事件等等。

ae.h中的解释

//文件读写事件回调函数
`typedef void aeFileProc(struct aeEventLoop *eventLoop, int fd, void *clientData, int mask);`

//定时器回调函数
`typedef int aeTimeProc(struct aeEventLoop *eventLoop, long long id, void *clientData);`
//事件结束回调函数，析构一些资源
`typedef void aeEventFinalizerProc(struct aeEventLoop *eventLoop, void *clientData);`
//不是很清楚，应该是进程结束前做的回调函数
`typedef void aeBeforeSleepProc(struct aeEventLoop *eventLoop);`
<!--more-->
``` cpp
//文件事件回调函数
 typedef struct aeFileEvent {
     int mask; /* one of AE_(READABLE|WRITABLE) */ //文件事件类型 读/写
     aeFileProc *rfileProc;
     aeFileProc *wfileProc;
     void *clientData;
 } aeFileEvent;

 /* A fired event */
 typedef struct aeFiredEvent {
     int fd;    ////已出现的事件的文件号对应的事件描述在aeEventLoop.events[]中的下标
     int mask;  //文件事件类型 AE_WRITABLE||AE_READABLE
 } aeFiredEvent;

 typedef struct aeTimeEvent {
     long long id; /* time event identifier. */  //由aeEventLoop.timeEventNextId进行管理
     long when_sec; /* seconds */
     long when_ms; /* milliseconds */
     aeTimeProc *timeProc;
     aeEventFinalizerProc *finalizerProc;
     void *clientData;
     struct aeTimeEvent *next;
 } aeTimeEvent;

 /* State of an event based program */
 typedef struct aeEventLoop {
     int maxfd;   //监听的最大文件号  
     int setsize; //跟踪的文件描述符最大数量
     long long timeEventNextId;     //定时器事件的ID编号管理（分配ID号所用）
     time_t lastTime;     /* Used to detect system clock skew */
     aeFileEvent *events; //注册的文件事件，这些是需要进程关注的文件
     aeFiredEvent *fired;  //poll结果，待处理的文件事件的文件号和事件类型  
     aeTimeEvent *timeEventHead; //定时器时间链表
     int stop;   //时间轮询是否结束？
     void *apidata;  //polling API 特殊的数据
     aeBeforeSleepProc *beforesleep;  //休眠前的程序
 } aeEventLoop;
```
 /* Prototypes */
 //创建eventLoop结构
 `aeEventLoop *aeCreateEventLoop(int setsize);`
 //删除eventloop
 `void aeDeleteEventLoop(aeEventLoop *eventLoop);`
 //事件派发停止
 `void aeStop(aeEventLoop *eventLoop);`
 //添加文件读写事件
 `int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
     aeFileProc *proc, void *clientData);`
 //删除文件读写事件
 `void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask);`
 //获取文件事件对应类型(读或写)
 `int aeGetFileEvents(aeEventLoop *eventLoop, int fd);`
//创建定时器事件
 `long long aeCreateTimeEvent(aeEventLoop *eventLoop, 
     long long milliseconds,aeTimeProc *proc, void *clientData,
     aeEventFinalizerProc *finalizerProc);`
 //删除定时器事件
 `int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id);`
 //派发事件
 `int aeProcessEvents(aeEventLoop *eventLoop, int flags);`
 //等待millionseconds直到文件描述符可读或者可写
 `int aeWait(int fd, int mask, long long milliseconds);`
 //ae事件轮询主函数
 `void aeMain(aeEventLoop *eventLoop);`
 //获取当前网络模型
 `char *aeGetApiName(void);`
 //进程休眠前回调函数
 `void aeSetBeforeSleepProc(aeEventLoop *eventLoop, 
     aeBeforeSleepProc *beforesleep);`
 //获取eventloop所有的事件个数
 `int aeGetSetSize(aeEventLoop *eventLoop);`
 //重新设置eventloop事件个数
 `int aeResizeSetSize(aeEventLoop *eventLoop, int setsize);`

 ae.cpp中，一个函数一个函数解析

 

``` cpp
//定义了几个宏，根据不同的宏加载
//不同的网络模型
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
#ifdef HAVE_EPOLL
#include "ae_epoll.c"
#else
#ifdef HAVE_KQUEUE
#include "ae_kqueue.c"
#else
#include "ae_select.c"
#endif
#endif
#endif
```
 `aeCreateEventLoop,主要负责eventloop结构的创建和初始化，以及模型的初始化`

``` cpp
aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;
    int i;
    //创建eventloop
    if ((eventLoop = zmalloc(sizeof(*eventLoop))) == NULL) goto err;
    //为进程要注册的文件开辟空间
    eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);
    //为激活的要处理的文件开辟空间
    eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);
    //开辟失败报错
    if (eventLoop->events == NULL || eventLoop->fired == NULL) goto err;
    //设置监听事件总数
    eventLoop->setsize = setsize;
    //更新为当前时间
    eventLoop->lastTime = time(NULL);
    eventLoop->timeEventHead = NULL;
    eventLoop->timeEventNextId = 0;
    eventLoop->stop = 0;
    eventLoop->maxfd = -1;
    eventLoop->beforesleep = NULL;
    //将不同模式的api注册到eventloop里
    if (aeApiCreate(eventLoop) == -1) goto err;
    /* Events with mask == AE_NONE are not set. So let's initialize the
     * vector with it. */
    for (i = 0; i < setsize; i++)
        //将所有文件事件类型初始为空
        eventLoop->events[i].mask = AE_NONE;
    return eventLoop;

err:
    if (eventLoop) {
        zfree(eventLoop->events);
        zfree(eventLoop->fired);
        zfree(eventLoop);
    }
    return NULL;
}
```
``` cpp
//事件队列大小和重置 
 
//获取eventloop事件队列大小
int aeGetSetSize(aeEventLoop *eventLoop) {
    return eventLoop->setsize;
}

//重新设置大小
int aeResizeSetSize(aeEventLoop *eventLoop, int setsize) {
    int i;

    if (setsize == eventLoop->setsize) return AE_OK;
    if (eventLoop->maxfd >= setsize) return AE_ERR;
    //不同的网络模型调用不同的resize
    if (aeApiResize(eventLoop,setsize) == -1) return AE_ERR;
    //重新开辟空间
    eventLoop->events = zrealloc(eventLoop->events,sizeof(aeFileEvent)*setsize);
    eventLoop->fired = zrealloc(eventLoop->fired,sizeof(aeFiredEvent)*setsize);
    eventLoop->setsize = setsize;

    /* Make sure that if we created new slots, they are initialized with
     * an AE_NONE mask. */
    //重新初始化事件类型
    for (i = eventLoop->maxfd+1; i < setsize; i++)
        eventLoop->events[i].mask = AE_NONE;
    return AE_OK;
}
```
 `删除eventloop和stop事件轮询`

``` cpp
//删除eventloop结构
void aeDeleteEventLoop(aeEventLoop *eventLoop) {
    aeApiFree(eventLoop);
    zfree(eventLoop->events);
    zfree(eventLoop->fired);
    zfree(eventLoop);
}

//设置eventloop停止标记
void aeStop(aeEventLoop *eventLoop) {
    eventLoop->stop = 1;
}
```
 `创建监听事件`

``` cpp
//创建监听事件
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
                      aeFileProc *proc, void *clientData)
{
    //判断fd大于eventloop设置的事件队列大小
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }

    //取出对应的aeFileEvent事件
    aeFileEvent *fe = &eventLoop->events[fd];

    //添加读写事件到不同的模型
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    //文件类型按位或
    fe->mask |= mask;
    //根据最终的类型设置读写回调函数
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    //fe中读写操作的clientdata
    fe->clientData = clientData;
    //如果fd大于当前最大的eventLoop maxfdfd
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```
 `删除监听事件`

``` cpp
void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask)
{
    if (fd >= eventLoop->setsize) return;
    aeFileEvent *fe = &eventLoop->events[fd];
    if (fe->mask == AE_NONE) return;
    //网络模型里删除对应的事件
    aeApiDelEvent(eventLoop, fd, mask);
    //清除对应的类型标记
    fe->mask = fe->mask & (~mask);
    //如果删除的fd是maxfd，并且对应的事件为空，那么更新maxfd
    if (fd == eventLoop->maxfd && fe->mask == AE_NONE) {
        /* Update the max fd */
        int j;

        for (j = eventLoop->maxfd-1; j >= 0; j--)
            if (eventLoop->events[j].mask != AE_NONE) break;
        eventLoop->maxfd = j;
    }
}
```
``` cpp
//获取文件类型
int aeGetFileEvents(aeEventLoop *eventLoop, int fd) {
    if (fd >= eventLoop->setsize) return 0;
    aeFileEvent *fe = &eventLoop->events[fd];
    //返回对应的类型标记
    return fe->mask;
}
```
`事件派发函数`

``` cpp
//派发事件的函数
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    /* Nothing to do? return ASAP */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

   
    //为了休眠，直到有时间事件触发，即便是没有文件事件处理，我们也会
    //调用对应的事件时间
    //这部分不是很清楚，知道大体意思是设置时间，
    //为了aeApiPoll设置等待的时间
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;

        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);
        if (shortest) {
            long now_sec, now_ms;

            /* Calculate the time missing for the nearest
             * timer to fire. */
            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;
            tvp->tv_sec = shortest->when_sec - now_sec;
            if (shortest->when_ms < now_ms) {
                tvp->tv_usec = ((shortest->when_ms+1000) - now_ms)*1000;
                tvp->tv_sec --;
            } else {
                tvp->tv_usec = (shortest->when_ms - now_ms)*1000;
            }
            if (tvp->tv_sec < 0) tvp->tv_sec = 0;
            if (tvp->tv_usec < 0) tvp->tv_usec = 0;
        } else {
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }
        //调用不同的网络模型poll事件
        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            //轮询处理就绪事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;
            //可读就绪事件
            if (fe->mask & mask & AE_READABLE) {
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            //可写就绪事件
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
            processed++;
        }
    }
    /* Check time events */
    //处理所有定时器事件
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```
 

``` cpp
/等待millionseconds，直到有可读或者可写事件触发
int aeWait(int fd, int mask, long long milliseconds) {
    struct pollfd pfd;
    int retmask = 0, retval;

    memset(&pfd, 0, sizeof(pfd));
    pfd.fd = fd;
    if (mask & AE_READABLE) pfd.events |= POLLIN;
    if (mask & AE_WRITABLE) pfd.events |= POLLOUT;

    if ((retval = poll(&pfd, 1, milliseconds))== 1) {
        if (pfd.revents & POLLIN) retmask |= AE_READABLE;
        if (pfd.revents & POLLOUT) retmask |= AE_WRITABLE;
    if (pfd.revents & POLLERR) retmask |= AE_WRITABLE;
        if (pfd.revents & POLLHUP) retmask |= AE_WRITABLE;
        return retmask;
    } else {
        return retval;
    }
}                                    
```
``` cpp
//ae主函数
void aeMain(aeEventLoop *eventLoop) {
    //stop初始为0
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        //调用beforesleep函数
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        //派发所有的事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}

//获取api名字
char *aeGetApiName(void) {
    return aeApiName();
}

//sleep之前的回调函数
void aeSetBeforeSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *beforesleep) {
    eventLoop->beforesleep = beforesleep;
}
```
 

这就是ae文件里大体的几个api，其他的没理解的还在研究。