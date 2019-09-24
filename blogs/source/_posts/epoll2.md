---
title: IO多路复用之epoll（二）
date: 2017-08-07 17:18:23
categories: [网络编程]
tags: [网络编程]
---
前一篇介绍了`epoll的LT模式`，`LT模式注意epollout事件在数据全部写成功后需要取消关注`，或者更改为EPOLLIN。

而这次`epoll的ET模式`，要注意的是`在读和写的过程中要在循环中写完或者读完所有数据`，`确保不要丢掉一些数据`。

因为`epoll ET模式只在两种边缘更改的时候触发`，对于读事件只在内核缓冲区由空变为

非空通知一次用户，对于写事件，内核缓冲区只在由满变为非满的情况通知用户一次。
<!--more-->
下面是代码
``` cpp
int main()
{

	int eventsize = 20;
	struct epoll_event * epoll_eventsList = (struct epoll_event *)malloc(sizeof(struct epoll_event)

	*eventsize);

	//打开一个空的描述符
	int idlefd = open("/dev/null",O_RDONLY|O_CLOEXEC);	
	cout << "idlefd" <<idlefd <<endl;	
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
	servaddr.sin_port = htons(6667);
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
	event.events = EPOLLIN|EPOLLET;
	//将listenfd加入到epollfd管理的表里
	epoll_ctl(epollfd, EPOLL_CTL_ADD, listenfd, &event);

	//用于接收新连接的客户端地址
	struct sockaddr_in peeraddr;
	socklen_t peerlen;
	int connfd;
	//为了简单起见写了个很大的数组，根据文件描述符存储内容
	//其实很多项目代码采epoll.data.ptr回调具体的读写

	vector<string> recievebuf;

	for(int i = 0 ; i < 22222; i++)
	{
		recievebuf.push_back("");
	}

	while(1)
	{
		int nready = epoll_wait(epollfd, epoll_eventsList, eventsize, -1);
		if (nready == -1)
		{
			if (errno == EINTR)
			continue;

			ERR_EXIT("epoll_wait");
		}
		if (nready == 0)
		continue;

		//大小不够重新开辟

		if ((size_t)nready == eventsize)
		{
			if(eventsize * 2 >= 22222)
			{
				ERR_EXIT("too many fds");
			}

			struct epoll_event * epoll_eventsList2 = (struct epoll_event *)malloc(sizeof(struct epoll_event) * 
			eventsize *2);
			if(epoll_eventsList2)
			{
				memcpy(epoll_eventsList2,epoll_eventsList,sizeof(struct epoll_event) * eventsize);
				eventsize = eventsize * 2;
				free(epoll_eventsList);
				epoll_eventsList = epoll_eventsList2;

			}


		}


		for (int i = 0; i < nready; ++i)
		{
			//判断wait返回的events数组状态是否正常
			if ((epoll_eventsList[i].events & EPOLLERR) ||
			(epoll_eventsList[i].events & EPOLLHUP))
			{
				fprintf (stderr, "epoll error\n");
				close (epoll_eventsList[i].data.fd);
				continue;

			}

			if (epoll_eventsList[i].data.fd == listenfd)
			{
				peerlen = sizeof(peeraddr);
				//ET模式accept放在while循环里

				do
				{
					connfd = ::accept4(listenfd, (struct sockaddr*)&peeraddr,
					&peerlen, SOCK_NONBLOCK | SOCK_CLOEXEC);

					if(connfd <= 0)
					break;

					std::cout<<"ip="<<inet_ntoa(peeraddr.sin_addr)<<
					" port="<<ntohs(peeraddr.sin_port)<<std::endl;
					clients.push_back(connfd);

					//将connd加入epoll表里，关注读事件

					event.data.fd = connfd;
					event.events = EPOLLIN |EPOLLET;
					epoll_ctl(epollfd, EPOLL_CTL_ADD, connfd, &event);
					cout << "loop" <<endl;	
					cout << "loop" << connfd << endl;
				}while(1);
				//accept失败，判断是否接收全所有的fd
				cout << connfd << endl;
				if (connfd == -1){
					if (errno != EAGAIN && errno != ECONNABORTED
						&& errno != EPROTO && errno != EINTR)
						{
							cout << "error" <<endl;
							ERR_EXIT("accept");
						}	

					}

					//所有请求都处理完成
					cout << "continue"<<endl;
					continue;

				}//endif
				else if(epoll_eventsList[i].events & EPOLLIN)
				{
					connfd = epoll_eventsList[i].data.fd;	
					if(connfd > 22222)
					{
						close(connfd);
						event = epoll_eventsList[i];
						epoll_ctl(epollfd, EPOLL_CTL_DEL, connfd, &event);
						clients.erase(std::remove(clients.begin(), clients.end(), connfd), clients.end());
						continue;

					}

					char buf[1024] = {0};
					if(connfd < 0)
					continue;
					int ret = 0;
					int total = 0;
					std::string strtemp;
					while(1)
					{
						cout << "begin read" <<endl;
						ret = read(connfd, buf, 1024);
						if(ret <= 0)
						{
							break;
						}

						strtemp += string(buf);
						total += ret;
						memset(buf, 0, 1024);

						if(ret < 1024)
						{
							break;
						}

					}//endwhile(1)

					cout << "end read" <<endl;
					recievebuf[connfd] = strtemp.c_str();
					cout << "buff data :" << recievebuf[connfd]<<endl; 
					if(ret == -1)
					{
						if((errno == EAGAIN) ||
						(errno == EWOULDBLOCK))
						{
							//由于内核缓冲区空了，下次有数据到来是会触发epollin
							continue;
						}


						ERR_EXIT("read");

					}//endif ret == -1

					//连接断开
					if(ret == 0)
					{
						std::cout<<"client close"<<std::endl;
						close(connfd);
						event = epoll_eventsList[i];
						epoll_ctl(epollfd, EPOLL_CTL_DEL, connfd, &event);
						clients.erase(std::remove(clients.begin(), clients.end(), connfd), clients.end());
						continue;

					}

					cout << "turn to write" << endl;
					//更改为写模式

					event.data.fd = connfd;
					event.events = EPOLLOUT | EPOLLET;

					epoll_ctl(epollfd, EPOLL_CTL_MOD, connfd, &event);
					cout << "epoll mod change success" << endl;

				}//end elif

				else //写事件
				{

					if(epoll_eventsList[i].events & EPOLLOUT)
					{
						cout << "begin write" <<endl;
						connfd = epoll_eventsList[i].data.fd;
						int count = 0;
						int totalsend = 0;
						char buf[1024];
						strcpy(buf, recievebuf[connfd].c_str());

						cout << "write buff" <<buf<<endl;
						while(1)
						{
							int totalcount = strlen(buf);
							int pos = 0;
							count = write(epoll_eventsList[i].data.fd, buf + pos, totalcount);
							cout << "write count:" << count;
							if(count < 0)
							{
								break;
							}

							if(count < totalcount)
							{
								totalcount = totalcount - count;
								pos += count;

							}
							else
							{
								break;

							}

						}//end while


						if(count == -1)
						{
							if((errno == EAGAIN) ||
							(errno == EWOULDBLOCK))
							{
								//由于内核缓冲区满了
								//于内核缓冲区满了
								continue;
							}

							ERR_EXIT("write");
						}

						if(count == 0)
						{
							std::cout<<"client close"<<std::endl;
							close(connfd);
							event = epoll_eventsList[i];
							epoll_ctl(epollfd, EPOLL_CTL_DEL, connfd, &event);
							clients.erase(std::remove(clients.begin(), clients.end(), connfd), 
							clients.end());
							continue;

						}

						event.data.fd = connfd;
						event.events = EPOLLIN|EPOLLET;
						epoll_ctl(epollfd, EPOLL_CTL_MOD, connfd, &event);

					}

				}//end eles 写事件


			}

		}
	}
	```


源代码下载地址：[http://download.csdn.net/detail/secondtonone1/9486222](http://download.csdn.net/detail/secondtonone1/9486222)
我的微信公众号：
![1.jpg](1.jpg)