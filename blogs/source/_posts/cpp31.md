---
title: C++ 面试常问问题(一)
date: 2022-03-02 16:50:16
tags: C++
categories: C++
---
这篇文章讲解C++ 面试常问的几个问题。本文通过demo讲解初始化列表，继承，字符串等常问问题。看下边这个例子
## 初始化列表
``` cpp
//基类
class Base
{
public:
    Base() : m_nbase(0), m_nbase2(m_nbase + 100) {}
    Base(int n) : m_nbase(n), m_nbase2(m_nbase + 100)
    {
        cout << "this is Base construct " << endl;
    }
    ~Base()
    {
        cout << "this is Base destruct " << endl;
    }

   virtual void printData()
    {
        cout << "this is Base printData " << endl;
        cout << "data is " << m_nbase << endl;
        // cout << "base2 data is " << m_nbase2 << endl;
    }

    void printData2()
    {
        cout << "base2 data is " << m_nbase2 << endl;
    }

    int SizeOf(char p[])
    {
        return sizeof(p);
    }

    int SizeOf2(char *p)
    {
        return sizeof(p);
    }

private:
    int m_nbase;
    int m_nbase2;
};
```
实现了一个类Base，类的构造函数采用了初始化列表，初始化列表按顺序初始化初始类成员。
接下来在main函数里调用如下
<!--more-->
``` cpp
Base b1(1);
b1.printData();
b1.printData2();
```
1 问Base的初始化列表是否会报错？
回答：
不会有问题，因为初始化列表按顺序初始化类成员。所以会分别输出m_nbase的值为1, m_nbase2的值为101
## 继承问题
继承问题常问到的是基类和子类的关系，继承先构造基类再构造子类。析构时先析构子类，再析构基类。
我们实现一个Derive类继承自Base
``` cpp
class Derive : public Base
{
public:
    Derive(int n) : Base(n), m_nderive(n) 
    { cout << "this is Derive construct " << endl; }

     ~Derive()
    {
        cout << "this is Derive destruct " << endl;
    }

    void printData()
    {
        cout << "this is Derive printData" << endl;
    }

private:
    int m_nderive;
};
```
Derive类继承了Base类，接下来我们在主函数中调用如下
``` cpp
Derive d1(2);
d1.printData();
cout << " ................." << endl;
```
1  问 上面程序输出什么
答 输出如下
``` cmd
this is Base construct 
this is Derive construct 
this is Derive printData
this is Derive destruct 
this is Base destruct 
```
因为构造时，先构造基类再构造子类，析构时先析构子类再析构基类
如果在主函数中调用如下
``` cpp
Base *b1 = new Derive(2);
b1->printData();
delete b1;
```
2  问上面程序输出什么？
答输出如下
``` cmd
this is Base construct 
this is Derive construct 
this is Derive printData
this is Base destruct 
```
因为构造时同样先构造基类，再构造子类。
由于创建子类对象返回基类指针，调用虚函数printData会触发多态机制，调用了子类的printData。
析构时由于析构函数不是虚函数，所以不会调用子类的析构函数！！！
所以只会输出Base的析构函数。
3  问如何解决多态情况下子类无法析构问题？
答 将基类和子类的析构函数都设置为虚析构函数即可，少一个都不行！！！
修改如下
``` cpp
class Base
{
public:
    Base() : m_nbase(0), m_nbase2(m_nbase + 100) {}
    Base(int n) : m_nbase(n), m_nbase2(m_nbase + 100)
    {
        cout << "this is Base construct " << endl;
    }
    ~Base()
    {
        cout << "this is Base destruct " << endl;
    }

   virtual void printData()
    {
        cout << "this is Base printData " << endl;
        cout << "data is " << m_nbase << endl;
        // cout << "base2 data is " << m_nbase2 << endl;
    }
    //....省略
};

class Derive : public Base
{
public:
    Derive(int n) : Base(n), m_nderive(n) 
    { cout << "this is Derive construct " << endl; }

    virtual  ~Derive()
    {
        cout << "this is Derive destruct " << endl;
    }

    void printData()
    {
        cout << "this is Derive printData" << endl;
    }

private:
    int m_nderive;
};
```
此时再调用
``` cpp
Base *b1 = new Derive(2);
b1->printData();
delete b1;
```
程序输出
``` cpp
this is Base construct 
this is Derive construct 
this is Derive printData
this is Derive destruct 
this is Base destruct 
```
## 字符串sizeof
字符串的sizeof也是常问的问题
``` cpp
char carry[100] = "Hello World";
cout << "sizeof(carry)  " << sizeof(carry) << endl;
char *pstr = "Hello World";
cout << "sizeof(pstr)   " << sizeof(pstr) << endl;
cout << "sizeof(*pstr)  " << sizeof(*pstr) << endl;
Base b1;
cout << "b1.SizeOf(carry)  " << b1.SizeOf(carry) << endl;
cout << "b1.SizeOf2(carry) " << b1.SizeOf2(carry) << endl;
```
1  问上边程序输出什么
答 输出如下
``` cpp
sizeof(carry)  100
sizeof(pstr)  4
sizeof(*pstr)  1
b1.SizeOf(carry) 4
b1.SizeOf2(carry) 4
```
carry是个数组，所以sizeof(carry)是数组的大小为100
pstr是个char指针，所以sizeof(pstr)是指针大小为4字节，当然在64位机器为8
我的机器上输出的是8，这个根据机器不同而不同，记住是指针大小就行。
pstr指向字符串首地址，也是第一个字符的地址,*pstr为第一个字符
sizeof(*pstr)为一个字符的大小，所以为1
b1.SizeOf以及b1.SizeOf2内部调用的都是sizeof()一个char指针，所以都为4，
我的机器上输出为8
## bool，float和0比较
bool和0比较
``` cpp
bool bval = true;
if(!bval){
    //...
    //bval为0的逻辑
}else{
    //...
}
```
float和0比较
``` cpp
    float f1 = 0.1;
    if (f1 <= FLT_EPSILON && f1 >= -FLT_EPSILON)
    {
        cout << "this is float 0" << endl;
    }
    else
    {
        cout << "this is not float 0" << endl;
    }

    double d1 = 0.0;
    if (d1 <= DBL_EPSILON && d1 >= -DBL_EPSILON)
    {
        cout << "this is double 0" << endl;
    }
    else
    {
        cout << "this is not double 0" << endl;
    }
```
## 总结
本文介绍了一些面试常见问题
源码链接
[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
想系统学习更多C++知识,可点击下方链接。
[C++基础](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/24H7mZgZw57zCqGoULERZ2mQWOM)



