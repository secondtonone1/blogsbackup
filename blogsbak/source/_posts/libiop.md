---
title: libiop通讯流程和api讲解
date: 2017-08-07 12:51:49
categories: 技术开发
tags: [网络编程]
---
上一篇讲到了`libiop基本结构`,这次根据libiop提供的test跟踪下消息和运行流程

``` cpp
void echo_server_test()
{
    int keepalive_timeout = 60;
    iop_base_t *base = iop_base_new(10240);
    printf("create a new iop_base_t object.\n"); 
    iop_add_tcp_server(base,"0.0.0.0",7777,
        my_echo_parser,my_echo_processor,
        my_echo_on_connect,my_echo_on_destroy,my_echo_on_error,
        keepalive_timeout);
    printf("create a new tcp server on port 7777.\n");
    printf("start iop run loop.\n");
    iop_run(base);
    
}
```
`echo_server_test` 函数内部添加了一个tcpserver,将函数一层一层展开,展开`iop_add_tcp_server`
<!--more-->
``` cpp
int iop_add_tcp_server(iop_base_t *base, const char *host, unsigned short port,
                       iop_parser parser, iop_processor processor,
                       iop_cb on_connect,iop_cb on_destroy,
                       iop_err_cb on_error, int keepalive_timeout)
{
    
    iop_tcp_server_arg_t *sarg = 0;
    io_handle_t h = INVALID_HANDLE;
    sarg = (iop_tcp_server_arg_t *)malloc(sizeof(iop_tcp_server_arg_t));
    if(!sarg){return -1;}
    memset(sarg,0,sizeof(iop_tcp_server_arg_t));
    h = iop_tcp_server(host,port);
    if(h == INVALID_HANDLE){return -1;}
    sarg = (iop_tcp_server_arg_t *)malloc(sizeof(struct tag_iop_tcp_server_arg_t));
    if(!sarg)
    {
        iop_close_handle(h);
        return -1;
    }
#ifdef WIN32
    strcpy_s(sarg->host,sizeof(sarg->host)-1, host);
#else
    strcpy(sarg->host,host);
#endif
    sarg->port = port;
    sarg->timeout = keepalive_timeout;
    sarg->on_connect = on_connect;
    sarg->on_destroy = on_destroy;
    sarg->on_error = on_error;
    sarg->parser = parser;
    sarg->processor = processor;
    _list_add_before(base->tcp_protocol_list_head, _list_node_new(sarg));         
    return iop_add(base,h,EV_TYPE_READ,_iop_tcp_server_cb,(void *)sarg,-1);
}
```

解读`iop_add_tcp_server`,函数参数`iop_base_t` 是`iop基本事件结构`，前面有说过， 

``` cpp
struct tag_iop_base_t
{
    iop_t *iops;        /*所有iop*/
    int maxio;            /*最大并发io数,包括定时器在内*/
    int maxbuf;            /*单个发送或接收缓存的最大值*/
    int free_list_head;    /*可用iop列表*/
    int free_list_tail; /*最后一个可用iop*/
    int io_list_head;    /*已用io类型的iop列表*/
    int timer_list_head;    /*已用timer类型的iop列表*/
    int connect_list_head;  /*异步连接的iop列表*/
    volatile int exit_flag;    /*退出标志*/

    int dispatch_interval;        /*高度的间隔时间*/
    iop_op_t op_imp;           /*事件模型的内部实现*/
    void *model_data;         /*事件模型特定的数据*/

    iop_time_t cur_time;        /*当前调度时间*/
    iop_time_t last_time;        /*上次调度时间*/
    iop_time_t last_keepalive_time; /*上次检查keepalive的时间*/

    _list_node_t * tcp_protocol_list_head;    /*use for advance tcp server model.*/
};
```

第二个参数`host是主机地址`，`port是端口号`，剩下的`parser为解析函数指针`,`processor为处理函数指针`，`on_connect为连接的回调函数指针`，`on_destroy,为

销毁功能函数指针`，`on_error为错误情况下函数指针`，`keepalive_timeout为超时时间`

这几个函数指针的类型如下，基本都类似的

``` cpp
/*tcp连接事件回调函数*/
typedef void (*iop_cb)(iop_base_t *,int,void *);
void iop_default_cb(iop_base_t *base, int id, void *arg);



/*
*   返回-1代表要删除事件,返回0代表不删除
*/
typedef int (*iop_err_cb)(iop_base_t *,int,int,void *);

int iop_default_err_cb(iop_base_t *base, int id, int err, void *arg);


/************************************
*协议解析器，
* parameters:
*    char *buf:数据
*    int len:数据长度
*return:
*    返回0代表还要收更多数据以代解析，-1代表协议错误，>0代表解析成功一个数据包
***********************************/
typedef int (*iop_parser)(char *, int);
int iop_default_parser(char *buf, int len);


/*
*数据处理器
*parameters:
*    base:iop_base_t 指针
*    id:iop对象的id
*    buf:数据包起始点
*    len:数据包长度
*    arg:自带的参数
*return:
    -1: 代表要关闭连接,0代表正常
*/
typedef int (*iop_processor)(iop_base_t *,int,char *,int,void *);
```
回到`iop_add_tcp_server函数`里 

