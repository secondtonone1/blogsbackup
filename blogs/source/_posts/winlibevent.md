---
title: windows环境libevent搭建和demo分析
date: 2017-08-07 10:53:56
categories: [网络编程]
tags: [网络编程]
---
`libevent框架`之前有做过分析，这次是谈谈如何将libevent搭建在`vs工作环境下`，并且编写一个demo进行测试。测试过程中会再一次带大家分析消息是怎么传递的。我的libevent版本`libevent-2.0.22-stable`，用对应的vs命令工具进入该目录
<!--more-->
![1.png](1.png)

我的是`Visual Studio 2008版本`的Command Prompt
![2.png](2.png)
执行成功后在libevent目录下生成三个lib
![3.png](3.png)
之后用vs创建控制台项目
![4.png](4.png)
生成成功后在项目目录里创建Include和Lib两个文件夹
![5.png](5.png)
分别进入libevent这两个目录里边
![6.png](6.png)
将内部的所有文件拷贝到Include文件夹里，event内容重复可以合并我们项目目录Include文件夹下的内容为
![7.png](7.png)
将libevent库中的三个lib拷贝到项目的Lib文件夹里
下一步配置项目属性，完成编译
`1、配置头文件包含路径，C++/General/Additional Include Directories  配置为相对路径的Include（因配置的路径不同而异）`
![8.png](8.png)
`2、配置代码生成`
C/C++ /Code Generation RuntimeLibrary 设置为MTD,因为库的生成是按照这个MTD模式生成的，所以要匹配
![9.png](9.png)
`3、配置 C/C++ /Advanced/Compile As Compile as C++ Code (/TP) `
（因为我的工程用到C++的函数所以配置这个）网上有人推荐配置成TC的也可以，自己根据项目需要
![10.png](10.png)
`4、配置库目录`
Linker/General/Additional Library Directories   ..\Lib(根据自己的Lib文件夹和项目相对位置填写)
![11.png](11.png)
5`配置 Linker\Input\AdditionalLibraries `   ws2_32.lib;wsock32.lib;libevent.lib;libevent_core.lib;libevent_extras.lib;
![12.png](12.png)
6 `配置忽略项，可以不配置` 输入\忽略特定默认库 libc.lib;msvcrt.lib;libcd.lib;libcmtd.lib;msvcrtd.lib;%(IgnoreSpecificDefaultLibraries)
生成lib后，不带调试信息，无法单步进函数里，所以要修改脚本：`Makefile.nmake第二行CFLAGS=$(CFLAGS) /Od /W3 /wd4996 /nologo /Zi`
到此为止项目配置好了，我们来写相关的demo代码

主函数

``` cpp
int main(int argc, char **argv)
{
    struct event_base *base;
    struct evconnlistener *listener;
    struct event *signal_event;

    struct sockaddr_in sin;

#ifdef WIN32
    WSADATA wsa_data;
    WSAStartup(0x0201, &wsa_data);
#endif
    //创建event_base
    base = event_base_new();
    if (!base) 
    {
        fprintf(stderr, "Could not initialize libevent!\n");
        return 1;
    }

    memset(&sin, 0, sizeof(sin));
    sin.sin_family = AF_INET;
    sin.sin_port = htons(PORT);
    sin.sin_addr.s_addr = inet_addr("192.168.1.99");
    //std::string ipstr = inet_ntoa(sin.sin_addr);
    //std::cout << ipstr.c_str();
    
    //基于eventbase 生成listen描述符并绑定
    //设置了listener_cb回调函数，当有新的连接登录的时候
    //触发listener_cb
    listener = evconnlistener_new_bind(base, listener_cb, (void *)base,
        LEV_OPT_REUSEABLE|LEV_OPT_CLOSE_ON_FREE, -1,
        (struct sockaddr*)&sin,
        sizeof(sin));

    if (!listener) 
    {
        fprintf(stderr, "Could not create a listener!\n");
        return 1;
    }
    
    //设置终端信号，当程序收到SIGINT后调用signal_cb
    signal_event = evsignal_new(base, SIGINT, signal_cb, (void *)base);

    if (!signal_event || event_add(signal_event, NULL)<0) 
    {
        fprintf(stderr, "Could not create/add a signal event!\n");
        return 1;
    }
    //event_base消息派发
    event_base_dispatch(base);

    //释放生成的evconnlistener
    evconnlistener_free(listener);
    //释放生成的信号事件
    event_free(signal_event);
    //释放event_base
    event_base_free(base);

    printf("done\n");
    return 0;
}
```
listener回调函数

