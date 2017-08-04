---
title: libevent学习文档(三)working with event
date: 2017-08-04 18:23:33
categories: 技术开发
tags: [网络编程,Linux环境编程]
---
`Events` have similar lifecycles. Once you call a Libevent function to set up an event and associate it with an `event base`, it becomes initialized. At this point, you can add, which makes it pending in the base. When the event is pending, if the conditions that would trigger an event occur (e.g., its file descriptor changes state or its timeout expires), the event becomesactive, and its (user-provided) callback function is run. If the event is configured persistent, it remains pending. If it is not persistent, it stops being pending when its callback runs. You can make a pending event non-pending by deleting it, and you can add a non-pending event to make it pending again.

事件有相似的生命周期，一旦你调用libevent函数设置event和event_base关联后，event被初始化了。add这个事件会使它阻塞，当事件阻塞时，有触发事件的条件出现，事件会激活，回调函数会被调用。`如果事件被设置为永久，它保持阻塞`。`如果不是永久，当事件的回调函数调用的时候就不阻塞了`。可以通过删除一个事件使它由阻塞变为非阻塞。通过添加使它由非阻塞变为阻塞。
<!--more-->
 
``` cpp
#define EV_TIMEOUT      0x01
#define EV_READ         0x02
#define EV_WRITE        0x04
#define EV_SIGNAL       0x08
#define EV_PERSIST      0x10
#define EV_ET           0x20

typedef void (*event_callback_fn)(evutil_socket_t, short, void *);

struct event *event_new(struct event_base *base, evutil_socket_t fd,
    short what, event_callback_fn cb,
    void *arg);

void event_free(struct event *event);
```

通过event_new创建事件，通过event_free释放。 参数base 表示event绑定在那个event_base上, fd表示event关联的描述符， what表示事件的类型，是一个bitfield, 上面那些宏按位或，cb是事件回调函数，事件就绪后可以触发。event_free释放event事件。

All new events are initialized and non-pending. To make an event pending, call event_add() (documented below).

To deallocate an event, call event_free(). It is safe to call event_free() on an event that is pending or active: doing so makes the event non-pending and inactive before deallocating it.

所有新创建的事件都是初始化的，并且非阻塞，调用event_add可以让一个事件变为阻塞。
调用event_free释放event，在事件阻塞或者激活状态下调用event_free是安全的，这个函数会在释放event之前将事件变为非阻塞并且非激活状态。

 

`EV_TIMEOUT`

This flag indicates an event that becomes active after a timeout elapses.

`EV_READ`

This flag indicates an event that becomes active when the provided file descriptor is ready for reading.

`EV_WRITE`

This flag indicates an event that becomes active when the provided file descriptor is ready for writing.

`EV_SIGNAL`

Used to implement signal detection. See "Constructing signal events" below.

`EV_PERSIST`

Indicates that the event is persistent. See "About Event Persistence" below.

`EV_ET`

Indicates that the event should be edge-triggered, if the underlying event_base backend supports edge-triggered events.

`EV_TIMEOUT`：表示过一段事件后event变为active。

`EV_READ`：当文件描述可读的时候变为就绪。

`EV_WRITE`：当文件描述符可写的时候变为就绪。

`EV_SIGNAL`：信号事件的标记

`EV_PERSIST`：永久事件，下面会介绍。

`EV_ET`：如果后端支持边缘触发事件，那么事件是边缘触发的。

`About Event Persistence`

By default, whenever a pending event becomes active (because its fd is ready to read or write, or because its timeout expires), it becomes non-pending right before its callback is executed. Thus, if you want to make the event pending again, you can call event_add() on it again from inside the callback function.

If the EV_PERSIST flag is set on an event, however, the event is persistent. This means that event remains pending even when its callback is activated. If you want to make it non-pending from within its callback, you can call event_del() on it.

The timeout on a persistent event resets whenever the event’s callback runs. Thus, if you have an event with flags EV_READ|EV_PERSIST and a timeout of five seconds, the event will become active:

Whenever the socket is ready for reading.

Whenever five seconds have passed since the event last became active.

默认情况下，当一个阻塞事件变为active时，(读事件可读，写事件可写，超时间到期等)，在事件对应的回调函数调用前该事件就会变为非阻塞的。因此，如果想要将事件变为阻塞，需要在事件的回调函数里调用event_add()

如果设置了EV_PERSIST 标记位， 那么事件就变味永久的，这意味着事件在回调函数触发时任然保持pending，如果你想要在回调函数调用后该事件变为非阻塞，需要调用event_del()。

当事件回调函数调用后超时会被重置，因此，如果事件带有EV_READ|EV_PERSIST标记，并且有5秒的超时值，如下情况事件会变为active：

`1当socket可读时`

`2从上次变为active之后过了5秒后事件会变为active。`

当事件的回调函数需要用到自己作为参数时候，需要将参数传递为

void *event_self_cbarg();

