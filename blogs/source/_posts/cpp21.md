---
title: 智能指针shared_ptr
date: 2022-01-17 15:40:22
tags: C++
categories: C++
---
## 指针
C++提供了对指针操作的方法，当我们用new开辟指定类型的空间后，就生成了一个指针。
``` cpp
void use_pointer()
{
    //开辟整形指针，指向一个值为5的元素
    int *pint = new int(5);
    //开辟指向字符串的指针
    string *pstr = new string("hello zack");
}
```
<!--more-->
通过new + 类型构造的方式可以生成指针对象，但是开辟的指针对象所占用的空间在堆空间上。需要手动回收。
可以通过delete 指针对象的方式回收
``` cpp
void use_pointer()
{
    //开辟整形指针，指向一个值为5的元素
    int *pint = new int(5);
    //开辟指向字符串的指针
    string *pstr = new string("hello zack");
    //释放pint指向的空间
    if (pint != nullptr)
    {
        delete pint;
        pint = nullptr;
    }
    //释放指针指向的空间。
    if (pstr != nullptr)
    {
        delete pstr;
        pstr = nullptr;
    }
}
```
通过delete 指针对象回收其指向的堆空间。为了防止double free，所以将释放后的对象分别置为nullptr。
指针存在很多隐患:
1  当一个函数返回局部变量的指针时，外部使用该指针可能会造成崩溃或逻辑错误。因为局部变量随着函数的右}释放了。
2  如果多个指针指向同一个堆空间，其中一个释放了堆空间，使用其他的指针时会造成崩溃。
3  对一个指针多次delete，会造成double free问题。
4  两个类对象A和B，分别包含对方类型的指针成员，互相引用时如何释放是个问题。

