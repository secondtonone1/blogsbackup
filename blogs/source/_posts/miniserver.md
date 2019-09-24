---
title: 基于epoll封装的事件回调miniserver
date: 2017-08-07 17:06:12
categories: [网络编程]
tags: [网络编程]
---
`epoll`技术前两节已经阐述过了，目前主要做一下封装，很多epoll的服务器都是采用`事件回调`方式处理，

其实并没有什么复杂的，我慢慢给大家阐述下原理。

在`networking.h和networking.cpp`里，这两个文件主要实现了一些文件读写功能的回调函数

![1.png](1.png)

`acceptCallBack` 负责新的描述符连接上来进行回调，

`readCallBack` 负责读操作回调
<!--more-->
`writeCallBack` 负责写操作回调

`initListenSocket` 负责初始化描述符的基本信息

`setNonblock`负责设置描述符非阻塞

`bindListenSocket`负责绑定描述符

实现如下
![2.png](2.png)

![3.png](3.png)

![4.png](4.png)

![5.png](5.png)

![6.png](6.png)

![7.png](7.png)
这些回调函数会赋值给EventLoop 的proc里，实现绑定，因为以后要回调
![8.png](8.png)

`creatEventLoop`为了创建并且初始化一个eventLoop

`deleteEventLoop`删除一个eventloop

`CreateFileEvent`将一个fd绑定到对应的epoll事件中，并且绑定fd对应的FileEvent事件类型和回调函数

`DeleteFileEvent`将一个fd解绑epoll事件，并且解绑fd对应的fileEvent事件类型和回调函数

`ProcessEvents`，轮询处理所有epoll就绪事件，并且调用之前注册好的回调函数

实现如下
![9.png](9.png)

![10.png](10.png)

![11.png](11.png)

![12.png](12.png)

上一部分将文件描述符加入epoll监听队列，以及从监听队列删除对应fd，或者轮询就绪事件都被我封装在apiepoll.h
和apiepoll.cpp里。
![13.png](13.png)

`ApiState`是基于epoll封装的结构体

`epfd表示epoll create产生的句柄`

`events表示epoll监听的事件队列`，这个大小可以自己开辟，一般都是

最大的客户端连接数+保留的一部分空间

`ApiCreate`表示创建epoll结构和句柄，将数据存储到eventLoop里

`ApiResize`重新开辟epoll序列大小

`ApiFree`释放epoll events

`ApiAddEvent` 将事件类型添加到epoll监听序列里

`ApiDelEvent` 将事件类型从epoll监听序列里删除

`ApiPoll`，就是epoll调用epoll_wait，返回就绪事件队列

轮询处理，回调函数就可以了
![14.png](14.png)
 
![15.png](15.png)

![16.png](16.png)

![17.png](17.png)

 源代码下载地址：[http://download.csdn.net/detail/secondtonone1/9502252](http://download.csdn.net/detail/secondtonone1/9502252)

关注我的公众号平台，定期推送技术总结
![1.jpg](1.jpg)