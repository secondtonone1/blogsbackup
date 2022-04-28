---
title: C++ 右值引用与移动构造函数
date: 2022-02-10 17:12:07
tags: C++
categories: C++
---
## 右值与右值引用
不能修改的值就是右值，右值一般为临时变量。常见的右值有字面常量值，返回右值的表达式。
所谓右值引用就是必须绑定到右值的引用。我们通过&&来获得右值引用。
右值引用有一个重要的性质——只能绑定到一个将要销毁的对象。
因此，我们可以自由地将一个右值引用的资源“移动”到另一个对象中。
<!--more-->
``` cpp
void right_references()
{
    int i = 42;
    //r 为i的引用，左值引用
    int &r = i;
    //rr 不可以引用左值i，
    //因为其是右值引用
    //int &&rr = i;
    //表达式是一个右值
    //不能用左值引用
    //int &r2 = i*42;
    //可以将const引用绑定到右值上
    const int &r3 = i * 42;
    //将rr2绑定到乘法结果上
    int &&rr2 = i * 42;
}
```
上述代码右值引用只能捕获右值，左值引用只能捕获左值。const 引用可以捕获右值。
左值更长久，右值是短暂的。
``` cpp
void right_references()
{
    //右值引用捕获字面常量值
    int &&rr1 = 42;
    //右值引用不能捕获左值
    //因为rr1为左值，下面操作非法
    //int &&rr2 = rr1;
}
```
变量是左值，因此我们不能将一个右值引用直接绑定到一个变量上，即使这个变量是右值引用类型也不行。
## move操作
可以通过move操作将一个左值转化为右值引用。
``` cpp
void right_references()
{
    //右值引用捕获字面常量值
    int &&rr1 = 42;
    //通过move将左值转化为右值引用
    int &&rr3 = std::move(rr1);
}
```
move调用告诉编译器：我们有一个左值，但我们希望像一个右值一样处理它。
我们必须认识到，调用move就意味着承诺：除了对rr1赋值或销毁它外，我们将不再使用它。
在调用move之后，我们不能对移后源对象的值做任何假设。
我们可以销毁一个移后源对象，也可以赋予它新值，但不能使用一个移后源对象的值。
## 移动构造函数
在一些场景，我们不仅需要使用拷贝构造函数，还需要移动构造函数，移动构造函数减少拷贝带来的开销。
类似拷贝构造函数，移动构造函数的第一个参数是该类类型的一个引用。
不同于拷贝构造函数的是，这个引用参数在移动构造函数中是一个右值引用。
与拷贝构造函数一样，任何额外的参数都必须有默认实参。
完善之前我们实现的strvec类，实现一个移动构造函数
``` cpp
//移动构造函数
//声明noexcept就是不抛出异常
StrVec(StrVec &&src) noexcept : elements(src.elements), first_free(src.first_free), cap(src.cap)
{
    //将src的成员设置为空
    src.elements = src.first_free = src.cap = nullptr;
}
```
与拷贝构造函数不同，移动构造函数不分配任何新内存；它接管给定的StrVec中的内存。
在接管内存之后，它将给定对象中的指针都置为nullptr。
这样就完成了从给定对象的移动操作，此对象将继续存在。
最终，移后源对象会被销毁，意味着将在其上运行析构函数。
移动构造函数通常不会被标准库调用，因为移动构造函数可能抛出异常，导致移动过程中数据出现问题。
比如vector，我们调用push_back时，通常vector会利用拷贝构造函数完成数据的移动而不是利用移动构造函数。
这么做是为了保证移动的过程中出现异常后，清除新空间的数据，保留旧数据。
说白了就是怕vector调用push_back添加元素时扩容，将旧数据移动到新空间时崩溃造成旧数据丢失。
除非我们通过noexcept关键字通知标准库我们这个类的移动构造函数不会抛出异常，这样vector才会优先采用移动构造函数。
移动操作对移后源对象中留下的值没有任何要求。因此，我们的程序不应该依赖于移后源对象中的数据。

如果一个类定义了自己的拷贝构造函数、拷贝赋值运算符或者析构函数，编译器就不会为它合成移动构造函数和移动赋值运算符了。
如果没定义析构函数，则编译器不会为该类定义合成的移动构造函数，因为使用移动构造函数的前提时编译器知道怎么回收对象。
当一个类既有移动构造函数又有拷贝构造函数的时候，编译器会根据条件选择最合适的。
完善之前的strvec类，实现移动赋值运算符
``` cpp
//移动赋值运算符
StrVec &StrVec::operator=(StrVec &&src)
{
    cout << "this is move operator = " << endl;
    if (this != &src)
    {
        //释放自己的空间操作
        this->free();
        //接管源对象资源
        this->elements = src.elements;
        this->first_free = src.first_free;
        this->cap = src.cap;
        //将源对象成员赋值为空
        src.elements = src.first_free = src.cap = nullptr;
    }

    return *this;
}
```
如果一个类有一个可用的拷贝构造函数而没有移动构造函数，则其对象是通过拷贝构造函数来“移动”的。
拷贝赋值运算符和移动赋值运算符的情况类似。
建议定义拷贝构造函数，拷贝赋值运算符以及析构函数后，在定义移动构造和移动赋值，交给编译器自动调用合理的赋值和构造函数。
之前实现过消息和文件夹的两个类，现在为Message类增加移动构造函数和移动赋值函数，在此之前先实现工具函数帮助我们将Message类的文件夹信息
移动给另一个Message类对象
``` cpp
   //将m的Folders交接给本类对象
    //并且实现Folders和本类对象的关联
    //接触Folders和m的关联
void Message::move_Folders(Message *m)
{
    //将m的Folders交接给本对象
    this->folders = m->folders;
    //将本对象和folders关联
    for (auto f : folders)
    {
        //解除folders和m的关联
        f->remMsg(*m);
        //添加本对象和folders的关联
        f->addMsg(*this);
    }
    //清除m的folders
    m->folders.clear();
}

//利用move_Folders函数实现移动构造函数
Message::Message(Message &&m) : contents(std::move(m.contents))
{
    move_Folders(&m);
}

Message &Message::operator=(Message &&m)
{
    if (this != &m)
    {
        remove_from_Folders();
        contents = std::move(m.contents);
        move_Folders(&m);
    }
    return *this;
}

void Message::printMsg()
{
    cout << "contents is " << contents << endl;ss
}
```
接下来在主函数中调用如下函数。
``` cpp
    auto msg4 = new Message("msg4");
    msg4->printMsg();
    auto msg3 = new Message("msg3");
    msg3->printMsg();
    *msg4 = std::move(*msg3);
    msg4->printMsg();
    delete msg4;
    delete msg3;
```

可以看到在调用move之后msg4输出的内容变为msg3，我们的移动操作是成功的。
## 总结
本文通过移动操作移动赋值实现了移动构造函数，以及其注意事项。
源码链接
[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
想系统学习更多C++知识,可点击下方链接。
[C++基础](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/24H7mZgZw57zCqGoULERZ2mQWOM)