``` cpp
static void listener_cb(struct evconnlistener *listener, evutil_socket_t fd,
    struct sockaddr *sa, int socklen, void *user_data)
{
    struct event_base *base = (struct event_base *)user_data;
    struct bufferevent *bev;
    //生成一个bufferevent，用于读或者写
    bev = bufferevent_socket_new(base, fd, BEV_OPT_CLOSE_ON_FREE);
    if (!bev) 
    {
        fprintf(stderr, "Error constructing bufferevent!");
        event_base_loopbreak(base);
        return;
    }
    //设置了写回调函数和事件的回调函数
    bufferevent_setcb(bev, NULL, conn_writecb, conn_eventcb, NULL);
    //bufferevent设置写事件回调
    bufferevent_enable(bev, EV_WRITE);
    //bufferevent关闭读事件回调
    bufferevent_disable(bev, EV_READ);
    //将MESSAGE字符串拷贝到outbuffer里
    bufferevent_write(bev, MESSAGE, strlen(MESSAGE));
}
```
一些基本参数

static const char MESSAGE[] = "Hello, NewConnection!\n";

static const int PORT = 9995;
bufferevent的写回调函数

``` cpp
static void conn_writecb(struct bufferevent *bev, void *user_data)
{
    //取出bufferevent 的output数据
    struct evbuffer *output = bufferevent_get_output(bev);
    //长度为0，那么写完毕，释放空间
    if (evbuffer_get_length(output) == 0) 
    {
        printf("flushed answer\n");
        bufferevent_free(bev);
    }
}
```
bufferevent的事件回调函数

``` cpp
//仅仅作为事件回调函数，写自己想要做的功能就行
//最后记得释放buffevent空间
static void conn_eventcb(struct bufferevent *bev, short events, void *user_data)
{
    if (events & BEV_EVENT_EOF) 
    {
        printf("Connection closed.\n");
    } 
    else if (events & BEV_EVENT_ERROR) 
    {
        printf("Got an error on the connection: %s\n",
            strerror(errno));/*XXX win32*/
    }
    /* None of the other events can happen here, since we haven't enabled
     * timeouts */
    bufferevent_free(bev);
}
```
信号终止函数

``` cpp
//程序捕捉到信号后就让baseloop终止
static void signal_cb(evutil_socket_t sig, short events, void *user_data)
{
    struct event_base *base = (struct event_base *)user_data;
    struct timeval delay = { 2, 0 };

    printf("Caught an interrupt signal; exiting cleanly in two seconds.\n");

    event_base_loopexit(base, &delay);
}
```
整个demo完成了。

下面分析下libevent如何做的消息传递和回调注册函数从main函数中的evconnlistener_new_bind入手

``` cpp
struct evconnlistener *
evconnlistener_new_bind(struct event_base *base, evconnlistener_cb cb,
    void *ptr, unsigned flags, int backlog, const struct sockaddr *sa,
    int socklen)
{
    struct evconnlistener *listener;
    evutil_socket_t fd;
    int on = 1;
    int family = sa ? sa->sa_family : AF_UNSPEC;

    if (backlog == 0)
        return NULL;

    fd = socket(family, SOCK_STREAM, 0);
    if (fd == -1)
        return NULL;

    if (evutil_make_socket_nonblocking(fd) < 0) {
        evutil_closesocket(fd);
        return NULL;
    }

    if (flags & LEV_OPT_CLOSE_ON_EXEC) {
        if (evutil_make_socket_closeonexec(fd) < 0) {
            evutil_closesocket(fd);
            return NULL;
        }
    }

    if (setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, (void*)&on, sizeof(on))<0) {
        evutil_closesocket(fd);
        return NULL;
    }
    if (flags & LEV_OPT_REUSEABLE) {
        if (evutil_make_listen_socket_reuseable(fd) < 0) {
            evutil_closesocket(fd);
            return NULL;
        }
    }

    if (sa) {
        if (bind(fd, sa, socklen)<0) {
            evutil_closesocket(fd);
            return NULL;
        }
    }
    //cb = listener_cb,  ptr = struct event_base *base;
    listener = evconnlistener_new(base, cb, ptr, flags, backlog, fd);
    if (!listener) {
        evutil_closesocket(fd);
        return NULL;
    }

    return listener;
}
```
evconnlistener_new_bind 完成了socket生成和绑定，并且内部调用evconnlistener_new

