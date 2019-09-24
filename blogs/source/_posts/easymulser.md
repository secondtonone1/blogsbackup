---
title: 简单的并发服务器（多个线程各自accept）
date: 2017-08-07 16:49:17
categories: [网络编程]
tags: [网络编程]
---
基于之前讲述的简单循环服务器，做一个多个线程各自accept的服务器demo由于多个线程各自accept，容易造成数据错误，需要在accept前后枷锁
先看下客户端
客户端创建socket，初始化服务器地址信息，然后进行连接
![1.png](1.png)
<!--more-->
连接成功后发送信息给服务器，并且接受服务器回传的信息
![2.png](2.png)
服务器部分：

服务器静态的的初始化一个线程锁，然后去写一个处理连接请求的函数，在循环里accept客户端的连接，接受客户端的消息，并且发送给客户端事件，由于多个线程各自accept，所以要在accept处加锁，避免产生的socket有误
![3.png](3.png)
处理连接，这个主要是服务器创建了几个线程，各自线程的回调函数是handle_request，参数是socket , pthread_join是等待线程安全退出后进程结束
![4.png](4.png)
服务器的main函数
![5.png](5.png)
这是个阻塞模式下的多线程accept，实际生产和项目中都采用非阻塞模型，只是用来理解socket通信

原理，并无太大用处。

源代码下载地址： [http://download.csdn.net/detail/secondtonone1/9510570](http://download.csdn.net/detail/secondtonone1/9510570)
我的微信公众号，谢谢
![1.jpg](1.jpg)