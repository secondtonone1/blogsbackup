---
title: C++单例模式的几种实现
date: 2022-03-20 15:40:09
tags: C++
categories: C++
---
本文介绍C++单例模式的集中实现方式，以及利弊
<!--more-->
## 局部静态变量方式
``` cpp
//通过静态成员变量实现单例
//懒汉式
class Single2
{
private:
    Single2()
    {
    }
    Single2(const Single2 &) = delete;
    Single2 &operator=(const Single2 &) = delete;

public:
    static Single2 &GetInst()
    {
        static Single2 single;
        return single;
    }
};
```
上述代码通过局部静态成员single实现单例类，原理就是函数的局部静态变量生命周期随着进程结束而结束。上述代码通过懒汉式的方式实现。
调用如下
``` cpp
void test_single2()
{
    //多线程情况下可能存在问题
    cout << "s1 addr is " << &Single2::GetInst() << endl;
    cout << "s2 addr is " << &Single2::GetInst() << endl;
}
```
程序输出如下
``` cmd
sp1  is  0x1304b10
sp2  is  0x1304b10
```
确实生成了唯一实例，上述单例模式存在隐患，对于多线程方式生成的实例可能时多个。
## 静态成员变量指针方式
可以定义一个类的静态成员变量，用来控制实现单例
``` cpp
//饿汉式
class Single2Hungry
{
private:
    Single2Hungry()
    {
    }
    Single2Hungry(const Single2Hungry &) = delete;
    Single2Hungry &operator=(const Single2Hungry &) = delete;

public:
    static Single2Hungry *GetInst()
    {
        if (single == nullptr)
        {
            single = new Single2Hungry();
        }
        return single;
    }

private:
    static Single2Hungry *single;
};
```
这么做的一个好处是我们可以通过饿汉式的方式避免线程安全问题
``` cpp
//饿汉式初始化
Single2Hungry *Single2Hungry::single = Single2Hungry::GetInst();
void thread_func_s2(int i)
{
    cout << "this is thread " << i << endl;
    cout << "inst is " << Single2Hungry::GetInst() << endl;
}
void test_single2hungry()
{
    cout << "s1 addr is " << Single2Hungry::GetInst() << endl;
    cout << "s2 addr is " << Single2Hungry::GetInst() << endl;

    for (int i = 0; i < 3; i++)
    {
        thread tid(thread_func_s2, i);
        tid.join();
    }
}
int main(){
    test_single2hungry()
}
```
程序输出如下
``` cmd
s1 addr is 0x1e4b00
s2 addr is 0x1e4b00
this is thread 0
inst is 0x1e4b00
this is thread 1
inst is 0x1e4b00
this is thread 2
inst is 0x1e4b00
```
可见无论单线程还是多线程模式下，通过静态成员变量的指针实现的单例类都是唯一的。饿汉式是在程序启动时就进行单例的初始化，这种方式也可以通过懒汉式调用，无论饿汉式还是懒汉式都存在一个问题，就是什么时候释放内存？多线程情况下，释放内存就很难了，还有二次释放内存的风险。
我们定义一个单例类并用懒汉式方式调用
``` cpp
//懒汉式指针
//即使创建指针类型也存在问题
class SinglePointer
{
private:
    SinglePointer()
    {
    }

    SinglePointer(const SinglePointer &) = delete;
    SinglePointer &operator=(const SinglePointer &) = delete;

public:
    static SinglePointer *GetInst()
    {
        if (single != nullptr)
        {
            return single;
        }

        s_mutex.lock();
        if (single != nullptr)
        {
            s_mutex.unlock();
            return single;
        }
        single = new SinglePointer();
        s_mutex.unlock();
        return single;
    }

private:
    static SinglePointer *single;
    static mutex s_mutex;
};
```
在cpp文件里初始化静态成员,并定义一个测试函数
``` cpp
//懒汉式
//在类的cpp文件定义static变量
SinglePointer *SinglePointer::single = nullptr;
std::mutex SinglePointer::s_mutex;

void thread_func_lazy(int i)
{
    cout << "this is lazy thread " << i << endl;
    cout << "inst is " << SinglePointer::GetInst() << endl;
}

void test_singlelazy()
{
    for (int i = 0; i < 3; i++)
    {
        thread tid(thread_func_lazy, i);
        tid.join();
    }

    //何时释放new的对象？造成内存泄漏
}

int main(){
    test_singlelazy();
}
```
函数输出如下
``` cpp
this is lazy thread 0
inst is 0xbc1700
this is lazy thread 1
inst is 0xbc1700
this is lazy thread 2
inst is 0xbc1700
```
此时生成的单例对象的内存空间还没回收，这是个问题，另外如果多线程情况下多次delete也会造成崩溃。
## 智能指针方式
可以利用智能指针自动回收内存的机制设计单例类
``` cpp
//利用智能指针解决释放问题
class SingleAuto
{

private:
    SingleAuto()
    {
    }

    SingleAuto(const SingleAuto &) = delete;
    SingleAuto &operator=(const SingleAuto &) = delete;

public:
    ~SingleAuto()
    {
        cout << "single auto delete success " << endl;
    }

    static std::shared_ptr<SingleAuto> GetInst()
    {
        if (single != nullptr)
        {
            return single;
        }

        s_mutex.lock();
        if (single != nullptr)
        {
            s_mutex.unlock();
            return single;
        }
        single = std::shared_ptr<SingleAuto>(new SingleAuto);
        s_mutex.unlock();
        return single;
    }

private:
    static std::shared_ptr<SingleAuto> single;
    static mutex s_mutex;
};
```
SingleAuto的GetInst返回std::shared_ptr<SingleAuto>类型的变量single。因为single是静态成员变量，所以会在进程结束时被回收。智能指针被回收时会调用内置指针类型的析构函数，从而完成内存的回收。
在主函数调用如下测试函数
``` cpp
// 智能指针方式
std::shared_ptr<SingleAuto> SingleAuto::single = nullptr;
mutex SingleAuto::s_mutex;
void test_singleauto()
{
    auto sp1 = SingleAuto::GetInst();
    auto sp2 = SingleAuto::GetInst();
    cout << "sp1  is  " << sp1 << endl;
    cout << "sp2  is  " << sp2 << endl;
    //此时存在隐患，可以手动删除裸指针，造成崩溃
    // delete sp1.get();
}
int main(){
    test_singleauto();
}
```
程序输出如下
``` cpp
sp1  is  0x1174f30
sp2  is  0x1174f30
```
智能指针方式不存在内存泄漏，但是有一个隐患就是单例类的析构函数时public的，如果被人手动调用会存在崩溃问题，比如将上边test_singleauto中的注释打开，程序会崩溃。
## 辅助类智能指针单例模式
智能指针在构造的时候可以指定删除器，所以可以传递一个辅助类或者辅助函数帮助智能指针回收内存时调用我们指定的析构函数。
``` cpp
// safe deletor
//防止外界delete
//声明辅助类
//该类定义仿函数调用SingleAutoSafe析构函数
//不可以提前声明SafeDeletor，编译时会提示incomplete type
// class SafeDeletor;
//所以要提前定义辅助类
class SingleAutoSafe;
class SafeDeletor
{
public:
    void operator()(SingleAutoSafe *sf)
    {
        cout << "this is safe deleter operator()" << endl;
        delete sf;
    }
};
class SingleAutoSafe
{
private:
    SingleAutoSafe() {}
    ~SingleAutoSafe()
    {
        cout << "this is single auto safe deletor" << endl;
    }
    SingleAutoSafe(const SingleAutoSafe &) = delete;
    SingleAutoSafe &operator=(const SingleAutoSafe &) = delete;
    //定义友元类，通过友元类调用该类析构函数
    friend class SafeDeletor;

public:
    static std::shared_ptr<SingleAutoSafe> GetInst()
    {
        if (single != nullptr)
        {
            return single;
        }

        s_mutex.lock();
        if (single != nullptr)
        {
            s_mutex.unlock();
            return single;
        }
        //额外指定删除器
        single = std::shared_ptr<SingleAutoSafe>(new SingleAutoSafe, SafeDeletor());
        //也可以指定删除函数
        // single = std::shared_ptr<SingleAutoSafe>(new SingleAutoSafe, SafeDelFunc);
        s_mutex.unlock();
        return single;
    }

private:
    static std::shared_ptr<SingleAutoSafe> single;
    static mutex s_mutex;
};
```
SafeDeletor要写在SingleAutoSafe上边，并且SafeDeletor要声明为SingleAutoSafe类的友元类，这样就可以访问SingleAutoSafe的析构函数了。
我们在构造single时制定了SafeDeletor(),single在回收时，会调用SingleAutoSafe的仿函数，从而完成内存的销毁。
并且SingleAutoSafe的析构函数为私有的无法被外界手动调用了。
``` cpp
//智能指针初始化为nullptr
std::shared_ptr<SingleAutoSafe> SingleAutoSafe::single = nullptr;
mutex SingleAutoSafe::s_mutex;

void test_singleautosafe()
{
    auto sp1 = SingleAutoSafe::GetInst();
    auto sp2 = SingleAutoSafe::GetInst();
    cout << "sp1  is  " << sp1 << endl;
    cout << "sp2  is  " << sp2 << endl;
    //此时无法访问析构函数，非常安全
    // delete sp1.get();
}

int main(){
    test_singleautosafe();
}
```
程序输出如下
``` cpp
sp1  is  0x1264f30
sp2  is  0x1264f30
```
通过辅助类调用单例类的析构函数保证了内存释放的安全性和唯一性。这种方式时生产中常用的。如果将test_singleautosafe函数的注释打开，手动delete sp1.get()编译阶段就会报错，达到了代码安全的目的。因为析构被设置为私有函数了。
## 通用的单例模板类
我们可以通过声明单例的模板类，然后继承这个单例模板类的所有类就是单例类了。达到泛型编程提高效率的目的。
``` cpp
template <typename T>
class Single_T
{
protected:
    Single_T() = default;
    Single_T(const Single_T<T> &st) = delete;
    Single_T &operator=(const Single_T<T> &st) = delete;
    ~Single_T()
    {
        cout << "this is auto safe template destruct" << endl;
    }

public:
    static std::shared_ptr<T> GetInst()
    {
        if (single != nullptr)
        {
            return single;
        }

        s_mutex.lock();
        if (single != nullptr)
        {
            s_mutex.unlock();
            return single;
        }
        //额外指定删除器
        single = std::shared_ptr<T>(new T, SafeDeletor_T<T>());
        //也可以指定删除函数
        // single = std::shared_ptr<SingleAutoSafe>(new SingleAutoSafe, SafeDelFunc);
        s_mutex.unlock();
        return single;
    }

private:
    static std::shared_ptr<T> single;
    static mutex s_mutex;
};

//模板类的static成员要放在h文件里初始化
template <typename T>
std::shared_ptr<T> Single_T<T>::single = nullptr;

template <typename T>
mutex Single_T<T>::s_mutex;
```
我们定义一个网络的单例类，继承上述模板类即可，并将构造和析构设置为私有，同时设置友元保证自己的析构和构造可以被友元类调用.
``` cpp
//通过继承方式实现网络模块单例
class SingleNet : public Single_T<SingleNet>
{
private:
    SingleNet() = default;
    SingleNet(const SingleNet &) = delete;
    SingleNet &operator=(const SingleNet &) = delete;
    ~SingleNet() = default;
    friend class SafeDeletor_T<SingleNet>;
    friend class Single_T<SingleNet>;
};
```
在主函数中调用如下
``` cpp
void test_singlenet()
{
    auto sp1 = SingleNet::GetInst();
    auto sp2 = SingleNet::GetInst();
    cout << "sp1  is  " << sp1 << endl;
    cout << "sp2  is  " << sp2 << endl;
}
```
程序输出如下
``` cmd
sp1  is  0x1164f30
sp2  is  0x1164f30
```
## 总结
本文介绍了一些面试常见问题
源码链接
[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
想系统学习更多C++知识,可点击下方链接。
[C++基础](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/24H7mZgZw57zCqGoULERZ2mQWOM)
