---
title: C++ IO流
date: 2021-12-29 17:49:19
tags: C++
categories: C++
---
## 流的状态
C++流包括istream, ostream，基于istream继承实现了istringstream和ifstream，基于ostream继承实现了ostringstream和ofstream。
由于不能拷贝IO对象，因此我们也不能将形参或返回类型设置为流类型。
进行IO操作的函数通常以引用方式传递和返回流。读写一个IO对象会改变其状态，因此传递和返回的引用不能是const的。
<!--more-->
IO库定义了一个与机器无关的iostate类型，它提供了表达流状态的完整功能。
IO库定义了4个iostate类型的constexpr值，表示特定的位模式。这些值用来表示特定类型的IO条件，可以与位运算符一起使用来一次性检测或设置多个标志位。

badbit表示系统级错误，如不可恢复的读写错误。通常情况下，一旦badbit被置位，流就无法再使用了
。在发生可恢复错误后，failbit被置位，如期望读取数值却读出一个字符等错误。这种问题通常是可以修正的，流还可以继续使用。
如果到达文件结束位置，eofbit和failbit都会被置位。
goodbit的值为0，表示流未发生错误。
如果badbit、failbit和eofbit任一个被置位，则检测流状态的条件会失败。

流对象的rdstate成员返回一个iostate值，对应流的当前状态。
setstate操作将给定条件位置位，表示发生了对应错误。
clear成员是一个重载的成员：它有一个不接受参数的版本，而另一个版本接受一个iostate类型的参数。
clear不接受参数的版本清除（复位）所有错误标志位。执行clear()后，调用good会返回true。
``` cpp
    //保留当前cin状态
    auto old_state = cin.rdstate();
    //使cin有效
    cin.clear();
    //将cin置为原有状态
    cin.setstate(old_state);
```
## 管理输出缓冲

如果想在每次输出操作后都刷新缓冲区，我们可以使用unitbuf操纵符。它告诉流在接下来的每次写操作之后都进行一次flush操作。而nounitbuf操纵符则重置流，使其恢复使用正常的系统管理的缓冲区刷新机制。
``` cpp
    //所有输出操作都立即刷新缓冲区
    cout << unitbuf;
    //恢复到正常模式
    cout << nounitbuf;
```
## 关联输入和输出流
tie有两个重载的版本：一个版本不带参数，返回指向输出流的指针。如果本对象当前关联到一个输出流，则返回的就是指向这个流的指针，如果对象未关联到流，则返回空指针。tie的第二个版本接受一个指向ostream的指针，将自己关联到此ostream。即，x.tie（&o）将流x关联到输出流o。

我们既可以将一个istream对象关联到另一个ostream，也可以将一个ostream关联到另一个ostream：
``` cpp
    //将cin和cout绑定在一起
    cin.tie(&cout);
    // old_tie指向当前关联到cin的流(如果有的话)
    // cin不再与其他流关联
    ostream *old_tie = cin.tie(nullptr);
    //将cin和cerr关联
    //读取cin会刷新cerr
    cin.tie(&cerr);
    //重建cin和cout的关联
    cin.tie(old_tie);
```
在这段代码中，为了将一个给定的流关联到一个新的输出流，我们将新流的指针传递给了tie。为了彻底解开流的关联，我们传递了一个空指针。每个流同时最多关联到一个流，但多个流可以同时关联到同一个ostream。
## fstream
对文件操作可以使用fstream，fstream分为ofstream和ifstream，ofstream是向文件里写，ifstream是从文件里读
``` cpp
//输入流关联opfile路径的文件
ifstream(opfile);
//输出流未关联任何文件
ofstream();
```
之前我们的Sale_data类是从istream读数据，现在改为从文件读数据
![https://cdn.llfc.club/1640920217%281%29.jpg](https://cdn.llfc.club/1640920217%281%29.jpg)

fstream包含成员函数open和close
如果我们定义了一个空文件流对象，可以随后调用open来将它与文件关联起来：
![https://cdn.llfc.club/1640920542.jpg](https://cdn.llfc.club/1640920542.jpg)
一旦一个文件流已经打开，它就保持与对应文件的关联。实际上，对一个已经打开的文件流调用open会失败，并会导致failbit被置位。随后的试图使用文件流的操作都会失败。为了将文件流关联到另外一个文件，必须首先关闭已经关联的文件。一旦文件成功关闭，我们可以打开新的文件：
![https://cdn.llfc.club/1640920775%281%29.jpg](https://cdn.llfc.club/1640920775%281%29.jpg)
文件模式
![https://cdn.llfc.club/1640921452%281%29.jpg](https://cdn.llfc.club/1640921452%281%29.jpg)
当我们打开一个ofstream时，文件的内容会被丢弃。阻止一个ofstream清空给定文件内容的方法是同时指定app模式：
![https://cdn.llfc.club/1640921967%281%29.jpg](https://cdn.llfc.club/1640921967%281%29.jpg)
## stringstream
istringstream从string读取数据，ostringstream向string写入数据，而头文件stringstream既可从string读数据也可向string写数据。
定义一个PersonInfo的公有类
![https://cdn.llfc.club/1640932091%281%29.jpg](https://cdn.llfc.club/1640932091%281%29.jpg)
我们的程序会读取数据文件，并创建一个PersonInfo的vector。vector中每个元素对应文件中的一条记录。我们在一个循环中处理输入数据，每个循环步读取一条记录，提取出一个人名和若干电话号码：
![https://cdn.llfc.club/1640932377%281%29.jpg](https://cdn.llfc.club/1640932377%281%29.jpg)
这里我们用getline从标准输入读取整条记录。如果getline调用成功，那么line中将保存着从输入文件而来的一条记录。在while中，我们定义了一个局部PersonInfo对象，来保存当前记录中的数据。
我们可以使用ostream管理要输出的字符串
![https://cdn.llfc.club/1640933439%281%29.jpg](https://cdn.llfc.club/1640933439%281%29.jpg)
在此程序中，我们假定已有两个函数，valid和format，分别完成电话号码验证和改变格式的功能。程序最有趣的部分是对字符串流formatted和badNums的使用。我们使用标准的输出运算符（<<）向这些对象写入数据，但这些“写入”操作实际上转换为string操作，分别向formatted和badNums中的string对象添加字符。
源码链接[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
我的公众号，谢谢关注
![https://cdn.llfc.club/gzh.jpg](https://cdn.llfc.club/gzh.jpg)