开辟arg的空间

`sarg = (iop_tcp_server_arg_t *)malloc(sizeof(iop_tcp_server_arg_t));`
绑定端口和地址

`h = iop_tcp_server(host,port);`
对arg赋值
``` cpp
sarg->port = port;
    sarg->timeout = keepalive_timeout;
    sarg->on_connect = on_connect;
    sarg->on_destroy = on_destroy;
    sarg->on_error = on_error;
    sarg->parser = parser;
    sarg->processor = processor;
```
下面这句代码最重要

`iop_add(base,h,EV_TYPE_READ,_iop_tcp_server_cb,(void *)sarg,-1);
这句代码将socket h绑定了一个读事件，当有读事件就绪时会触发iop_tcp_server_cb这个函数`。

如何将h和iop_tcp_server_cb绑定的，展开iop_add

``` cpp
int iop_add(iop_base_t *base,io_handle_t handle,unsigned int events,iop_event_cb evcb,void *arg,int timeout)
{
    int r = 0;
    iop_t *iop = _iop_base_get_free_node(base);
    if(!iop){return -1;}
    iop->handle = handle;
    iop->events = events;
    iop->timeout = timeout;
    iop->evcb = evcb;
    iop->last_dispatch_time = base->cur_time;
    iop->arg = arg;
    //io 事件
    if(handle != INVALID_HANDLE)
    {
        //LOG_DBG("iop_add io, id=%d.\n", iop->id);
        iop->prev = -1;
        iop->next = base->io_list_head;
        base->io_list_head = iop->id;
        iop->iop_type = IOP_TYPE_IO;
            iop_set_nonblock(handle);
        r = (*(base->op_imp.base_add))(base, iop->id, handle, events);
        if(r != 0)
        {
            iop_del(base,iop->id);
            return -1;
        }
    }
    else
    {
        /*timer*/
        //LOG_DBG("iop_add timer, id=%d.\n", iop->id);
        iop->prev = -1;
        iop->next = base->timer_list_head;
        base->timer_list_head = iop->id;
        iop->iop_type = IOP_TYPE_TIMER;
    }
    return iop->id;    
}
```
iop_add 形参不做解释，其中形参evcb也是函数指针

`/*事件回调函数,返回-1代表要删除对象,返回0代表正常*/
typedef int (*iop_event_cb)(iop_base_t *,int,unsigned int,void *);`
`在iop_add内部完成iop回调函数evcb的绑定和基本参数赋值,然后判断是io事件还是定时器事件,对于IO事件，要通知网络层(epoll,select等不同模型)进行绑定，
调用base中op_imp成员的base_add函数指针完成绑定。`

`r = (*(base->op_imp.base_add))(base, iop->id, handle, events);`
之所以能调用是因为之前op_imp.base_add被赋值了。回到
``` cpp
void echo_server_test()
{
    int keepalive_timeout = 60;
    iop_base_t *base = iop_base_new(10240);
        ...
}

iop_base_t* iop_base_new(int maxio)
{
#ifdef _HAVE_EVENT_PORTS_
#endif
#ifdef _HAVE_WORKING_KQUEUE_
#endif
#ifdef _HAVE_EPOLL_
    return iop_base_new_special(maxio,"epoll");
#endif
#ifdef _HAVE_DEVPOLL_
#endif
#ifdef _HAVE_POLL_
    return iop_base_new_special(maxio,"poll");
#endif
#ifdef _HAVE_SELECT_
    return iop_base_new_special(maxio,"select");
#endif
    return NULL;
}
```
一层一层看

``` cpp
iop_base_t* iop_base_new_special(int maxio,const char *model)
{
    int r = -1;
    iop_base_t *base = NULL;
    if(strcmp(model,"epoll")==0)
    {
        base = _iop_base_new(maxio);
        if(base)
        {
            r = iop_init_epoll(base, maxio);
        }
    }
        ......  
}
```
``` cpp
int iop_init_epoll(void *iop_base, int maxev)
{
    ...
    //模型内部实现，不同模型不同的函数指针和名字
    iop_op->name = "epoll";
    iop_op->base_free = epoll_free;
    iop_op->base_dispatch = epoll_dispatch;
    iop_op->base_add = epoll_add;
    iop_op->base_del = epoll_del;
    iop_op->base_mod = epoll_mod;
    
    //1024 is not the max events limit.
    //创建epoll表句柄
   ...
    //iop_epoll_data_t类型的数据存在base的model_data里
    //方便回调
    base->model_data = iop_data;
    
    return 0;
}
```
上面就是在new函数里实现的一层一层函数指针的绑定，所以之后才可以调用对应的函数指针。

