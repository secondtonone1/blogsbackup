---
title: C++单例模式设计和实现
date: 2017-08-04 11:44:16
categories: 技术开发
tags: C++
---
C++单例模式主要用途就是整个程序中只实例化一个对象，之后获取到的都是该对象本身进行处理问题。

`单例模式`一般都是在函数中采用局部静态变量完成的，因为`局部的静态变量`生命周期是随着程序的生命周期

一起结束，所以不用担心会失效。另外局部的静态变量`作用域`仅限于该函数内部，别的函数不会直接使用。

第三点就是局部的静态变量跟所有的静态变量一样，放在`全局区(静态区)`，只被`初始化一次`。
<!-- more -->
下面是我结合模板设计的单例类

``` cpp
#ifndef _SINGLETON_CLASS_H_
#define _SINGLETON_CLASS_H_

template <class Type>
class Singleton
{
protected :
    Singleton(){}

public:
    static Type & getSingleton()
    {        
        return singleton;
    }

private:
        
    Singleton(const Singleton & temp){
        singleton = temp.singleton;
    }

private:
    static Type singleton;
};

template <class Type>
Type Singleton<Type>::singleton;

#endif
```
其余的类继承就可以了。

需要注意类的静态成员变量，如果不是integer type，需要在类外完成初始化。

int属于integer type，在类内可以完成初始化。

其余的类继承该类：

``` cpp
class NetWorkSystem : public Singleton<NetWorkSystem>
{
public:
    NetWorkSystem():m_nListenfd(0),m_pEvent_base(NULL),m_nConnId(0){}
    bool initial();
    static  void tcpread_cb(struct bufferevent *bev, void *ctx);
    static  void tcpwrite_cb(struct bufferevent *bev, void *ctx);
    static  void tcperror_cb(struct bufferevent *bev, short what, void *ctx);
    static  void listener_read_cb(evutil_socket_t fd, short what, void *p);
    void run();
    void release();
    //...   
 
};
```
使用时使用getsinggleton这个函数即可。

这是我服务器中截取的代码，可以从github中下载该服务器源码。

下载地址：[https://github.com/secondtonone1/smartserver](https://github.com/secondtonone1/smartserver)

服务器自己做的，还在不断地完善之中。