生成了evconnlistener* listener,将listener和socket绑定在一起。

``` cpp
struct evconnlistener *
evconnlistener_new(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    evutil_socket_t fd)
{
    struct evconnlistener_event *lev;
    if (backlog > 0) {
        if (listen(fd, backlog) < 0)
            return NULL;
    } else if (backlog < 0) {
        if (listen(fd, 128) < 0)
            return NULL;
    }
       //开辟evconnlistener_event大小区域
    lev = mm_calloc(1, sizeof(struct evconnlistener_event));
    if (!lev)
        return NULL;
    //lev -> base 表示  evconnlistener 
    //evconnlistener     evconnlistener_ops 基本回调参数和回调函数结构体赋值
    lev->base.ops = &evconnlistener_event_ops;
    //evconnlistener_cb 设置为listener_cb
    lev->base.cb = cb;
    //ptr表示event_base 指针
    lev->base.user_data = ptr;
    lev->base.flags = flags;
    lev->base.refcnt = 1;

    if (flags & LEV_OPT_THREADSAFE) {
        EVTHREAD_ALLOC_LOCK(lev->base.lock, EVTHREAD_LOCKTYPE_RECURSIVE);
    }

    //  lev   is evconnlistener_event       
    //lev->listener is event
    //为lev->listener设置读回调函数和读关注事件，仅进行设置并没加入event队列
    event_assign(&lev->listener, base, fd, EV_READ|EV_PERSIST,
        listener_read_cb, lev);
    //实际调用了event_add将事件加入event队列
    evconnlistener_enable(&lev->base);

    return &lev->base;
}

lev = mm_calloc(1, sizeof(struct evconnlistener_event));开辟了一个

evconnlistener_event* 空间，evconnlistener_event类型如下

struct evconnlistener_event {
    struct evconnlistener base;
    struct event listener;
};
```

该结构包含一个evconnlistener和event事件结构体

evconnlistener结构如下

``` cpp
struct evconnlistener {
    //listener基本操作封装成一个结构体
    //结构体包含操作的函数指针
    const struct evconnlistener_ops *ops;
    void *lock;
    //listener回调函数，有新的连接到来会触发
    evconnlistener_cb cb;
    //listener有错误会触发这个函数
    evconnlistener_errorcb errorcb;
    //存储一些回调函数用到的参数
    void *user_data;
    unsigned flags;
    short refcnt;
    unsigned enabled : 1;
};

struct evconnlistener_ops {
　　　　int (*enable)(struct evconnlistener *);
　　　　int (*disable)(struct evconnlistener *);
　　　　void (*destroy)(struct evconnlistener *);
　　　　void (*shutdown)(struct evconnlistener *);
　　　　evutil_socket_t (*getfd)(struct evconnlistener *);
　　　　struct event_base *(*getbase)(struct evconnlistener *);
};

```
 
lev->base.ops = &evconnlistener_event_ops;这句就是对这个结构体指针赋值，evconnlistener_event_ops是一个实例化的结构体对象，里面包含定义好的操作函数

``` cpp
static const struct evconnlistener_ops evconnlistener_event_ops = {
    event_listener_enable,
    event_listener_disable,
    event_listener_destroy,
    NULL, /* shutdown */
    event_listener_getfd,
    event_listener_getbase
};
```

对lev->base其余参数的赋值就不一一解释了。接下来看一下event_assign函数内部实现

