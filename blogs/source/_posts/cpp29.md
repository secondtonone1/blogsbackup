---
title: C++ 动态内存管理示例
date: 2022-02-09 10:45:54
tags: C++
categories: C++
---
## 动态内存管理
之前我们讲述过动态内存的开辟，可以通过new, malloc，以及alloc等方式，本文通过介绍alloc方式，构造一个StrVec类，这个类的功能类似于一个vector，实现字符串的管理，其中包含push一个字符串，动态扩容，析构，回收内存等操作。
## StrVec类实现细节
StrVec类实现如下
``` cpp
class StrVec
{
public:
    //无参构造函数
    StrVec() : elements(nullptr), first_free(nullptr),
               cap(nullptr) {}
    //拷贝构造函数
    StrVec(const StrVec &);
    //拷贝赋值运算符
    StrVec &operator=(const StrVec &);
    //析构函数
    ~StrVec();
    //拷贝元素
    void push_back(const std::string &);
    //返回元素个数
    size_t size() const { return first_free - elements; }
    //返回总容量
    size_t capacity() const { return cap - elements; }
    //返回首元素地址
    std::string *begin() const
    {
        return elements;
    }
    //返回第一个空闲元素地址
    //也是最后一个有效元素的下一个位置
    std::string *end() const
    {
        return first_free;
    }

private:
    //判断容量不足开辟新空间
    void chk_n_alloc()
    {
        if (size() == capacity())
            reallocate();
    }
    //重新开辟空间
    void reallocate();
    // copy指定范围的元素到新的内存中
    std::pair<std::string *, std::string *> alloc_n_copy(
        const std::string *, const std::string *);
    //释放空间
    void free();
    //数组首元素的指针
    std::string *elements;
    //指向数组第一个空闲元素的指针
    std::string *first_free;
    //指向数组尾后位置的指针
    std::string *cap;
    //构造string类型allocator静态成员
    static std::allocator<std::string> alloc;
};
```
<!--more-->
1  elements成员，该成员指向StrVec内部数组空间的第一个元素
2  first_free成员指向第一个空闲元素，也就是有效元素的下一个元素，该元素开辟空间但未构造。
3  cap 指向最后一个元素的下一个位置。
4  alloc为静态成员，主要负责string类型数组的开辟工作。
5  无参构造函数将三个指针初始化为空，并且默认够早了alloc。
6  alloc_n_copy私有函数的功能是将一段区域的数据copy到新的空间，
并且返回新开辟的空间地址以及第一个空闲元素的地址(第一个未构造元素的地址)。
7  chk_n_alloc私有函数检测数组大小是否达到容量，如果达到则调用reallocate重新开辟空间。
8  reallocate重新开辟空间
9  capacity返回总容量
10 size返回元素个数
11 push_back 将元素放入开辟的类似于数组的连续空间中。
12 begin返回首元素地址
13 end返回第一个空闲元素地址,也是最后一个有效元素的下一个位置
无论我们实现push操作还是拷贝构造操作，都要实现realloc，当空间不足时要开辟空间将旧数据移动到新的数据
``` cpp
//重新开辟空间
void StrVec::reallocate()
{
    string *newdata = nullptr;
    //数组为空的情况
    if (elements == nullptr || cap == nullptr || first_free == nullptr)
    {
        newdata = alloc.allocate(1);
        // elements和first_free都指向首元素
        elements = newdata;
        first_free = newdata;
        // cap指向数组尾元素的下一个位置。
        cap = newdata + 1;
        return;
    }
    //不为空则扩充两倍空间
    newdata = alloc.allocate(size() * 2);
    //新内存空闲位置
    auto dest = newdata;
    //旧内存有效位置
    auto src = elements;
    //通过移动操作将旧数据放到新内存中
    for (size_t i = 0; i != size(); ++i)
    {
        alloc.construct(dest++, std::move(*src++));
    }
    //移动后旧内存数据无效，一定要删除
    free();
    //更新数据位置
    elements = newdata;
    //更新第一个空闲位置
    first_free = dest;
    //更新容量
    cap = elements + size() * 2;
}
```
reallocate函数内部判断是否为刚初始化指针却没开辟空间的空数组，如果是则开辟1个大小的空间。
否则则开辟原有空间的两倍，将旧数据移动到新空间，采用了std::move操作，这么做减少拷贝造成的性能开销。
move之后原数据就无效了，所以要调用私有函数free()进行释放。我们实现该free操作
``` cpp
//释放操作
void StrVec::free()
{
    //判断elements是否为空
    if (elements == nullptr)
    {
        return;
    }

    auto dest = elements;
    //要先遍历析构每一个对象
    for (size_t i = 0; i < size(); i++)
    {
        // destroy会调用每一个元素的析构函数
        alloc.destroy(dest++);
    }
    //再整体回收内存
    alloc.deallocate(elements, cap - elements);
}
```
先通过遍历destroy销毁内存，从而调用string的析构函数，最后在deallocate回收内存。
``` cpp
// copy指定范围的元素到新的内存中,返回新元素的地址和第一个空闲元素地址的pair
std::pair<std::string *, std::string *> StrVec::alloc_n_copy(
    const std::string *b, const std::string *e)
{
    auto newdata = alloc.allocate(e - b);
    //将原数据用来初始化新空间
    auto first_free = uninitialized_copy(b, e, newdata);
    return {newdata, first_free};
}
```
这样利用alloc_n_copy，我们就可以实现拷贝构造和拷贝赋值了
``` cpp
//拷贝构造函数
StrVec::StrVec(const StrVec &strtmp)
{
    //将形参数据拷贝给自己
    auto rsp = alloc_n_copy(strtmp.begin(), strtmp.end());
    //更新elements, cap，first_free
    elements = rsp.first;
    first_free = rsp.second;
    cap = rsp.second;
}
```
但是拷贝赋值要注意一点，就是自赋值的情况，所以我们提前判断是否为自赋值，如不是则进行和拷贝构造相同的操作
``` cpp
//拷贝赋值运算符
StrVec &StrVec::operator=(const StrVec &strtmp)
{
    //防止自赋值
    if (this == &strtmp)
    {
        return *this;
    }
    //将形参数据拷贝给自己
    auto rsp = alloc_n_copy(strtmp.begin(), strtmp.end());
    //更新elements, cap，first_free
    elements = rsp.first;
    first_free = rsp.second;
    cap = rsp.second;
}
```
我们可以利用free实现析构函数
``` cpp
//析构
StrVec::~StrVec()
{
    free();
}
```
接下来我们实现push_back，将指定字符串添加到数组空间,以及抛出元素
``` cpp
//添加元素
void StrVec::push_back(const std::string &s)
{
    chk_n_alloc();
    alloc.construct(first_free++, s);
}

//抛出元素
void StrVec::pop_back(std::string &s)
{
    if (first_free == nullptr)
    {
        return;
    }

    if (size() == 1)
    {
        
        s = *elements;
        alloc.destroy(elements);
        first_free = nullptr;
        elements = nullptr;
        return;
    }

    s = *(--first_free);
    alloc.destroy(first_free);
}
```
接下来实现测试函数，测试上述操作
``` cpp
void test_strvec()
{
    auto str1 = StrVec();
    str1.push_back("hello zack");
    StrVec str2(str1);
    str2.push_back("hello rolin");
    StrVec str3 = str1;
    string strtmp;
    str3.pop_back(strtmp);
}
```
在主函数调用上面test_strvec，运行稳定。
## 总结
本文通过allocator实现了一个类似于vector的类，管理string变量。演示了拷贝构造，拷贝赋值要注意的事项，同时演示了如何手动开辟内存并管理内存空间。
源码链接
[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
想系统学习更多C++知识,可点击下方链接。
[C++基础](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/24H7mZgZw57zCqGoULERZ2mQWOM)