---
title: libevent文档学习(一)多线程接口和使用
date: 2017-08-07 10:44:51
categories: [网络编程]
tags: [网络编程,Linux环境编程]
---
参考libevent官方提供的文档： [http://www.wangafu.net/~nickm/libevent-book/Ref1_libsetup.html](http://www.wangafu.net/~nickm/libevent-book/Ref1_libsetup.html)

这一篇主要翻译`libevent多线程`的使用接口和文档。

As you probably know if you’re writing multithreaded programs, it isn’t always safe to access the same data from multiple threads at the same time.

Libevent structures can generally work three ways with multithreading.

Some structures are inherently single-threaded: it is never safe to use them from more than one thread at the same time.

Some structures are optionally locked: you can tell Libevent for each object whether you need to use it from multiple threads at once.

Some structures are always locked: if Libevent is running with lock support, then they are always safe to use from multiple threads at once.

 

当你编写多线程程序的时候，多个线程访问同一块数据并不安全。对于多线程libevent通常采取以下三种方式工作，

`1 一些结构内不是单线程的，多线程同时访问这个结构是不安全的。`

`2一些结构内部是选择性加锁的，你需要通知libevent，对于每个对象你是否采用多线程的方式使用它。`

`3一些结构总是加锁的，如果libevent设置了加锁的模式，采用多线程方式是安全的。`

 

To get locking in Libevent, you must tell Libevent which locking functions to use. You need to do this before you call any Libevent function

that allocates a structure that needs to be shared between threads.

If you are using the pthreads library, or the native Windows threading code, you’re in luck. There are pre-defined functions that will set

Libevent up to use the right pthreads or Windows functions for you.

 

如果要使用libevent多线程锁的功能，需要开辟一个线程共享的结构，如果您使用windows或者linux提供的pthread库，libevent已经定义好了。

``` cpp
#ifdef WIN32
int evthread_use_windows_threads(void);
#define EVTHREAD_USE_WINDOWS_THREADS_IMPLEMENTED
#endif
#ifdef _EVENT_HAVE_PTHREADS
int evthread_use_pthreads(void);
#define EVTHREAD_USE_PTHREADS_IMPLEMENTED
#endif
```
libevent针对win32平台定义了`evthread_use_windows_threads`,

libevent针对Linux thread库 定义了`evthread_use_pthreads`

evthread_use_pthread函数是这样的
<!--more-->
``` cpp
int
evthread_use_pthreads(void)
{
    //pthread lock callback结构体对象
    struct evthread_lock_callbacks cbs = {
        //锁的版本
        EVTHREAD_LOCK_API_VERSION,
        //锁的属性
        EVTHREAD_LOCKTYPE_RECURSIVE,
        //创建锁
        evthread_posix_lock_alloc,
        //回收锁
        evthread_posix_lock_free,
        //加锁回调函数
        evthread_posix_lock,
        //解锁回调函数
        evthread_posix_unlock
    };
    //条件变量回调结构体对象
    struct evthread_condition_callbacks cond_cbs = {
        //条件变量的版本
        EVTHREAD_CONDITION_API_VERSION,
        //创建条件变量
        evthread_posix_cond_alloc,
        //回收条件变量
        evthread_posix_cond_free,
        //激活条件的回调函数
        evthread_posix_cond_signal,
        //条件不满足阻塞的回调函数
        evthread_posix_cond_wait
    };

    //设置互斥锁的属性
    /* Set ourselves up to get recursive locks. */
    if (pthread_mutexattr_init(&attr_recursive))
        return -1;
    if (pthread_mutexattr_settype(&attr_recursive, PTHREAD_MUTEX_RECURSIVE))
        return -1;
    //将cbs的属性设置到全局变量中，分为调试和正常模式
    evthread_set_lock_callbacks(&cbs);
    //同样将cond_cbs设置到全局变量
    evthread_set_condition_callbacks(&cond_cbs);
    //设置线程id到全局变量
    evthread_set_id_callback(evthread_posix_get_id);
    return 0;
}
```

这几个结构体如下

``` cpp
#define EVTHREAD_WRITE  0x04
#define EVTHREAD_READ   0x08
#define EVTHREAD_TRY    0x10

#define EVTHREAD_LOCKTYPE_RECURSIVE 1
#define EVTHREAD_LOCKTYPE_READWRITE 2

#define EVTHREAD_LOCK_API_VERSION 1

struct evthread_lock_callbacks {
       int lock_api_version;
       unsigned supported_locktypes;
       void *(*alloc)(unsigned locktype);
       void (*free)(void *lock, unsigned locktype);
       int (*lock)(unsigned mode, void *lock);
       int (*unlock)(unsigned mode, void *lock);
};

int evthread_set_lock_callbacks(const struct evthread_lock_callbacks *);

void evthread_set_id_callback(unsigned long (*id_fn)(void));

struct evthread_condition_callbacks {
        int condition_api_version;
        void *(*alloc_condition)(unsigned condtype);
        void (*free_condition)(void *cond);
        int (*signal_condition)(void *cond, int broadcast);
        int (*wait_condition)(void *cond, void *lock,
            const struct timeval *timeout);
};

int evthread_set_condition_callbacks(
        const struct evthread_condition_callbacks *);
```

The evthread_lock_callbacks structure describes your locking callbacks and their abilities. For the version described above, the lock_api_version field must be set to

EVTHREAD_LOCK_API_VERSION. The supported_locktypes field must be set to a bitmask of the EVTHREAD_LOCKTYPE_* constants to describe which lock types you can support.

(As of 2.0.4-alpha, EVTHREAD_LOCK_RECURSIVE

is mandatory and EVTHREAD_LOCK_READWRITE is unused.) The alloc function must return a new lock of the specified type. The free function must release all resources held by a lock of the

specified type. The lock function must try to acquire the lock in the specified mode, returning 0 on success and nonzero on failure. The unlock function must try to unlock the lock, returning 0 on success and nonzero on failure.

理解：

`evthread_lock_callbacks` 包括了锁的回调函数和他们的功能，`lock_api_version` 要被设置为EVTHREAD_LOCK_API_VERSION，

`supported_locktypes` 应该设置为自己需要的EVTHREAD_LOCKTYPE_开头的几个类型的bitmask按位或alloc 函数返回指定类型的锁，free

 函数释放指定类型的锁的所有资源，lock 函数试图获取制定模式的锁，成功返回0，失败返回非0.

`unlock` 解锁函数释放指定的锁，成功返回0，失败返回非0

 

Recognized lock types are:

0
A regular, not-necessarily recursive lock.

EVTHREAD_LOCKTYPE_RECURSIVE
A lock that does not block a thread already holding it from requiring it again. Other threads can acquire the lock once the thread holding it has unlocked it as many times

as it was initially locked.

`EVTHREAD_LOCKTYPE_READWRITE`
A lock that allows multiple threads to hold it at once for reading, but only one thread at a time to hold it for writing. A writer excludes all readers.

 

0表示常规锁，不可以被重复上锁

 

`EVTHREAD_LOCKTYPE_RECURSIVE`：这种锁当一个线程持有，该线程可以继续获取他而不被阻塞，其他线程需要等到该线程释放这个锁后可以获取到这个锁，

并且可以多次加锁。

 

`EVTHREAD_LOCKTYPE_READWRITE`：这种锁多线程可以在读的时候都获取到他，但是写操作时只允许一个线程持有。

 

Recognized lock modes are:

`EVTHREAD_READ`
For READWRITE locks only: acquire or release the lock for reading.

`EVTHREAD_WRITE`
For READWRITE locks only: acquire or release the lock for writing.

`EVTHREAD_TRY`
For locking only: acquire the lock only if the lock can be acquired immediately.

`EVTHREAD_READ和EVTHREAD_WRITE`都是针对READWRITE 锁的获取和释放。

`EVTHREAD_TRY`：这个模式只在能立即获得锁的时候获取锁，否则不等待。

 

The id_fn argument must be a function returning an unsigned long identifying what thread is calling the function. It must always return the same number for the same thread, and must not ever return the same number for two different threads if they are both executing at the same time.

id_fn函数返回一个unsigned long标识调用该函数的线程。不同线程的返回值不同，同一个线程的返回值相同。

The evthread_condition_callbacks structure describes callbacks related to condition variables. For the version described above, the lock_api_version field must be set to EVTHREAD_CONDITION_API_VERSION. The alloc_condition function must return a pointer to a new condition variable. It receives 0 as its argument. The free_condition function must release storage and resources held by a condition variable. The wait_condition function takes three arguments: a condition allocated by alloc_condition, a lock allocated by the evthread_lock_callbacks.alloc function you provided, and an optional timeout. The lock will be held whenever the function is called; the function must release the lock, and wait until the condition becomes signalled or until the (optional) timeout has elapsed. The wait_condition function should return -1 on an error, 0 if the condition is signalled, and 1 on a timeout. Before it returns, it should make sure it holds the lock again. Finally, the signal_condition function should cause one thread waiting on the condition to wake up (if its broadcast argument is false) and all threads currently waiting on the condition to wake up (if its broadcast argument is true). It will only be held while holding the lock associated with the condition.

`evthread_condition_callbacks` 描述了几个跟条件变量相关的回调函数。lock_api_version 应该被设置为EVTHREAD_CONDITION_API_VERSION，alloc_condition 喊回一个指向新的环境变量的指针，

`free_condition` 释放条件变量的资源，`wait_condition` 带有三个参数，分别是alloc_condition开辟的条件变量，`evthread_lock_callbacks`开辟的锁，以及一个可选的超时值，在调用这个函数时lock要提前加锁，

之后，函数内部必须释放锁，等待条件被唤醒或者超时，wait_condition 在错误时返回-1，超时返回1，被激活返回0.在该函数内部返回之前，他要自己上锁。signal_condition 激活单个线程，broadcast 参数设为true时

所有等待该条件的线程被激活。只有持有和条件相关的锁的时候线程才会被挂起。

 

libevent中开辟锁和释放等等的回调函数以及条件变量的相关函数实现比较简单，就不展开了，可以查看evthread_pthread.c文件。

下面看下系统是如何调用

`evthread_set_condition_callbacks`()和`evthread_set_lock_callbacks()`分别将条件回调的结构体对象和锁相关功能的结构体对象

赋值给全局变量

 

_evthread_cond_fns和_evthread_lock_fns,

libevent封装了几个通过`_evthread_cond_fns`和 `_evthread_lock_fns` 调用锁和条件变量的接口，

都在evthread-internal.h文件里。
``` cpp
EVTHREAD_ALLOC_LOCK(lockvar, locktype);

EVTHREAD_FREE_LOCK(lockvar, locktype);

EVLOCK_LOCK(lockvar,mode);

EVLOCK_UNLOCK(lockvar,mode);

EVTHREAD_ALLOC_COND(condvar);

EVTHREAD_FREE_COND(cond);

EVTHREAD_COND_SIGNAL(cond);

EVTHREAD_COND_WAIT(cond, lock);
```
等等，就不展开了，读者自己阅读。