代码例子
``` cpp
#include <event2/event.h>

static int n_calls = 0;

void cb_func(evutil_socket_t fd, short what, void *arg)
{
    struct event *me = arg;

    printf("cb_func called %d times so far.\n", ++n_calls);

    if (n_calls > 100)
       event_del(me);
}

void run(struct event_base *base)
{
    struct timeval one_sec = { 1, 0 };
    struct event *ev;
    /* We're going to set up a repeating timer to get called called 100
       times. */
    ev = event_new(base, -1, EV_PERSIST, cb_func, event_self_cbarg());
    event_add(ev, &one_sec);
    event_base_dispatch(base);
}
```
For performance and other reasons, some people like to allocate events as a part of a larger structure. For each use of the event, this saves them:

The memory allocator overhead for allocating a small object on the heap.

The time overhead for dereferencing the pointer to the struct event.

The time overhead from a possible additional cache miss if the event is not already in the cache.

有时候开辟event作为一个较大结构体的一部分，可以节省在堆上开辟小对象的内存，也可以节省间接引用事件指针的事件和额外内存流失的处理。

文档的作者并不提倡用event_assign这个函数，推荐使用event_new，而且对于一些问题event_assign并不好调试

下面是使用event_assign的例子
``` cpp
struct event_pair {
         evutil_socket_t fd;
         struct event read_event;
         struct event write_event;
};
void readcb(evutil_socket_t, short, void *);
void writecb(evutil_socket_t, short, void *);
struct event_pair *event_pair_new(struct event_base *base, evutil_socket_t fd)
{
        struct event_pair *p = malloc(sizeof(struct event_pair));
        if (!p) return NULL;
        p->fd = fd;
        event_assign(&p->read_event, base, fd, EV_READ|EV_PERSIST, readcb, p);
        event_assign(&p->write_event, base, fd, EV_WRITE|EV_PERSIST, writecb, p);
        return p;
}
```
WARNING
Never call event_assign() on an event that is already pending in an event base. Doing so can lead to extremely hard-to-diagnose errors. If the event is already initialized and pending, call event_del() on it before you call event_assign() on it again.

 

event在event_base中阻塞时不要调用event_assign()，否则会造成很难查找分析的问题，如果一个事件已经初始化并且pending了，需要调用event_del()删除他，然后再次调用event_assign()。

evtimer_assign和evsignal_assign分别是定时器和信号的注册函数。


由于调用event_assign()可能会造成版本兼容的问题，调用如下函数，可以获取到event运行时大小。
size_t event_get_struct_event_size(void);


This function returns the number of bytes you need to set aside for a struct event. As before, you should only be using this function if you know that heap-allocation is actually a significant problem in your program, since it can make your code much harder to read and write.

这个函数返回event结构体旁边的偏移位置的字节数，只有在你觉得堆开辟确实是一个难题的时候才采用这个方法。因为这么做会是你的代码更难去读和写。


`事件的添加：`
int event_add(struct event *ev, const struct timeval *tv);
Calling event_add on a non-pending event makes it pending in its configured base. The function returns 0 on success, and -1 on failure. If tv is NULL, the event is added with no timeout. Otherwise, tv is the size of the timeout in seconds and microseconds.

If you call event_add() on an event that is already pending, it will leave it pending, and reschedule it with the provided timeout. If the event is already pending, and you re-add it with the timeout NULL, event_add() will have no effect.

调用event_add会让一个event变得pending，返回0表示成功，-1表示失败。如果tv设置为NULL，表示没有超时检测。否则，tv表示超时的秒数和毫秒。
如果在一个pending的event上调用add，会使它pengding，并且根据超时值重新计时。

`事件的删除：`
int event_del(struct event *ev);
 


事件删除函数，会将一个阻塞或者激活的事件变为非阻塞和非激活的，如果事件是非阻塞的或者非激活的，调用这个函数并没有什么影响。同样，返回0表示成功，-1表示失败。


`优先级设置：`
int event_priority_set(struct event *event, int priority);
 


每个event_base有priorities，event可以设置从0到这个值之间的一个数，0表示成功，-1表示失败。
优先级高的先处理，优先级低的后处理。
如果不设置优先级，默认值为event_base中队列大小除以2
``` cpp
#include <event2/event.h>

void read_cb(evutil_socket_t, short, void *);
void write_cb(evutil_socket_t, short, void *);

void main_loop(evutil_socket_t fd)
{
  struct event *important, *unimportant;
  struct event_base *base;

  base = event_base_new();
  event_base_priority_init(base, 2);
  /* Now base has priority 0, and priority 1 */
  important = event_new(base, fd, EV_WRITE|EV_PERSIST, write_cb, NULL);
  unimportant = event_new(base, fd, EV_READ|EV_PERSIST, read_cb, NULL);
  event_priority_set(important, 0);
  event_priority_set(unimportant, 1);

  /* Now, whenever the fd is ready for writing, the write callback will
     happen before the read callback.  The read callback won't happen at
     all until the write callback is no longer active. */
}
 
```

