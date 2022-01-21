---
title: 智能指针 unique_ptr
date: 2022-01-19 10:05:23
tags: C++
categories: C++
---
## unique_ptr
unique_ptr和shared_ptr不同，unique_ptr不允许所指向的内容被其他指针共享，所以unique_ptr是不允许拷贝构造和赋值的。
``` cpp
void use_uniqueptr()
{
    //指向double类型的unique指针
    unique_ptr<double> udptr;
    //一个指向int类型的unique指针
    unique_ptr<int> uiptr(new int(42));
    // unique不支持copy
    // unique_ptr<int> uiptr2(uiptr);
    // unique不支持赋值
    // unique_ptr<int> uiptr3 = uiptr;
}
```
虽然我们不能拷贝或赋值unique_ptr，但可以通过调用release或reset将指针的所有权从一个（非const）unique_ptr转移给另一个unique：
<!--more-->
``` cpp
void use_uniqueptr()
{
    //定义一个upstr
    unique_ptr<string> upstr(new string("hello zack"));
    // upstr.release()返回其内置指针，并将upstr置空
    // 用upstr返回的内置指针初始化了upstr2
    unique_ptr<string> upstr2(upstr.release());
    unique_ptr<string> upstr3(new string("hello world"));
    //将upstr3的内置指针转移给upstr2
    // upstr2放弃原来的内置指针，指向upstr3返回的内置指针。
    upstr2.reset(upstr3.release());
}
```
unique_ptr有一个成员方法就是release，release可以返回unique_ptr的内置指针，并将unique_ptr置为空。
上述代码将upstr的内置指针转移给upstr2了。同样的道理，通过reset操作, upstr2将upstr3的内置指针绑定了。
release()操作提供了返回unique_ptr的内置指针的方法，但要注意release过后unique_ptr被置空，那返回的内置指针要么手动释放，要么交给其他的智能指针管理。
``` cpp
void use_uniqueptr()
{
    //定义一个upstr
    unique_ptr<string> upstr(new string("hello zack"));
    //获取upstr的内置指针
    string *inerp = upstr.release();
    //因为此时upstr已经通过release交出内置指针使用权
    //所以要手动释放内置指针的内存
    delete inerp;
}
```
不能拷贝unique_ptr的规则有一个例外：我们可以拷贝或赋值一个将要被销毁的unique_ptr。
最常见的例子是从函数返回一个unique_ptr：
``` cpp
unique_ptr<int> clone_unique(int a)
{
    return unique_ptr<int>(new int(a));
}

void use_uniqueptr()
{
    int a = 1024;
    unique_ptr<int> mp = clone_unique(a);
    cout << *mp << endl;
}
```
## 删除器
类似shared_ptr，我们可以为unique_ptr指定删除器，但与之不同的是，为unique_ptr指定删除器时要在尖括号里指定删除器类型
``` cpp
//p 指向一个类型为objT的对象，并使用一个类型为delT的对象释放objT对象
//它会调用一个名为fcn的delT类型对象 
unique_ptr<objT, delT> p(new objT, fcn);
```
作为一个更具体的例子，我们这样演示,先定义一个unique_deleter
``` cpp
void unique_deleter(int *p)
{
    cout << "this is unique deleter" << endl;
    cout << "inner pointer data is " << *p << endl;
}
```
再基于删除器定义一个unique_ptr
``` cpp
void use_uniqueptr()
{
    unique_ptr<int, decltype(unique_deleter) *> mp(new int(1024), unique_deleter);
}
```
我们在主函数调用use_uniqueptr会输出如下
``` cpp
this is unique deleter
inner pointer data is 1024
```
在本例中我们使用了decltype来指明函数指针类型。由于decltype返回一个函数类型，所以我们必须添加一个＊来指出我们正在使用该类型的一个指针。
## weak_ptr
weak_ptr是一种不控制所指向对象生存期的智能指针，它指向由一个shared_ptr管理的对象。
将一个weak_ptr绑定到一个shared_ptr不会改变shared_ptr的引用计数。
一旦最后一个指向对象的shared_ptr被销毁，对象就会被释放。
即使有weak_ptr指向对象，对象也还是会被释放，因此，weak_ptr的名字抓住了这种智能指针“弱”共享对象的特点。
weak_ptr同样包括reset()，use_count()等方法。
与shared_ptr不同的是，weak_ptr提供expired()方法，该方法在use_count为0时返回true, 否则返回false。所以可以通过expired方法去判断weak_ptr的内置指针是否被释放。
weak_ptr通过lock()方法返回一个shared_ptr，shared_ptr内置指针指向的空间和weak_ptr内置指针指向相同。由于weak_ptr的弱共享特点，其内置指针可能被回收，所以当expired为true时， lock()返回一个空的shared_ptr，否则返回一个shared_ptr，该shared_ptr的内置指针与weak_ptr的内置指针指向相同。
我们通过如下几个例子阐述weak_ptr的特性
1  不增加shared_ptr的引用计数
``` cpp
void use_weakptr()
{
    //构造shared_ptr
    auto psint = make_shared<int>(1024);
    //用shared_ptr构造weak_ptr
    weak_ptr<int> pwint(psint);
    //打印shared_ptr的引用计数
    cout << "shared_ptr use count is " << psint.use_count() << endl;
}
```
上述代码输出shared_ptr use count is  1
因为weak_ptr不占用引用计数。
2  通过expired判断内置指针是否被释放
``` cpp
weak_ptr<int> clone_weakptr(int num)
{
    shared_ptr<int> psint(new int(num));
    return weak_ptr<int>(psint);
}

void use_weakptr()
{
    auto wptr = clone_weakptr(1024);
    if (wptr.expired())
    {
        cout << "wptr inner pointer has been deleted" << endl;
    }
    else
    {
        cout << "wptr inner pointer data is " << *(wptr.lock()) << endl;
    }
}
```
在主函数中调用use_weakptr将会输出"wptr inner pointer has been deleted"。
因为clone_weakptr返回的weak_ptr引用了局部变量psint，psint随着函数clone_weakptr结束而释放，所以wptr.expired()返回true
3 通过lock生成shared_ptr
``` cpp
void use_weakptr()
{
    shared_ptr<int> psint(new int(1022));
    //也可以通过赋值，将shared_ptr赋值给weak_ptr
    weak_ptr<int> pwint = psint;
    //通过weak_ptr生成shared_ptr
    shared_ptr<int> psint2 = pwint.lock();
    cout << "psint use count is " << psint.use_count() << endl;
    cout << "psint2 use count is " << psint2.use_count() << endl;
}
```
可以看到通过赋值初始化pwint，pwint.lock()返回另一个shared_ptr，这样两个shared_ptr引用计数相同，都为2.
源码连接
[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
想系统学习更多C++知识,可点击下方链接。
[C++基础](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/23uSgIjfVfmwfhNGMprDFxL0uKL)