在iop_add 函数绑定成功后，整个iop_add_tcp_server流程走完了。

我们下一步看看如何`派发消息`

``` cpp
void echo_server_test()
{
    int keepalive_timeout = 60;
    iop_base_t *base = iop_base_new(10240);
    ... 
    iop_add_tcp_server(...,...);
    ...
    iop_run(base);
    
}
```
`iop_run函数完成消息轮询和派发`

``` cpp
void iop_run(iop_base_t *base)
{
    while(base->exit_flag == 0)
    {
        iop_dispatch(base);
    }
    iop_base_free(base);
}
```
 
`
iop_dispatch消息派发函数

iop_base_free iop_base释放
`
 

``` cpp
int iop_dispatch(iop_base_t *base)
{
    int cur_id = 0;
    int next_id = 0;
    int r = 0;
    iop_t *iop = (iop_t *)0;
    //调用不同模型的函数指针实现消息派发
    dispatch_imp_cb dispatch_cb = base->op_imp.base_dispatch;
    r = (*dispatch_cb)(base,base->dispatch_interval);
    if( r == -1)
    {
        return -1;
    }

    //检测定时器时间，定时调用
    if(base->cur_time > base->last_time)
    {
        //check timers...
        cur_id = base->timer_list_head;
        while(cur_id != -1)
        {
            iop = base->iops + cur_id;
            next_id = iop->next;
            if(base->cur_time > iop->last_dispatch_time + iop->timeout)
            {
                IOP_CB(base,iop,EV_TYPE_TIMER);
            }
            cur_id = next_id;
        }



        /*********check for connect list.*********************/

        cur_id = base->connect_list_head;
        while(cur_id != -1)
        {
            iop = base->iops + cur_id;
            next_id = iop->next;
            if(base->cur_time > iop->last_dispatch_time + iop->timeout)
            {
                IOP_CB(base,iop,EV_TYPE_TIMEOUT);
            }
            cur_id = next_id;
        }

        //超时检测
        /*********clear keepalive, 60 seconds per times***********************/
        if(base->cur_time > base->last_keepalive_time+60)
        {
            base->last_keepalive_time = base->cur_time;
            cur_id = base->io_list_head;
            while(cur_id != -1)
            {
                iop = base->iops+cur_id;
                next_id = iop->next;
                if(iop->timeout > 0 && iop->last_dispatch_time + iop->timeout < base->cur_time)
                {
                    IOP_CB(base,iop,EV_TYPE_TIMEOUT);
                }
                cur_id = next_id;
            }
        }

        base->last_time = base->cur_time;
    }

    return r;
}
```
 

`这句代码是消息派发的关键`

  //调用不同模型的函数指针实现消息派发
   ` dispatch_imp_cb dispatch_cb = base->op_imp.base_dispatch;
    r = (*dispatch_cb)(base,base->dispatch_interval);`
 

`base->op_imp.base_dispatch之前在epoll_init里完成过初始化`

`其实调用的是epoll的dispatch`

``` cpp
static int epoll_dispatch(iop_base_t * base, int timeout)
{
    int i;
    int id = 0;
    iop_t *iop = NULL;
    //iop_base中取出模型数据
    iop_epoll_data_t *iop_data = (iop_epoll_data_t *)(base->model_data);
    int n = 0;
    do{
        n = epoll_wait(iop_data->epfd, iop_data->events, iop_data->nevents, timeout);    
    }while((n < 0) && (errno == EINTR));
    base->cur_time = time(NULL);
    for(i = 0; i < n; i++)
    {
        //取出iop的id
        id = (int)((iop_data->events)[i].data.u32);
        if(id >= 0 && id < base->maxio)
        {
            iop = (base->iops)+id;
            //这个宏是调用绑定在iop的事件回调函数（accept,read,write等）
            IOP_CB(base,iop,from_epoll_events(iop_data->events[i].events));
        }
    }
    return n;
}
```
 

`这句话完成绑定在iop的回调函数调用，基本功能就是accept，read或者write等`

 

//这个宏是调用绑定在iop的事件回调函数（accept,read,write等）
`IOP_CB(base,iop,from_epoll_events(iop_data->events[i].events));`
这样就是整个libiop通讯流程和事件驱动机制