所以C++提出了智能指针的用法，可以解决上述隐患。
shared_ptr允许多个指针指向同一个对象；
unique_ptr则“独占”所指向的对象。
标准库还定义了一个名为weak_ptr的伴随类，它是一种弱引用，指向shared_ptr所管理的对象。
这三种类型都定义在memory头文件中。
``` cpp
    //我们定义一个指向整形5得指针
    auto psint2 = make_shared<int>(5);
    //判断智能指针是否为空
    if (psint2 != nullptr)
    {
        cout << "psint2 is " << *psint2 << endl;
    }

    auto psstr2 = make_shared<string>("hello zack");
    if (psstr2 != nullptr && !psstr2->empty())
    {
        cout << "psstr2 is " << *psstr2 << endl;
    }
```
对于智能指针得使用和普通的内置指针没什么区别，通过判断指针是否为nullptr可以判断是否为空指针。
通过`->`可以取指针内部得成员方法或者成员变量。
make_shared函数将参数为对象类型的构造函数的参数，将此参数传递给模板中得对象类型的构造函数，从而构造出对象类型得智能指针，节省了对象在函数传递得开销。
当我们需要获取内置类型时，可以通过智能指针的方法get()返回其底层的内置指针。
``` cpp
    int *pint = psint2.get();
    cout << "*pint  is " << *pint << endl;
```
不要手动回收智能指针get返回的内置指针，要交给智能指针自己回收即可，否则会造成double free或者 使用智能指针产生崩溃等问题。
也不要用get()返回得内置指针初始化另一个智能指针，因为两个智能指针引用一个内置指针会出现问题，比如一个释放了另一个不知道就会导致崩溃等问题。
shared_ptr会根据引用计数管理内置指针，当引用计数为0时就自动删除内置指针。
当将一个智能指针p赋值给另一个智能指针q时，p引用计数就-1，q引用计数就+1
``` cpp
void use_sharedptr()
{
    //我们定义一个指向整形5得指针
    auto psint2 = make_shared<int>(5);
    auto psstr2 = make_shared<string>("hello zack");
    //将psint2赋值给psint3,他们底层的内置指针相同
    // psint3和psint2引用计数相同，引用计数+1，都为2
    shared_ptr<int> psint3 = psint2;
    //打印引用计数
    cout << "psint2 usecount is " << psint2.use_count() << endl;
    cout << "psint3 usecount is " << psint3.use_count() << endl;
    // psint3引用计数为1
    psint3 = make_shared<int>(1024);
    // psint2引用计数-1，变为1
    //打印引用计数
    cout << "psint2 usecount is " << psint2.use_count() << endl;
    cout << "psint3 usecount is " << psint3.use_count() << endl;
}
```
程序输出
``` cmd
psint2 usecount is 2
psint3 usecount is 2
psint2 usecount is 1
psint3 usecount is 1
```
可以利用shared_ptr实现数据共享，我们定义一个StrBlob类，这个类仅又一个成员shared_ptr成员，用来管理vector,记录有多少个StrBlob类对象使用vector，当所有的StrBlob销毁时，vector自动回收。
``` cpp
class StrBlob
{
public:
    //定义类型
    typedef std::vector<string>::size_type size_type;
    StrBlob();
    //通过初始化列表构造
    StrBlob(const initializer_list<string> &li);
    //返回vector大小
    size_type size() const { return data->size(); }
    //判断vector是否为空
    bool empty()
    {
        return data->empty();
    }
    //向vector写入元素
    void push_back(const string &s)
    {
        data->push_back(s);
    }

    //从vector弹出元素
    void pop_back();
    //访问头元素
    std::string &front();
    //访问尾元素
    std::string &back();

private:
    shared_ptr<vector<string>> data;
};
```
因为StrBlob未重载赋值运算符，也没有实现拷贝构造函数，所以StrBlob对象之间的赋值就是浅copy，因而内部成员data会随着StrBlob对象的赋值修改引用计数，默认情况下，当我们拷贝、赋值或销毁一个StrBlob对象时，它的shared_ptr成员会被拷贝、赋值或销毁。
当然我们也可以实现拷贝构造和赋值操作，让大家更好的理解智能指针随着类对象赋值等操作达到共享的效果。
运算符重载之后介绍，为了让程序更完善，这里给出拷贝构造和运算符重载的完整类声明。
``` cpp
class StrBlob
{
public:
    //定义类型
    typedef std::vector<string>::size_type size_type;
    StrBlob();
    //通过初始化列表构造
    StrBlob(const initializer_list<string> &li);
    //拷贝构造函数
    StrBlob(const StrBlob &sb);
    StrBlob &operator=(const StrBlob &sb);

    //返回vector大小
    size_type size() const { return data->size(); }
    //判断vector是否为空
    bool empty()
    {
        return data->empty();
    }
    //向vector写入元素
    void push_back(const string &s)
    {
        data->push_back(s);
    }

    //从vector弹出元素
    void pop_back();
    //访问头元素
    std::string &front();
    //访问尾元素
    std::string &back();

private:
    shared_ptr<vector<string>> data;
    //检测i是否越界
    void check(size_type i, const string &msg) const;
};
```
接下来实现三个构造函数
``` cpp
StrBlob::StrBlob()
{
    data = make_shared<vector<string>>();
}

StrBlob::StrBlob(const StrBlob &sb)
{
    data = sb.data;
}

StrBlob::StrBlob(const initializer_list<string> &li)
{
    data = make_shared<vector<string>>(li);
}
```
默认构造函数初始化data指向了一个空的vector，拷贝构造函数将sb的data赋值给自己，初始化列表方式的构造函数是用初始化列表构造data。接下来实现赋值运算符的重载
``` cpp
StrBlob &StrBlob::operator=(const StrBlob &sb)
{
    if (&sb != this)
    {
        this->data = sb.data;
    }

    return *this;
}
```
将sb的data赋值给this->data，这样this->data和sb.data引用计数相同。
我们实现检查越界的函数
``` cpp
//检测i是否越界
void StrBlob::check(size_type i, const string &msg) const
{
    if (i >= data->size())
    {
        throw out_of_range(msg);
    }
}
```
接下来实现front
``` cpp
string &StrBlob::front()
{
    //不要返回局部变量的引用
    // if (data->size() <= 0)
    // {
    //     return string("");
    // }
    // 1 可以用一个局部变量返回异常情况
    if (data->size() <= 0)
    {
        return badvalue;
    }
    return data->front();
}
```
要考虑队列为空的情况，此时返回空字符串。但是如果我们直接构造一个空字符串返回，这样就返回了局部变量的引用，局部变量会随着函数结束而释放，造成安全隐患。所以我们可以返回类的成员变量badvalue，作为队列为空的标记。当然如果不能容忍队列为空的情况，可以通过抛出异常来处理，那我们用这种方式改写front
``` cpp
string &StrBlob::front()
{
    check(0, "front on empty StrBlob");
    return data->front();
}
```
同样我们实现back()和pop_back()
``` cpp
string &StrBlob::back()
{
    check(0, "back on empty StrBlog");
    return data->back();
}

void StrBlob::pop_back()
{
    check(0, "back on pop_back StrBlog");
    data->pop_back();
}
```
这样我们通过定义StrBlob类，达到共享vector<string>的方式。多个StrBlob操作的是一个vector<string>向量。
我们新增一个打印shared_ptr引用计数的方法
``` cpp
void StrBlob::printCount()
{
    cout << "shared_ptr use count is " << data.use_count() << endl;
}
```
下面测试以下
``` cpp
void test_StrBlob()
{
    StrBlob strblob1({"hello", "zack", "good luck"});
    StrBlob strblob2;
    try
    {
        auto str2front = strblob2.front();
    }
    catch (std::out_of_range &exc)
    {
        cout << exc.what() << endl;
    }
    catch (...)
    {
        cout << "unknown exception" << endl;
    }

    strblob2 = strblob1;
    auto str1front = strblob1.front();
    cout << "strblob1 front is " << str1front << endl;

    strblob2.printCount();
    strblob1.printCount();
}
```
程序输出
``` cmd
front on empty StrBlob
strblob1 front is hello
shared_ptr use count is 2
shared_ptr use count is 2
```
因为strblob2的队列为空，所以会抛出异常，当执行strblob2 = strblob1之后，strblob2和strblob1的data的引用计数相同都为2。
## shared_ptr和new结合
之前的方式都是通过make_shared<类型>(构造函数列表参数)的方式构造的shared_ptr，也可以通过new 生成的内置指针初始化生成shared_ptr。
``` cpp
    auto psint = shared_ptr<int>(new int(5));
    auto psstr = shared_ptr<string>(new string("hello zack"));
```
接受指针参数的智能指针构造函数是explicit的。因此，我们不能将一个内置指针隐式转换为一个智能指针，必须使用直接初始化形式来初始化一个智能指针：
``` cpp
    //错误，不能用内置指针隐式初始化shared_ptr
    // shared_ptr<int> psint2 = new int(5);
    //正确，显示初始化
    shared_ptr<string> psstr2(new string("good luck"));
```
除了智能指针之间的赋值，可以通过一个智能指针构造另一个
``` cpp
    shared_ptr<string> psstr2(new string("good luck"));
    //可以通过一个shared_ptr 构造另一个shared_ptr
    shared_ptr<string> psstr3(psstr2);
    cout << "psstr2 use count is " << psstr2.use_count() << endl;
    cout << "psstr3 use count is " << psstr3.use_count() << endl;
```
程序输出
``` cmd
psstr2 use count is 2
psstr3 use count is 2
```
通过一个指针构造另一个智能指针，两个指针共享底层内置指针，所以引用计数为2.
在构造智能指针的同时，可以指定自定义的删除方法替代shared_ptr自己的delete操作
``` cpp
    //可以设置新的删除函数替代delete
    shared_ptr<string> psstr4(new string("good luck for zack"), delfunc);
```
我们为psstr4指定了delfunc删除函数，这样当psstr4被释放时就会执行delfunc函数，而不是delete操作。
``` cpp
void delfunc(string *p)
{
    if (p != nullptr)
    {
        delete (p);
        p = nullptr;
    }

    cout << "self delete" << endl;
}
```
我们实现了自己的delfunc函数作为删除器，回收了内置指针，并且打印了删除信息。这样当psstr4执行析构时，会打印"self delete"。
推荐使用make_shared的方式构造智能指针。
如果通过内置指针初始化生成智能指针，那一定要记住不要手动回收内置指针。
当将一个shared_ptr绑定到一个普通指针时，我们就将内存的管理责任交给了这个shared_ptr。
一旦这样做了，我们就不应该再使用内置指针来访问shared_ptr所指向的内存了。
以下代码存在问题
``` cpp
void process(shared_ptr<int> psint)
{
    cout << "psint data is " << *psint << endl;
}

int main()
{
    int *p = new int(5);
    process(shared_ptr<int>(p));
    //危险，p已经被释放，会造成崩溃或者逻辑错误
    cout << "p data is " << *p << endl;
    return 0;
}
```
程序输出
``` cmd
psint data is 5
p data is 10569024
```
因为p构造为shared_ptr，那么它的回收就交给了shared_ptr，而shared_ptr是process的形参，形参在process运行结束会释放，那么p也被回收，之后再访问p会产生逻辑错误，所以打印了一个非法内存的数值。

