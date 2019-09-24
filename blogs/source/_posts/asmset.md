---
title: win7 64位环境下配置汇编环境和程序设计
date: 2017-08-12 11:08:17
categories: [汇编]
tags: [汇编]
---
下载`dosbox`，并解压安装

下载地址： [http://pan.baidu.com/s/1eRJbJAq](http://pan.baidu.com/s/1eRJbJAq)

默认安装到C:\Program Files (x86)\DOSBox-0.74

安装成功后，双击该目录下DOSBox 0.74 Options.bat文件，弹出配置选项文本文档，

找到`[autoexec]`选项，在下面添加如下字段：
``` cpp
MOUNT C D:\masmpro 
set PATH=$PATH$;D:\masmpro
```

D：\masmpro是我创建的汇编程序目录，这样每次启动dosbox，自动挂载到我自己的项目目录里。

当然也可以手动设置，启动dosbox，手动输入MOUNT C D:\masmpro 然后回车

效果如下：
![1.png](1.png)
<!--more-->
接下来输入C:

切换到C：目录下，此时输入dir可以看到D：\masmpro里的文件
![2.png](2.png)
下面配置masmpro，下载地址：[http://pan.baidu.com/s/1jI4WENk](http://pan.baidu.com/s/1jI4WENk)

解压即可，将文件夹MASM内的内容拷贝到D：masmpro里，这其中包括
![3.png](3.png)
之后再masmpro文件夹下建立一个hello.asm的汇编程序
``` asm
data    segment                ;数据段
hello    db    'Hello,World!$',0
data    ends

code    segment                ;代码段
    assume    cs:code,ds:data
start:                    ;入口
    mov    ax,data
    mov    ds,ax
    lea    dx,hello
    mov    ah,9h
    int    21h
    mov    ah,4ch
    int    21h

code    ends

end start            ;标志入口点
```
打开dosbox，输入masm hello.asm
![4.png](4.png)
接着输入link hello链接目标文件
![5.png](5.png)
最后运行hello.exe，输入hello直接运行
![6.png](6.png)
到此为止windows 7 64位环境下汇编环境搭建成功，并且可以开始汇编语言的学习了。

我的公众号
![1.jpg](1.jpg)