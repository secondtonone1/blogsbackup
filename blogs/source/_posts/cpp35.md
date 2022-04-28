---
title: C++ 类的继承封装和多态
date: 2022-04-28 20:55:54
tags: C++
categories: C++
---
## C++ 特性
C++ 三大特性，封装继承多态。我们先实现一个Quote作为基类
<!--more-->
``` cpp
class Quote
{
public:
    Quote() = default;
    Quote(const std::string &book, double sales_price)
    {
        price = sales_price;
        bookNo = book;
    }
    std::string isbn() const
    {
        return bookNo;
    }

    virtual double net_price(std::size_t n) const
    {
        cout << "this is Quote net_price" << endl;
        return n * price;
    }
    static void PrintHello()
    {
        cout << "hello world" << endl;
    }

    Quote(const Quote &quote) : bookNo(quote.bookNo), price(quote.price) {}

    void printMem()
    {
        cout << "price is " << price << " bookNo is " << bookNo << endl;
    }

    virtual ~Quote()
    {
        cout << "this is Quote destruct" << endl;
    }

    // final 阻止其他继承Quote的类重写f3函数
    virtual void f3() final {}

private:
    std::string bookNo;

protected:
    double price = 0.0;
};
```
net_price是一个虚函数，实现了基类的计算规则。同时我们实现了一个虚函数f3，但是f3末尾用final标识了，表示继承Quote的子类不能重写f3函数。
我们实现子类
``` cpp
class BulkQuote : public Quote
{
public:
    BulkQuote() = default;
    BulkQuote(const std::string &book, double p, std::size_t qty, double disc) : Quote(book, p), min_qty(qty), discount(disc)
    {
    }

    // override 是C11提供的继承关系检测工具，检测函数类型是否匹配，是否为虚函数等。
    double net_price(std::size_t) const override;
    // void f3() {}

private:
    //打折后最多买多少
    std::size_t min_qty = 0;
    //折扣额度
    double discount = 0.0;
};
```
子类无法重写f3所以注释了。子类BulkQuote重写了net_price,该函数后边用了overide关键字,C11规定写了override关键字的函数必须符合基类的规则，包括函数参数类型相同，返回值相同，函数名一致等。
基类Quote有一个静态函数PrintHello，子类继承Quote也将PrintHello继承过来。
可以通过如下方式调用
``` cpp
void use_base_static()
{
    Quote::PrintHello();
    BulkQuote::PrintHello();
}
```
可以将子类赋值给基类或者将子类对象传给基类的构造函数，这么做的结果是基类构造时只拷贝子类的基类部分。
``` cpp
void use_derive_to_base()
{
    BulkQuote bulkquote(string("Live"), 1.2, 100, 0.8);
    //子类传给基类构造函数，或者子类赋值给基类
    //就会调用基类构造函数，只构造基类部分。
    Quote quote(bulkquote);
    quote.printMem();
    quote = bulkquote;
    quote.printMem();
}
```
当我们将一个子类对象传递给一个基类引用，或者将一个子类对象的指针传递给一个基类指针，通过基类的指针或引用调用虚函数，会动态调用子类对象的虚函数版本，这种特性叫做多态。我们先实现一个全局函数
``` cpp
void print_total(ostream &os, const Quote &quote, std::size_t n)
{
    os << quote.net_price(n) << endl;
}
```
多态特性会让编译器根据动态类型绑定虚函数调用的版本，所谓动态类型就是运行时才确定的类型。
``` cpp
void use_derive_param()
{
    BulkQuote bulkquote(string("Live"), 1.2, 100, 0.8);
    Quote quote(string("Quote"), 1.2);
    print_total(cout, quote, 100);
    print_total(cout, bulkquote, 100);
}
```
上面的程序会根据传给print_total具体的实参类型调用各自的虚函数net_price。
## 纯虚类
如果一个类只包含纯虚函数，不包含成员变量，则该类为纯虚类。所谓纯虚函数就是只有声明，函数体为=0的形式。
``` cpp
//纯虚类
class VirtualBase
{
public:
    VirtualBase() = default;
    virtual void mem() = 0;
    virtual void test() = 0;
};
```
纯虚类类似于Go语言的interface，当我们继承纯虚类后一定要重写其所有的纯虚函数。
``` cpp
class DeriveFromBase : public VirtualBase
{
    virtual void mem()
    {
    }
    virtual void test() {}
};
```
## 封装性
子类只可以访问基类的protected和public成员，不能访问private成员。子类的友元函数可以访问子类的私有变量，公有变量以及受保护的变量，当时不能访问基类的私有变量和protected变量。
``` cpp
// protected
class ProBase
{
public:
    ProBase() = default;
    ProBase(int n) : prot_mem(n) {}
    void mem_func()
    {
        cout << "this is ProBase mem_func" << endl;
    }

protected:
    int prot_mem;

private:
    int priv_mem;
};
```
ProBase包括一个私有变量priv_mem和一个受保护变量prot_mem。我们定义子类继承它
``` cpp
class Sneaky : public ProBase
{
public:
    Sneaky() = default;
    Sneaky(int n) : ProBase(1024), prot_mem(n) {}
    //子类可以使用基类的public和protected成员
    void UsePro()
    {
        cout << prot_mem << endl;
    }

    //子类无法使用基类的private成员。
    // void UsePriv()
    // {
    //     cout << priv_mem << endl;
    // }
    friend void clobber(Sneaky &);
    friend void clobber(ProBase &);

    void GetMem()
    {
        cout << "this is ProBase prot_mem: " << ProBase::prot_mem << endl;
        cout << "this is Sneaky prot_mem: " << prot_mem << endl;
    }

    void mem_func(int n)
    {
        cout << "this is Sneaky mem_func" << endl;
    }

private:
    int self_mem;
    int prot_mem;
};
```
通过继承Sneaky拥有了基类ProBase的私有变量priv_mem和受保护变量prot_mem。又定义了自己的私有变量self_mem和prot_mem。
可以看到即使在Sneaky的类声明中UsePriv这个函数里也无权访问基类的私有变量priv_mem。我们在如下函数测试
``` cpp
void use_probase()
{
    Sneaky sk(11);
    // sk.prot_mem;
    sk.GetMem();
    //调用子类的mem_func(int n)
    sk.mem_func(100);
    ProBase pb;
    //调用基类的mem_func()
    pb.mem_func();
    //基类的mem_func()被覆盖了
    //sk.mem_func();
    //想使用基类的mem_func()需要添加基类作用域
    sk.ProBase::mem_func();
}
```
在类的声明之外，通过对象的方式无法直接使用sk的私有变量prot_mem。因为子类实现了mem_func(int n)版本，所以把基类的mem_func(void)覆盖了，想调用基类版本的mem_func()需要通过sk.ProBase::mem_func()显示调用基类版本。
接下来我们实现Sneaky的两个友元函数
``` cpp
void clobber(Sneaky &s)
{
    s.prot_mem = 100;
    s.self_mem = 1000;
}
//子类友元无法访问基类受保护成员
void clobber(ProBase &b)
{
    // b.prot_mem = 10;
}
```
子类的友元函数无法访问基类的私有成员和保护成员。
## 重写和隐藏
子类继承基类，重新实现基类的虚函数就叫做重写，重写要求必须和基类的虚函数完全匹配，包括参数类型返回值等。
对于基类的非虚函数，子类实现了同名的函数，只要名字相同，即使参数不同，也可以覆盖基类函数，叫做隐藏。
``` cpp
class VBase
{
public:
    virtual int fcn()
    {
        cout << "this is VBase fcn()" << endl;
    }
};

class VD1 : public VBase
{
public:
    // VD1自己定义的fcn(int)，因为和基类VBase的fcn参数不同
    //但是VD1也继承了VBase  fcn()这个版本
    //隐藏了基类的fcn()
    int fcn(int)
    {
        cout << "this is VD1 fcn(int)" << endl;
    }

    // VD1自己新定义的虚函数
    virtual void f2()
    {
        cout << "this is VD1 f2()" << endl;
    }
};

class VD2 : public VD1
{
public:
    //隐藏了VD1版本的fcn(int),因为VD1中fcn(int)不是虚函数
    int fcn(int)
    {
        cout << "this is VD2 fcn(int)" << endl;
    }

    //重写,因为VD1从VBase中继承了虚函数fcn()
    int fcn()
    {
        cout << "this is VD2 fcn()" << endl;
    }

    //重写了VD1的虚函数

    void f2()
    {
        cout << "this is VD2 f2()" << endl;
    }
};
```
如下函数展示了覆盖和重写等情况时调用规则
``` cpp
void use_hiddenbase()
{
    VD1 vd1;
    //调用基类VBase版本
    vd1.VBase::fcn();
    //调用VD1版本
    vd1.fcn(100);

    VD2 vd2;
    VBase *pvb = &vd1;
    //会调用基类的VBase::fcn()
    pvb->fcn();
    VBase *pvb2 = &vd2;
    //多态调用VD2::fcn()
    pvb2->fcn();

    VD1 *pvd1 = &vd2;
    //调用VD2版本的f2()
    pvd1->f2();
}
```
## 类的继承和多态总结

