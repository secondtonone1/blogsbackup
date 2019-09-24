---
title: IO多路复用之epoll(一)讲解
date: 2017-08-07 17:25:29
categories: [网络编程]
tags: [网络编程]
---
网络通信中`socket`有自己的`内核发送缓冲区`和`内核接受缓冲区`，好比是一个水池，当用户发送数据的时候会从`用户缓冲区`拷贝到`socket的内核发送缓冲区`，然后从

socket发送缓冲区发出去, 当用户要读取数据时，就是从`socket内核读缓冲区`读到`用户缓冲区`。所以TCP中`recv`, `send`, `read`, `write`等函数并不是真的直接读写

发送报文，而是将数据分别写到socket内核缓冲区，或者从socket内核缓冲区读到用户区。

对于读有两种状态，`可读`和`不可读`，当用户从socket中读取数据的时候，如果socket读缓冲区内容为空，那么读数据会失败，阻塞情况下会等待数据到来，非阻塞情况下

会返回一个`EWOULDBLOCK` 或者`EAGAIN`，反之如果读缓冲区内容不为空，那么就会返回读到的字节数。

对于用户向socket中写数据，存在`可写`和`不可写`，如果socket写缓冲区内容满了，此时向socket写缓冲区里写数据会失败，阻塞情况下会等待socket的写缓冲区非满(有空闲空间)，非阻塞情况下回返回-1，根据errono 为`EWOULDBLOCK`或者`EAGAIN`，`判断写缓冲区满`

反之，如果socket写缓冲区此时非满，那么写会成功，返回写入的字节数。

综上所述，对于读的情况，我们关注socket读缓冲区空还是非空，对于写的情况，我们关注
<!--more-->
写缓冲区满还是非满。

讲述epoll 的几个参数含义

`EPOLLIN`：关联文件用来监听读事件
`EPOLLOUT`:关联文件用来监听写事件

`POLLRDNORM`，有普通数据可读。
`POLLRDBAND`，有优先数据可读。
`POLLPRI`，有紧迫数据可读。

`POLLWRNORM`，写普通数据不会导致阻塞。
`POLLWRBAND`，写优先数据不会导致阻塞。
`POLLMSG`，SIGPOLL消息可用。
此外，还可能返回下列事件：
`POLLER`，指定的文件描述符发生错误。
`POLLHUP`，指定的文件描述符挂起事件。
`POLLNVAL`，指定的文件描述符非法。

EPOLL有两种模式`LT（水平触发模式）ET（边缘触发模式）`

### LT：水平触发 一直触发
`1 内核中的socket接受缓冲区不为空，那么有数据可读，一直出发读事件`
`2 内核中的socket发送缓冲区不满，可以继续写入数据，写事件一直出发`
试想一下，如果一开始就关注EPOLLOUT，会是什么情况呢？对于LT模式，由于刚开始socket的发送缓冲区肯定是空的，那么
由于socket的发送缓冲区可以一直写数据，所以写事件会一直触发。正确的做法是，我们向socket中写数据，当返回EAGAIN(socket
发送缓冲区满了)的时候，这个时候如果关注EPOLLOUT，当socket发送缓冲区由满变为非满，会触发写事件。这个时候，就从应用
缓冲区取出数据拷贝到内核缓冲区。如果应用缓冲区的数据已经全部拷贝到内核缓冲区，说明数据全部在socket的发送缓冲区，
就取消EPOLLOUT的关注，如果应用缓冲区还有数据没拷贝完全，就不需要取消对EPOLLOUT的关注。

`优点是对于read操作比较简单，只要有read事件就读，读多读少都可以。`
`缺点是write相关操作较复杂， 由于socket在空闲状态发送缓冲区一定是不满的，
故若socket一直在epoll wait列表中，则epoll会一直通知write事件`，
所以必须保证没有数据要发送的时候，要把socket的write事件从epoll wait列表中删除。
而在需要的时候在加入回去，这就是LT模式的最复杂部分。

