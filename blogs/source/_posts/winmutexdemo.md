---
title: Windows互斥锁demo和分析
date: 2017-08-03 20:36:08
categories: 技术开发
tags: [C++,Windows环境编程]
---
## 一：windows创建锁接口

创建互斥锁的方法是调用函数`CreateMutex`
``` cpp
HANDLE CreateMutex(
LPSECURITY_ATTRIBUTESlpMutexAttributes, // 指向安全属性的指针
BOOLbInitialOwner, // 初始化互斥对象的所有者
LPCTSTRlpName // 指向互斥对象名的指针
);
```
第一个参数是一个指向`SECURITY_ATTRIBUTES`结构体的指针，一般的情况下，可以是nullptr。

第二个参数类型为BOOL，表示`互斥锁创建出来后是否被当前线程持有`。

第三个参数类型为字符串（const TCHAR*），是这个`互斥锁的名字`，如果是nullptr，则互斥锁是匿名的。

例子：

`HANDLE hMutex = CreateMutex(nullptr, FALSE, nullptr);`
<!-- more -->  
## 二：windows持有锁接口：

DWORD `WaitForSingleObject`(

HANDLE hHandle,
DWORD dwMilliseconds
);
 

这个函数的作用比较多。这里只介绍第一个参数为互斥锁句柄时的作用。

它的作用是等待，直到一定时间之后，或者，其他线程均不持有hMutex。第二个参数是等待的时间（单位：毫秒），如果该参数为INFINITE，则该函数会一直等待下去。

## 三：释放锁

BOOL WINAPI `ReleaseMutex`(
HANDLE hMutex
);
## 四：销毁

BOOL `CloseHandle`(
HANDLE hObject
);

下面是网上的一个案例，根据我自己做服务器的需求，模仿者写了一个：

 
''' cpp
//各种类型的锁的基类
class BaseLock
{
public:
    BaseLock(){}
    virtual ~BaseLock(){}
    virtual void lock() const = 0 ;
    virtual void unlock() const = 0 ;
};

 


//互斥锁继承基类
class Mutex :public BaseLock
{
public:
    Mutex();
    ~Mutex();
    virtual void lock() const;
    virtual void unlock() const;
private:
#if defined _WIN32
    HANDLE m_hMutex;
#endif
};

 

//互斥锁实现文件：


//在构造函数里创建锁
Mutex::Mutex()
{
    #if defined _WIN32
        m_hMutex = ::CreateMutex(NULL, FALSE, NULL);
    #endif
}

//析构函数里销毁锁
Mutex::~ Mutex()
{
    #if defined _WIN32
        ::CloseHandle(m_hMutex);
    #endif
}

//互斥锁上锁
void Mutex::lock() const
{
    #if defined _WIN32
      DWORD d = WaitForSingleObject(m_hMutex, INFINITE);
    #endif
}

//互斥锁解锁
void Mutex::unlock() const
{
    #if defined _WIN32
      ::ReleaseMutex(m_hMutex);
    #endif
}

class CLock
{
public:
    CLock(const BaseLock & baseLock):m_cBaseLock(baseLock){
        //构造函数里通过基类锁调用加锁函数(多态)
        m_cBaseLock.lock();
    }
    ~CLock(){
        //析构函数先解锁
        m_cBaseLock.unlock();
    }
private:
    //常引用变量，需要在初始化列表初始
    //多态机制
    const BaseLock& m_cBaseLock;
};
'''
 

CLock是留给外界使用的接口类，可以实现自动加锁和解锁。构造函数传入不同类型的锁，目前只实现了互斥锁，通过基类类型的引用成员可以实现多态调用不同的lock和unlock，而CLock析构函数因为会调用基类的unlock，从而实现不同类型的解锁。那么读者可能会有疑问互斥锁什么时候会销毁？互斥锁的销毁写在互斥锁类的析构函数里，当调用互斥锁的析构函数就会自动销毁这把锁了。什么时候调用互斥锁的析构函数呢？之前有介绍过，析构函数的调用顺序，先析构子类对象，然后析构子类对象中包含的其他类型的对象，最后析构基类对象，所以整个流程是先调用Mutex的构造函数，将Mutex构造的对象传入CLock的构造函数，这样实现自动加锁，当CLock析构的时候先析构CLock对象，之后析构CLock类里的BaseLock对象，因为是多态，会自动根据虚析构函数调用子类也就是MutexLock的析构函数，完成销毁锁的操作。

下面是我服务器中的一段代码截取，算是这个锁的示例

``` cpp
void NetWorker::pushNodeInStream(TcpHandler * tcpHandler)
{
        //加锁处理消息加入到instream里
        Mutex mutexlock;
        CLock mylock(mutexlock);    
        list<MsgNode *> * msgList = tcpHandler->getListMsgs();


}
```
因为函数`}`会释放局部变量，那么就会调用CLock析构函数，接着调用Mutex析构函数。依次完成解锁和销毁锁的操作。我的服务器还在制作当中，基本框架制作完毕会做一些服务器设计的研究。

 