---
title: python学习(20) 网络编程
date: 2018-01-02 16:57:42
categories: 技术开发
tags: [python]
---
python 网络编程和基本的C语言编程一样，效率不是很高，如果为了封装通信库
建议采用C/C++做底层封装，采用epoll、poll、iocp等网络模型封装，编译成网络库
供其他模块使用。这里在python学习过程中介绍一下
## TCP 编程 服务器端
1 `创建套接字`
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
AF_INET表示网络通信，SOCK_STRAM表示面向字节流TCP方式通信。
2 `绑定端口和地址`
s.bind(('127.0.0.1',9999))
3 `监听套接字`
s.listen(5)
5,表示监听队列最大多长。
4 `接收客户端连接`
sock, addr = s.accept()
s为生成的socket，accept接收客户端连接，返回对端描述符和地址
5 `接收数据`
data = sock.recv(1024)
sock为对端socket，recv接收数据，最多接收1024
6 `发送数据`
sock.send(('Hello, %s' %data.decode('utf-8')).encode('utf-8'))
7 `关闭描述符`
sock.close()
<!--more-->
## TCP客户端编程
1 `创建一个socket`
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
2 `建立连接`、
s.connect(('127.0.0.1', 9999))
3 `接收消息和发送消息`
s.send(data)
s.recv(1024).decode('utf-8')
4 `关闭描述符`
s.close()

服务端案例：
``` python
import socket
import threading
import time

#线程处理函数
def tcplink(sock, addr):
	print('Accept new connection from %s:%s...' % addr)
	sock.send(b'Welcome!')
	while True:
		data = sock.recv(1024)
		time.sleep(1)
		if not data or data.decode('utf-8')=='exit':
			break
		sock.send(('Hello, %s' %data.decode('utf-8')).encode('utf-8'))
	sock.close()
	print('Connection from %s:%s closed.'%addr)



#服务器tcp编程流程
#创建套接字
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
#绑定套接字
s.bind(('127.0.0.1',9999))
#监听套接字
s.listen(5)



print('Waiting for connection...')
# 调用accept接受连接
while True:
	# 接收新的连接
	sock, addr = s.accept()
	# 创建新的线程处理TCP
	t = threading.Thread(target=tcplink,args=(sock,addr))
	t.start()
```
客户端：
``` python
import socket

#创建一个socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
# 建立连接:
s.connect(('127.0.0.1', 9999))
# 接收欢迎消息:
print(s.recv(1024).decode('utf-8'))
for data in [b'Michael', b'Tracy', b'Sarah']:
    # 发送数据:
    s.send(data)
    print(s.recv(1024).decode('utf-8'))
s.send(b'exit')
s.close()
```
## UDP通信
udp通信和tcp通信不一样，很简单。服务器端不需要监听和accept，
客户端也不需要connect。
服务端 示例：

``` python
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
# 绑定端口:
s.bind(('127.0.0.1', 9999))


print('Bind UDP on 9999...')
while True:
    # 接收数据:
    data, addr = s.recvfrom(1024)
    print('Received from %s:%s.' % addr)
    s.sendto(b'Hello, %s!' % data, addr)
```
服务器端绑定好端口和地址后，调用recvfrom返回数据和对端地址
客户端 示例：
``` python
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
for data in [b'Michael', b'Tracy', b'Sarah']:
    # 发送数据:
    s.sendto(data, ('127.0.0.1', 9999))
    # 接收数据:
    print(s.recv(1024).decode('utf-8'))
s.close()
```
客户端发送数据和接收数据和之前一样，仅仅是不需要调用connect连接服务器了。

谢谢关注我的公众号：
![wxgzh.jpg](wxgzh.jpg)