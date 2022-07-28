---
title: 模板类型的原样转发和可变参数
date: 2022-06-16 17:11:28
tags: C++
categories: C++
---
## 原样转发的意义
前文我们实现了一个my_move函数，用来模拟stl的move操作，实现去引用的功能。其内部的原理就是通过remove_reference实现去引用操作。
有时我们也需要保留原类型的左值或者右值属性，进行原样转发，此时就要用forward实现转发功能。
我们先定义一个模板函数
<!--more-->
``` cpp
template <typename F, typename T1, typename T2>
void flip1(F f, T1 t1, T2 t2)
{
    f(t2, t1);
}
```
flip1内部调用了函数f
我们写一个函数测试
``` cpp

void ftemp(int v1, int &v2)
{
    cout << v1 << " " << ++v2 << endl;
}

void use_ftemp(){
    int j = 100;
    int i = 99;
    flip1(ftemp, j, 42);
    cout << "i is " << i << " j is " << j << endl;
}
```
通过打印发现i和j的值没有变化，因为ftemp的v2参数虽然是引用，但是是flip1的形参t1的引用
t1只是形参，修改t1并不能影响外边的实参j。
想要达到修改实参的目的，需要将flip1的参数修改为引用，我们先实现修改后的版本flip2
``` cpp
template <typename F, typename T1, typename T2>
void flip2(F f, T1 &&t1, T2 &&t2)
{
    f(t2, t1);
}
```
我们定义了一个flip2函数，t1和t2分别是右值引用类型。接下来用一个测试函数进行测试
``` cpp
int j = 100;
int i = 99;
flip2(ftemp, j, 42);
cout << "i is " << i << " j is " << j << endl;
```
这次我们发现j被修改了，因为flip2的t1参数类型为T1的右值引用，当把实参j赋值给flip2时，T1变为int&,
t1的类型就是int& &&，通过折叠t1变为int&类型。这样t1就和实参j绑定了，在flip2内部修改t1，就达到了修改j的目的。
但是flip2同样存在一个问题，如果flip2的第一个参数f，如果f是一个接受右值引用参数的函数，会出现编译错误。
为说明这一点，我们实现一个接纳模板参数右值引用类型的函数
``` cpp
void gtemp(int &&i, int &j)
{
    cout << "i is " << i << " j is " << j << endl;
}
```
此时如果我们将gtemp作为参数传递给flip2会报错
``` cpp
int j = 100;
int i = 99;
// flip2(gtemp, j, 42) 会报错
// 因为42作为右值纯递给flip2，t2会被折叠为int&类型
// t2传递给gtemp第一个参数时，int&&无法绑定int&类型
//flip2(gtemp, i, 42);
cout << "i is " << i << " j is " << j << endl;
```
当我们将42传递给flip2第二个参数时，T2被实例化为int类型，t2就变为int && 类型，通过折叠t2变为int&类型。
t2作为参数传递给gtemp的第一个参数时会报错，
cannot bind rvalue reference of type 'int&&' to lvalue of type 'int'
因为t2是一个左值，右值无法绑定该左值。
解决的办法就是实现一个flip函数，内部实现对T2，T1类型的原样转发。
``` cpp
template <typename F, typename T1, typename T2>
void flip(F f, T1 &&t1, T2 &&t2)
{
    f(std::forward<T2>(t2), std::forward<T1>(t1));
}
```
通过forward将t2类型转化为和T2类型一样的类型，也就是int的右值类型，接下来的调用就不会出问题了
``` cpp
void use_ftemp()
{
    int j = 100;
    int i = 99;
    flip(gtemp, i, 42);
    cout << "i is " << i << " j is " << j << endl;
}
```
## 模板的可变参数
模板同样支持可变参数
``` cpp
//可变参数的函数模板
template <typename T>
ostream &print(ostream &os, const T &t)
{
    return os << t; //输出最后一个元素
}

template <typename T, typename... Args>
ostream &print(ostream &os, const T &t, const Args &...rest)
{
    os << t << ", ";
    return print(os, rest...);
}
```
Args是可变的模板参数包， 然后再用Args定义rest变量，这是一个可变参数列表。
我们的模板函数print内部调用stl的print函数，通过对rest...实现展开操作。
调用过程可按如下的方式
``` cpp
void use_printtemp()
{
    int i = 100;
    string s = "hello zack!!!";
    print(cout, i, s, 42);
}
```
第一次调用print实际是调用的可变参数的print，之后才调用没有可变参数的print函数。
## 总结
本文介绍了模板类型的原样转发，以及多模板参数列表的使用。
视频链接[https://www.bilibili.com/video/BV1ES4y187Yc/?vd_source=8be9e83424c2ed2c9b2a3ed1d01385e9](https://www.bilibili.com/video/BV1ES4y187Yc/?vd_source=8be9e83424c2ed2c9b2a3ed1d01385e9)
源码链接 [https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)