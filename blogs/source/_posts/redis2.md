---
title: 对于redis框架的理解（二）
date: 2017-08-07 16:22:07
categories: 技术开发
tags: [网络编程]
---
之前梳理过`redis` main函数主体流程

`大体是 initServerConfig() -> loadServerConfig()

`-> daemonize() -> initServer() -> aeSetBeforeSleepProc()`

`->aeMain() -> aeDeleteEventLoop();`

`initServerConfig()` 初始化server的配置

`loadServerConfig()`会从配置文件里加载对应的配置

`daemonize()`创建守护进程

看一下daemonize的函数组成
``` cpp
void daemonize(void) {
　　int fd;

　　if (fork() != 0) exit(0); /* parent exits */
　　//设置为首进程
　　//之前parent和child运行在同一个session里,

　　//parent是会话（session）的领头进程,
　　//parent进程作为会话的领头进程，如果exit结束执行的话，

　　//那么子进程会成为孤儿进程，并被init收养。
　　//执行setsid()之后,child将重新获得一个新的会话(session)id。
　　setsid(); /* create a new session */

　　/* Every output goes to /dev/null. If Redis is daemonized but
　　* the 'logfile' is set to 'stdout' in the configuration file
　　* it will not log at all. */

　　//  /dev/null相当于黑洞文件，所有写入他的数据都会消失，

　　//从他里面读不出数据
　　if ((fd = open("/dev/null", O_RDWR, 0)) != -1) {
　　　　dup2(fd, STDIN_FILENO);    //将标准输入重定向到fd
　　　　dup2(fd, STDOUT_FILENO); //将标准输出重定向到fd
　　　　dup2(fd, STDERR_FILENO); //将错误输出重定向到fd
　　　　if (fd > STDERR_FILENO)

　　　　　　close(fd);

　　　}
}
```
<!--more--> 

着重解释下`dup2`这个函数

`int dup(int oldfd);`

复制oldfd所指向的文件描述符，返回系统目前未使用的最小的文件描述符。

新的文件描述符和oldfd指向一个文件，

他们`共享读写，枷锁等权限`，`当一个文件描述符操作lseek，另一个也会随着偏移`。

但是他们`不共享close-on-exec`。

`int dup2(int oldfd, int newfd);`

`复制oldfd到newfd，如果newfd指向的文件打开，那么会关闭该文件。`

`dup2失败后返回-1，成功则共享文件状态。`

 

接下来看看initServer函数做了些什么 下面是initServer内部的几个步骤

//由于initserver是守护进程，忽略sighup
//sighub在控制终端关闭的时候会发给session首进程
//session首进程退出时，该信号被发送到该session中的前台进程组中的每一个进程
`signal(SIGHUP, SIG_IGN);`

//写管道发现读进程终止时产生sigpipe信号，

//写已终止的SOCK_STREAM套接字同样会产生此信号
`signal(SIGPIPE, SIG_IGN);`
``` cpp
server.pid = getpid();
server.current_client = NULL;
server.clients = listCreate(); //创建客户队列
server.clients_to_close = listCreate(); //创建关闭队列
server.slaves = listCreate(); //创建从机队列
server.monitors = listCreate(); //创建监控队列
server.slaveseldb = -1; /* Force to emit the first SELECT command. */
server.unblocked_clients = listCreate(); //创建非堵塞客户队列
server.ready_keys = listCreate(); //创建可读键队列
server.clients_waiting_acks = listCreate(); //客户等待回包队列
server.get_ack_from_slaves = 0;
server.clients_paused = 0;
```

//创建别的模块公用的对象
`createSharedObjects(); //创建共享对象`
`adjustOpenFilesLimit(); //改变可打开文件的最大数量`

