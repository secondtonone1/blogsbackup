---
title: C++ 拷贝构造 赋值 和析构
date: 2022-01-24 14:20:56
tags: C++
categories: C++
---
## 拷贝构造函数
一个类可以不定义拷贝构造函数，系统会默认提供一个拷贝构造函数，叫做合成拷贝构造函数。与默认构造函数不同的是，即使我们定义了其他构造函数，系统也会为我们生成合成拷贝构造函数。合成的拷贝构造函数会将其参数的成员逐个拷贝到正在创建的对象中。编译器从给定对象中依次将每个非static成员拷贝到正在创建的对象中。对类类型的成员，会使用其拷贝构造函数来拷贝；内置类型的成员则直接拷贝。
为了方便举例，我们手动实现一个mystring类
``` cpp
class mystring_
{
private:
    /* data */
public:
    mystring_(/* args */);
    mystring_(const mystring_ &mstr);
    mystring_(char *m_str);
    ~mystring_();

private:
    char *m_str;
};
```
<!--more-->
我们为mystring_类声明了一个无参构造函数，一个拷贝构造函数，一个有参构造函数以及一个析构函数。
我们实现这几个构造函数
``` cpp
mystring_::mystring_(/* args */) : m_str(nullptr)
{
}

mystring_::mystring_(const mystring_ &mystr)
{
    if (&mystr == this)
    {
        return;
    }
    size_t len = strlen(mystr.m_str);
    m_str = new char(len + 1);
    strcpy(m_str, mystr.m_str);
    m_str[len] = '\0';
}

mystring_::mystring_(char *mstr)
{
    size_t len = strlen(mstr);
    m_str = new char(len + 1);
    strcpy(m_str, mstr);
    m_str[len] = '\0';
}
```
定义了无参构造函数，无参构造函数里只将m_str赋值为空指针.
拷贝构造函数的参数是接受一个const mstring_ 类型的引用，内部判断是否是自己，防止循环拷贝。不是自己则将对方数据拷贝给自己。
有参构造函数接受一个char* 参数，利用字符串构造mystring_ 类。
我们实现一个函数测试一下构造函数是否生效
``` cpp
extern void use_mystr();
class mystring_
{
private:
    /* data */
public:
    mystring_(/* args */);
    mystring_(const mystring_ &mstr);
    mystring_(char *m_str);
    ~mystring_();
    friend ostream &operator<<(ostream &os, mystring_ &mystr1);

private:
    char *m_str;
};
```
我们完善了mystring_类的声明，新增了友元函数并重载`<<`运算符，输出mystring_类对象的内容。
增加了全局函数use_mystr()用来测试。
``` cpp
ostream &operator<<(ostream &os, mystring_ &mystr1)
{
    if (mystr1.m_str == nullptr)
    {
        os << "mystring_ data is null" << endl;
        return os;
    }
    os << "mystring_ data is " << mystr1.m_str << endl;
    return os;
}

void use_mystr()
{
    auto mystr1 = mystring_("hello zack");
    auto mystr2(mystr1);
    auto mystr3 = mystring_();
    cout << mystr1 << mystr2 << mystr3 << endl;
}
```
上述代码输出
``` cmd
mystring_ data is hello zack
mystring_ data is hello zack
mystring_ data is null
```
我们先用有参构造函数构造了mystr1，然后用拷贝构造函数构造了mystr2，最后用无参构造函数构造了mystr3。
## 拷贝初始化
当我们显示调用拷贝构造函数时会选择最合适的拷贝构造函数完成初始化，如上例中的mystr2(mystr1),我们称这种方式为直接初始化，调用拷贝构造函数完成直接初始化。
还有另一种情况，就是拷贝初始化，隐式调用了拷贝构造函数。
``` cpp
void use_mystr()
{
    //直接初始化
    mystring_ mystr1("hello zack");
    //直接初始化
    auto mystr2(mystr1);
    //拷贝初始化
    auto mystr3 = mystr2;
    //拷贝初始化
    mystring_ mystr4 = "hello world!";
    //拷贝初始化
    auto mystr5 = mystring_("hello everyone");
    cout << mystr1 << mystr2 << mystr3 << mystr4 << mystr5 << endl;
}
```
程序输出如下
``` cmd
mystring_ data is hello zack
mystring_ data is hello zack
mystring_ data is hello zack
mystring_ data is hello world!
mystring_ data is hello everyone
```
用构造函数显示指明参数构造生成的对象就是直接初始化，用赋值运算符隐式调用构造函数生成对象这种方式叫做拷贝初始化。