``` cpp
{
    if (!base)
        base = current_base;

    _event_debug_assert_not_added(ev);
       //属于哪个event_base
    ev->ev_base = base;
       //事件回调函数
    ev->ev_callback = callback;
    //回调函数的参数
    ev->ev_arg = arg;
    //event关注哪个fd
    ev->ev_fd = fd;
    //event事件类型
    ev->ev_events = events;
    ev->ev_res = 0;
    ev->ev_flags = EVLIST_INIT;
    //被调用过几次
    ev->ev_ncalls = 0;
    ev->ev_pncalls = NULL;

    if (events & EV_SIGNAL) {
        if ((events & (EV_READ|EV_WRITE)) != 0) {
            event_warnx("%s: EV_SIGNAL is not compatible with "
                "EV_READ or EV_WRITE", __func__);
            return -1;
        }
        ev->ev_closure = EV_CLOSURE_SIGNAL;
    } else {
        if (events & EV_PERSIST) {
            evutil_timerclear(&ev->ev_io_timeout);
            ev->ev_closure = EV_CLOSURE_PERSIST;
        } else {
            ev->ev_closure = EV_CLOSURE_NONE;
        }
    }

    min_heap_elem_init(ev);

    if (base != NULL) {
        /* by default, we put new events into the middle priority */
        //优先级的设置
        ev->ev_pri = base->nactivequeues / 2;
    }

    _event_debug_note_setup(ev);

    return 0;
}
```
`event_assign`内部实现可以看出该函数仅仅对`event`的属性进行设置

`event_assign`主要对event设置了`listener_read_cb`回调函数，这是

很重要的一个细节，我们看下listener_read_cb内部实现

``` cpp
static void
listener_read_cb(evutil_socket_t fd, short what, void *p)
{
    struct evconnlistener *lev = p;
    int err;
    evconnlistener_cb cb;
    evconnlistener_errorcb errorcb;
    void *user_data;
    LOCK(lev);
    while (1) {
        struct sockaddr_storage ss;
#ifdef WIN32
        int socklen = sizeof(ss);
#else
        socklen_t socklen = sizeof(ss);
#endif
        //调用accept生成新的fd
        evutil_socket_t new_fd = accept(fd, (struct sockaddr*)&ss, &socklen);
        if (new_fd < 0)
            break;
        if (socklen == 0) {
            /* This can happen with some older linux kernels in
             * response to nmap. */
            evutil_closesocket(new_fd);
            continue;
        }
        //设置非阻塞
        if (!(lev->flags & LEV_OPT_LEAVE_SOCKETS_BLOCKING))
            evutil_make_socket_nonblocking(new_fd);

        if (lev->cb == NULL) {
            evutil_closesocket(new_fd);
            UNLOCK(lev);
            return;
        }
        ++lev->refcnt;
        //cb 就 是  listener_cb
        cb = lev->cb;
        user_data = lev->user_data;
        UNLOCK(lev);
        //触发了listener_cb
        
        //完成了eventbuffer注册写和事件函数  
        cb(lev, new_fd, (struct sockaddr*)&ss, (int)socklen,
            user_data);
        
        LOCK(lev);
        if (lev->refcnt == 1) {
            int freed = listener_decref_and_unlock(lev);
            EVUTIL_ASSERT(freed);
            return;
        }
        --lev->refcnt;
    }
   。。。。。。。。。
}
``` 
在evconnlistener_new中 
//evconnlistener_cb 设置为listener_cb
lev->base.cb = cb;

`lev->base`就是`evconnlistener`对象

`listener_read_cb`内部回调用绑定在`evconnlistener`的`listener_cb`。

记得之前我所说的绑定么？`evconnlistener_new`这个函数里生成的`lev`，

之后对lev，这里的lev就是 evconnlistener_event对象，lev->listener是

event对象，通过调用`event_assign(&lev->listener, base, fd, EV_READ|EV_PERSIST,
listener_read_cb, lev);`

lev->listener绑定的就是`listener_read_cb`。也就是是说

`listener_read_cb`调用后，从而调用了绑定在`evconnlistener`的`listener_cb`。

那么我们只要知道lev->listener（event对象）的读事件是如何派发的就可以梳理此

流程了。

之前我们梳理过`event_dispatch`里进行的事件派发，调用不同模型的dispatch，

稍后再梳理一遍。因为调用`event_assign`仅仅对event设置了属性，还没有加到

事件队列里。

在`evconnlistener_new`函数里调用完`event_assign`，之后调用的是

`evconnlistener_enable`，evconnlistener_enable这个函数完成了事件

添加到事件队列的功能。

``` cpp
int
evconnlistener_enable(struct evconnlistener *lev)
{
    int r;
    LOCK(lev);
    lev->enabled = 1;
    if (lev->cb)
        //调用evconnlistener 的ops的enable函数
        //lev->ops 此时指向evconnlistener_event_ops
        //enable函数为  event_listener_enable
        r = lev->ops->enable(lev);
    else
        r = 0;
    UNLOCK(lev);
    return r;
}
```
上面有evconnlistener_event_ops结构体，那几个函数也列出来了。

