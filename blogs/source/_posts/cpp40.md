---
title: 模板特例化
date: 2022-07-24 15:07:17
tags: C++
categories: C++
---
## 特例化介绍
模板特例化主要是用于在模板特定情况下的一些特殊定义，用来完善模板在特定情况的调用
我们先实现一个函数模板
<!--more-->
``` cpp
template <typename T>
int compare(const T &v1, const T &v2)
{
    cout << "use compare T&" << endl;
    if (v1 < v2)
        return -1;
    if (v2 < v1)
        return 1;
    return 0;
}
```
接下来我们实现一个带字面值常量的特例化版本
``` cpp
//带字面常量的比较函数
template <size_t N, size_t M>
int compare(const char (&a1)[N], const char (&a2)[M])
{
    cout << "use const char (&)[N]" << endl;
    strcmp(a1, a2);
}
```
我们实现一个testcompare函数测试普通版和特例话版本的函数调用
``` cpp
void testcompare()
{
    const char *p1 = "h1";
    const char *p2 = "mom";
    //调用特例化版本
    compare(p1, p2);
    //调用第二个版本
    compare("hi", "mom");
}
```
当调用特例化版本时，N会被设定为"h1"的长度，M会被设定为"mom"长度。
但是我们发现使用通用模板类型的函数compare在被叫指针p1和p2时不是很理想，可以单独实现针对p1和p2指针特定版的模板函数
``` cpp
template <>
int compare(const char* &v1, const char* &v2)
{
    cout << "use compare char * " << endl;
    if (strlen(v1) < strlen(v2))
        return -1;
    else if (strlen(v2) < strlen(v1))
        return 1;
    else 
        return strcmp(v1, v2);
}
```
对于char * 版本我们实现了自己的比较规则，如果长度长的那个就是大值，相等则依次比较字符串中的每个字符。
## 类模板
类模板的使用和函数模板类似，我们先声明两个模板类，然后为模板类声明一个比较函数重载运算符
``` cpp
template <typename>
class BlobPtr;

template <typename>
class Blob;

template <typename T>
bool operator==(const Blob<T> &, const Blob<T> &);
```
我们实现`Blob<T>`模板类
``` cpp
//定义模板类型的blob
template <typename T>
class Blob
{
public:
    typedef T value_type;
    typedef typename std::vector<T>::size_type size_type;
    // T类型的BlobPtr是T类型的Blob的友元类
    friend class BlobPtr<T>;
    //重载==运算符
    friend bool operator==(const Blob<T> &, const Blob<T> &);
    //构造函数
    Blob()
    {
        data = make_shared<std::vector<T>>();
    }
    Blob(std::initializer_list<T> il)
    {
        data = make_shared<std::vector<T>>(il);
    }

    template <typename It>
    Blob(It b, It e);
    // Blob 中元素数目
    size_type size() const { return data->size(); }
    bool empty() const { return data->empty(); }
    //添加和删除元素
    void push_back(const T &t) { data->push_back(t); }
    //移动版本的push_back
    void push_back(const T &&t) { data->push_back(std::move(t)); }
    //删除元素
    void pop_back();
    //元素访问
    T &back();
    T &operator[](size_type i);

private:
    std::shared_ptr<std::vector<T>> data;
    //校验数据是否有效
    void check(size_type i, const std::string &msg) const;
};
```
接下来实现Blob模板类的几个成员函数
``` cpp
template <typename T>
void Blob<T>::check(size_type i, const std::string &msg) const
{
    if (i >= data->size())
        throw std::out_of_range(msg);
}

template <typename T>
void Blob<T>::pop_back()
{
    if (data->empty())
    {
        return;
    }
    data->pop_back();
}

template <typename T>
T &Blob<T>::back()
{
    return data->back();
}

template <typename T>
T &Blob<T>::operator[](size_type i)
{
    check(i, "index out of range");
    return (*data)[i];
}
```
实现了pop_back, back, check等操作，以及下标索引等函数，接下来实现比较运算符重载
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
此时我们还没有实现迭代器版本的构造函数，与类模板的普通成员函数不同，成员函数有自己的模板，所以要写两个模板名
``` cpp
//与模板类的普通成员不同，成员模板是函数模板
//模板类的T类型
template <typename T>
//成员函数模板It类型
template <typename It>
Blob<T>::Blob(It b, It e)
{
    //通过迭代器构造
    data = std::make_shared<std::vector<T>>(b, e);
}
```
这样我们就可以通不同类型的vector初始化Blob的构造函数了
``` cpp
void use_tempmemfunc()
{
    int ia[] = {0, 1, 2, 3, 4};
    vector<long> vi = {7, 6, 5, 4};
    list<const char *> w = {"now", "zack", "lov u"};
    // Blob<T> T被实例化为int，
    //函数模板It被实例化为 int *
    Blob<int> a1(begin(ia), end(ia));
    // It为vi的迭代器类型vector<long>::iterator T为long类型
    Blob<long> a2(vi.begin(), vi.end());
    //实例化Blob<string>以及list<const char *>::iterator参数
    Blob<string> a3(w.begin(), w.end());
}
```
接下来我们实现BlobPtr这个模板类
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
BlobPtr实现了根据Blob构造自己的成员wptr以及curr，因为wptr是一个弱指针，所以只做弱关联。
接下来我们实现前置++和后置++
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
前置++很容易理解，后置++理解较为困难，这里做一下说明，后置++的函数里先用一个BlobPtr引用类型的临时变量rt存储了*this，因为this不会被释放，所以rt就是`*this`的引用，所引用的内容不会释放，这样外界接受到rt后同样是引用的`*this`。这样即使rt被回收了也没关系，因为外部已经捕获到`*this`的引用了。然后对curr++操作，这就是我们看到的先返回`*this`的引用，后++。
模板类没有实现拷贝赋值时，默认用拷贝构造完成构造初始化
``` cpp
void use_classtemp()
{
    Blob<int> ia;
    Blob<int> ia2 = {0, 1, 2, 3, 5};
    Blob<string> ia3 = {"hello ", "zack", "nice"};
    for (size_t i = 0; i < ia2.size(); i++)
    {
        ia2[i] = i * i;
    }

    for (size_t i = 0; i < ia2.size(); i++)
    {
        cout << ia2[i] << endl;
    }

    for (size_t i = 0; i < ia3.size(); i++)
    {
        string_upper(ia3[i]);
    }

    for (size_t i = 0; i < ia3.size(); i++)
    {
        cout << ia3[i] << endl;
    }

    const auto &data = ia3.back();
    cout << data << endl;

    ia3.pop_back();
    const auto &data2 = ia3.back();
    cout << data2 << endl;
}
```
## 模板的友元
模板类也支持友元类的访问，以下列举了几种情况
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

