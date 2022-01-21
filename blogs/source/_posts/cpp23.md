---
title: C++ 动态数组
date: 2022-01-20 10:51:31
tags: C++
categories: C++
---
C++语言和标准库提供了两种一次分配一个对象数组的方法。C++语言定义了另一种new表达式语法，可以分配并初始化一个对象数组。标准库中包含一个名为allocator的类，允许我们将分配和初始化分离。使用allocator通常会提供更好的性能和更灵活的内存管理能力。
<!--more-->
## new和数组
为了让new分配一个对象数组，我们要在类型名之后跟一对方括号，在其中指明要分配的对象的数目。在下例中，new分配要求数量的对象并（假定分配成功后）返回指向第一个对象的指针：
``` cpp
int get_size_new()
{
    return 42;
}
void new_array()
{
    int *p_array = new int[get_size_new()]();
    for (int i = 0; i < get_size_new(); i++)
    {
        cout << *(p_array + i) << " ";
    }

    delete[] p_array;
    cout << endl;
}
```
在main函数中调用new_array会输出42个0,因为new 分配的数组初始值都为0。
为了释放动态数组，我们使用一种特殊形式的delete——在指针前加上一个空方括号对.
方括号中的大小必须是整型，但不必是常量。也可以用一个表示数组类型的类型别名，来分配一个数组，这样，new表达式中就不需要方括号了：
``` cpp
void new_array()
{
    //定义数组类型
    typedef int array_type[10];
    //动态开辟数组空间
    int *p_array = new (array_type);
    delete[] p_array;
}
```
虽然我们通常称`new T[]`分配的内存为“动态数组”，但这种叫法某种程度上有些误导。
当用new分配一个数组时，我们并未得到一个数组类型的对象，而是得到一个数组元素类型的指针。
即使我们使用类型别名定义了一个数组类型，new也不会分配一个数组类型的对象。new返回的是一个元素类型的指针。
由于分配的内存并不是一个数组类型，因此不能对动态数组调用begin或end。
要记住我们所说的动态数组并不是数组类型，这是很重要的。
可以通过{}初始化动态数组
``` cpp
void new_array()
{
    //通过{}初始化动态数组
    int *p_array = new int[10]{1, 2, 3, 4};
    //释放动态数组
    delete[] p_array;
}
```
如果{}初始化列表小于数组长度，则默认补充空值，int补充0，string补充空字符串。
动态分配一个大小为0的数组是合法的
``` cpp
void new_array()
{
    int n = 0;
    //开辟一个大小为0的数组
    int *p_array = new int[n];
    for (int *p = p_array; p != n + p_array; p++)
    {
        cout << *p << " ";
    }

    delete[] p_array;
}
```
当n为0时，开辟了一个长度为0的动态数组，因为循环条件p != n+p_array，所以不会进入循环。
当我们用new分配一个大小为0的数组时，new返回一个合法的非空指针。此指针保证与new返回的其他任何指针都不相同。
对于零长度的数组来说，此指针就像尾后指针一样，我们可以像使用尾后迭代器一样使用这个指针。
可以用此指针进行比较操作，就像上面循环代码中那样。可以向此指针加上（或从此指针减去）0，也可以从此指针减去自身从而得到0。但此指针不能解引用——毕竟它不指向任何元素。
## 智能指针和动态数组
标准库提供了一个可以管理new分配的数组的unique_ptr版本。为了用一个unique_ptr管理动态数组，我们必须在对象类型后面跟一对空方括号：
``` cpp
void unique_array()
{
    //开辟一个20个整形的动态数组，用unique_ptr管理它。
    auto unarray = unique_ptr<int[]>(new int[20]);
    //释放这个动态数组
    unarray.release();
}
```
类型说明符中的方括号`<int[]>`指出up指向一个int数组而不是一个int。由于unarray指向一个数组，当unarray销毁它管理的指针时，会自动使用delete[]。
当一个unique_ptr指向一个数组时，我们可以使用下标运算符来访问数组中的元素：
``` cpp
void unique_array()
{
    //开辟一个20个整形的动态数组，用unique_ptr管理它。
    auto unarray = unique_ptr<int[]>(new int[20]);

    //可以通过下标访问数组元素
    for (size_t i = 0; i < 10; i++)
    {
        unarray[i] = 1024;
    }

    //释放这个动态数组
    unarray.release();
}
```
shared_ptr也可以管理动态数组，这一点在C++ primer 第5版里没有提及，但是我自己测试好用
``` cpp
void shared_array()
{
    // 开辟一个5个整形的动态数组，用shared_ptr管理它
    auto sharray = shared_ptr<int[]>(new int[5]{1, 2, 3, 4, 5});
    for (int i = 0; i < 5; i++)
    {
        cout << sharray[i] << " ";
    }
    sharray.reset();
    cout << endl;
}
```
C++ primer 第5版推荐的用法如下
``` cpp
void use_shared_array()
{
    shared_ptr<int> sharray = shared_ptr<int>(new int[5], [](int *p)
                                              { delete[] p; });
    sharray.reset();
}
```
上例中，shared_ptr管理一个动态数组并提供了删除器。
## allocator类
当分配一大块内存时，我们通常计划在这块内存上按需构造对象。在此情况下，我们希望将内存分配和对象构造分离。这意味着我们可以分配大块内存，但只在真正需要时才真正执行对象创建操作（同时付出一定开销）。

