---
title: golang面试题汇总(一)
date: 2021-12-03 16:26:44
categories: [golang]
tags: [golang]
---
## 简介
陆续总结一些面试常常会问到的问题，对知识体系做一个梳理
<!--more-->
## 面试题
### 1 简述go协程调度原理
go协程调度模型图
[https://llfc.club/articlepage?id=21XXEPY1IoqquP7phaftDjf9Rt4](https://llfc.club/articlepage?id=21XXEPY1IoqquP7phaftDjf9Rt4)
简述原理:
go的协程是通过MPG模型调用的，M为内核态线程,G为用户态协程,P为处理器,系统会通过调度器从全局队列找到G分配给空闲的M，P会选择一个M来运行，M和G的数量不等，P会有一个本地队列表示M未处理的G,M本身有一个正在处理的G，M每次处理完一个G就从本地队列里取一个G，并且更新P的schedtick字段，如果本地队列没有G，则从全局队列一次性取走G/P个数的G，如果全局队列里也没有，就从其他的P的本地队列取走一半。go1.2之前goroutine是轮询式调度，之后改为抢占式，弱抢占式。
弱抢占式:
如果某个P的schedtick一直没有递增，说明这个P一直在执行一个G任务，如果超过一定时间就会为G增加标记，并且该G执行非内联函数时中断自己并把自己加到队尾。
### 2 struct能否比较
要看struct包含的内容，如果struct包含了指针，chan,map等即使比较也是比较底层元素指针的地址，不同的指针指向同一块内存空间，但是比较出来的结果会不相等。
### 3 defer关键字
一个函数定义了多个defer函数