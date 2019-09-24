---
title: Libevent学习笔记(四) bufferevent 的 concepts and basics
date: 2017-08-04 18:08:08
categories: 技术开发
tags: [网络编程,Linux环境编程]
---
`Bufferevents and evbuffers`

Every bufferevent has an input buffer and an output buffer. These are of type "struct evbuffer". When you have data to write on a bufferevent, you add it to the output buffer; when a bufferevent has data for you to read, you drain it from the input buffer.

每个`bufferevents`都会包含一个input的buffer和output的buffer,都是`struct evbuffer`类型， 如果你有数据想通过bufferevent发送，需要将数据放入output buffer里。如果要从bufferevent中读数据，

需要从input buffer里读取。

`Callbacks and watermarks`

Every bufferevent has two data-related callbacks: a read callback and a write callback. By default, the read callback is called whenever any data is read from the underlying transport, and the write callback is called whenever enough data from the output buffer is emptied to the underlying transport. You can override the behavior of these functions by adjusting the read and write "watermarks" of the bufferevent.

每一个bufferevent都有两个回调函数，一个`读回调`和一个`写回调`函数，默认情况下，只要从底层传输读取数据就回触发读回调函数，当output buffer中足够多的数据排出就会触发写回调函数。而我的理解是，这里的足够多，默认情况下是排空，

用户可以通过调整读写水位来达到控制这些函数触发。
<!--more-->

下面是libevent源码中的一些注释

``` cpp
/**
   A read or write callback for a bufferevent.

   The read callback is triggered when new data arrives in the input
   buffer and the amount of readable data exceed the low watermark
   which is 0 by default.

   The write callback is triggered if the write buffer has been
   exhausted or fell below its low watermark.

   @param bev the bufferevent that triggered the callback
   @param ctx the user-specified context for this bufferevent
 */
```
 

到此为止总结下，bufferevent实际上是libevent为我们准备的一个缓存结构，将接受的数据缓存到input buffer供用户从中读取，用户将发送的数据缓存到output buffer进行发送。bufferevent的input buffer中有数据到来并且可读数据达到或超过读数据的低水位就会触发读回调函数，读数据的低水位默认是0,写回调函数会在outbuffer中的数据低于写的低水位时会触发。bufferevent会自己讲output中的内容发送出去，将接收到的内容放到input中。

`Read low-water mark`
Whenever a read occurs that leaves the bufferevent’s input buffer at this level or higher, the bufferevent’s read callback is invoked. Defaults to 0, so that every read results in the read callback being invoked.

只要读事件让bufferevent超过或者到达这个Read low-water 水位或者更高，读回调会被触发。默认情况下 Read low-water 是0，所以每次bufferevent 读数据都会导致读回调触发。

`Read high-water mark`
If the bufferevent’s input buffer ever gets to this level, the bufferevent stops reading until enough data is drained from the input buffer to take us below it again. Defaults to unlimited, so that we never stop reading because of the size of the input buffer.

如果bufferevent 的 input buffer达到Read high-water水平，那么bufferevent会停止读，直到有数据从input中被用户取出使input buffer的水位线低于Read high-water， 默认情况下Read high-water是无穷大的，因此我们不会因为input buffer不够停止读。 

`Write low-water mark`
Whenever a write occurs that takes us to this level or below, we invoke the write callback. Defaults to 0, so that a write callback is not invoked unless the output buffer is emptied.

当bufferevent写数据导致数据大小低于 Write low-water水平线，就回触发写回调函数。默认这个Write low-water值为0，因此只有当output buffer数据为空时才会触发写回调函数。

`Write high-water mark`
Not used by a bufferevent directly, this watermark can have special meaning when a bufferevent is used as the underlying transport of another bufferevent. See notes on filtering bufferevents below.

这个参数不会直接使用，当一个bufferevent作为参数传递给另一个bufferevent时可以做为特殊意义。filtering bufferevents可能会用到。