### ET：边沿触发模式
`由未就绪变为就绪状态才可触发`，比如写事件由缓冲区满变为缓冲区非满，读事件由缓冲区空到非空才触发
`1.内核中socket接受缓冲区不为空，那么有数据可读，可以触发读事件，但是对于边缘模式，只触发一次`
例如，当listen一个文件描述符，当有很多新的连接连上来的时候，只触发一次。所以要在循环中accept，取出所以新连接上来的socket
例如，当监听的socket有读事件，一定要读完所有的数据，否则之后到来的数据就会被漏掉。解决办法是在循环体内读取数据，
直到读完或者读出的返回值为EAGIN（缓冲区为空）才可以。对于写事件，发送数据，直到发送完或者产生EAGIN（缓冲区为满）
ET模式下，只有socket的状态发生变化时才会通知，也就是读取缓冲区由无数据到有数据时通知read事件，发送缓冲区由满变成未满通知write事件。

`缺点是epoll read事件触发时，必须保证socket的读取缓冲区数据全部读完（事实上这个要求很容易达到）`
`优点：对于write事件，发送缓冲区由满到未满时才会通知，若无数据可写，忽略该事件，若有数据可写，直接写。`
当向socket写数据，返回的值小于传入的buffer大小或者write系统调用返回EWouldBlock时，表示发送缓冲区已满。


平时大家使用 epoll 时都知道其事件触发模式有默认的 level-trigger 模式和通过 EPOLLET 启用的 edge-trigger 模式两种。
从 epoll 发展历史来看，它刚诞生时只有 edge-trigger 模式，后来因容易产生 race-cond 且不易被开发者理解，
又增加了 level-trigger 模式并作为默认处理方式。二者的差异在于 level-trigger 模式下只要某个 fd 
处于 readable/writable 状态，无论什么时候进行 epoll_wait 都会返回该 fd；
而 edge-trigger 模式下只有某个 fd 从 unreadable 变为 readable 或从 unwritable 变为 writable 时，
epoll_wait 才会返回该 fd。

### 然后我们来说说epoll几个函数
epoll的接口非常简单，一共就三个函数：
`1 int epoll_create(int size);`
创建一个epoll句柄，size用来告诉内核这个监听的数目一共有多大。size为最大监听数量+1，因为创建好epoll句柄后，他就会占用一个fd值，
在linux下如果查看/proc/进程/id/fd/，能看到这个fd，使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

`2 int epoll_ctl(int epfd, int op, int fd, struct epoll_event * event);`
epoll的事件注册函数，先注册想要监听的事件类型，第一个参数是epoll_create()的返回值，第二个参数表示动作，
用三个宏来表示：

`EPOLL_CTL_ADD`：注册新的fd到epfd中
`EPOLL_CTL_MOD`:修改已经注册的fd的监听事件；
`EPOLL_CTL_DEL`:从epfd中删除一个fd
第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：
``` cpp
typedef union epoll_data {
void *ptr;
int fd;
__uint32_t u32;
__uint64_t u64;
} epoll_data_t;

struct epoll_event {
__uint32_t events; /* Epoll events */
epoll_data_t data; /* User data variable */
};
```
 
events可以是以下几个宏的集合：
`EPOLLIN` ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
`EPOLLOUT`：表示对应的文件描述符可以写；
`EPOLLPRI`：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
`EPOLLERR`：表示对应的文件描述符发生错误；
`EPOLLHUP`：表示对应的文件描述符被挂断；
`EPOLLET`： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
`EPOLLONESHOT`：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

 

 

`3 int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int tiemout);`
等待事件产生，参数events用来从内核中得到事件的集合，maxevents告知内核这个events有多大，这个maxevents的值不能大于创建epoll_create()
的size，参数timeout是超时时间，毫秒，0表示立即返回，-1会永久阻塞。该函数返回需要处理的事件数目，如果返回0，表示已超时。

 

下面我查阅相关资料，仿照别人写了一个类似的epoll demo代码，这个是LT的，下一篇写ET的