template <typename Type>
class Bar
{
    //将访问权限授予用来实例化Bar的类型
    friend Type;
};
```
## 类模板的别名
类模板的别名定义有以下几种方式
``` cpp
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
类模板的静态成员要在非内联文件中初始化，也就是说在类模板声明的.h文件初始化。
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

template <typename T>
size_t Foo<T>::ctr = 0;
```
## 告知编译器模板的子类型
对于string::size_type , size_type是一个类型
默认情况下，C++语言假定通过作用域运算符访问的名字不是类型。
因此，如果我们希望使用一个模板类型参数的类型成员，就必须显式告诉编译器该名字是一个类型。
我们通过使用关键字typename来实现这一点：

我们用下面的例子显示指名模板名作用域下的是类型不是名字
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
## 巧用模板类完成析构
有时候我们可以利用模板类型实现()的重载，这样通过仿函数传递给智能指针的第二个参数，可以帮助智能指针回收内存
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
DebugDelete实现了仿函数，接下来写一个函数调用这个仿函数
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
## 模板类型推断
有时候对于模板函数返回的类型表示起来很复杂时，可以通过auto 配合尾置类型推断返回数据类型
比如我们我们想返回迭代器指向类型的引用
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
通过decltype(*beg)返回迭代器beg指向的元素的引用类型。
如果想要返回指向元素的副本类型，不是引用类型可以通过remove_reference去引用
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
## 模板的左值和右值
函数模板同样存在左值和右值
``` cpp
//接受左值引用的模板函数
template <typename T>
void f1(T &t)
{
}

//接受右值引用的模板函数
template <typename T>
void f2(T &&t)
{
}
```
f2(42) T就被推断为int
int i = 100; f2(i) T就被推断为int& 参数类型就变为int& &&
当模板函数的参数是一个T类型的右值引用
1 传递给该参数的实参是一个右值时，T就是该右值类型
2 传递给该参数的实参是一个左值时，T就是该左值引用类型

折叠规则
X&& 、X&&& 都会被折叠为X&
X&& && 会被折叠为X&&

所以我们可以推断move的实现原理，其参数一定是T&&类型，因为其能接受左值和右值两种类型。其返回值一定是实参类型的右值引用类型。
``` cpp
template <typename T>
typename remove_reference<T>::type &&my_move(T &&t)
{
    return static_cast<typename remove_reference<T>::type &&>(t);
}
```
## 为什么要有原样转发
stl::forward是用来做原样转发的，将原有类型保持原样传递给其他函数，这种机制尤为重要。因为如果不进行原样转发，传递的参数变为左值，传递给一个接受右值引用的函数会出现编译报错。
比如我们实现一个flip函数，既能接受左值又能接受右值，并且在函数内部修改这个值会同步到外部实参的效果，那他的实现一定是通过模板类型T&&实现的，通过折叠达到适配左值和右值的目的
``` cpp
void flip(F f, T1 &&t1, T2 &&t2)
{
    f(std::forward<T2>(t2), std::forward<T1>(t1));
}
```
flip 函数内部调用了函数f, 将t1和t2的类型原样转发。
``` cpp
void gtemp(int &&i, int &j)
{
    cout << "i is " << i << " j is " << j << endl;
}

void use_ftemp()
{
    int j = 100;
    int i = 99;
   
    // flip(gtemp, j, 42) 会报错
    // 因为42作为右值纯递给flip，t2会被折叠为int类型,
    // j作为左值传递给flip, T1会绑定为int&，通过折叠t1变为int&类型
    // 如果不进行原样转发，t2传递给gtemp第一个参数时，t2虽然是右值引用类型的变量
    // 但是t2作为左值传递给了gtemp第一个参数，编译器会报错，int&&无法绑定int类型
    // 所以无论右值引用类型还是左值引用类型的变量当成参数传递给其他函数时，这个变量就是一个左值。
    // 通过原样转发就保证了这个值在传递给其他函数时不改变其左值引用类型或者右值引用类型
    // 这样即使编译报错也是实参层面传递出了错误。
    flip(gtemp, i, 42);
    cout << "i is " << i << " j is " << j << endl;
}
```
## 总结
本文模拟实现了vector的功能。
视频链接[https://www.bilibili.com/video/BV15t4y1W7ZL/?vd_source=8be9e83424c2ed2c9b2a3ed1d01385e9](https://www.bilibili.com/video/BV15t4y1W7ZL/?vd_source=8be9e83424c2ed2c9b2a3ed1d01385e9)
源码链接 [https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