A bufferevent also has an "error" or "event" callback that gets invoked to tell the application about non-data-oriented events, like when a connection is closed or an error occurs. The following event flags are defined:

一个bufferevent也会有类似错误或者特殊事件的回调函数，这类函数触发后通知应用程序一些和数据无关的事件，比如当有连接关闭，或者错误出现可以触发这类函数。

`BEV_EVENT_READING`
An event occured during a read operation on the bufferevent. See the other flags for which event it was.

在bufferevent上进行读操作

`BEV_EVENT_WRITING`
An event occured during a write operation on the bufferevent. See the other flags for which event it was.

在bufferevent上进行写操作

`BEV_EVENT_ERROR`
An error occurred during a bufferevent operation. For more information on what the error was, call EVUTIL_SOCKET_ERROR().

bufferevent操作产生错误，通过EVUTIL_SOCKET_ERROR可以详细查看错误源

`BEV_EVENT_TIMEOUT`
A timeout expired on the bufferevent.

超时事件

`BEV_EVENT_EOF`
We got an end-of-file indication on the bufferevent.

读到文件结束符

`BEV_EVENT_CONNECTED`
We finished a requested connection on the bufferevent.

完成连接

Deferred callbacks

By default, a bufferevent callbacks are executed immediately when the corresponding condition happens. (This is true of evbuffer callbacks too; we’ll get to those later.) This immediate invocation can make trouble when dependencies get complex. For example, suppose that there is a callback that moves data into evbuffer A when it grows empty, and another callback that processes data out of evbuffer A when it grows full. Since these calls are all happening on the stack, you might risk a stack overflow if the dependency grows nasty enough.

To solve this, you can tell a bufferevent (or an evbuffer) that its callbacks should be deferred. When the conditions are met for a deferred callback, rather than invoking it immediately, it is queued as part of the event_loop() call, and invoked after the regular events' callbacks.

(Deferred callbacks were introduced in Libevent 2.0.1-alpha.)

默认情况下，bufferevent回调函数在条件符合时会立即执行。当情况复杂时这种立即出发的方式可能会造成问题，例如，假设有一个回调函数功能是在evbuffer A变空时向evbuffer A中放入数据，

另一个回调函数是在evbuffer A数据满的时候从中取出数据，由于这些操作是在栈上进行，这种做法会造成造成栈溢出，解决方法就是将回调函数延迟触发，条件满足时，将回调函数作为event_loop的一个部分

通过有规律的事件回调触发。

Option flags for bufferevents

You can use one or more flags when creating a bufferevent to alter its behavior. Recognized flags are:

创建bufferevent时可以选择如下选项进行个性化设置

`BEV_OPT_CLOSE_ON_FREE`
When the bufferevent is freed, close the underlying transport. This will close an underlying socket, free an underlying bufferevent, etc.

当bufferevent被释放，关闭底层传输，这个将会关闭底层socket，释放底层的bufferevent等。

`BEV_OPT_THREADSAFE`
Automatically allocate locks for the bufferevent, so that it’s safe to use from multiple threads.

为bufferevent自动上锁，多线程模式会安全。

`BEV_OPT_DEFER_CALLBACKS`
When this flag is set, the bufferevent defers all of its callbacks, as described above.

设置bufferevent的callback延迟处理。

`BEV_OPT_UNLOCK_CALLBACKS`
By default, when the bufferevent is set up to be threadsafe, the bufferevent’s locks are held whenever the any user-provided callback is invoked. Setting this option makes Libevent release the bufferevent’s lock when it’s invoking your callbacks.

如果设置了线程安全选项，设置这个选项会使得调用我们的回调函数时释放锁。

Working with socket-based bufferevents

`1创建bufferevent，成功返回bufferevent的指针，失败返回空，options是上面提到的选项。`
``` cpp
struct bufferevent *bufferevent_socket_new(
    struct event_base *base,
    evutil_socket_t fd,
    enum bufferevent_options options);
 ```

 `2连接函数`

int bufferevent_socket_connect(struct bufferevent *bev,
    struct sockaddr *address, int addrlen);
 

