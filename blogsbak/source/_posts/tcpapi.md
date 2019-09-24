---
title: TCP/IP地址格式转换API
date: 2017-08-04 15:30:02
categories: 技术开发
tags: [网络编程]
---
1、htonl ()和ntohl( )

`ntohl( )`-----`网络顺序转换成主机顺序(长整型)`

u_long PASCAL FAR ntohl (u_long netlong); 

`htonl ()`-----`主机顺序转换成网络顺序 (长整型)`

u_long PASCAL FAR htonl (u_long hostlong);
<!--more-->

2、htons ()和ntohs( )  

`htons()`------`主机顺序转换成网络顺序(短整型)`

u_short PASCAL FAR htons (u_short hostshort);

`ntohs()`------`网络顺序转换成主机顺序(短整型)`

u_short PASCAL FAR ntohs (u_short netshort);

3、inet_addr( )和inet_ntoa ( )

unsigned long PASCAL FAR inet_addr (const char FAR * cp);

char FAR * PASCAL FAR inet_ntoa (struct in_addr in);

`inet_addr`函数需要一个字符串作为其参数，该字符串指定了以点分十进制格式表示的IP地址（例如：192.168.1.161）。而且inet_addr函数会返回一个适合分配给S_addr的u_long类型的数值。

``` cpp
int sockfd; 
struct sockaddr_in my_addr; 
sockfd = socket(AF_INET, SOCK_STREAM, 0); 

my_addr.sin_family = AF_INET; /* 主机字节序 */ 
my_addr.sin_port = htons(9925); /* short, 网络字节序 */ 
my_addr.sin_addr.s_addr = inet_addr("192.168.0.1"); 

bzero(&(my_addr.sin_zero), 8); 

bind(sockfd, (struct sockaddr *)&my_addr, sizeof(struct sockaddr));
```

inet_ntoa函数会完成相反的转换，它接受一个in_addr结构体类型的参数并返回一个以点分十进制格式表示的IP地址字符串。

服务器accept收到一个连接后，可以通过inet_ntoa找到对方ip，输出为字符串格式的点分十进制

``` 1c
        client = accept(serverSocket, (struct sockaddr *)&clientAddr, (socklen_t *)&addr_len);  
        if (client < 0)  
        {  
            perror("accept");  
          
        }  
      
        printf("\nRecv client data...\n");  
        printf("IP is %s\n", inet_ntoa(clientAddr.sin_addr));  
```
 
sockaddr_in , sockaddr , in_addr区别
``` 1c
struct   sockaddr   {  
                unsigned   short   sa_family;     
                char   sa_data[14];     
        };  
```
  上面是通用的socket地址，具体到Internet   socket，用下面的结构，二者可以进行类型转换  
``` 1c        
  struct   sockaddr_in   {  
                short   int   sin_family;     
                unsigned   short   int   sin_port;     
                struct   in_addr   sin_addr;     
                unsigned   char   sin_zero[8];     
        };  
   
```
struct   in_addr就是32位IP地址。  
``` 1c       
struct   in_addr   {  
                union {
                        struct { u_char s_b1,s_b2,s_b3,s_b4; } S_un_b;
                        struct { u_short s_w1,s_w2; } S_un_w;
                        u_long S_addr; 
                } S_un;

                #define s_addr  S_un.S_addr
        };  
```

inet_addr()是将一个点分制的IP地址(如192.168.0.1)转换为上述结构中需要的32位IP地址(0xC0A80001)。

填值的时候使用sockaddr_in结构，而作为函数（如socket, listen, bind等）的参数传入的时候转换成sockaddr结构就行了，毕竟都是16个字符长。

名词解析：

主机字节序：

不同的CPU有不同的字节序类型，这些字节序是指整数在内存中保存的顺序，这个叫做主机序。最常见的有两种

1．Little endian：低字节存高地址，高字节存低地址

2．Big endian：低字节存低地址，高字节存高地址

网络字节序：

网络字节顺序是TCP/IP中规定好的一种数据表示格式，它与具体的CPU类型、操作系统等无关，从而可以保证数据在不同主机之间传输时能够被正确解释。网络字节顺序采用big endian排序方式。

为了进行转换bsd socket提供了转换的函数，有下面四个网络与主机字节转换函数:htons ntohs htonl ntohl (s 就是short l是long h是host n是network)

htons 把unsigned short类型从主机序转换到网络序，htonl 把unsigned long类型从主机序转换到网络序，ntohs 把unsigned short类型从网络序转换到主机序，ntohl 把unsigned long类型从网络序转换到主机序。

在使用little endian的系统中 这些函数会把字节序进行转换 在使用big endian类型的系统中这些函数会定义成空宏