``` cpp
#include <unistd.h>
#include <sys/types.h>
#include <fcntl.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <fcntl.h>
#include <sys/wait.h>
#include <sys/epoll.h>

#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

#include <vector>
#include <algorithm>
#include <iostream>


#define ERR_EXIT(m) \
do \
{ \
perror(m); \
exit(EXIT_FAILURE); \
} while(0)

//先定义一个类型，epoll_event连续的存储空间，可以是数组，也可以是vector
//当然也可以malloc一块连续内存 (struct epoll_event *)malloc(n * sizeof(struct epoll_event));
typedef std::vector<struct epoll_event> EventList;

int main()
{
　　//打开一个空的描述符
　　int idlefd = open("/dev/null",O_RDONLY|O_CLOEXEC);

　　//生成listen描述符
　　int listenfd = socket(PF_INET, SOCK_CLOEXEC | SOCK_STREAM | SOCK_NONBLOCK, IPPROTO_TCP);

　　if(listenfd < 0)
　　{
　　　　ERR_EXIT("socketfd");
　　}

　　//初始化地址信息
　　struct sockaddr_in servaddr;
　　memset(&servaddr,0 ,sizeof(struct sockaddr_in));
　　servaddr.sin_family = AF_INET;
　　servaddr.sin_port = htons(6666);
　　servaddr.sin_addr.s_addr = htonl(INADDR_ANY);

　　int on = 1;
　　if (setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0)
　　ERR_EXIT("setsockopt");

　　if(bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)))
　　ERR_EXIT("bindError");

　　if (listen(listenfd, SOMAXCONN) < 0)
　　ERR_EXIT("listen");

　　//记录客户端连接对应的socket
　　std::vector<int> clients;
　　//创建epollfd, 用于管理epoll事件表
　　int epollfd;
　　epollfd = epoll_create1(EPOLL_CLOEXEC);

　　struct epoll_event event;
　　event.data.fd = listenfd;
　　event.events = EPOLLIN;
　　//将listenfd加入到epollfd管理的表里
　　epoll_ctl(epollfd, EPOLL_CTL_ADD, listenfd, &event);

　　//创建长度为16的epoll_event队列
　　EventList events(16);
　　//用于接收新连接的客户端地址
　　struct sockaddr_in peeraddr;
　　socklen_t peerlen;
　　int connfd;

　　int nready;
　　while(1)
　　{
　　　　nready = epoll_wait(epollfd, &*events.begin(), static_cast<int>(events.size()), -1);
　　　　if (nready == -1)
　　　　{
　　　　　　if (errno == EINTR)
　　　　　　continue;

　　　　　　ERR_EXIT("epoll_wait");
　　　　}
　　　　if (nready == 0)	// nothing happended
　　　　　　continue;

　　　　//大小不够了就重新开辟

　　　　if ((size_t)nready == events.size())
　　　　events.resize(events.size()*2);

　　
　　　　for (int i = 0; i < nready; ++i)
　　　　{
　　　　　　//判断wait返回的events数组状态是否正常
　　　　　　if ((events[i].events & EPOLLERR) ||
　　　　　　　　(events[i].events & EPOLLHUP) ||
　　　　　　　　(!(events[i].events & EPOLLIN)))
　　　　　　　　{
　　　　　　　　　　fprintf (stderr, "epoll error\n");
　　　　　　　　　　close (events[i].data.fd);
　　　　　　　　　　continue;
　　　　　　　　}


　　　　　　　　if (events[i].data.fd == listenfd)
　　　　　　　　{
　　　　　　　　　　peerlen = sizeof(peeraddr);
　　　　　　　　　　//LT模式accept不用放在while循环里
　　　　　　　　　　connfd = ::accept4(listenfd, (struct sockaddr*)&peeraddr,
　　　　　　　　　　　　&peerlen, SOCK_NONBLOCK | SOCK_CLOEXEC);

　　　　　　　　　　//accept失败，判断是否是文件描述符达到上限

　　　　　　　　　　if (connfd == -1)
　　　　　　　　　　{
　　　　　　　　　　　　if (errno == EMFILE)
　　　　　　　　　　　　{
　　　　　　　　　　　　　　//关闭之前打开的描述符，重新accept，然后关闭这个accept得到的描述符，

　　　　　　　　　　　　　　//因为LT模式，如果socket读缓冲区有数据一直读取失败会造成busyloop　　　　　　　　　

　　　　　　　　　　　　　　close(idlefd);
　　　　　　　　　　　　　　idlefd = accept(listenfd, NULL, NULL);
　　　　　　　　　　　　　　close(idlefd);
　　　　　　　　　　　　　　idlefd = open("/dev/null", O_RDONLY | O_CLOEXEC);
　　　　　　　　　　　　　　continue;
　　　　　　　　　　　　}
　　　　　　　　　　　　else
　　　　　　　　　　　　　　ERR_EXIT("accept4");
　　　　　　　　　}


　　　　　　　　　　　　std::cout<<"ip="<<inet_ntoa(peeraddr.sin_addr)<<
　　　　　　　　　　　　　　" port="<<ntohs(peeraddr.sin_port)<<std::endl;

　　　　　　　　　　　　　　　　clients.push_back(connfd);


　　　　　　　　　　　　//将connd加入epoll表里，关注读事件
　　　　　　　　　　　　event.data.fd = connfd;
　　　　　　　　　　　　event.events = EPOLLIN ;
　　　　　　　　　　　　epoll_ctl(epollfd, EPOLL_CTL_ADD, connfd, &event);
　　　　　　　　}
　　　　　　　　else if (events[i].events & EPOLLIN)
　　　　　　　　{
　　　　　　　　　　connfd = events[i].data.fd;
　　　　　　　　　　if (connfd < 0)
　　　　　　　　　　continue;

　　　　　　　　　　char buf[1024] = {0};
　　　　　　　　　　int ret = read(connfd, buf, 1024);
　　　　　　　　　　if (ret == -1)
　　　　　　　　　　//ERR_EXIT("read");
　　　　　　　　　　{
　　　　　　　　　　　　if((errno == EAGAIN) ||
　　　　　　　　　　　　　　(errno == EWOULDBLOCK))
　　　　　　　　　　　　{
　　　　　　　　　　　　　　//由于内核缓冲区空了，下次再读，
　　　　　　　　　　　　　　//这个是LT模式不需要重新加入EPOLLIN事件,下次还会通知
　　　　　　　　　　　　　　continue;
　　　　　　　　　　　　}

　　　　　　　　　　　　ERR_EXIT("read"); 
　　　　　　　　　　}
　　　　　　　　　　if (ret == 0)
　　　　　　　　　　{
　　　　　　　　　　　　std::cout<<"client close"<<std::endl;
　　　　　　　　　　　　close(connfd);
　　　　　　　　　　　　event = events[i];
　　　　　　　　　　　　epoll_ctl(epollfd, EPOLL_CTL_DEL, connfd, &event);
　　　　　　　　　　　　clients.erase(std::remove(clients.begin(), clients.end(), connfd), clients.end());
　　　　　　　　　　　　continue;
　　　　　　　　　　}
　　　　　　　　　　//
　　　　　　　　　　std::cout<<buf;
　　　　　　　　　　write(connfd, buf, strlen(buf));
　　　　　　　　}
　　　　　　　　else //写事件
　　　　　　　　　　if(events[i].events & EPOLLOUT)
　　　　　　　　　　{
　　　　　　　　　　　　connfd = events[i].data.fd;
　　　　　　　　　　　　char buf[512];
　　　　　　　　　　　　int count = write(events[i].data.fd, buf, strlen(buf));
　　　　　　　　　　　　if(count == -1)
　　　　　　　　　　　　{
　　　　　　　　　　　　　　if((errno == EAGAIN) ||
　　　　　　　　　　　　　　　　(errno == EWOULDBLOCK))
　　　　　　　　　　　　　　{
　　　　　　　　　　　　　　　　//由于内核缓冲区满了，下次再写，
　　　　　　　　　　　　　　　　//这个是LT模式不需要重新加入EPOLLOUT事件
　　　　　　　　　　　　　　　　　continue;
　　　　　　　　　　　　　　}

　　　　　　　　　　　　　　ERR_EXIT("write"); 

　　　　　　　　　　　　}

　　　　　　　　　　　　　　//写完要记得从epoll内核中删除，因为LT模式写缓冲区不满就会触发EPOLLOUT事件，防止busyloop
　　　　　　　　　　　　　　event.data.fd = connfd;
　　　　　　　　　　　　　　event.events = EPOLLOUT;
　　　　　　　　　　　　　　epoll_ctl (epollfd, EPOLL_CTL_DEL, events[i].data.fd, &event);
　　　　　　　　　　　　　　//close 操作会将events[i].data.fd从epoll表里删除，所以上面的操作可以不写
　　　　　　　　　　　　　　//close(events[i].data.fd); 此处是关闭了和客户端的连接，不关闭也可以，只和策划需求有关
　　　　　　　　　　}
　　　　　　}
　　　　}

return 0;	
}
```

 好了，这就是epoll 的LT模式，demo源代码下载地址 [http://download.csdn.net/detail/secondtonone1/9484752](http://download.csdn.net/detail/secondtonone1/9484752)。
 谢谢关注我的公众号
 ![1.jpg](1.jpg)