我们看下event_listener_enable函数

``` cpp
static int
event_listener_enable(struct evconnlistener *lev)
{
    //通过evconnlistener* 找到evconnlistener_event *
    struct evconnlistener_event *lev_e =
        EVUTIL_UPCAST(lev, struct evconnlistener_event, base);
    //将
    return event_add(&lev_e->listener, NULL);
}
```

里面调用了event_add

``` cpp
//添加事件操作
int
event_add(struct event *ev, const struct timeval *tv)
{
    int res;

    if (EVUTIL_FAILURE_CHECK(!ev->ev_base)) {
        event_warnx("%s: event has no event_base set.", __func__);
        return -1;
    }

    EVBASE_ACQUIRE_LOCK(ev->ev_base, th_base_lock);
    //添加事件核心函数
    res = event_add_internal(ev, tv, 0);

    EVBASE_RELEASE_LOCK(ev->ev_base, th_base_lock);

    return (res);
}
```
对于`event_add`内部判断是否枷锁，进行加锁，然后调用

`event_add_internal完成事件添加`

``` cpp
static inline int
event_add_internal(struct event *ev, const struct timeval *tv,
    int tv_is_absolute)
{
    struct event_base *base = ev->ev_base;
    int res = 0;
    int notify = 0;

   
    //根据不同的事件类型将事件放到evmap里，调用不同模型的add函数
    //将事件按照EV_READ或者EV_WRITE或者EV_SIGNAL放入evmap事件队列里
    //将ev按照EVLIST_INSERTED放入用户的事件队列里
    if ((ev->ev_events & (EV_READ|EV_WRITE|EV_SIGNAL)) &&
        !(ev->ev_flags & (EVLIST_INSERTED|EVLIST_ACTIVE))) {
        if (ev->ev_events & (EV_READ|EV_WRITE))
            //将事件按照读写  IO方式加入evmap里，并且调用不同网络模型的add完成事件添加
            res = evmap_io_add(base, ev->ev_fd, ev);
        else if (ev->ev_events & EV_SIGNAL)
            //将事件按照信号方式添加
            res = evmap_signal_add(base, (int)ev->ev_fd, ev);
        //将事件插入轮询的事件队列里
        if (res != -1)
            event_queue_insert(base, ev, EVLIST_INSERTED);
        if (res == 1) {
            /* evmap says we need to notify the main thread. */
            notify = 1;
            res = 0;
        }
    }
}
```
由于lev->listener（event）类型的事件是I/O 读事件，所以会进入evmap_io_add完成读事件的添加

``` cpp
int
evmap_io_add(struct event_base *base, evutil_socket_t fd, struct event *ev)
{
    const struct eventop *evsel = base->evsel;
    struct event_io_map *io = &base->io;
    struct evmap_io *ctx = NULL;
    int nread, nwrite, retval = 0;
    short res = 0, old = 0;
    struct event *old_ev;

    EVUTIL_ASSERT(fd == ev->ev_fd);

    if (fd < 0)
        return 0;
//windows情况下use a hashtable instead of an array
#ifndef EVMAP_USE_HT
    if (fd >= io->nentries) {
        if (evmap_make_space(io, fd, sizeof(struct evmap_io *)) == -1)
            return (-1);
    }
#endif

    //从io中找到下标为fd的结构体数据 evmap_io * 赋值给ctx
    //如果没有找到就调用evmap_io_init初始化
    GET_IO_SLOT_AND_CTOR(ctx, io, fd, evmap_io, evmap_io_init,
                         evsel->fdinfo_len);

    nread = ctx->nread;
    nwrite = ctx->nwrite;

    if (nread)
        old |= EV_READ;
    if (nwrite)
        old |= EV_WRITE;

    if (ev->ev_events & EV_READ) {
        if (++nread == 1)
            res |= EV_READ;
    }
    if (ev->ev_events & EV_WRITE) {
        if (++nwrite == 1)
            res |= EV_WRITE;
    }
   ......if (res) {
        void *extra = ((char*)ctx) + sizeof(struct evmap_io);
        /* XXX(niels): we cannot mix edge-triggered and
         * level-triggered, we should probably assert on
         * this. */
         //这里就是调用了不同模型的demultiplexer的添加操作
         //调用不同的网络模型add接口
        if (evsel->add(base, ev->ev_fd,
            old, (ev->ev_events & EV_ET) | res, extra) == -1)
            return (-1);
        retval = 1;
    }

    ctx->nread = (ev_uint16_t) nread;
    ctx->nwrite = (ev_uint16_t) nwrite;
    //哈希表对应的event事件队列加入ev
    TAILQ_INSERT_TAIL(&ctx->events, ev, ev_io_next);

    return (retval);
}
```
该函数内部的一些函数就不展开，挺好理解的。到此为止我们了解了listen描述符的回调函数和读事件的绑定。

