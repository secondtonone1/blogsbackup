---
title: makefile学习笔记
date: 2017-08-12 10:54:02
categories: 技术开发
tags: [Linux环境编程]
---
之前都是手动编译的，最近也学了下`makefile`相关的知识，文件结构是这样的在server文件夹里有eventloop.h,eventloop.cpp, networking.h, networking.cpp,

api_epoll.h, api_epoll.cpp,以及文件夹main,在main文件夹里有一个main.cpp文件。

 先写一个简单的makefile

``` cpp
./server: eventloop.o api_epoll.o networking.o main.o
    g++ -o ./server eventloop.o api_epoll.o networking.o  main.o

main.o: ./main/main.cpp eventloop.h api_epoll.h networking.h
    g++ -c ./main/main.cpp -o main.o 

eventloop.o: eventloop.cpp eventloop.h api_epoll.h
    g++ -c eventloop.cpp -o eventloop.o

api_epoll.o: api_epoll.cpp api_epoll.h eventloop.h
    g++ -c api_epoll.cpp -o api_epoll.o

networking.o: networking.cpp networking.h eventloop.h
    g++ -c networking.cpp -o networking.o

clean:
    rm -f $(OBJS) $(TARGET)
```
`makefile基本写法是`

`目标:依赖项`

`　　command`

<!--more-->
`command表示编译规则`，就是g++/gcc编译语句，：左边表示目标，右边表示依赖项要生成目标文件，必须找到对应的依赖项。

上述makefile要生成./server需要eventloop.o networking.o api_epoll.o.如果不存在就先生成这三个.o文件。或者这三个.o文件比./server新，那么就需要重新

生成./server。同理，要生成这些.o文件，需要依赖一些.h和.cpp，那么如果.h或者.cpp有改动，就需要重新生成.o文件，从而生成最新的./server文件。这是最直白的makefile。

对于一些编译规则和目标文件，依赖文件，我们可以采用变量的形式来替代，可以节省时间。

先了解下变量

`CC表示用gcc编译`

`CXX表示用g++编译`

`INC_DIR表示包含的路径`

`CFLAGS表示C语言编译的选项`：`-c表示编译.c/.cpp文件成为.o文件`

`-Wall表示编译显示警告`

`-g表示允许gdb调试`

`-I表示包含路径`

`CXXFLAGS表示g++的编译选项`

`CPPFLAGS表示C语言预编译选项`

`HEADS 表示头文件`

`SOURCES表示源文件`

`OBJS 表示objs文件集合,所有.o文件`

`TARGET表示目标文件，最重要生成的文件`

`all 表示最终的目标`

`clean 是个为目标，make并不产生作用。`

调用make clean 可以执行下面的命令行

`rm -f $(OBJS) $(TARGET)`

为了能读懂makefile，还需要了解一些makefile用到的函数

`$(SOURCES:.cpp=.o)表示将SOURCES中的cpp`文件全部替换为.o为后缀的文件，这样可以达到不用一个一个去写.o文件的作用。

`$(wilcard ./*.cpp)表示在目录./*.cpp下的所有文件，包含路径显示出来。`

patsubst函数格式

`$(patsubst pattern, replacement, text)`

用法如下

`$(patsubst %.c, %.o, $(wildcard *.c))`

将当前目录下.c结尾的文件替换为.o结尾的文件除此之外，还需要了解的就是一些自动变量

`$< 表示依赖项中第一个依赖项的名字`

`$@ 表示目标项中目标的名称`

`$^ 依赖性中所有不重复的依赖项文件，中间用空格区分`

从而我们有了一个改良版本

