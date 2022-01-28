---
title: 拷贝控制和资源管理
date: 2022-01-27 11:07:00
tags: C++
categories: C++
---
## 拷贝控制
前文我们介绍了HasPtr类的拷贝控制，实现了行为像值的类，所谓行为像值的类就是我们所说的深拷贝，将一个类对象拷贝给另一个类对象时，其所有的成员都作为副本在新的类对象创建一遍，如果是指针类型的成员，则将指针指向的空间的数据复制一份给新对象，这样两个对象所有的成员都不关联，实现了深拷贝，不会受彼此影响。比如之前的HasPtr的拷贝构造
``` cpp
HasPtr::HasPtr(const HasPtr &hp)
{
    cout << "this is copy construtor" << endl;
    if (&hp != this)
    {
        this->m_str = new string(string(*hp.m_str));
        int seconds = time((time_t *)NULL);
        _curnum++;
        this->m_index = _curnum;
    }

    return;
}
```
<!--more-->
下面我们介绍另一只拷贝控制，行为类似于指针的类，所谓行为类似于指针的类就是在类对象进行拷贝构造时，如果有指针成员，则浅拷贝，将指针的值赋值给新的对象，这两个对象共享同一个指针成员的指针。
我们同样实现另一个类SharePtr类，其内部有一个指针成员ps，现在我们实现这样的需求，多个SharePtr之间拷贝构造或者赋值，将ps做浅拷贝，也就是直接赋值给新对象，当所有SharePtr对象销毁后，才销毁成员ps。简单来讲，SharePtr的声明如下
``` cpp 
class SharePtr
{
public:
    //构造函数根据字符串值构造一个字符串指针，并且初始化引用计数为1
    SharePtr(const std::string str = "") : pstr(new string(str)),
                                           usecount(new size_t(1)) {}
    SharePtr(const SharePtr &sptr);
    SharePtr &operator=(const SharePtr &sptr);

private:
    //共享的字符串指针
    string *pstr;
    //引用计数
    size_t *usecount;
};
```
usecount用来管理引用计数，表示多少个SharePtr类对象共享字符串指针。
pstr字符串指针，在SharePtr对象拷贝构造和赋值等操作时，将pstr赋值给新对象，实现浅拷贝。
接下来我们实现拷贝构造和拷贝赋值
``` cpp 
SharePtr &SharePtr::operator=(const SharePtr &sptr)
{
    //如果是自赋值则跳过
    if (&sptr == this)
    {
        return *this;
    }

    //减少自己原来的引用计数
    (*this->usecount)--;

    //判断引用计数是否为0
    if (*(this->usecount) == 0)
    {
        //回收原来指向的内存
        delete (this->pstr);
        //回收引用计数内存
        delete (this->usecount);
    }

    //其他类对象拷贝给自己
    this->pstr = sptr.pstr;

    //对方引用计数增加
    (*sptr.usecount)++;
    //拷贝对方的引用计数给自己
    this->usecount = sptr.usecount;
    return *this;
}

SharePtr::SharePtr(const SharePtr &sptr)
{
    //如果是自赋值则跳过
    if (&sptr == this)
    {
        return;
    }
    //其他类对象拷贝给自己
    this->pstr = sptr.pstr;
    //引用计数增加
    (*sptr.usecount)++;
    //拷贝对方的引用计数给自己
    this->usecount = sptr.usecount;
}
```
实现了拷贝构造和拷贝赋值，首先都要判断是否是自拷贝或者自赋值，如果不是自拷贝，那就将对方的引用计数增加1并且赋值给自己。对于赋值运算，先将自己原来指向的引用计数-1，如果引用计数为0则回收原来开辟的内存。然后将被拷贝的对象引用计数+1,然后赋值给自己。
对于析构函数，先减少引用计数，如果引用计数为0则回收内存
``` cpp
SharePtr::~SharePtr()
{
    //引用计数-1
    (*this->usecount)--;
    //引用计数为0，销毁内存
    if (*(this->usecount) == 0)
    {
        cout << "use count is 0 dealloc"
             << endl;
        delete usecount;
        delete pstr;
        return;
    }
}
```
接下来我们实现一个打印引用计数的函数，一个测试函数
``` cpp
//获取引用计数
size_t SharePtr::getUseCount()
{
    return *(this->usecount);
}

void test_share()
{
    SharePtr sptr1("hello zack");
    SharePtr sptr2(sptr1);
    cout << "sptr1 use count is "
         << sptr1.getUseCount() << endl;
    cout << "sptr2 use count is "
         << sptr2.getUseCount() << endl;
    SharePtr sptr3("hello world");
    cout << "sptr3 use count is "
         << sptr3.getUseCount() << endl;
    sptr2 = sptr3;
    cout << "sptr1 use count is "
         << sptr1.getUseCount() << endl;
    cout << "sptr2 use count is "
         << sptr2.getUseCount() << endl;
    cout << "sptr3 use count is "
         << sptr3.getUseCount() << endl;
}
```
在主函数中调用test_share结果输出如下
``` cpp
sptr1 use count is 2
sptr2 use count is 2
sptr3 use count is 1
sptr1 use count is 1
sptr2 use count is 2
sptr3 use count is 2
use count is 0 dealloc
use count is 0 dealloc
```
因为我们只开辟了两份对象，sptr1和sptr3,所以最后会回收两个对象。sptr2是拷贝构造生成的，共享了sptr1的数据。
可以看到拷贝赋值减少了=左边的引用计数，增加了=右边的引用计数。拷贝构造增加了引用计数，两个对象都共享一套数据。
## 总结
这个例子实现了类指针行为的类SharePtr，其行为很像智能指针，只不过智能指针通过模板实现了泛型而已。所以通过这个例子我们可以理解智能指针的工作原理和内部实现了。
源码链接
[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
想系统学习更多C++知识,可点击下方链接。
[C++基础](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/23uSgIjfVfmwfhNGMprDFxL0uKL)