回到main函数，看下`event_base_dispatch就知道绑定在lev->listener（event）类型的读事件listener_read_cb`

是如何派发的，进而在读事件里完成了evconnlistener的listener_cb的调用。
``` cpp
int
event_base_dispatch(struct event_base *event_base)
{
    return (event_base_loop(event_base, 0));
}
```

`int
event_base_loop(struct event_base *base, int flags)这个函数内部`

我们只看关键代码

``` cpp
int
event_base_loop(struct event_base *base, int flags)
{
　　
　　const struct eventop *evsel = base->evsel;
　　　　//不同模型的派发函数
　　　　//evsel就是指向ops数组的某一种模型


　　res = evsel->dispatch(base, tv_p);
   
　　
　　if (N_ACTIVE_CALLBACKS(base)) {

　　　　//处理激活队列中的事件
　　　　int n = event_process_active(base);
　　　　if ((flags & EVLOOP_ONCE)
　　　　　　&& N_ACTIVE_CALLBACKS(base) == 0
　　　　　　　　&& n != 0)
　　　　　　done = 1;
　　　　} else if (flags & EVLOOP_NONBLOCK)
　　　　　　done = 1;



}
```
之前有说过evsel初始化和模型选择的代码，这里重新梳理下`const struct eventop *evsel`;

`event_base_new或者event_init内部调用了event_base_new_with_config，`

`event_base_new_with_config函数调用了base->evsel = eventops[i]`;

`eventops属主封装了几种模型结构体的指针`

``` cpp
static const struct eventop *eventops[] = {
#ifdef _EVENT_HAVE_EVENT_PORTS
    &evportops,
#endif
#ifdef _EVENT_HAVE_WORKING_KQUEUE
    &kqops,
#endif
#ifdef _EVENT_HAVE_EPOLL
    &epollops,
#endif
#ifdef _EVENT_HAVE_DEVPOLL
    &devpollops,
#endif
#ifdef _EVENT_HAVE_POLL
    &pollops,
#endif
#ifdef _EVENT_HAVE_SELECT
    &selectops,
#endif
#ifdef WIN32
    &win32ops,
#endif
    NULL
};
```
举个例子，看下`epollops`

``` cpp
const struct eventop epollops = {
    "epoll",
    epoll_init,
    epoll_nochangelist_add,
    epoll_nochangelist_del,
    epoll_dispatch,
    epoll_dealloc,
    1, /* need reinit */
    EV_FEATURE_ET|EV_FEATURE_O1,
    0
};
```
`eventop 类型`
``` cpp
struct eventop {
    /** The name of this backend. */
    const char *name;
   
    void *(*init)(struct event_base *);

    int (*add)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
    
    int (*del)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
   
    int (*dispatch)(struct event_base *, struct timeval *);
    /** Function to clean up and free our data from the event_base. */
    void (*dealloc)(struct event_base *);
    
    int need_reinit;
   
    enum event_method_feature features;
 
    size_t fdinfo_len;
};
```
`epollops是eventop类型的变量，实现了增加删除，初始化，销毁，派发等功能`。

所以`当模型选择epoll时,res = evsel->dispatch(base, tv_p)`;实际调用的是epoll的派发函数

``` cpp
static int epoll_dispatch(struct event_base *base, struct timeval *tv)
{

　　struct epollop *epollop = base->evbase;
　　　　struct epoll_event *events = epollop->events;

　　res = epoll_wait(epollop->epfd, events, epollop->nevents, timeout);

　　for (i = 0; i < res; i++) {
　　　　　　int what = events[i].events;
　　　　　　short ev = 0;

　　　　　　if (what & (EPOLLHUP|EPOLLERR)) {
　　　　　　　　ev = EV_READ | EV_WRITE;
　　　　　　} else {
　　　　　　　　　　if (what & EPOLLIN)
　　　　　　　　　　　　ev |= EV_READ;
　　　　　　　　　　if (what & EPOLLOUT)
　　　　　　　　　　ev |= EV_WRITE;
　　　　　　　　}

　　　　　　　　if (!ev)
　　　　　　　　　　continue;

　　　　　　　　//更新evmap，并且将事件放入active队列
　　　　　　　　evmap_io_active(base, events[i].data.fd, ev | EV_ET);
　　　　　　}

}
```