除此之外，libevent还为我们提供了一些接口访问当前event_base 和event属性。
``` cpp
int event_pending(const struct event *ev, short what, struct timeval *tv_out);

#define event_get_signal(ev) /* ... */
evutil_socket_t event_get_fd(const struct event *ev);
struct event_base *event_get_base(const struct event *ev);
short event_get_events(const struct event *ev);
event_callback_fn event_get_callback(const struct event *ev);
void *event_get_callback_arg(const struct event *ev);
int event_get_priority(const struct event *ev);

void event_get_assignment(const struct event *event,
        struct event_base **base_out,
        evutil_socket_t *fd_out,
        short *events_out,
        event_callback_fn *callback_out,
        void **arg_out);
 
```
 
event_pending 获取当前event对应的what属性事件是否pending或者被激活，If it is, and any of the flags EV_READ, EV_WRITE, EV_SIGNAL, and EV_TIMEOUT are set in the whatargument, the function returns all of the flags that the event is currently pending or active on

任类型都可以设置到what参数里，这个函数返回当前pending或者激活状态的标记按位或。event_get_signal和event_get_fd返回event关联的信号id和文件描述符id， event_get_base返回event绑定的event_base，
event_get_events返回event监听的事件集合，event_get_callback返回event的回调函数，以及
event_get_callback_arg返回回调函数参数，
event_get_priority返回event的优先级
event_get_assignment这个函数返回event所有绑定的信息到对应的指针域，如果形参为NULL，表示忽略。\
``` cpp
#include <event2/event.h>
#include <stdio.h>

/* Change the callback and callback_arg of 'ev', which must not be
 * pending. */
int replace_callback(struct event *ev, event_callback_fn new_callback,
    void *new_callback_arg)
{
    struct event_base *base;
    evutil_socket_t fd;
    short events;

    int pending;

    pending = event_pending(ev, EV_READ|EV_WRITE|EV_SIGNAL|EV_TIMEOUT,
                            NULL);
    if (pending) {
        /* We want to catch this here so that we do not re-assign a
         * pending event.  That would be very very bad. */
        fprintf(stderr,
                "Error! replace_callback called on a pending event!\n");
        return -1;
    }

    event_get_assignment(ev, &base, &fd, &events,
                         NULL /* ignore old callback */ ,
                         NULL /* ignore old callback argument */);

    event_assign(ev, base, fd, events, new_callback, new_callback_arg);
    return 0;
}
 
```

还有个能只调用一次事件的创建接口

int event_base_once(struct event_base *, evutil_socket_t, short,
  void (*)(evutil_socket_t, short, void *), void *, const struct timeval *);

这个函数不支持EV_SIGNAL 和 EV_PERSIST ，这个事件也不支持手动删除和激活。当该事件对应的回调函数触发后，该事件会自动从event_base中移除，并且libevent会析构掉该event。

`激活event的接口`

void event_active(struct event *ev, int what, short ncalls);
 
这个函数可以根据what(EV_READ, EV_WRITE,EV_TIMER等)将event设置为active，调用函数前，event是否为pengding并不影响，
并且激活它并且变为非阻塞状态。
在同一个事件递归的调用event_active会导致内存耗尽。

下面是一个错误例子
``` cpp
struct event *ev;

static void cb(int sock, short which, void *arg) {
        /* Whoops: Calling event_active on the same event unconditionally
           from within its callback means that no other events might not get
           run! */

        event_active(ev, EV_WRITE, 0);
}

int main(int argc, char **argv) {
        struct event_base *base = event_base_new();

        ev = event_new(base, -1, EV_PERSIST | EV_READ, cb, NULL);

        event_add(ev, NULL);

        event_active(ev, EV_WRITE, 0);

        event_base_loop(base, 0);

        return 0;
}
```


有两种改进的方式，一种是采用定时器，另一种是采用libevent提供的event_config_set_max_dispatch_interval

定时器的就是只调用一次loop，之后的回调函数cb会反复调用，因为cb内部发现event不是阻塞状态了，就要将event删除后再加入，loop内部检测到新的event，继续调用cb，反复调用cb
``` cpp
struct event *ev;
struct timeval tv;

static void cb(int sock, short which, void *arg) {
   if (!evtimer_pending(ev, NULL)) {
       event_del(ev);
       evtimer_add(ev, &tv);
   }
}

int main(int argc, char **argv) {
   struct event_base *base = event_base_new();

   tv.tv_sec = 0;
   tv.tv_usec = 0;

   ev = evtimer_new(base, cb, NULL);

   evtimer_add(ev, &tv);

   event_base_loop(base, 0);

   return 0;
}
 


event_config_set_max_dispatch_interval设置了dispatch的时间间隔，每个一段时间才派发就绪时间，这样就不会导致递归造成的资源耗尽了。

struct event *ev;

static void cb(int sock, short which, void *arg) {
        event_active(ev, EV_WRITE, 0);
}

int main(int argc, char **argv) {
        struct event_config *cfg = event_config_new();
        /* Run at most 16 callbacks before checking for other events. */
        event_config_set_max_dispatch_interval(cfg, NULL, 16, 0);
        struct event_base *base = event_base_new_with_config(cfg);
        ev = event_new(base, -1, EV_PERSIST | EV_READ, cb, NULL);

        event_add(ev, NULL);

        event_active(ev, EV_WRITE, 0);

        event_base_loop(base, 0);

        return 0;
}
 
```


今天的学习就到这里，这是我的公众号