``` cpp
CC = gcc
CXX = g++

INC_DIR = ./


CFLAGS = -c -Wall
CFLAGS += -g
CFLAGS += -I$(INC_DIR)

CXXFLAGS = $(CFLAGS)

HEADS = $(wildcard $(INC_DIR)/*.h)

SOURCES = $(wildcard ./*.cpp)
OBJS = $(SOURCES:.cpp=.o)
TARGET = ./server

all:$(TARGET)
$(TARGET): $(OBJS)
    $(CXX) -o $@ $(OBJS)  ./main/main.cpp

eventloop.o: eventloop.cpp eventloop.h api_epoll.h
    $(CXX) $(CXXFLAGS) $< -o $@

api_epoll.o: api_epoll.cpp api_epoll.h eventloop.h
     $(CXX) $(CXXFLAGS) $< -o $@

networking.o: networking.cpp networking.h eventloop.h
     $(CXX) $(CXXFLAGS) $< -o $@

clean:
    rm -f $(OBJS) $(TARGET)
```

makefile有个自动推导规则，那就是会根据目标文件.o推导出对应名称的.c依赖项，连command编译规则也会自动生成。

举个例子
``` cpp
eventloop.o:eventloop.cpp eventloop.h api_epoll.h

　　$(CXX) $(CXXFLAGS) $< -o $@
```

makefile 会自动搜寻eventloop.cpp 自动执行$(CXX) $(CXXFLAGS) $< -o $@

所以上面两句可以简化为

eventloop.o: eventloop.h api_epoll.h从而生成了一个我们简化的自动推导版本

``` cpp
CC = gcc
CXX = g++

INC_DIR = ./


CFLAGS = -c -Wall
CFLAGS += -g
CFLAGS += -I$(INC_DIR)

CXXFLAGS = $(CFLAGS)

HEADS = $(wildcard $(INC_DIR)/*.h)

SOURCES = $(wildcard ./*.cpp)
OBJS = $(SOURCES:.cpp=.o)
TARGET = ./server

all:$(TARGET)
$(TARGET): $(OBJS)
    $(CXX) -o $@ $(OBJS)  ./main/main.cpp

eventloop.o:  eventloop.h api_epoll.h

api_epoll.o:  api_epoll.h eventloop.h

networking.o: networking.h eventloop.h

clean:
    rm -f $(OBJS) $(TARGET)
```
 
下面是静态规则

makefile有个静态规则，也相当于推导规则

格式如下
``` cpp
 <targets ...>: <target-pattern>: <prereq-patterns ...>
            <commands>
            ...
```

    `targets定义了一系列的目标文件，可以有通配符。是目标的一个集合。`

    `target-parrtern是指明了targets的模式，也就是的目标集模式。`

    `prereq-parrterns是目标的依赖模式，它对target-parrtern形成的模式再进行一次依赖目标的定义`

举个例子
``` cpp
$(OBJS):%.o:%.cpp 
　　$(CXX) $(CXXFLAGS) $< -o $@
```
OBJS中.o文件是目标的模式， %.c是依赖项的模式

在OBJS中找到eventloop.o，那么推断出eventloop.cpp为对应的依赖文件,从而采取$(CXX) $(CXXFLAGS) $< -o $@编译

下面是根据静态模式写的makefile

``` cpp
CC = gcc
CXX = g++

INC_DIR = ./

CFLAGS = -c -Wall
CFLAGS += -g

CFLAGS += -I$(INC_DIR)
CXXFLAGS = $(CFLAGS)

HEADS = $(wildcard $(INC_DIR)/*.h)

SOURCES = $(wildcard ./*.cpp)

OBJECTS = $(SOURCES:.cpp=.o)

TARGET = ./server

all:$(TARGET)
$(TARGET): $(OBJECTS) $(SOURCES) $(HEADS)
    $(CXX) -o $@ $(OBJECTS) ./main/main.cpp

$(OBJECTS):%.o:%.cpp $(HEADS)
    $(CXX) $(CXXFLAGS) $< -o $@

clean:
    rm -f $(OBJECTS)  $(TARGET)
```
至此，makefile我就了解了这么多，也可以写一些中小型项目makefile了
我的公众号，谢谢关注
![1.jpg](1.jpg)