the address and addrlen arguments are as for the standard call connect(). If the bufferevent does not already have a socket set, calling this function allocates a new stream socket for it, and makes it nonblocking.

If the bufferevent does have a socket already, calling bufferevent_socket_connect() tells Libevent that the socket is not connected, and no reads or writes should be done on the socket until the connect operation has succeeded.

It is okay to add data to the output buffer before the connect is done.

如果bufferevent没有设置socket，调用这个函数会为它创建一个非阻塞socket

如果bufferevent有socket，调用这个函数会通知libevent在连接成功前不能调用读或者写。

在连接成功前可以向output buffer中添加数据。

下面是例子：

``` cpp
#include <event2/event.h>
#include <event2/bufferevent.h>
#include <sys/socket.h>
#include <string.h>

void eventcb(struct bufferevent *bev, short events, void *ptr)
{
    if (events & BEV_EVENT_CONNECTED) {
         /* We're connected to 127.0.0.1:8080.   Ordinarily we'd do
            something here, like start reading or writing. */
    } else if (events & BEV_EVENT_ERROR) {
         /* An error occured while connecting. */
    }
}

int main_loop(void)
{
    struct event_base *base;
    struct bufferevent *bev;
    struct sockaddr_in sin;

    base = event_base_new();

    memset(&sin, 0, sizeof(sin));
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = htonl(0x7f000001); /* 127.0.0.1 */
    sin.sin_port = htons(8080); /* Port 8080 */

    bev = bufferevent_socket_new(base, -1, BEV_OPT_CLOSE_ON_FREE);

    bufferevent_setcb(bev, NULL, NULL, eventcb, NULL);

    if (bufferevent_socket_connect(bev,
        (struct sockaddr *)&sin, sizeof(sin)) < 0) {
        /* Error starting connection */
        bufferevent_free(bev);
        return -1;
    }

    event_base_dispatch(base);
    return 0;
}
```

Note that you only get a BEV_EVENT_CONNECTED event if you launch the connect() attempt using bufferevent_socket_connect(). If you call connect() on your own, the connection gets reported as a write.

If you want to call connect() yourself, but still get receive a BEV_EVENT_CONNECTED event when the connection succeeds, call bufferevent_socket_connect(bev, NULL, 0) after connect() returns -1 with errno equal to EAGAIN or EINPROGRESS.

如果通过bufferevent_socket_connect连接，那么返回的事件是BEV_EVENT_CONNECTED ，

如果通过connect连接，那么返回的是write事件。如果调用了connect，还想捕捉到BEV_EVENT_CONNECTED 事件，可以继续调用bufferevent_socket_connect(bev,NULL, 0),返回值为-1，errno为EAGAIN或者EINPROGRESS

`3释放bufferevent`

void bufferevent_free(struct bufferevent *bev);
This function frees a bufferevent. Bufferevents are internally reference-counted, so if the bufferevent has pending deferred callbacks when you free it, it won’t be deleted until the callbacks are done.

The bufferevent_free() function does, however, try to free the bufferevent as soon as possible. If there is pending data to write on the bufferevent, it probably won’t be flushed before the bufferevent is freed.

If the BEV_OPT_CLOSE_ON_FREE flag was set, and this bufferevent has a socket or underlying bufferevent associated with it as its transport, that transport is closed when you free the bufferevent.

当bufferevent有延迟回调函数没处理，调用bufferevent_free并不会立即释放bufferevent，bufferevent内部是引用计数的，他能做到的是尽快释放，如果bufferevent上存在阻塞的数据还没有写，这部分数据在bufferevent释放前是不会被释放的。

`4 其他的一些设置函数`

是指bufferevent回调函数，以及获取回调函数

