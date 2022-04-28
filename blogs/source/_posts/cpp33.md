---
title: C++ 运算符重载
date: 2022-04-10 14:52:06
tags: C++
categories: C++
---
本文介绍了C++ 运算符重载的用法，以我们构造的string类为例子，说明重载的用法。
## 构造我们自己的string类
声明如下
``` cpp
class mystring_
{
public:
    mystring_(/* args */);
    mystring_(const mystring_ &mstr);
    mystring_(const char *m_str);
    mystring_(const string);
    ~mystring_();

    friend ostream &operator<<(ostream &os, const mystring_ &mystr1);
    mystring_ &operator=(const mystring_ &mystr);
    mystring_ &operator=(string str);
    mystring_ &operator=(const char *cstr);
    friend mystring_ operator+(const mystring_ &str1, const mystring_ &str2);
    char operator[](unsigned int index);
    friend class mystringOpr_;

private:
    char *m_str;
};
```
<!--more-->
在string类里重载了输出运算符`<<`，赋值运算符=, 加法运算符+, 取下标运算符[], 又声明了友元类mystringOpr。
``` cpp
class mystringOpr_
{
public:
    bool operator()(const mystring_ &, const mystring_ &);
};
```
该类重载了()运算符，这样mystringOpr的实例对象就是可调用对象了，可以当作仿函数使用。
## mystring_类的具体实现
我们先实现构造函数和析构函数
``` cpp
mystring_::mystring_(/* args */) : m_str("")
{
}
mystring_::mystring_(const mystring_ &mystr)
{
    if (&mystr == this)
    {
        return;
    }

    if (mystr.m_str == nullptr)
    {
        m_str = nullptr;
        return;
    }
    size_t len = strlen(mystr.m_str);
    m_str = new char[len + 1];
    strcpy(m_str, mystr.m_str);
    m_str[len] = '\0';
}

mystring_::mystring_(const char *mstr)
{
    cout << "use mystring_ construct , param is const char *" << endl;
    size_t len = strlen(mstr);
    m_str = new char[len + 1];
    strcpy(m_str, mstr);
    m_str[len] = '\0';
}

mystring_::mystring_(const string str)
{
    cout << "use mystring_ construct string str" << endl;
    size_t len = str.length();
    m_str = new char[len + 1];
    strncpy(m_str, str.c_str(), len);
    m_str[len] = '\0';
}

mystring_::~mystring_()
{
    if (m_str == "" || m_str == nullptr)
    {
        cout << "begin destruct , str is null or empty" << endl;
    }
    else
    {
        cout << "begin destruct " << m_str << endl;
    }

    if (m_str == nullptr || m_str == "")
    {
        return;
    }

    delete[] m_str;
    m_str = nullptr;
}
```
实现了形参不同的拷贝构造函数，包括const string类型，const char`*`类型，const mystring_ &类型，以及默认的无参构造函数。
接下来重载+运算符
``` cpp
mystring_ operator+(const mystring_ &str1, const mystring_ &str2)
{
    size_t len = strlen(str1.m_str) + strlen(str2.m_str) + 1;
    mystring_ strtotal;
    strtotal.m_str = new char[len + 1];
    memset(strtotal.m_str, 0, len);
    memcpy(strtotal.m_str, str1.m_str, strlen(str1.m_str));
    strcat(strtotal.m_str, str2.m_str);
    return strtotal;
}
```
重载=运算符
``` cpp
mystring_ &mystring_::operator=(const mystring_ &mystr)
{
    if (&mystr == this)
    {
        return *this;
    }

    if (this->m_str != nullptr)
    {
        delete[] m_str;
        this->m_str = nullptr;
    }

    if (mystr.m_str == nullptr)
    {
        m_str = nullptr;
    }

    size_t len = strlen(mystr.m_str);
    m_str = new char[len + 1];
    strcpy(m_str, mystr.m_str);
    m_str[len] = '\0';
    return *this;
}

mystring_ &mystring_::operator=(string str)
{
    cout << "use operator = string str" << endl;
    if (this->m_str != nullptr)
    {
        delete[] m_str;
        this->m_str = nullptr;
    }

    size_t len = str.length();
    m_str = new char[len + 1];
    strncpy(m_str, str.c_str(), len);
    m_str[len] = '\0';
    return *this;
}

mystring_ &mystring_::operator=(const char *cstr)
{
    cout << "use operator = const char*" << endl;
    if (this->m_str != nullptr)
    {
        delete[] m_str;
        this->m_str = nullptr;
    }

    size_t len = strlen(cstr);
    m_str = new char[len + 1];
    strncpy(m_str, cstr, len);
    m_str[len] = '\0';
    return *this;
}
```
重载输出运算符
``` cpp
ostream &operator<<(ostream &os, const mystring_ &mystr1)
{
    if (mystr1.m_str == nullptr)
    {
        os << "mystring_ data is null" << endl;
        return os;
    }
    os << "mystring_ data is " << mystr1.m_str << endl;
    return os;
}
```
重载取下表运算符[]
``` cpp
char mystring_::operator[](unsigned int index)
{
    if (index >= strlen(m_str))
    {
        throw "index out of range!!!";
    }

    return m_str[index];
}
```
我们写一个函数测试重载效果
``` cpp
void use_mystr_1()
{
    auto mystr1 = mystring_("hello zack");
    auto mystr2(mystr1);
    auto mystr3 = mystring_();
    cout << mystr1 << mystr2 << mystr3 << endl;
    mystring_ mystr4 = ", i love u";
    auto mystr5 = mystr1 + mystr4;
    cout << "mystr4 is " << mystr4 << endl;
    cout << "mystr5 is " << mystr5 << endl;

    mystring_ mystr6 = "";
    auto mystr7 = mystr5 + mystr6;
    cout << "mystr7 is " << mystr7 << endl;

    auto ch = mystr1[4];
    cout << "index is 4, char is " << ch << endl;
}
```
测试结果如下
``` cpp
use mystring_ construct , param is const char *
mystring_ data is hello zack
mystring_ data is hello zack
mystring_ data is

use mystring_ construct , param is const char *
mystr4 is mystring_ data is , i love u

mystr5 is mystring_ data is hello zack, i love u

use mystring_ construct , param is const char *
mystr7 is mystring_ data is hello zack, i love u

index is 4, char is o
```
可以看出调用不同的构造函数会打印不同的日志，重载+运算符实现了字符串的拼接，重载赋值实现了拷贝。
## 通过仿函数mystringOpr_实现排序
我们先实现仿函数，这样就可以对我们mystring_类对象排序了
``` cpp
bool mystringOpr_::operator()(const mystring_ &str1, const mystring_ &str2)
{
    if (strlen(str1.m_str) > strlen(str2.m_str))
    {
        return true;
    }

    if (strlen(str1.m_str) < strlen(str2.m_str))
    {
        return false;
    }

    for (unsigned int i = 0; i < strlen(str1.m_str); i++)
    {
        return str1.m_str[i] > str2.m_str[i];
    }
}
```
我们调用sort实现mystring_排序,在原来的基础上补充排序代码
``` cpp
void use_mystr_1()
{
    auto mystr1 = mystring_("hello zack");
    auto mystr2(mystr1);
    auto mystr3 = mystring_();
    cout << mystr1 << mystr2 << mystr3 << endl;
    mystring_ mystr4 = ", i love u";
    auto mystr5 = mystr1 + mystr4;
    cout << "mystr4 is " << mystr4 << endl;
    cout << "mystr5 is " << mystr5 << endl;

    mystring_ mystr6 = "";
    auto mystr7 = mystr5 + mystr6;
    cout << "mystr7 is " << mystr7 << endl;

    auto ch = mystr1[4];
    cout << "index is 4, char is " << ch << endl;

    std::vector<mystring_> vec_mystring_;
    vec_mystring_.push_back(mystr1);
    vec_mystring_.push_back(mystr2);
    vec_mystring_.push_back(mystr3);
    vec_mystring_.push_back(mystr4);
    vec_mystring_.push_back(mystr5);
    vec_mystring_.push_back(mystr6);
    vec_mystring_.push_back(mystr7);
    sort(vec_mystring_.begin(), vec_mystring_.end(), mystringOpr_());
    cout << "====================after sort ..." << endl;
    for_each(vec_mystring_.begin(), vec_mystring_.end(), [](const mystring_ &str)
             { cout << str << endl; });
}
```
上述代码通过sort排序vector里的mystring_对象，并用lambda表达式输出，可以看到输出
``` cmd
use mystring_ construct , param is const char *
mystring_ data is hello zack
mystring_ data is hello zack
mystring_ data is

use mystring_ construct , param is const char *
mystr4 is mystring_ data is , i love u

mystr5 is mystring_ data is hello zack, i love u

use mystring_ construct , param is const char *
mystr7 is mystring_ data is hello zack, i love u

index is 4, char is o
====================after sort ...
mystring_ data is hello zack, i love u

mystring_ data is hello zack, i love u

mystring_ data is hello zack

mystring_ data is hello zack

mystring_ data is , i love u

mystring_ data is

mystring_ data is
```
我们通过仿函数和sort实现了mystring_从大到小的排序。
## 总结
源码链接：[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
视频链接: [https://www.bilibili.com/video/BV1zu411e7DW/](https://www.bilibili.com/video/BV1zu411e7DW/)