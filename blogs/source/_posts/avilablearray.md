---
title: 柔性数组探索和应用
date: 2017-08-04 12:48:23
categories: 技术开发
tags: [C++]
---
`redis`字符串可以实现通过地址偏移找到所在结构体的首地址，`struct sdshdr *sh = (void *)(s - (sizeof(struct sdshdr)))`
![1](avilablearray/1.png)
也就是通过buf地址可以找到sdshdr的地址，这个我一直不理解，写了代码测试下
<!--more-->
![2](avilablearray/2.png)
地址一次间隔4，结构体总大小为8，最后一个buf是空数组，没大小。之前自己一直错误的认为buf的大小按照char *开辟，这次打印出来大小为0
将结构体buf成员改为char *类型
![3](avilablearray/3.png)
这次大小变为12了，也就是char* 占用了四个字节
 现在回到最初的结构
![4](avilablearray/4.png)
 我尝试给buf开辟空间
![5](avilablearray/5.png)
编译是不允许的，这是个零大小的数组，但是buf[lenth]这种方式可以访问，只是数组越界罢了。
那就要一次给这个结构体开辟好空间，通过buf位移取出数据
![6](avilablearray/6.png)
结果
![7](avilablearray/7.png)  
  
![8](avilablearray/8.png)  
&buf和buf所指向的地址一个地址。因为它本身没有空间