类似vector，allocator是一个模板。为了定义一个allocator对象，我们必须指明这个allocator可以分配的对象类型。
当一个allocator对象分配内存时，它会根据给定的对象类型来确定恰当的内存大小和对齐位置：
``` cpp
void use_allocator()
{
    allocator<string> alloc;
    // allocator分配5个string类型对象的空间
    // 这些空间未构造
    auto const p = alloc.allocate(5);
    //销毁开辟的空间
    alloc.deallocate(p, 5);
}
```
上述代码用allocator构造alloc对象，说明开辟的空间是为string对象准备的，然后调用allocate开辟空间，但是这些空间不能直接使用，需要调用构造函数才能使用，我们用allocator类的construct来构造对象。
``` cpp
void use_allocator()
{
    allocator<string> alloc;
    // allocator分配5个string类型对象的空间
    // 这些空间未构造
    auto p = alloc.allocate(5);
    auto q = p;
    string str = "c";
    for (; q != p + 5; q++)
    {
        //构造字符串，每次字符串增加c字符
        alloc.construct(q, str);
        str += "c";
    }
    // //打印构造的字符串列表
    for (q = p; q != p + 5; q++)
    {
        cout << *q << endl;
    }

    //销毁开辟的空间
    alloc.deallocate(p, 5);
}
```
循环中通过construct为每个q指向的空间构造string对象，对象的内容就是str的内容，str会随着循环每次增加c，所以上面的代码输出如下
``` bash
c
cc
ccc
cccc
ccccc
```
另外stl也提供了一些拷贝和填充内存的算法
``` cpp
void use_allocator()
{
    vector<int> ivec = {1, 2, 3, 4, 5};
    allocator<int> alloc;
    //开辟2倍ivec大小的空间
    auto p = alloc.allocate(ivec.size() * 2);
    //将ivec的内容copy至alloc开辟的空间里
    //返回q指向剩余未构造的内存空间的起始地址
    auto q = uninitialized_copy(ivec.begin(), ivec.end(), p);
    //将剩余元素初始化为42
    uninitialized_fill_n(q, ivec.size(), 42);
}
```
通过uninitialized_copy将ivec元素拷贝到p指向的空间，同样完成了构造。
uninitialized_fill_n将剩余ivec大小未构造的空间全部初始化为42。
## 总结
本文介绍了动态数组开辟的方法，利用new关键字可以开辟动态数组，利用delete[]可以回收数组。
也实现了通过shared_ptr和unique_ptr等智能指针管理动态数组的方案。
最后通过列举allocator的一些方法，展示了如何实现开辟空间和构造对象分离的方式动态构造对象。
源码连接
[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
想系统学习更多C++知识,可点击下方链接。
[C++基础](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/23uSgIjfVfmwfhNGMprDFxL0uKL)