可以看看`adjustOpenFilesLimit() `里边做了什么
``` cpp
void adjustOpenFilesLimit(void) {
　　　　rlim_t maxfiles = server.maxclients+REDIS_MIN_RESERVED_FDS;
　　　　struct rlimit limit;

　　　　//获取每个进程能创建的各种系统资源的最大数量
　　　　if (getrlimit(RLIMIT_NOFILE,&limit) == -1) {
　　　　redisLog(REDIS_WARNING,"Unable to obtain the current NOFILE limit (%s),

　　　　assuming 1024 and setting the max clients configuration accordingly.",
　　　　strerror(errno));
　　　　//1024 - 目前redis最小保留的用于其他功能的描述符
　　　　server.maxclients = 1024-REDIS_MIN_RESERVED_FDS;
　　　　} else {
　　　　　　　　//系统当前（软）限制
　　　　　　　　rlim_t oldlimit = limit.rlim_cur;

　　　　　　　　/* Set the max number of files if the current limit is not enough
　　　　　　　　* for our needs. */
　　　　　　　　if (oldlimit < maxfiles) {
　　　　　　　　//最大软件限制
　　　　　　　　rlim_t bestlimit;
　　　　　　　　int setrlimit_error = 0;

　　　　　　　　/* Try to set the file limit to match 'maxfiles' or at least
　　　　　　　　* to the higher value supported less than maxfiles. */
　　　　　　　　//尝试设置限制数符合maxfiles或者最接近
　　　　　　　　bestlimit = maxfiles;
　　　　　　　　//循环判断，如果最大的连接数大于当前系统能使用的最大软件限制
　　　　　　　　while(bestlimit > oldlimit) {
　　　　　　　　//设置每次递减的数量
　　　　　　　　rlim_t decr_step = 16;
　　　　　　　　//设置当前软件限制
　　　　　　　　limit.rlim_cur = bestlimit;
　　　　　　　　limit.rlim_max = bestlimit;
　　　　　　　　//设置成功则断开，不成功继续循环，找到最接近的最大限制
　　　　　　　　if (setrlimit(RLIMIT_NOFILE,&limit) != -1) break;
　　　　　　　　//设置限制错误码
　　　　　　　　setrlimit_error = errno;

　　　　　　　　/* We failed to set file limit to 'bestlimit'. Try with a
　　　　　　　　* smaller limit decrementing by a few FDs per iteration. */

　　　　　　　　//最大限制递减，如果小于规定值那么退出循环
　　　　　　　　if (bestlimit < decr_step) break;
　　　　　　　　bestlimit -= decr_step;
　　　　　　　　　　　　/* Assume that the limit we get initially is still valid if

　　　　　　* our last try was even lower. */
　　　　　　//最大连接数小于当前系统允许的资源数量，那么扩充为系统允许的数量
　　　　　　if (bestlimit < oldlimit) bestlimit = oldlimit;

　　　　　　//这个条件用于处理最大连接数
　　　　　　if (bestlimit < maxfiles) {
　　　　　　//保存当前服务器客户队列最大数量
　　　　　　　　int old_maxclients = server.maxclients;
　　　　　　　　//服务器客户队列最大数量
　　　　　　　　server.maxclients = bestlimit-REDIS_MIN_RESERVED_FDS;
　　　　　　　　if (server.maxclients < 1) {
　　　　　　　　　　redisLog(REDIS_WARNING,"Your current 'ulimit -n' "
　　　　　　　　　　"of %llu is not enough for Redis to start. "
　　　　　　　　　　"Please increase your open file limit to at least "
　　　　　　　　　　"%llu. Exiting.",
　　　　　　　　　　(unsigned long long) oldlimit,
　　　　　　　　　　(unsigned long long) maxfiles);
　　　　　　　　　　exit(1);
　　　　　　　　}

　　　}

```
回到initServer函数内部

adjustOpenFilesLimit()函数过后是调用

`server.el = aeCreateEventLoop(server.maxclients+REDIS_EVENTLOOP_FDSET_INCR);`

这个函数是创建基本的时间循环结构。这个api写在Ae.c中，这个文件主要负责创建事件轮询的结构，创建文件读写事件，删除文件读写事件，删除事件轮询结构，派发文件读写功能等，下一节再讲。

然后是创建了一个定时器回调函数

`if(aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
redisPanic("Can't create the serverCron time event.");
exit(1);
}`

接着创建了 TCP的回调函数，主要用于有新的连接到来触发acceptTcpHandler

`for (j = 0; j < server.ipfd_count; j++) {
if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
	acceptTcpHandler,NULL) == AE_ERR)
	{
	redisPanic(
	"Unrecoverable error creating server.ipfd file event.");
	}
}`

 

//然后添加了udp的回调
`if (server.sofd > 0 && aeCreateFileEvent(server.el,server.sofd,AE_READABLE,
acceptUnixHandler,NULL) == AE_ERR) redisPanic("Unrecoverable error creating server.sofd file event.");`

 

这就是initServer大体的流程