智能指针类型定义了一个名为get的函数，它返回一个内置指针，指向智能指针管理的对象。
此函数是为了这样一种情况而设计的：我们需要向不能使用智能指针的代码传递一个内置指针。
使用get返回的指针的代码不能delete此指针。
``` cpp
void bad_use_sharedptr()
{
    shared_ptr<int> p(new int(5));
    //通过p获取内置指针q
    //注意q此时被p绑定，不要手动delete q
    int *q = p.get();
    {
        //两个独立的shared_ptr m和p都绑定q
        auto m = shared_ptr<int>(q);
    }

    //上述}结束则m被回收，其绑定的q也被回收
    //此时使用q是非法操作，崩溃或者逻辑错误
    cout << "q data is " << *q << endl;
}
```
上述代码虽然没有手动delete q但是，两个独立的shared_ptr m和p都绑定了q，导致其中一个m被回收时q的内存也回收所以之后访问*q会出现崩溃或者数据异常。
注意，以下代码和上面是不同的，m和p此时共享q,并且引用计数是共享同步的。
``` cpp
void good_use_sharedptr()
{
    shared_ptr<int> p(new int(5));
    //通过p获取内置指针q
    //注意q此时被p绑定，不要手动delete q
    int *q = p.get();
    {
        // m和p的引用计数都为2
        shared_ptr<int> m(p);
    }

    //上述}结束则m被回收，其绑定的q也被回收
    //此时使用q是非法操作，崩溃或者逻辑错误
    cout << "q data is " << *q << endl;
}
```
所以总结以下：
get用来将指针的访问权限传递给代码，你只有在确定代码不会delete指针的情况下，才能使用get。
特别是，永远不要用get初始化另一个智能指针或者为另一个智能指针赋值。
## reset
reset的功能是为shared_ptr重新开辟一块新的内存，让shared_ptr绑定这块内存
``` cpp
    shared_ptr<int> p(new int(5));
    // p重新绑定新的内置指针
    p.reset(new int(6));
```
上述代码为p重新绑定了新的内存空间。
reset常用的情况是判断智能指针是否独占内存，如果引用计数为1，也就是自己独占内存就去修改，否则就为智能指针绑定一块新的内存进行修改，防止多个智能指针共享一块内存，一个智能指针修改内存导致其他智能指针受影响。
``` cpp
    //如果引用计数为1，unique返回true
    if (!p.unique())
    {
        //还有其他人引用，所以我们为p指向新的内存
        p.reset(new int(6));
    }

    // p目前是唯一用户
    *p = 1024;
```
使用智能指针的另一个好处，就是当程序一场崩溃时，智能指针也能保证内存空间被回收
``` cpp
void execption_shared()
{
    shared_ptr<string> p(new string("hello zack"));
    //此处导致异常
    int m = 5 / 0;
    //即使崩溃也会保证p被回收
}
```
即使运行到 m = 5 / 0处，程序崩溃，智能指针p也会被回收。
有时候我们传递个智能指针的指针不是new分配的，那就需要我们自己给他传递一个删除器
``` cpp
void delfuncint(int *p)
{
    cout << *p << " in del func" << endl;
}

void delfunc_shared()
{
    int p = 6;
    shared_ptr<int> psh(&p, delfuncint);
}
```
如果不传递delfuncint，会造成p被智能指针delete，因为p是栈空间的变量，用delete会导致崩溃。
## 总结
智能指针陷阱智能指针可以提供对动态分配的内存安全而又方便的管理，但这建立在正确使用的前提下。
为了正确使用智能指针，我们必须坚持一些基本规范：
· 不使用相同的内置指针值初始化（或reset）多个智能指针。
· 不delete get（）返回的指针。
· 不使用get（）初始化或reset另一个智能指针。
· 如果你使用get（）返回的指针，记住当最后一个对应的智能指针销毁后，你的指针就变为无效了。
· 如果你使用智能指针管理的资源不是new分配的内存，记住传递给它一个删除器。

源码连接
[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)