拷贝初始化不仅在我们用=定义变量时会发生，在下列情况下也会发生
· 将一个对象作为实参传递给一个非引用类型的形参
· 从一个返回类型为非引用类型的函数返回一个对象
· 用花括号列表初始化一个数组中的元素或一个聚合类中的成员
. 某些类类型还会对它们所分配的对象使用拷贝初始化。
例如，当我们初始化标准库容器或是调用其insert或push成员时，容器会对其元素进行拷贝初始化。与之相对，用emplace成员创建的元素都进行直接初始化。
## 重载赋值运算符
有时我们需要重载赋值运算符达到将一个对象赋值给另一个对象的目的。如果我们不重载赋值运算符，编译器会为我们生成一个合成拷贝赋值运算符，类似默认构造函数，系统默认提供的赋值操作，但是系统默认提供的赋值操作是浅拷贝，要实现数组，指针等数据的深拷贝，需要我们手动重载实现赋值运算符。
重载运算符本质上是函数，其名字由operator关键字后接表示要定义的运算符的符号组成。因此，赋值运算符就是一个名为operator=的函数。
为了与内置类型的赋值保持一致，赋值运算符通常返回一个指向其左侧运算对象的引用。
另外值得注意的是，标准库通常要求保存在容器中的类型要具有赋值运算符，且其返回值是左侧运算对象的引用。
``` cpp
class mystring_
{
private:
    /* data */
public:
    mystring_(/* args */);
    mystring_(const mystring_ &mstr);
    mystring_(char *m_str);
    ~mystring_();
    friend ostream &operator<<(ostream &os, mystring_ &mystr1);
    mystring_ &operator=(const mystring_ &mystr);

private:
    char *m_str;
};
```
我们再次完善类的声明，增加了operator=重载函数，该函数接受一个同类型的const mystring_ 引用对象，返回一个mystring_引用。
``` cmd
mystring_ &mystring_::operator=(const mystring_ &mystr)
{
    if (&mystr == this)
    {
        return *this;
    }

    size_t len = strlen(mystr.m_str);
    m_str = new char(len + 1);
    strcpy(m_str, mystr.m_str);
    m_str[len] = '\0';
}
```
同样需要判断是否为自赋值，防止进入自己赋值自己的死循环影响效率。如果不是自赋值，那就开辟空间，将m_str指向的数据拷贝过来，实现深拷贝。
## 析构函数
析构函数用来指明类对象销毁时需要进行的回收操作。析构函数是隐式调用的，如果我们不实现析构函数，系统也会为我们实现默认的析构函数。
如果一个类中有一个指针类型的成员，系统提供的默认的析构函数隐式销毁该类对象时并不会回收指针所指的空间。
比如我们上面的mystring_类包含m_str成员，如果我们不实现析构函数则m_str指向的空间不会被回收。
我们实现mystring_的析构函数
``` cpp
mystring_::~mystring_()
{
    if (m_str == nullptr)
    {
        return;
    }

    delete (m_str);
    m_str = nullptr;
}
```
如果我们定义了析构函数，那么理论上也需要实现拷贝构造和拷贝赋值操作。举个例子
``` cpp
class HasPtr
{
public:
    HasPtr(const string &str);
    ~HasPtr();

private:
    string *m_str;
};
```
上面给出了HasPtr的声明，包含一个有参构造函数和一个析构函数，接下来给出实现
``` cpp
HasPtr::HasPtr(const string &str) : m_str(new string(str)) {}

HasPtr::~HasPtr()
{
    if (m_str != nullptr)
    {
        delete m_str;
        m_str = nullptr;
    }
}
```
HasPtr必须要实现拷贝构造和拷贝赋值，否则会存在安全问题，看下面的例子
``` cpp
HasPtr f(HasPtr hp)
{
    HasPtr copyptr = hp;
    return copyptr;
}

void use_mystr()
{
    HasPtr ptr("hello zack!");
    f(ptr);
}
```
在主函数调用use_mystr,会引发崩溃。
因为函数f的形参hp在调用结束后会被析构，而f内部将hp赋值给copystr是浅拷贝，当f调用结束copystr也会调用析构函数，这样就会double free导致崩溃。
解决得办法就是为HasPtr实现拷贝赋值和拷贝构造，进行成员的深拷贝。
``` cpp
HasPtr::HasPtr(const HasPtr &hp)
{
    if (&hp != this)
    {
        this->m_str = new string(string(*hp.m_str));
    }

    return;
}

HasPtr &HasPtr::operator=(const HasPtr &hp)
{
    if (&hp != this)
    {
        this->m_str = new string(string(*hp.m_str));
    }

    return *this;
}
```
如果我们实现了拷贝构造函数，一般来说也要实现赋值运算符。比如上面的例子HasPtr新增一个int成员index，表示唯一标识。我们通过拷贝构造构建新对象时，要单独生成唯一的index。
我们先完善HasPtr声明
``` cpp
class HasPtr
{
public:
    HasPtr(const string &str);
    ~HasPtr();
    HasPtr(const HasPtr &hp);
    HasPtr &operator=(const HasPtr &hp);
    friend ostream &operator<<(ostream &os, const HasPtr &);

private:
    string *m_str;
    int m_index;
    static int _curnum;
};
```
我们新增了两个变量，一个m_index成员标识唯一索引，一个_curnum静态成员，用来自增生成唯一数字。
我们在cpp文件中初始化类的成员变量_curnum
``` cpp
int HasPtr::_curnum = 0;
```
然后修改几个构造函数
``` cpp
HasPtr::HasPtr(const string &str) : m_str(new string(str)), m_index(++_curnum)
{
    cout << "this is param constructor" << endl;
}

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
然后我们在重载赋值运算符的函数里增加输出信息,再重载一个输出运算符
``` cpp
HasPtr &HasPtr::operator=(const HasPtr &hp)
{
    cout << "this is operator = " << endl;
    if (&hp != this)
    {
        this->m_str = new string(string(*hp.m_str));
    }

    return *this;
}

