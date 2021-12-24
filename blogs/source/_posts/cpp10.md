---
title: 类基础
date: 2021-12-24 09:53:46
tags: C++
categories: C++
---
## 类
类就是对一类对象的抽象，比如鹦鹉，麻雀都是鸟，鸟就是类，而鹦鹉，麻雀等就是对象。我们期待实现一个Sales_data类，用来管理图书录入系统，通过录入Sales_data对象信息，达到统计销量和收入的目的。源码链接[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)。如果我们实现Sales_data类，外部调用是这样的
``` cpp
void dealSales()
{
    //保存当前求和结果的变量
    Sales_data total;
    //读入第一笔交易
    if (read(cin, total))
    {
        //保存下一条交易数据的变量
        Sales_data trans;
        //读入下一条交易数据
        while (read(cin, trans))
        {
            //判断isbn
            if (total.isbn() == trans.isbn())
            {
                //更新变量total的值
                total.combine(trans);
            }
            else
            {
                //输出结果
                print(cout, total) << endl;
                // isbn号不一样，说明是新类型的书
                // 处理新类型的书
                total = trans;
            }
        }
        //输出最后一条交易
        print(cout, total) << endl;
    }
    else
    {
        //没有输出任何信息
        cerr << "No data?!" << endl;
    }
}
```
<!--more-->
Sales_data的接口应该包含以下操作：
· 一个isbn成员函数，用于返回对象的ISBN编号
· 一个combine成员函数，用于将一个Sales_data对象加到另一个对象上
· 一个名为add的函数，执行两个Sales_data对象的加法· 一个read函数，将数据从istream读入到Sales_data对象中
· 一个print函数，将Sales_data对象的值输出到ostream
我们定义Sales_data类如下
``` cpp
#ifndef __CLASS_H__
#define __CLASS_H__
class Sales_data
{
public:
    //通过default实现默认构造
    // Sales_data() = default;
    //显示实现默认构造
    Sales_data() : bookNo(""), units_sold(0), revenue(0.0) {}
    // copy构造，根据Sales_data类型对象构造一个新对象
    Sales_data(const Sales_data &sa);
    //返回图书号
    std::string isbn() const { return bookNo; }
    //获取平均单价
    double avg_price() const;
    //将一个Sales_data对象合并到当前类对象
    Sales_data &combine(const Sales_data &);

private:
    //图书编号
    std::string bookNo;
    //销量
    unsigned units_sold = 0;
    //收入
    double revenue = 0.0;
};
// Sales_data的非成员接口
extern Sales_data add(const Sales_data &, const Sales_data &);
extern std::ostream &print(std::ostream &, const Sales_data &);
extern std::istream &read(std::istream &, Sales_data &);
extern void dealSales();
#endif
```
接下来我们逐个部分介绍
## 成员函数
isbn就是Sales_data类的成员函数，该函数返回类对象的bookNo，因为成员函数内部有一个隐藏的this指针形参, this指向了Sales_data对象，所以返回的是this->bookNo，this具体指向哪个Sales_data对象要看是谁调用的isbn，比如a.isbn()，那么this就指向&a。
对于普通成员函数，this的类型为Sales_data `*` const，因为编译器不允许我们在成员函数内部修改this的值。默认情况下，this的类型是指向类类型非常量版本的常量指针。例如在Sales_data成员函数中，this的类型是Sales_data `*`const。尽管this是隐式的，但它仍然需要遵循初始化规则。
对于const成员函数，如isbn，this为const Sales_data `*` const类型，不允许通过*this修改其指向的对象。this是隐式的并且不会出现在参数列表中，所以在哪儿将this声明成指向常量的指针就成为我们必须面对的问题。C++语言的做法是允许把const关键字放在成员函数的参数列表之后，此时，紧跟在参数列表后面的const表示this是一个指向常量的指针。像这样使用const的成员函数被称作常量成员函数（const member function）。
所以常量对象，以及常量对象的引用或指针都只能调用常量成员函数。