``` cpp
typedef void (*bufferevent_data_cb)(struct bufferevent *bev, void *ctx);
typedef void (*bufferevent_event_cb)(struct bufferevent *bev,
    short events, void *ctx);

void bufferevent_setcb(struct bufferevent *bufev,
    bufferevent_data_cb readcb, bufferevent_data_cb writecb,
    bufferevent_event_cb eventcb, void *cbarg);

void bufferevent_getcb(struct bufferevent *bufev,
    bufferevent_data_cb *readcb_ptr,
    bufferevent_data_cb *writecb_ptr,
    bufferevent_event_cb *eventcb_ptr,
    void **cbarg_ptr);
```
 

`5 读写生效函数，读写失效函数`

void bufferevent_enable(struct bufferevent *bufev, short events);
void bufferevent_disable(struct bufferevent *bufev, short events);

short bufferevent_get_enabled(struct bufferevent *bufev);
You can enable or disable the events EV_READ, EV_WRITE, or EV_READ|EV_WRITE on a bufferevent. When reading or writing is not enabled, the bufferevent will not try to read or write data.

There is no need to disable writing when the output buffer is empty: the bufferevent automatically stops writing, and restarts again when there is data to write.

Similarly, there is no need to disable reading when the input buffer is up to its high-water mark: the bufferevent automatically stops reading, and restarts again when there is space to read.

By default, a newly created bufferevent has writing enabled, but not reading.

You can call bufferevent_get_enabled() to see which events are currently enabled on the bufferevent.

 可以设置bufferevent 读，写，或者读和写生效，当读或者写失效，bufferevent将不会尝试读或者写数据。在output为空时没必要让写操作失效，bufferevent会自动停止写，当有数据可写时才再次执行写操作。

同样的，在input buffer超过高水位时，没必要设置读失效，bufferevent 会自动停止读，而等到有空间(其实是input缓冲区非满)再次开始接收数据。

默认情况下，新创建的 bufferevent是可写的不可读的。(因为如果设置可读选项，那么会出现busyloop，因为input一直非满)

用户可以通过bufferevent_get_enabled获取当前哪些事件类型是允许的。

 

`6设置水位接口`

void bufferevent_setwatermark(struct bufferevent *bufev, short events,
    size_t lowmark, size_t highmark);
The bufferevent_setwatermark() function adjusts the read watermarks, the write watermarks, or both, of a single bufferevent. (If EV_READ is set in the events field, the read watermarks are adjusted. If EV_WRITE is set in the events field, the write watermarks are adjusted.)

A high-water mark of 0 is equivalent to "unlimited".

bufferevent_setwatermark函数调整读或者写的水位。

 

`7获取读写缓冲区的内容`

struct evbuffer *bufferevent_get_input(struct bufferevent *bufev);
struct evbuffer *bufferevent_get_output(struct bufferevent *bufev);
 

 

`8向bufferevent中写数据`

int bufferevent_write(struct bufferevent *bufev,
    const void *data, size_t size);
int bufferevent_write_buffer(struct bufferevent *bufev,
    struct evbuffer *buf);
 

调用bufferevent_write可以向bufev的output 的末尾追加写入。

bufferevent_write_buffer将buf中的数据移除，写入到bufevoutput中。

两个函数返回0表示成功，-1表示失败。

 

`9从bufferevent中读数据`

size_t bufferevent_read(struct bufferevent *bufev, void *data, size_t size);
int bufferevent_read_buffer(struct bufferevent *bufev,
    struct evbuffer *buf); 
 

bufferevent_read 从bufev中input buffer 读取size大小数据，存在data中。

bufferevent_read_buffer从bufev的input buffer中将所有数据取出放入到buf中。

两个函数返回0表示成功，-1表示失败。

`10设置读写事件的超时时间`

void bufferevent_set_timeouts(struct bufferevent *bufev,
    const struct timeval *timeout_read, const struct timeval *timeout_write);

`11强制刷新读或者写`

int bufferevent_flush(struct bufferevent *bufev,
    short iotype, enum bufferevent_flush_mode state);
 

 强制从底层读取数据，或将数据写入底层。The iotype argument should be EV_READ, EV_WRITE, or EV_READ|EV_WRITE to indicate whether bytes being read, written, or both should be processed.
