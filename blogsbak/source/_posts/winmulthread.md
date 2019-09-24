---
title: windows多线程接口介绍和使用
date: 2017-08-03 20:47:18
categories: 技术开发
tags: [C++,Windows环境编程]
---
## 一`windows多线程`接口：

### 1 创建线程

`CreateThread` 与 `_beginthreadex`都可以实现创建线程，两个函数的参数相同，

 ``` cpp
 HANDLEWINAPICreateThread(
 LPSECURITY_ATTRIBUTESlpThreadAttributes,
 SIZE_TdwStackSize,
  LPTHREAD_START_ROUTINElpStartAddress,
 LPVOIDlpParameter,
 DWORDdwCreationFlags,
 LPDWORDlpThreadId
);
```
<!-- more -->  
函数说明：  

第一个参数表示线程内核对象的`安全属性`，一般传入NULL表示使用默认设置。  

第二个参数表示线程栈空间大小。传入0表示使用默认大小（1MB）。  

第三个参数表示新线程所执行的线程函数地址，多个线程可以使用同一个函数地址。  

第四个参数是传给线程函数的参数。  

第五个参数指定额外的标志来控制线程的创建，为0表示线程创建之后立即就可以进行调度，如果为CREATE_SUSPENDED则表示线程创建后暂停运行，这样它就无法调度，直到调用ResumeThread()。  

第六个参数将返回线程的ID号，传入NULL表示不需要返回该线程ID号。  

函数返回值：

成功返回新线程的句柄，失败返回NULL。   

CreateThread 与 _beginthreadex的区别是_beginthreadex更安全一些，_beginthreadex会为每个线程分配一些独立的数据块，这个独立的数据块用于保存线程独有的信息，因

为在调用C的标准库时，有些函数是返回的是全局信息，这个全局信息容易被多线程干扰，_beginthreadex会规避这个问题。  

### 2线程等待函数WaitForSingleObject，

`WaitForSingleObject`这个函数使线程等待某个特定的对象，使线程进入等待状态，直到指定的内核对象被触发。

DWORDWINAPIWaitForSingleObject(
 HANDLEhHandle,
 DWORDdwMilliseconds
);

函数说明：  

第一个参数为要等待的内核对象。  

第二个参数为最长等待的时间，以毫秒为单位，如传入5000就表示5秒，传入0就立即返回，传入INFINITE表示无限等待。  

因为线程的句柄在线程运行时是未触发的，线程结束运行，句柄处于触发状态。所以可以用WaitForSingleObject()来等待一个线程结束运行。函数返回值：在指定的时间内对象被触发，函数返回WAIT_OBJECT_0。超过最长等待时间对象仍未被触发返回WAIT_TIMEOUT。传入参数有错误将返回WAIT_FAILED  

具体的使用

可以从我自己做的服务器里截取一部分代码看看

``` cpp
void BaseThread::startup(UInt32 stackSize)
{
    assert(m_nId == 0);
    #if defined _WIN32
    m_hThread =(HANDLE) _beginthreadex(NULL,0,threadFunc, this, 0, &m_nId);
    //cout << this <<endl;
     ::SetThreadPriority(::GetCurrentThread(), 2);
    //让线程跑起来后再退出函数
    // Sleep(1000);
    
    #endif
}

void BaseThread::join()
{
    #if defined _WIN32
     DWORD exitCode;
     while(1)
     {
        if(GetExitCodeThread(m_hThread, &exitCode) != 0)
        {
            if(exitCode != STILL_ACTIVE)
            {
                break;

            }
            else
            {
                // wait之前， 需要唤起线程， 防止线程处于挂起状态导致死等
                ResumeThread(m_hThread);
                WaitForSingleObject(m_hThread, INFINITE);
            }
        }
        else
        {
            break;
        }
     }
    
     CloseHandle(m_hThread);
    #endif

     m_nId = 0;
}
```