1  派生类向基类转换只在指针或引用时才生效
2  不存在默认的基类向子类转换，但是如果确认转换安全可以通过static_cast来转换。
3  类不想被继承，可以在类名后添加final关键字
4  如果子类无权访问基类构造函数，则无法实现子类对象向基类对象的转换。
5  子类对象可以向基类对象转换，默认只将子类对象中基类的成员赋值给基类对象。
6  多态就是将子类对象的指针赋值给基类对象的指针，通过调用基类的虚函数，实现动态绑定， 运行时调用了子类的虚函数。
 
7   final 声明的虚函数会阻止继承该类的类重写该函数
8   override 要求编译器检测重写的函数是否符合规则，是否为虚函数，是否为类型相同。
9   继承纯虚类，一定要实现它的所有纯虚方法，否则该类无法使用。
10  子类可以使用基类的public和protected成员，子类无法使用基类的private成员
11  proteced 和private成员不可被对象的方式访问。
12  子类的友元函数可以访问子类的私有变量和受保护变量，但是不能访问基类的受保护变量。
13  子类的友元函数可以访问子类自己定义的私有变量，但是不能访问从基类继承而来的私有变量。
14  子类和基类有相同名字的成员或者非虚函数非静态的成员函数，在使用的时候默认使用子类的，
    如果想使用基类的需要加上基类名字的作用域。
15  如果子类实现的函数和基类的虚函数同名，但是参数类型不同，就不是重写而是隐藏，重写要求子类的函数和
    基类的虚函数类型，名称完全一致。
16  针对一个普通的非虚函数的成员函数，子类实现了一个同名的函数，就是覆盖，会隐藏基类的同名函数
17  重载是对于一个类来讲，实现了多个同名函数，他们的参数不同。