---
title: ECONNRESET和WSAECONNRESET怎么产生的以及如何避免
date: 2017-08-04 17:02:28
categories: 技术开发
tags: [网络编程]
---
`ECONNRESET`是linux环境网络编程产生的错误，错误码为104，

`WSAECONNRESET`是windows环境网络编程产生的错误，错误码为10054

两者产生的原因都一样，分以下几种情况：

1`接收端recv或者read， 对端已经关闭连接，recv/read返回该错误`

2 对端重启连接，还未建立连接

3 `发送端已经断开连接，但是调用send会触发这个错误`

第二点第三点都可以通过判断返回值解决，第一点在一些看似正常情况下也会触发该错误。比如对端close(fd)，接收端调用recv并没有返回0，而是-1，打印错误码为104或

10054，按道理讲这种情况按照返回值为0处理是可以的，但是尽量将代码写的规范一些，避免不必要的错误。
<!--more-->

为什么close(fd)会导致接收端读到复位RST，也就是收到错误的104呢？

因为close(fd)只是将文件描述符关闭，并没有关闭tcp建立起来的连接，断开连接需要四次握手，倘若发送端发送缓冲区有数据未发送完或者接受缓冲区有数据未读完，调用close(fd)，那么连接并没有关闭，这样，接收端收到的就是所谓的104或10054错误了。如何避免这个错误呢，就需要我们判断发送端发送和接受操作是否进行完，也就是判断缓冲区是否有数据，如果有数据需要等待数据处理完毕在关闭，否则会出现上述错误。

有一个做法是通过调用shutdown(s,SHUT_WR);
关闭发送端的写端，这样发送端不发送数据，然后调用close这次会发送关闭连接的FIN标志，接收端接收到FIN，那么recv或者read返回的就是0.

int shutdown(int sockfd,int how);
　　Sockfd是需要关闭的socket的描述符。参数 how允许为shutdown操作选择以下几种方式：
    SHUT_RD：关闭连接的读端。也就是该套接字不再接受数据，任何当前在套接字接受缓冲区的数据将被丢弃。

　　进程将不能对该套接字发出任何读操作。

　　对 TCP套接字该调用之后接受到的任何数据将被确认然后无声的丢弃掉。
    SHUT_WR:关闭连接的写端，进程不能在对此套接字发出写操作
    SHUT_RDWR:相当于调用shutdown两次：首先是以SHUT_RD,然后以SHUT_WR

下面摘用网上的一段话来说明二者的区别：
close-----关闭本进程的socket id，但链接还是开着的，用这个socket id的其它进程还能用这个链接，能读或写这个socket id
shutdown--则破坏了socket 链接，读的时候可能侦探到EOF结束符，写的时候可能会收到一个SIGPIPE信号，这个信号可能直到
socket buffer被填充了才收到。

 close(sockfd);使用close中止一个连接，但它只是减少描述符的参考数，并不直接关闭连接，只有当描述符的参考数为0时才关闭连接。

而且shutdown只是处理连接关闭，并不能回收描述符，所以最终还是要调用close(fd)才能回收描述符，在所有描述符引用次数为0时发送

FIN消息给对端。

出了采取shutdown的方式，还可以通过设置socket属性，调用close时，检测在socket完成缓冲区读写后，才关闭连接

``` cpp
struct linger {
 
     int l_onoff; /* 0 = off, nozero = on */
 
     int l_linger; /* linger time */
 
};
```
有下列三种情况：
1、设置 l_onoff为0，则该选项关闭，l_linger的值被忽略，等于内核缺省情况，close调用会立即返回给调用者，如果可能将会传输任何未发送的数据；
 
2、设置 l_onoff为非0，l_linger为0，则套接口关闭时TCP夭折连接，TCP将丢弃保留在套接口发送缓冲区中的任何数据并发送一个RST给对方，而不是通常的四分组终止序列，这避免了TIME_WAIT状态；
 
3、设置 l_onoff 为非0，l_linger为非0，当套接口关闭时内核将拖延一段时间（由l_linger决定）。如果套接口缓冲区中仍残留数据，进程将处于睡眠状态，直 到（a）所有数据发送完且被对方确认，之后进行正常的终止序列（描述字访问计数为0）或（b）延迟时间到。
 
下面是代码：
``` cpp
int z;
 int s;      
 struct linger so_linger;
  
 so_linger.l_onoff = 1
 so_linger.l_linger = 30;
 z = setsockopt(s,SOL_SOCKET,SO_LINGER,&so_linger,
     sizeof so_linger);
 if ( z )
     perror("setsockopt(2)");
     close(s);
```

到目前为止，我觉得比较好的主动关闭方式是：

关闭端：

1确保发送缓存区没有数据未发送，调用shutdown(fd,SHUTWR);

2如果能接收到数据，继续接受，直到接收到对方的FIN，也就是

read返回0或者-1

3如果接收到关闭信号，那么调用close正常关闭。