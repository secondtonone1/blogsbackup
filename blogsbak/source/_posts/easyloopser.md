---
title: 简单循环服务器
date: 2017-08-07 16:57:52
categories: 技术开发
tags: [网络编程]
---
客户端部分：

比较简单

创建socket 然后connect服务器，进行通讯
![1.png](1.png)
<!--more-->
发送数据，并且接收数据，然后关闭
![2.png](2.png)
服务器部分：
服务器要做的是创建socket，初始化地址信息，并且绑定socket，然后进行监听
![3.png](3.png)
然互就是在循环里处理客户端连接上来的请求，并且接受信息，回发信息
![4.png](4.png)
循环服务器比较简单，而且是阻塞的，这种模型只是用于理解socket基本原理，并不能满足生产的需求

 源代码下载地址： [http://download.csdn.net/detail/secondtonone1/9510567](http://download.csdn.net/detail/secondtonone1/9510567)

感兴趣请关注我的微信公众号
![5.jpg](5.jpg)