编译器分两步处理类：首先编译成员的声明，然后才轮到成员函数体（如果有的话）。因此，成员函数体可以随意使用类中的其他成员而无须在意这些成员出现的次序。所以isbn可以访问bookNo不会报错。
当我们在类的外部定义成员函数时，成员函数的定义必须与它的声明匹配。也就是说，返回类型、参数列表和函数名都得与类内部的声明保持一致。如果成员被声明成常量成员函数，那么它的定义也必须在参数列表后明确指定const属性。同时，类外部定义的成员的名字必须包含它所属的类名。
``` cpp
double Sales_data::avg_price() const
{
    if (units_sold)
        return revenue / units_sold;
    else
        return 0;
}
```
定义返回this对象的combine函数
``` cpp
Sales_data &Sales_data::combine(const Sales_data &sa)
{
    this->units_sold += sa.units_sold;
    this->revenue += sa.revenue;
    //返回调用该函数的对象
    return *this;
}
```
该函数一个值得关注的部分是它的返回类型和返回语句。一般来说，当我们定义的函数类似于某个内置运算符时，应该令该函数的行为尽量模仿这个运算符。内置的赋值运算符把它的左侧运算对象当成左值返回。
## 类相关非成员函数
我们定义非成员函数的方式与定义其他函数一样，通常把函数的声明和定义分离开
如果函数在概念上属于类但是不定义在类中，则它一般应与类声明（而非定义）在同一个头文件内。在这种方式下，用户使用接口的任何部分都只需要引入一个文件。
``` cpp
std::ostream &print(std::ostream &os, const Sales_data &item)
{
    os << item.isbn() << " " << item.units_sold << " "
       << item.revenue << " " << item.avg_price();
    return os;
}

std::istream &read(std::istream &is, Sales_data &item)
{
    double price = 0;
    is >> item.bookNo >> item.units_sold >> price;
    return is;
}
```
此时编译会报错，因为非成员函数访问了类对象的私有成员,我们通过private属性将Sales_data的成员对象定义为私有，所以非成员函数无法访问，所以可以在类的声明中声明友元函数
``` cpp
    friend std::ostream &print(std::ostream &, const Sales_data &);
    friend std::istream &read(std::istream &, Sales_data &);
```
这样print和read声明为Sales_item类的友元函数，就代表其能访问Sales_item类的私有成员了。
接下来我们同样定义add函数.
``` cpp
Sales_data add(const Sales_data &sa1, const Sales_data &sa2)
{
    Sales_data total = sa1;
    total.combine(sa2);
    return total;
}
```
因为combine是public成员函数，所以add不用声明为Sales_item类的友元函数.我们定义了一个新的Sales_data对象并将其命名为total。total将用于存放两笔交易的和，我们用sa1的副本来初始化total。默认情况下，拷贝类的对象其实拷贝的是对象的数据成员。在拷贝工作完成之后，total的bookNo、units_sold和revenue将和sa1一致。接下来我们调用combine函数，将sa2的units_sold和revenue添加给total。最后，函数返回sum的副本。
值得注意的是Sales_data total = sa1;调用了Sales_data类的copy构造函数，所以我们要实现Sales_data类的copy构造函数
``` cpp
Sales_data::Sales_data(const Sales_data &sa)
{
    this->bookNo = sa.bookNo;
    this->revenue = sa.revenue;
    this->units_sold = sa.units_sold;
}
```
## 构造函数
构造函数的名字和类名相同。和其他函数不一样的是，构造函数没有返回类型；除此之外类似于其他的函数，构造函数也有一个（可能为空的）参数列表和一个（可能为空的）函数体。类可以包含多个构造函数，和其他重载函数差不多，不同的构造函数之间必须在参数数量或参数类型上有所区别。
不同于其他成员函数，构造函数不能被声明成const的。
当我们创建类的一个const对象时，直到构造函数完成初始化过程，对象才能真正取得其“常量”属性。因此，构造函数在const对象的构造过程中可以向其写值。
类通过一个特殊的构造函数来控制默认初始化过程，这个函数叫做默认构造函数（default constructor）。默认构造函数无须任何实参。
默认构造函数在很多方面都有其特殊性。其中之一是，如果我们的类没有显式地定义构造函数，那么编译器就会为我们隐式地定义一个默认构造函数。编译器创建的构造函数又被称为合成的默认构造函数（synthesized default constructor）。对于大多数类来说，这个合成的默认构造函数将按照如下规则初始化类的数据成员：
· 如果存在类内的初始值，用它来初始化成员。
· 否则，默认初始化该成员。

我的理解：
如果我们定义了一个类，不写任何构造函数，那么系统会提供默认的构造函数，这时我们定义一个类对象会优先用类内的初始值初始化成员,比如我们注释掉Sales_data的两个构造函数，那么revenue成员会被初始化为0.0, units_sold会被初始化为0，因为类内有他们的初始值，而对于bookNo没有初始值，则执行默认的空值也就是空字符串赋值给bookNo。

`某些类不能依赖于合成的默认构造函数`
对于一个普通的类来说，必须定义它自己的默认构造函数，原因有三：
第一个原因也是最容易理解的一个原因就是编译器只有在发现类不包含任何构造函数的情况下才会替我们生成一个默认的构造函数。一旦我们定义了一些其他的构造函数，那么除非我们再定义一个默认的构造函数，否则类将没有默认构造函数。这条规则的依据是，如果一个类在某种情况下需要控制对象初始化，那么该类很可能在所有情况下都需要控制。
`只有当类没有声明任何构造函数时，编译器才会自动地生成默认构造函数。`
第二个原因是对于某些类来说，合成的默认构造函数可能执行错误的操作。回忆我们之前介绍过的，如果定义在块中的内置类型或复合类型（比如数组和指针）的对象被默认初始化，则它们的值将是未定义的。
`如果类包含有内置类型或者复合类型的成员，则只有当这些成员全都被赋予了类内的初始值时，这个类才适合于使用合成的默认构造函数`
第三个原因是有的时候编译器不能为某些类合成默认的构造函数。例如，如果类中包含一个其他类类型的成员且这个成员的类型没有默认构造函数，那么编译器将无法初始化该成员。对于这样的类来说，我们必须自定义默认构造函数，否则该类将没有可用的默认构造函数。

在C++11新标准中，如果我们需要默认的行为，那么可以通过在参数列表后面写上= default来要求编译器生成构造函数。其中，= default既可以和声明一起出现在类的内部，也可以作为定义出现在类的外部。和其他函数一样，如果= default在类的内部，则默认构造函数是内联的；如果它在类的外部，则该成员默认情况下不是内联的。

对于有参数的构造函数，我们可以通过构造函数初始值列表进行初始化，构造函数初始值列表（constructor initialize list），它负责为新创建的对象的一个或几个数据成员赋初值。构造函数初始值是成员名字的一个列表，每个名字后面紧跟括号括起来的（或者在花括号内的）成员初始值。不同成员的初始化通过逗号分隔开来。
我们再为Sales_data添加几个带参数的构造函数
``` cpp
    Sales_data(const std::string &s) : bookNo(s) {}
    Sales_data(const std::string &s, unsigned n, double p)
        : bookNo(s), units_sold(n), revenue(p * n) {}
```
我们再实现一个构造函数，其参数为ostream类型
``` cpp
Sales_data::Sales_data(std::istream &is)
{
    read(is, *this);
}
```