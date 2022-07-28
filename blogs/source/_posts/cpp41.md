---
title: C++ 一个例子说明类继承知识点
date: 2022-07-28 22:19:54
tags: C++
categories: C++
---
## 基类实现
我们先实现一个基类
``` cpp
class BaseTest
{
private:
    virtual void display() { cout << "Base display" << endl; }
    void say() { cout << "Base say()" << endl; }

public:
    virtual void func() { cout << "Base func()" << endl; }
    void exec()
    {
        display();
        say();
    }
    void f1(string a) { cout << "Base f1(string)" << endl; }
    void f1(int a) { cout << "Base f1(int)" << endl; }

    void exec2()
    {
        display();
        say();
    }
};
```
<!--more-->
BaseTest类中我们实现了一个虚函数display和 func。
BaseTest类内部重载了f1函数，实现了两个版本，一个参数为string一个参数为int。
同一个类中的多个同名函数叫做重载。
实现了普通函数say，exec以及exec2函数。exec和exec2函数内部调用了display和say函数。
## 子类实现

子类DeriveA继承了基类
``` cpp
class DeriveA : public BaseTest
{
public:
    void display() { cout << "DeriveA display()" << endl; }
    void f1(int a, int b) { cout << "DeriveA f1(int,int)" << endl; }
    void say() { cout << "DeriveA say()" << endl; }
    virtual void func() { cout << "DeriveA func()" << endl; }
    void use_base_f1(int a, int b)
    {
        BaseTest::f1(2);
        BaseTest::f1("test");
        cout << "DeriveA f1(int, int)" << endl;
    }
    void exec2()
    {
        display();
        say();
    }
};
```
子类DeriveA 子类重新实现了display和func函数，子类重新实现父类的虚函数，叫做重写。
同样子类重新实现了f1和say函数，由于父类有f1和say，所以子类重新实现覆盖了父类的函数，
这种普通函数被子类重写导致父类的函数被隐藏了，叫做覆盖。
## 函数调用
接下来我们通过函数调用，看一下覆盖，重载和重写的区别
``` cpp
void derive_base_test1()
{
    DeriveA a;
    BaseTest *b = &a;
    shared_ptr<BaseTest> c = make_shared<BaseTest>();
    //输出DeriveA func()
    b->func();
    //输出DeriveA func()
    a.func();
    //输出Base f1(string)
    b->f1("abc");
    //输出Base f1(int)
    b->f1(3);
    //输出DeriveA f1(int,int)
    a.f1(3, 5);

    a.use_base_f1(2, 4);
    cout << "========================" << endl;
    //输出DeriveA display()
    //输出Base say()
    b->exec();
    //输出DeriveA display()
    //输出Base say()
    a.exec();
    //输出Base display
    //输出Base say()
    c->exec();
    cout << "======================== \n"
         << endl;
    //输出 DeriveA display()
    //输出 Base say()
    b->exec2();
    //输出 DeriveA display()
    //输出 DeriveA say()
    a.exec2();
    //输出 Base display
    //输出 Base say()
    c->exec2();
}
```
代码里我们生成了一个DeriveA的实例a, 并将该实例返回给基类BaseTest的指针b，所以：
1   b->func();会根据多态的效果调用子类DeriveA的func函数
2   a.func() 因为a是一个对象，所以调用子类DeriveA的func函数
3   b->f1("abc")  调用基类BaseTest的f1，因为f1是一个普通函数
4   a.f1(3, 5) 调用DeriveA的f1，因为a是一个普通对象。
5   当我们想在子类里调用基类的f1函数，可以通过基类作用域加函数名的方式，比如例子中的
a.use_base_f1就在函数内部通过BaseTest::f1调用了基类函数f1
6   b->exec，首先b是一个指针且exec为普通函数只在基类实现了，所以调用基类的exec，
但是exec内部调用了虚函数display，此时触发多态机制调用DeriveA的display函数，因为b是一个指向子类DeriveA对象的基类BaseTest指针，exec内部调用了普通函数display，因为display不是虚函数，所以调用BaseTest的display函数
7   a.exec(); a是一个DeriveA对象，DeriveA自己没有实现exec函数，所以调用基类BaseTest的exec函数，exec内部调用display虚函数时由于DeriveA重写了display函数，所以调用DeriveA的display函数，exec内部调用say函数时由于say是普通函数，所以此时调用的是BaseTest的say函数。
8  c->exec(); 因为c为BaseTest类型，所以调用的就是BaseTest的exec，内部执行的也是BaseTest的display和say。
9  b->exec2(); 因为b是一个子类BaseTest的指针，所以调用BaseTest的exec2函数，exec2内部调用display时触发多态机制调用DeriveA的display，调用say时因为say是普通函数，所以调用BaseTest的say函数。
10  a.exec2(); 因为a是DeriveA类对象，且DeriveA实现了exec2，所以a调用DeriveA的exec2，这样exec2内部调用的都是DeriveA的say和display
11  c->exec2(); c为BaseTest类对象，所以调用BaseTest类的exec2以及display和say函数。
## 总结

考察一个函数是被子类还是基类调用时应该分以下几种情况
1  该函数是虚函数并且被子类重写，如果是基类指针指向子类对象，调用该函数则引发多态机制，调用子类的虚函数
2  如果该函数时虚函数并且没有被重写，那么无论调用的对象是基类指针还是子类对象，还是基类对象，
还是子类指针都是调用基类的这个虚函数
3  如果该函数不是虚函数，如果该函数被子类覆盖(子类重新定义了同名函数)，那么调用规则就是子类调用子类的该函数，
基类调用该基类的函数。
4  如果该函数不是虚函数，并且子类没有定义同名函数(没有覆盖基类同名函数)，那么无论是子类还是基类的指针或者对象，
统一调用的是基类的函数。
5  如果第4点里基类的函数(没有被子类覆盖)，但是内部调用了基类的虚函数，并且该虚函数被子类重写，这时内部这个虚函数调用规则
就要看调用对象的实际类型，符合1的调用标准，多态就走子类，不是多态就走基类(此时符合2标准)
6  如果第3点里基类的函数(被子类覆盖)，但是内部调用了基类的虚函数，并且该虚函数被子类重写，这时内部这个虚函数调用规则
就要看调用对象的实际类型，符合1的调用标准，多态就走子类，不是多态就走基类(此时符合2标准)

## 资源链接
本文模拟实现了vector的功能。
视频链接[https://www.bilibili.com/video/BV19S4y1J7kV/?vd_source=8be9e83424c2ed2c9b2a3ed1d01385e9](https://www.bilibili.com/video/BV19S4y1J7kV/?vd_source=8be9e83424c2ed2c9b2a3ed1d01385e9)
源码链接 [https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)