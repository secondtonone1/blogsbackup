---
title: libevent学习文档(二)eventbase相关接口和参数
date: 2017-08-07 10:40:02
categories: [网络编程]
tags: [网络编程,Linux环境编程]
---
Setting up a default event_base

The `event_base_new()` function allocates and returns a new event base with the default settings. It examines the environment variables and returns a pointer to a new event_base.

If there is an error, it returns NULL.

When choosing among methods, it picks the fastest method that the OS supports.

event_base_new()创建默认`event_base`，他检测环境变量并且返回一个指向新的event_base的指针。

当调用方法时，选择操作系统支持的做快速的方法。

Interface

struct event_base *event_base_new(void);
Setting up a complicated event_base

If you want more control over what kind of event_base you get, you need to use an event_config. An event_config is an opaque structure that holds information about your preferences for an event_base. When you want an event_base, you pass the event_config to event_base_new_with_config().

想要创建指定类型的event_base，需要使用event_config，event_config是不透明的结构体，保存了您创建event_base时需要的特性。

用户调用`event_base_new_with_config`创建特殊类型的event_base
<!--more-->
Interface

struct event_config *event_config_new(void);
struct event_base *event_base_new_with_config(const struct event_config *cfg);
void event_config_free(struct event_config *cfg);
 

To allocate an event_base with these functions, you call event_config_new() to allocate a new event_config. Then, you call other functions on the event_config to tell it about your needs. Finally, you call event_base_new_with_config() to get a new event_base. When you are done, you can free the event_config with event_config_free().

为了创造一个有特定功能的event_base，你需要调用event_config_new()创建event_config，只后调用event_base_new_with_config()，之后调用event_config_free释放。

Interface：

``` cpp
int event_config_avoid_method(struct event_config *cfg, const char *method);

enum event_method_feature {
    EV_FEATURE_ET = 0x01,
    EV_FEATURE_O1 = 0x02,
    EV_FEATURE_FDS = 0x04,
};
int event_config_require_features(struct event_config *cfg,
                                  enum event_method_feature feature);

enum event_base_config_flag {
    EVENT_BASE_FLAG_NOLOCK = 0x01,
    EVENT_BASE_FLAG_IGNORE_ENV = 0x02,
    EVENT_BASE_FLAG_STARTUP_IOCP = 0x04,
    EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,
    EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST = 0x10,
    EVENT_BASE_FLAG_PRECISE_TIMER = 0x20
};
int event_config_set_flag(struct event_config *cfg,
    enum event_base_config_flag flag);
```
 

Calling event_config_avoid_method tells Libevent to avoid a specific available backend by name. Calling event_config_require_feature() tells Libevent not to use any backend that cannot supply all of a set of features. Calling event_config_set_flag() tells Libevent to set one or more of the run-time flags below when constructing the event base.

 

event_config_avoid_method 可以规避一些函数，通过传入函数的名字，event_config_require_feature告诉libevent不要使用不支持指定特性的功能。

event_config_set_flag告诉libevent设置一个或多个运行时标记，当你创建event base时。

特性的标记位如下：

`EV_FEATURE_ET`：要求后端方法支持edge-triggered IO

`EV_FEATURE_O1`：要求添加或者删除，或者激活某个事件的事件复杂度为O（1）

`EV_FEATURE_FDS`：要求后端方法支持任意类型的文件描述符，不仅仅是socket

`EVENT_BASE_FLAG_NOLOCK`：不为event_base设置锁，可以省去event_base加锁和释放锁的时间，但是对于多线程操作，并不安全。

`EVENT_BASE_FLAG_IGNORE_ENV`：当调用后端的方法时并不检查EVENT_*环境变量，用这个选项时注意，会导致程序和Libevent库之间调试麻烦

`EVENT_BASE_FLAG_STARTUP_IOCP`：仅对windows有效，通知Libevent在启动的时候就是用必要的IOCP 派发逻辑，而不是在需要时才用IOCP派发逻辑。

`EVENT_BASE_FLAG_NO_CACHE_TIME`：每次超时回调函数调用后检测当前时间，而不是准备调用超时回调函数前检测。

`EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST`：通知libevent，如果后台使用epoll模型，使用changelist是安全的，

`epoll-changelist` 能够避免在两次派发之间如果同一个fd的状态有改变，多次不必要的系统调用。换句话说在派发消息中间，如果同一个fd多次改动，那么最终只调用一次系统

调用。如果后台不是epoll模型，那么这个选项是没什么影响。同样可以设置EVENT_EPOLL_USE_CHANGELIST 环境变量达到这个目的。

`EVENT_BASE_FLAG_PRECISE_TIMER`：libevent默认使用快速的定时器机制，设置这个选项后，如果有一个慢速的但是定时器精度高的机制，那么就会切换到这个机制。

如果没有这个机制，那么这个选项没什么影响。

下面几个例子说明了这些api用法

``` cpp
static void
test_base_features(void *arg)
{
    struct event_base *base = NULL;
    struct event_config *cfg = NULL;
    //创建一个event_config
    cfg = event_config_new();
    //设置在epoll模型情况下支持et模式
    tt_assert(0 == event_config_require_features(cfg, EV_FEATURE_ET));
    //创建event_base
    base = event_base_new_with_config(cfg);
    if (base) {
        tt_int_op(EV_FEATURE_ET, ==,
            event_base_get_features(base) & EV_FEATURE_ET);
    } else {
        base = event_base_new();
        tt_int_op(0, ==, event_base_get_features(base) & EV_FEATURE_ET);
    }

end:
    //结束时要释放base和config
    if (base)
        event_base_free(base);
    if (cfg)
        event_config_free(cfg);
}
```

下面这个是测试方法的

``` cpp
static void
test_methods(void *ptr)
{
    //获取event_base支持的方法
    const char **methods = event_get_supported_methods();
    struct event_config *cfg = NULL;
    struct event_base *base = NULL;
    const char *backend;
    int n_methods = 0;

    tt_assert(methods);

    backend = methods[0];
    while (*methods != NULL) {
        TT_BLATHER(("Support method: %s", *methods));
        ++methods;
        ++n_methods;
    }

    cfg = event_config_new();
    assert(cfg != NULL);
     //event_base屏蔽backend的方法
    tt_int_op(event_config_avoid_method(cfg, backend), ==, 0);
    //并且设置了忽略ENV环境变量的选项
    event_config_set_flag(cfg, EVENT_BASE_FLAG_IGNORE_ENV);
    //创建event_base
    base = event_base_new_with_config(cfg);
    if (n_methods > 1) {
        tt_assert(base);
        tt_str_op(backend, !=, event_base_get_method(base));
    } else {
        tt_assert(base == NULL);
    }

end:
    if (base)
        event_base_free(base);
    if (cfg)
        event_config_free(cfg);
}
```
谢谢关注我的公众号：
![1.jpg](1.jpg)
 