ostream &operator<<(ostream &os, const HasPtr &hp)
{
    os << "index is " << hp.m_index << " , data is " << *(hp.m_str) << endl;
    return os;
}
```
在主函数中调用如下函数
``` cpp
void use_mystr()
{
    HasPtr hasptr1("hello zack");
    HasPtr hasptr2(hasptr1);
    HasPtr hasptr3 = hasptr2;
    HasPtr hasptr4("hello world");
    hasptr4 = hasptr3;
    cout << hasptr1 << hasptr2 << hasptr3 << hasptr4 << endl;
}
```
输出为
``` cmd
this is param constructor
index is 1 , data is hello zack

this is copy construtor
index is 2 , data is hello zack

this is copy construtor
index is 3 , data is hello zack

this is param constructor
this is operator =
index is 4 , data is hello zack
```
hasptr1用的是有参构造函数，输出index为1
hasptr2用的是拷贝构造函数, 输出index为2
hasptr3用的是拷贝构造函数，输出index为3
hasptr4用的是有参构造函数，赋值运算符。输出index为4，但是我们期望hasptr4的index为hasptr3的index值。
所以要重新实现赋值运算符，这也是我要强调的，一旦我们实现了拷贝构造，就要实现重载赋值运算，避免逻辑错误。
``` cpp
HasPtr &HasPtr::operator=(const HasPtr &hp)
{
    cout << "this is operator = " << endl;
    if (&hp != this)
    {
        this->m_str = new string(string(*hp.m_str));
        this->m_index = hp.m_index;
    }

    return *this;
}
```
再次打印就可以看到hasptr4的index为3了，因为通过赋值运算符我们将hasptr3的index赋值给hasptr4的index了。
我们可以通过default指定实现合成版本的构造函数
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

private:
    string *m_str;
    int m_index;
    static int _curnum;
};
```
## delete关键字
有时我们需要阻止拷贝，比如我们实现单例模式，阻止拷贝最好的方式就是使用delete关键字将拷贝构造函数和赋值运算符定义为删除函数。
我们先实现单例模式，类的声明写在singleton_.h中，如下
``` cpp
class Singleton_
{
public:
    Singleton_(const Singleton_ &) = delete;
    Singleton_ &operator=(const Singleton_ &) = delete;

    static shared_ptr<Singleton_> &getinstance()
    {
        //如果非空直接返回不加锁节省效率
        if (_inst != nullptr)
        {
            return _inst;
        }
        //最好做一个二次判断
        _mutex.lock();
        if (_inst != nullptr)
        {
            _mutex.unlock();
            return _inst;
        }
        _inst = shared_ptr<Singleton_>(new Singleton_());
        _mutex.unlock();
        return _inst;
    }

private:
    Singleton_() {}
    static shared_ptr<Singleton_> _inst;
    static mutex _mutex;
};
```
我们将Singleton_的拷贝构造函数和赋值操作声明为delete，这样就防止了拷贝和赋值操作。
我们将Singleton_的构造函数声明为私有，这样就可以避免外部显示调用构造函数。
要想创建Singleton_的对象必须调用getinstance函数，该函数内部先判断_inst是否为空指针，如果不是空指针则直接返回_inst即可。
这么做减少了加锁的开销。如果_inst为空，则加锁并进入构建_inst的逻辑，在构建之前又判断了一下_inst是否为空，不为空则直接返回。这么做主要是稳妥一点，防止在lock之前有其他线程已经创建了_inst，因为在第一次判断_inst是否为空和_mutex.lock()之间还是有一定时间空隙的。
我们构建_inst的方式是显示调用new Singleton_()来初始化智能指针，而不是
``` cpp
_inst = make_shared<Singleton_>();
```
因为make_shared会间接调用Singleton_的构造函数，而Singleton_构造函数是私有的，所以会报错。
所以显示调用new Singleton_()初始化智能指针，因为是在类的成员函数getinstance里调用，所以可以访问私有构造函数。
在类的声明里_inst和_mutex是类的静态变量，类的静态变量初始化是放在cpp文件中的，不然会出现重复定义的错误。
我们在singleton_.cpp文件中初始化这两个变量
``` cpp
shared_ptr<Singleton_> Singleton_::_inst = nullptr;
mutex Singleton_::_mutex;
```
我们先测试单线程情况下单例是否正常
``` cpp
void test_single()
{
    shared_ptr<Singleton_> inst1 = Singleton_::getinstance();
    cout << "inst1 get ptr is " << inst1.get() << endl;

    shared_ptr<Singleton_> inst2 = Singleton_::getinstance();
    cout << "inst2 get ptr is " << inst2.get() << endl;
}
```
程序输出
``` cmd
inst1 get ptr is 0xfa4150
inst2 get ptr is 0xfa4150
```
再测试下多线程的情况下
``` cpp
void thread_func(int i)
{
    cout << "this is thread " << i << endl;
    shared_ptr<Singleton_> inst = Singleton_::getinstance();
    cout << "inst ptr is " << inst.get() << endl;
}

void test_thread_single()
{
    for (int i = 0; i < 3; i++)
    {
        thread tid(thread_func, i + 1);
        tid.join();
    }
    cout << "main thread exit " << endl;
}
```
程序输出
``` cmd
this is thread 1
inst ptr is 0x10c4dc0
this is thread 2
inst ptr is 0x10c4dc0
this is thread 3
inst ptr is 0x10c4dc0
main thread exit
```
tid.join是让线程阻塞执行结束后再进行下一轮逻辑，所以会陆续输出线程id和指针地址。我们能看到无论多线程还是单线程，打印的inst的地址都是相同的，可见单例实现正确。
以后我们可以用模板实现一个更广范围的单例模式，这个例子只是为了说明将一些构造函数声明为delete的作用。
但是请记住，析构函数不能声明为delete，如果析构函数被删除，就无法销毁此类型的对象了。
对于类中包含const成员，引用成员的类，系统不会为其生成默认的拷贝构造和默认的拷贝赋值。
## 总结
本文介绍了拷贝构造函数，构造函数以及析构函数等用法，介绍了使用default和delete来管理构造函数等方式，最后利用C11智能指针实现了单例类
源码链接
[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
想系统学习更多C++知识,可点击下方链接。
[C++基础](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/23uSgIjfVfmwfhNGMprDFxL0uKL)