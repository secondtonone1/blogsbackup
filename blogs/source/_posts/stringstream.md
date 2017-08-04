---
title: stringstream使用方法
date: 2017-08-04 12:36:06
categories: 技术开发
tags: [C++]
---
C++ 有`stringstream`这个工具可以方便的进行数据类型的转换

使用时包含

`#include <sstream.h>`

`using namespace std;`

当需要将一个整形的数转换为字符串

``` cpp
stringstream mystream;

int a = 100;

mystream << a;

std::string numstr;

mystream >> mumstr;
```
<!--more-->

如果需要将一个字符串转化为整形数

再次使用mystream需要清除之前的状态位

调用 `mystream.clear();`

并且字符串置空

mystream.str("");

这样就可以使用了。

``` cpp
mystream.clear()
mystream.str("");

string strtest = "1234";
mystream << strtest;
int numconvert;
mystream >> numconvert;
```
除此之外 stringstream可以连续将输入的内容输出到指定变量

``` cpp
std::string str1 = "1221";
std::string str2 = "12.34";
std::string str3 = "899";

std::stringstream mystream;
mystream << str1 << str2 << str3;
int num1, num3;
double num2;
mystream >> num1 >> num2 >> num3;
```
大体上就是mystream常用的用法了。