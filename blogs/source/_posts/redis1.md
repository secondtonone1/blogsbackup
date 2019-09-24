---
title: 对于redis底层框架的理解(一)
date: 2017-08-07 17:35:13
categories: [网络编程]
tags: [网络编程]
---
近期学习了`redis底层框架`，好多东西之前都没听说过，算是大开眼界了。
先梳理下redis正常的通讯流程吧
首先服务器启动都有主函数main，这个main函数就在redis.c里

首先是`initserverconfig()`，在这里初始化了redisserver基本的配置信息，

接着调用`loadServerConfig(char *filename)` 对 server 全局变量重新初始化。

然后是调用`daemonize()`，实现守护进程，脱离了控制台，使这个进程成为独立的首领进程

接下来是`initServer()`，这个过程很重要，完成了事件驱动的注册和一些回调函数的绑定，回头仔细说这个函数里面的功能

初始化服务器过后`aeSetBeforeSleepProc()`，设置了服务器休眠之前会

调用`beforeSleep函数`,然后进入主要的事件轮询函数 `aeMain(server.el)`，在这里完成事件的派发

最后事件轮询过后我们调用`aeDeleteEventLoop`，释放之前开辟的内存，

结束进程。

中间略去一些琐碎的过程，我们总结一下

`initserverconfig() ----> loadServerConfig------> daemonize()------->`

`initServer()-----> aeSetBeforeSleepProc()------>aeMain()----->aeDeleteEventLoop`

<!--more-->

接下来详细说一下每一个步骤都做了什么。

看一下main()函数
``` cpp
int main(int argc, char **argv) {

       

    //设置时间，一般都是设置事件poll等待多长时间返回

     struct timeval tv;

 

    /* We need to initialize our libraries, and the server configuration. */

    #ifdef INIT_SETPROCTITLE_REPLACEMENT

        //进程重命名

        spt_init(argc, argv);

    #endif

   //好像是更改字符编码

    setlocale(LC_COLLATE,"");

    //设置多线程安全模式

    zmalloc_enable_thread_safeness();

    //注册内存使用过量报错的函数

    zmalloc_set_oom_handler(redisOutOfMemoryHandler);

    srand(time(NULL)^getpid());

    gettimeofday(&tv,NULL);

    //哈希种子

    dictSetHashFunctionSeed(tv.tv_sec^tv.tv_usec^getpid());

    //服务器的启动模式：单机模式、Cluster模式、sentinel模式

    server.sentinel_mode = checkForSentinelMode(argc,argv);

    initServerConfig();

    loadServerConfig(configfile,options);

    。。。

 

    

           //创建守护进程

            if (server.daemonize) daemonize();

            //初始化服务器

            initServer();

            //设置服务器sleep之前的函数调用

            aeSetBeforeSleepProc(server.el,beforeSleep);

            //主函数事件驱动

            aeMain(server.el);

            //删除事件循环的结构，释放空间

            aeDeleteEventLoop(server.el);

            return 0;

}
```
谢谢关注我的公众号：
![1.jpg](1.jpg)

