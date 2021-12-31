---
title: 构造函数
date: 2021-12-28 15:40:09
tags: C++
categories: C++
---
## 类成员初始化
类成员的初始化可以通过构造函数的参数列表初始化，也可以在构造函数中赋值完成初始化
``` cpp
Sales_data::Sales_data(const Sales_data &sa)
{
    this->bookNo = sa.bookNo;
    this->revenue = sa.revenue;
    this->units_sold = sa.units_sold;
}
```
<!--more-->
上面就是通过赋值完成了bookNo, revenue, units_sold等成员的初始化。
但不是所有成员都可以通过构造函数内赋值完成初始化，比如const, 引用类型的成员变量，这种类型需要用构造函数初始化列表初始化。
``` cpp
class ConstRef
{
public:
    ConstRef(int ii) : ci(ii), ri(ii) { i = ii; };

private:
    int i;
    const int ci;
    int &ri;
};
```
如果成员是const、引用，或者属于某种未提供默认构造函数的类类型，我们必须通过构造函数初始值列表为这些成员提供初值。
## 委托构造函数
委托构造函数就是将自己的构造需求委托给另一个构造函数，达到构造对象的目的。
我们可以优化一下之前实现的Sales_data构造函数，先实现一个全参数版本的构造函数，在让其他构造函数委托该构造函数完成构造
我们重新定义一个新的类Sales_new
``` cpp
class Sales_new
{
public:
    Sales_new(const std::string &s, unsigned n, double p)
        : bookNo(s), units_sold(n), revenue(p * n) {}

    Sales_new() : Sales_new("", 0, 0.0) {}
    // copy构造，根据Sales_new类型对象构造一个新对象
    Sales_new(const Sales_new &sa) : Sales_new(sa.bookNo, sa.units_sold, sa.revenue) {}

    Sales_new(std::istream &is) : Sales_new() { read(is, *this); }
    friend std::istream &read(std::istream &, Sales_new &);

private:
    //图书编号
    std::string bookNo;
    //销量
    unsigned units_sold = 0;
    //收入
    double revenue = 0.0;
};
```
上述代码只实现了一个全参数版本的构造函数，其余的构造函数都是间接调用这个构造函数完成的。
## 默认构造函数重要性
默认情况下系统会为每个类生成默认构造函数，但我们实现参数版构造函数后，系统提供的默认构造函数就会被隐藏，需要我们手动实现默认构造函数。
以下说明了不实现默认构造函数的问题：
![https://cdn.llfc.club/1640681237%281%29.jpg](https://cdn.llfc.club/1640681237%281%29.jpg)
## 隐式类型转换
如果构造函数只接受一个实参，则它实际上定义了转换为此类类型的隐式转换机制，有时我们把这种构造函数称作转换构造函数.
``` cpp
Sales_data sa;
auto sb = sa.combine(string("good luck"));
```
Sales_data类的combine函数参数为const Sales_data&类型
``` cpp
 Sales_data &combine(const Sales_data &);
```
调用combine函数时，编译器自动执行了隐式类型转换，实现了string对象构造一个临时的Sales_data对象。
在要求隐式转换的程序上下文中，我们可以通过将构造函数声明为explicit抑制构造函数定义的隐式转换：
![https://cdn.llfc.club/1640682604%281%29.jpg](https://cdn.llfc.club/1640682604%281%29.jpg)
关键字explicit只对一个实参的构造函数有效。需要多个实参的构造函数不能用于执行隐式转换，所以无须将这些构造函数指定为explicit的。只能在类内声明构造函数时使用explicit关键字，在类外部定义时不应重复：
![https://cdn.llfc.club/1640682739%281%29.jpg](https://cdn.llfc.club/1640682739%281%29.jpg)
我们可以通过static_cast显示转换，避免错误。
![https://cdn.llfc.club/1640741930%281%29.jpg](https://cdn.llfc.club/1640741930%281%29.jpg)
## 聚合类
聚合类使得用户可以直接访问其成员，并且具有特殊的初始化语法形式。
当一个类满足如下条件时，我们说它是聚合的：
· 所有成员都是public的。
· 没有定义任何构造函数。
· 没有类内初始值。
· 没有基类，也没有virtual函数
以下是聚合类
``` cpp
struct Data
{
    int ival;
    string s;
};
```
可以用初始化{}初始化聚合类
``` cpp
Data val2 = {0, "zack"};
```
初始值的顺序必须与声明的顺序一致，也就是说，第一个成员的初始值要放在第一个，然后是第二个，以此类推。
与初始化数组元素的规则一样，如果初始值列表中的元素个数少于类的成员数量，则靠后的成员被值初始化。初始值列表的元素个数绝对不能超过类的成员数量。
## constexpr构造函数

数据成员都是字面值类型的聚合类是字面值常量类。
如果一个类不是聚合类，但它符合下述要求，则它也是一个字面值常量类：

· 数据成员都必须是字面值类型。· 类必须至少含有一个constexpr构造函数。
· 如果一个数据成员含有类内初始值，则内置类型成员的初始值必须是一条常量表达式；或者如果成员属于某种类类型，则初始值必须使用成员自己的constexpr构造函数。
· 类必须使用析构函数的默认定义，该成员负责销毁类的对象

尽管构造函数不能是const的，但是字面值常量类的构造函数可以是constexpr函数。事实上，一个字面值常量类必须至少提供一个constexpr构造函数。
constexpr构造函数就必须既符合构造函数的要求不能有返回语句，所以要用参数列表的方式初始化类。constexpr构造函数体一般来说应该是空的。
![https://cdn.llfc.club/1640748401%281%29.jpg](https://cdn.llfc.club/1640748401%281%29.jpg)
