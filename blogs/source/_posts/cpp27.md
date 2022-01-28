---
title: swap操作
date: 2022-01-28 11:12:04
tags: C++
categories: C++
---
## swap操作
我们常用的交换两个数据的操作是这样
``` cpp
void swap_int(int &a, int &b)
{
    int temp = a;
    a = b;
    b = temp;
}
```
主函数调用是这样的
``` cpp
    int a = 100, b = 200;
    swap_int(a, b);
    cout << "a is " << a << endl;
    cout << "b is " << b << endl;
```
程序输出
``` cmd
a is 200
b is 100
```
<!--more-->
可见a和b交换了，stl为我们提供了swap函数，可以交换两个对象的数据。但大部分情况还是需要我们实现自己的swap函数，比如交换两个HasPtr对象
``` cpp

class HasPtr
{
public:
    HasPtr() = default;
    HasPtr(const string &str);
    ~HasPtr();
    HasPtr(const HasPtr &hp);
    HasPtr &operator=(const HasPtr &hp);
    friend ostream &operator<<(ostream &os, const HasPtr &);
    friend void swap(HasPtr &, HasPtr &);

private:
    string *m_str;
    int m_index;
    static int _curnum;
};

void swap(HasPtr &hptr1, HasPtr &hptr2)
{
    using std::swap;
    swap(hptr1.m_index, hptr2.m_index);
    swap(hptr1.m_str, hptr2.m_str);
}
```
我们在类的声明里添加了swap友元函数。然后利用了stl的swap函数完成了类内部成员的交换。交换后两个类的m_str指针指向交换了。
有了我们自己实现的swap函数，就可以简化之前的赋值运算符了。但是我们要注释掉原来版本实现的operator = ，实现新版本的
``` cpp
HasPtr &HasPtr::operator=(HasPtr hptr)
{
    // hptr是一个局部变量
    //交换后hptr.m_str指向this.m_str原来指向的内存
    swap(*this, hptr);
    // return返回*this后，hptr自动被回收
    return *this;
}
```
新版本的operator=参数为HasPtr类型，而不是引用类型，这样hptr是赋值运算符右侧变量的副本，而函数内部通过swap交换this和hptr的数据和成员，最后返回*this。随着函数返回，那么形参也就会自动销毁了，这么做的好处是代码简洁，并且不用考虑自赋值的情况。
## 总结
本文介绍了swap操作，合理利用swap，并为类实现swap操作，可以简化我们的操作。当我们要实现sort等排序操作，内部会用到交换逻辑，如果想实现定制化的swap就需要为类实现swap函数。
[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
想系统学习更多C++知识,可点击下方链接。
[C++基础](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/23uSgIjfVfmwfhNGMprDFxL0uKL)