`evmap_io_active函数内部调用event_active_nolock，event_active_nolock中调用

event_queue_insert(base, ev, EVLIST_ACTIVE);负责将event放入激活队列里，

并且更新event在evmap中的 标记状态。`

到目前为止我们了解了事件派发的流程，`event_base_loop循环执行网络模型的dispatch，内核返回就绪事件，dispatch内部调用evmap_io_active将就绪事件放入激活队列里`。

`在event_base_loop中调用event_process_active处理就绪队列中的event`。比如内核返回listen描述符读就绪事件，那么就会将listen的event放入就绪队列中，

`在event_process_active处理event的读事件`，调用了之前绑定的listener_read_cb回调函数。

下面看下

``` cpp
static int
event_process_active(struct event_base *base)
{
    /* Caller must hold th_base_lock */
    struct event_list *activeq = NULL;
    int i, c = 0;
    //循环处理就绪队列中的每一个就绪事件
    for (i = 0; i < base->nactivequeues; ++i) {
        if (TAILQ_FIRST(&base->activequeues[i]) != NULL) {
            base->event_running_priority = i;
            activeq = &base->activequeues[i];
            c = event_process_active_single_queue(base, activeq);
            if (c < 0) {
                base->event_running_priority = -1;
                return -1;
            } else if (c > 0)
                break; 
        }
    }
    //调用延时回调函数
    event_process_deferred_callbacks(&base->defer_queue,&base->event_break);
    base->event_running_priority = -1;
    return c;
}
```
 
`循环调用了event_process_active_single_queue`
``` cpp
switch (ev->ev_closure) {
        case EV_CLOSURE_SIGNAL:
            event_signal_closure(base, ev);
            break;
        case EV_CLOSURE_PERSIST:
            event_persist_closure(base, ev);
            break;
        default:
        case EV_CLOSURE_NONE:
            EVBASE_RELEASE_LOCK(base, th_base_lock);
            (*ev->ev_callback)(
                ev->ev_fd, ev->ev_res, ev->ev_arg);
            break;
        }
```
`(*ev->ev_callback)(ev->ev_fd, ev->ev_res, ev->ev_arg);就是调用绑定在event上的回调函数。`比如绑定在lev->listener（event）类型的读事件listener_read_cb;

从而调用了绑定在evconnlistener的listener_cb。

这样整个流程就跑通了。最上面有listener_cb函数的实现，整个消息传递流程不跟踪了，读者可以模仿上面的方式去跟踪消息。

这里简单表述下在bufferevent创建的时候调用了这个函数

``` cpp
struct bufferevent *
bufferevent_socket_new(struct event_base *base, evutil_socket_t fd,
    int options)
{
    struct bufferevent_private *bufev_p;
    struct bufferevent *bufev;

    ...
　　　　//设置bufferevent中 ev_read(event类型)回调函数
　　　　event_assign(&bufev->ev_read, bufev->ev_base, fd,
　　　　EV_READ|EV_PERSIST, bufferevent_readcb, bufev);
　　　　//设置bufferevent中 ev_write(event类型)回调函数
　　　　event_assign(&bufev->ev_write, bufev->ev_base, fd,
　　　　EV_WRITE|EV_PERSIST, bufferevent_writecb, bufev);

　　　　//为bufev->output(evbuffer类型)设置回调函数，回调函数内部将ev_write事件加入事件队列
　　　　evbuffer_add_cb(bufev->output, bufferevent_socket_outbuf_cb, bufev);

　　 ...

    return bufev;
}
```
`函数内部为ev_read和ev_write 设置默认的回调函数bufferevent_readcb和bufferevent_writecb。`接着为输出的 evbuffer绑定bufferevent_socket_outbuf_cb函数。

bufferevent_socket_outbuf_cb函数如果发现我们有可写的东西，并且没开始写，那么 将ev_

