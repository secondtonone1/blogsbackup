---
title: select网络模型知识总结
date: 2017-08-07 12:41:57
categories: 技术开发
tags: [网络编程]
---
`select模型支持IO多路复用`，select函数如下
``` cpp
int select (
 IN int nfds,                           //windows下无意义，linux有意义
 IN OUT fd_set* readfds,      //检查可读性
 IN OUT fd_set* writefds,     //检查可写性
 IN OUT fd_set* exceptfds,  //例外数据
 IN const struct timeval* timeout);    //函数的返回时间
```
逐个解释每个参数意义：

`nfds：一个整型变量，表示比最大文件描述符+1`

`readfds： 这个集合监测读事件的描述符`，将要监听读事件的文件描述符放入readfds中，通过调用select，readfds中将没有就绪的读事件文件描述符清除，留下

就绪的读事件描述符，可以通过read或者recv来处理

`writefds：这个集合监测写事件的描述符`，将要监听的写事件的文件描述符放入writefds中，通过调用select，writefds中没有就绪的写事件文件描述符被清除，留下

就绪的写事件描述符，可以通过write或者send来处理。
<!--more-->

`execptfds：这个集合在调用select后会存有错误的文件描述符`。根据Linux网络网络编程第二版中介绍，可以监视带外数据OOB，带外数据使用MSG_OOB标志发送

到套接字上，当select()函数返回的时候，readfds将清除其中的其他文件描述符，留下OOB数据

函数返回值：

`当返回0时表示超时，-1表示有错误，大于0表示没有错误`。

`当监视文件集中有文件描述符符合要求，即读文件描述符集合中有文件可读，写文件描述符集合中有文件可写，或者错误文件描述符集合中有错误的描述符，都会返回大于0的数`
 
``` cpp
timeval结构体解释

struct  timeval {
        long    tv_sec;        //秒
        long    tv_usec;     //毫秒
};
```
`timeval指针为NULL，表示一直等待，直到有符合条件的描述符触发select返回`

`如果timeval中个参数均为0，表示立即返回`，`否则在select没有符合条件的描述符，等待对应的时间和，然后返回`。另外需要了解一些select的操作宏函数

fd_set是一个SOCKET队列，以下宏可以对该队列进行操作：
`FD_CLR( s, fd_set *set) 从队列set删除句柄s;`
`FD_ISSET( s,fd_set *set) 检查句柄s是否存在与队列set中;`
`FD_SET( s, fd_set *set )把句柄s添加到队列set中;`
`FD_ZERO( fd_set *set ) 把set队列初始化成空队列.`

看一个select的使用示例

``` cpp
//前面是服务器socket的创建，绑定和监听

int  listenSocket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);     
 // Bind     
  
 local.sin_family = AF_INET;     
 local.sin_addr.S_un.S_addr = htonl(INADDR_ANY);     
 local.sin_port = htons(PORT);     
 bind(listenSocket, (sockaddr*)&local, sizeof(SOCKADDR_IN));     
  
 // Listen     
  
 listen(listenSocket, 3);
 //设置非阻塞
 fcntl(listenSocket, F_SETFL, O_NONBLOCK );
 
 //这个使用来统计所有已经连接上来的文件描述符，
 //也可以用数组表示，只是遍历的时候要根据数据结构进行更改
FD_SET socketSet;  
FD_SET writeSet;  
FD_SET readSet;
 
//清空 socketSet
FD_ZERO(&socketSet);
//将文件描述符放入 socketSet，
//用于accept  
FD_SET(listenSocket,&socketSet); 
//统计最大的socket 
int maxfd = listenSocket;
int conNum = 1;
//数组存储连接的socket
int connectArray[1024]={0};
while(true)  
{  
    //清空读写集合
    FD_ZERO(&readSet);  
    FD_ZERO(&writeSet);  
    //读写都监听
    readSet=socketSet;  
    writeSet=socketSet;  
  
    //同时检查套接字的可读可写性。 
    //为等待时间传入NULL，则永久等待。传入0立即返回。不要勿用。
    int ret=select(maxfd,&readSet,&writeSet,NULL,NULL);  
    if(ret==-1)  
    {  
        return false;  
    }  
    sockaddr_in addr;  
    int len=sizeof(addr);  
    //是否存在客户端的连接请求。  
    //在readset中会返回已经调用过listen的套接字
    if(FD_ISSET(listenSocket,&readSet))
    {  
        acceptSocket=accept(listenSocket,(sockaddr*)&addr,&len);  
        if(acceptSocket==INVALID_SOCKET)  
        {  
            return false;  
        }  
        else  
        {  
            //大于我们最大的监听数量了
            if(conNum > 1024)
            {
                return false;
            }
            //更新数组
            connectArray[conNum] = acceptSocket;
            //设置非阻塞
            fcntl(connectArray[conNum], F_SETFL, O_NONBLOCK );
            //加到socketset里，以后赋值给读写集合
            FD_SET(connectArray[conNum],&socketSet);
            if(acceptSocket > maxfd)
            {
                maxfd = acceptSocket;
            }
            conNum++;
        }  
    }  
  
    for(int i=0;i<conNum;i++)  
    {  
        //判断是否有读事件
       if(FD_ISSET(connectArray[i],&readSet))  
        {  
            //调用recv，接收数据。  
            //判断recv结果，为0则客户端断开，
            //那么调用FD_CLR并关闭对应的socket
        }  
        //判断是否有写事件
        if(FD_ISSET(connectArray[i],&writeSet)  
        {  
            //调用send，发送数据。
            //判断send结果，为0客户端断开，
            //那么调用FD_CLR并关闭对应的socket    
        }  
    }  
}  
 
```
 

上面的例子结合了网上提供的一些demo，`其实writeSet不一定要放入socket，当某个socket需要send内容时,再调用FD_SET(socket,&writeSet),写成功后再调用FD_CLR(socket,&writeSet);避免造成busyloop，`因为当缓冲区非空时，写事件是一直就绪的。

