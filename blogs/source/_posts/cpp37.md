---
title: C++ 模板类的友元和折叠规则
date: 2022-06-15 17:41:05
tags: C++
categories: C++
---
## 为模板类声明友元类
有时我们需要A类访问模板类B的私有成员，但是不想其他类访问，就要在模板类B里为A类声明友元。比如我们想要实现一个BlobPtr类，让BlobPtr类成为Blob类的友元，这样BlobPtr类就可以访问Blob类了。对于Blob类的声明和定义在前文已经阐述[https://llfc.club/articlepage?id=28Vv7hro3VVMPDepLTlLRLqYJhJ](https://llfc.club/articlepage?id=28Vv7hro3VVMPDepLTlLRLqYJhJ)。
我们省略Blob类的详细声明,只为它添加友元类BlobPtr类，并且为他添加友元函数operator==
<!--more-->
``` cpp
class Blob
{
public:
    typedef T value_type;
    typedef typename std::vector<T>::size_type size_type;
    // T类型的BlobPtr是T类型的Blob的友元类
    friend class BlobPtr<T>;
    //重载==运算符
    friend bool operator==(const Blob<T> &, const Blob<T> &);
};
```
## 实现友元类BlobPtr
接下来我们实现友元类BlobPtr,先对其进行声明
``` cpp
template <typename T>
class BlobPtr
{
public:
    BlobPtr() : curr(0) {}
    BlobPtr(Blob<T> &a, size_t sz = 0) : wptr(a.data), curr(sz) {}
    //递增和递减
    BlobPtr &operator++(); //前置运算符
                           // BlobPtr &operator--(); //前置运算符--

    BlobPtr &operator++(int);

private:
    std::shared_ptr<std::vector<T>>
    check(std::size_t, const std::string &) const;
    std::size_t curr; //数组中的当前位置
    //保存一个weak_ptr， 表示底层vector可能被销毁
    std::weak_ptr<std::vector<T>> wptr;
};
```
在实现其定义,这里只举例实现一部分函数，其余的读者自己实现即可。
``` cpp
template <typename T>
BlobPtr<T> &BlobPtr<T>::operator++()
{
    this->curr++;
    return *this;
}

template <typename T>
BlobPtr<T> &BlobPtr<T>::operator++(int)
{
    BlobPtr &rt = *this;
    this->curr++;
    return rt;
}
```
对于友元函数operator == 的定义可以按照如下实现
``` cpp
template <typename T>
bool operator==(const Blob<T> &b1, const Blob<T> &b2)
{
    if (b1.size() > b2.size())
    {
        return true;
    }

    if (b1.siz() < b2.size())
    {
        return false;
    }

    for (unsigned int i = 0; i < b1.size(); i++)
    {
        if (b1.data[i] == b2.data[i])
        {
            continue;
        }

        return b1.data[i] > b2.data[i];
    }

    return true;
}
```
模板类的友元还有一些特殊的用法，如下，读者可以自己体会
``` cpp
template <typename T>
class Pal
{
};

template <typename T>
class Pal2
{
};

class C
{
    // Pal<C>是C类的友元
    friend class Pal<C>;
    //所有类型的Pal2的类都是C的友元
    template <typename T>
    friend class Pal2;
};

// c2本身是一个模板类
template <typename T>
class C2
{
    //和C2同类型的Pal是C2的所有实例友元
    friend class Pal<T>;
    // Pal2的所有实例都是C2的所有实例友元
    template <typename X>
    friend class Pal2;
    // Pal3是一个普通类，他是C2的所有实例的友元
    friend class Pal3;
};
```
## 定义模板类别名
我们可以通过typedef和using等方式为一个模板类定义别名
``` cpp
template <typename Type>
class Bar
{
    //将访问权限授予用来实例化Bar的类型
    friend Type;
};

//定义模板类别名
typedef long long INT64;
//我们可以为实例好的模板类定义别名
typedef Bar<int> mytype;
// C11 可以为模板类定义别名
template <typename T>
using twin = pair<T, T>;
// authors 是一个pair<string, string>
twin<string> authors;
// infos 是一个pair<int, int>类型
twin<int> infos;
template <typename T>
using partNo = pair<T, unsigned>;
// books是pair<string, unsigned>类型
partNo<string> books;
```
## 类模板的静态成员
对于类模板的静态成员，其初始化要放在声明的.h文件中。
``` cpp
//类模板的static成员
template <typename T>
class Foo
{
public:
    static std::size_t count() { return ctr; }

private:
    static std::size_t ctr;
};

//初始化放在和声明所在的同一个.h文件中
template <typename T>
size_t Foo<T>::ctr = 0;
```
## 模板类的作用域访问
默认情况下，C++语言假定通过作用域运算符访问的名字不是类型。
因此，如果我们希望使用一个模板类型参数的类型成员，就必须显式告诉编译器该名字是一个类型。
我们通过使用关键字typename来实现这一点：
``` cpp
// 用typename 告知编译器T::value_type是一个类型
template <typename T>
typename T::value_type top(const T &c)
{
    if (!c.empty())
        return c.back();
    else
        return typename T::value_type();
}
```
我们定义了一个名为top的模板函数，通过T::value_type声明其返回类型，但是C++默认作用域下value_type是一个成员，
所以为了说明value_type是一个类型就需要用typename关键字做声明。
## 通用的函数对象
我们可以通过模板类实现通用的仿函数，也就是实现通用的函数对象，我们先实现一个DebugDelete类，用来删除各种类型的指针对象
``` cpp
//函数对象，给指定类型的指针执行析构
class DebugDelete
{
public:
    DebugDelete(std::ostream &s = std::cerr) : os(s) {}
    //我们定义一个仿函数，参数是T*类型
    template <typename T>
    void operator()(T *p) const
    {
        os << "deleting unique_str" << std::endl;
        delete p;
    }

private:
    std::ostream &os;
};
```
DebugDelete构造函数接纳一个输出流，用来在operator()调用时输出删除信息
接下来我们实现一个测试函数，用来说明DebugDelete的用法
``` cpp
void use_debugdel()
{
    double *p = new double;
    DebugDelete d;
    //调用DebugDelete的仿函数,delete p
    d(p);
    //析构多种类型
    int *np = new int;
    //构造DebugDelete对象后调用仿函数析构np
    DebugDelete()(np);
    //作为删除器析构智能指针
    // p 被delete时会执行DebugDelete的仿函数进行析构
    unique_ptr<int, DebugDelete> p3(new int, DebugDelete());
    // 用DebugDelete 的仿函数析构string的指针
    unique_ptr<string, DebugDelete> sp(new string, DebugDelete());
}
```
可以看出DebugDelete可以用来给智能指针作删除器用。
## 尾置类型的推断
C11新标准中提出了尾置类型推断
``` cpp
// func接受了一个int类型的实参，返回了一个指针，该指针指向一个含有10个整数的数组
auto func(int i) -> int (*)[10];
```
利用这个特性，我们可以用在模板函数中，同样实现一个尾置类型推断函数
``` cpp
//推断返回类型，通过尾置返回允许我们在参数列表之后的声明返回类型
template <typename It>
auto fcnrf(It beg, It end) -> decltype(*beg)
{
    //处理序列
    //返回迭代器beg指向的元素的引用
    return *beg;
}
```
fcnrf的返回类型时It指向元素的解引用(*beg)类型，通过decltype类型推断给出返回的类型。
我们也可以实现一个返回值类型的函数，去掉引用类型。
``` cpp
// remove_reference 是一个模板
// remove_reference<decltype(*beg)>::type
// type的类型就是beg指向元素的类型
// remove_reference<int&>::type type就是int
// remove_reference<string&>::type type就是string

template <typename It>
auto fcncp(It beg, It end) -> remove_reference<decltype(*beg)>
{
    //返回迭代器beg指向元素的copy
    return *beg;
}
```
## 引用折叠规则
我们可以用模板定义一个左值引用
``` cpp
//接受左值引用的模板函数
template <typename T>
void f1(T &t)
{
}
```
当我们用模板类型定义一个右值引用时，传递给该类型的实参类型，会根据C++标准进行折叠。
我们先声明一个右值引用的模板函数
``` cpp
//接受右值引用的模板函数
template <typename T>
void f2(T &&t)
{
}
```
如果我们调用f2(42), T就被推断为int
int i = 100; f2(i) T就被推断为int&  进行参数展开参数就变为int& &&
折叠后就变为int &
所以我们可以做如下归纳：
当模板函数的实参是一个T类型的右值引用
1  传递给该参数的实参是一个右值时， T就是该右值类型
2  传递给该参数的实参是一个左值时， T就是该左值引用类型。

//折叠规则
//X&& ,X&&& 都会被折叠为X&
//X&& && 会被折叠为X&&
所以根据这个规律，我们可以实现一个类似于stl的move操作
``` cpp
template<typename T>
typename remove_reference<T>::type && my_move(T&& t){
    return static_cast<typename remove_reference<T>::type &&>(t);
}
```
如果我们在函数中作如下调用
``` cpp
void use_tempmove()
{
    int i = 100;
    my_move(i);
    //推断规则
    /*
    1  T被推断为int &
    2  remove_reference<int &>的type成员是int
    3  my_move 的返回类型是int&&
    4  推断t类型为int& && 通过折叠规则t为int&类型
    5  最后这个表达式变为 int && my_move(int &t)
    */

    auto rb = my_move(43);
    //推断规则
    /*
    1  T被推断为int
    2  remove_reference<int>的type成员是int
    3  my_move 的返回类型为int&&
    4  my_move 的参数t类型为int &&
    5  最后这个表达式变为 int && my_move(int && t)
    */
}
```
## 总结
这篇文章介绍了模板参数类型的折叠规则和友元类的声明和使用。
视频链接[https://www.bilibili.com/video/BV1cF41177hK?vd_source=8be9e83424c2ed2c9b2a3ed1d01385e9](https://www.bilibili.com/video/BV1cF41177hK?vd_source=8be9e83424c2ed2c9b2a3ed1d01385e9)
源码链接 [https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)