write事件加入event队列，跟上面的轮询一样，有可写就绪事件就会触发绑定在ev_write上的bufferevent-writecb函数。如果没有添加写的数据，就跳出函数。之后调用。由于这时处于bufferevent刚创建状态，那么说明没有数据写入bufferevent，所以这时是不会将ev_write加入event队列的。回到listener_cb函数。接着调用bufferevent_setcb 函数设置bufferevent的读，写，事件，

回调函数。

调用bufferevent_enable使写事件生效，

内部调用

`bufev->be_ops->enable(bufev, impl_events)；`

bufferevent注册好的回调函数如下

``` cpp
const struct bufferevent_ops bufferevent_ops_socket = {
    "socket",
    evutil_offsetof(struct bufferevent_private, bev),
    be_socket_enable,
    be_socket_disable,
    be_socket_destruct,
    be_socket_adj_timeouts,
    be_socket_flush,
    be_socket_ctrl,
};
```

``` cpp
static int
be_socket_enable(struct bufferevent *bufev, short event)
{
    if (event & EV_READ) {
        if (be_socket_add(&bufev->ev_read,&bufev->timeout_read) == -1)
            return -1;
    }
    if (event & EV_WRITE) {
        if (be_socket_add(&bufev->ev_write,&bufev->timeout_write) == -1)
            return -1;
    }
    return 0;
}
```
eventbuffer读写事件加入到event队列里，此处为添加ev_write写事件，当写事件就绪，

轮询可以出发绑定的bufferevent_writecb回调函数。

当调用`bufferevent_writecb`这个函数时，我们把内部代码简化分析

``` cpp
static void
bufferevent_writecb(evutil_socket_t fd, short event, void *arg)

{

//计算bufferevent能写的最大数量
    atmost = _bufferevent_get_write_max(bufev_p);

    if (bufev_p->write_suspended)
        goto done;

    if (evbuffer_get_length(bufev->output)) {
        evbuffer_unfreeze(bufev->output, 1);
        //bufferevent调用写操作，将outbuffer中的内容发送出去
        res = evbuffer_write_atmost(bufev->output, fd, atmost);
        evbuffer_freeze(bufev->output, 1);
        if (res == -1) {
            int err = evutil_socket_geterror(fd);
            if (EVUTIL_ERR_RW_RETRIABLE(err))
                goto reschedule;
            what |= BEV_EVENT_ERROR;
        } else if (res == 0) {
            /* eof case
               XXXX Actually, a 0 on write doesn't indicate
               an EOF. An ECONNRESET might be more typical.
             */
            what |= BEV_EVENT_EOF;
        }
        if (res <= 0)
            goto error;
        //bufferevent减少发送的大小，留下未发送的，下次再发送
        _bufferevent_decrement_write_buckets(bufev_p, res);
    }

    //计算是否将outbuf中的内容发送完，发完了就删除写事件
    if (evbuffer_get_length(bufev->output) == 0) {
        event_del(&bufev->ev_write);
    }

    /*
     * Invoke the user callback if our buffer is drained or below the
     * low watermark.
     */
     //将buffer中的内容发完，或者低于low 水位，那么调用用户注册的写回调函数
    if ((res || !connected) &&
        evbuffer_get_length(bufev->output) <= bufev->wm_write.low) {
        _bufferevent_run_writecb(bufev);
    }

}
```
`_bufferevent_run_writecb内部调用了bufev->writecb(bufev, bufev->cbarg);也就是说我们自己实现的`

``` cpp
static void
conn_writecb(struct bufferevent *bev, void *user_data)
{
    struct evbuffer *output = bufferevent_get_output(bev);
    if (evbuffer_get_length(output) == 0) {
        printf("flushed answer\n");
        bufferevent_free(bev);
    }
}
```
到此为止整个bufferevent消息走向梳理出来。最后有一点需要陈述，在listener_cb中最后调用bufferevent_write(bev, MESSAGE, strlen(MESSAGE));

`其内部调用evbuffer_add，该函数内部evbuffer_invoke_callbacks(buf);会调用bufferevent_socket_outbuf_cb，进而调用

bufferevent_write。`

`所以我认为是调用evbuffer_add向outbuf中添加数据后，调用了evbuffer_invoke_callbacks,触发bufferevent_write，
或者ev_write先检测到写就绪事件，然后调用buffervent_write.这两者先后并不清楚`。

整个流程就是这样，需要继续研究然后梳理。