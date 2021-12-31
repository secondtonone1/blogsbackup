---
title: 静态成员
date: 2021-12-29 15:35:44
tags: C++
categories: C++
---
## 声明静态成员
我们通过在成员的声明之前加上关键字static使得其与类关联在一起。和其他成员一样，静态成员可以是public的或private的。静态数据成员的类型可以是常量、引用、指针、类类型等。
<!--more-->
举个例子，我们定义一个类，用它表示银行的账户记录
``` cpp
class Account
{
public:
    void calculate()
    {
        amount += amount * interestRate;
    }

    static double rate()
    {
        return interestRate;
    }

    static void rate(double);

private:
    std::string owner;
    double amount;
    static double interestRate;
    static double initRate();
};
```
interestRate是类Account的静态成员变量，被所有对象共享。
静态成员函数也不与任何对象绑定在一起，它们不包含this指针。作为结果，静态成员函数不能声明成const的，而且我们也不能在static函数体内使用this指针。这一限制既适用于this的显式使用，也对调用非静态成员的隐式使用有效。
静态成员对象需要在类的非内联文件中被定义，所谓非内联文件就是类的cpp文件。
``` cpp
double Account::interestRate;

void Account::rate(double ds)
{
    interestRate = ds;
}

double Account::initRate()
{
    interestRate = 0.1;
    return interestRate;
}
```
和其他的成员函数一样，我们既可以在类的内部也可以在类的外部定义静态成员函数。当在类的外部定义静态成员时，不能重复static关键字，该关键字只出现在类内部的声明语句,所以我在类外定义了initRate函数，在类外前面没有static关键字。

要想确保对象只定义一次，最好的办法是把静态数据成员的定义与其他非内联函数的定义放在同一个文件中。
可以为静态成员提供const整数类型的类内初始值，不过要求静态成员必须是字面值常量类型的constexpr。
初始值必须是常量表达式，因为这些成员本身就是常量表达式，所以它们能用在所有适合于常量表达式的地方。
例如，我们可以用一个初始化了的静态数据成员指定数组成员的维度
![https://cdn.llfc.club/1640765986%281%29.jpg](https://cdn.llfc.club/1640765986%281%29.jpg)
即使一个常量静态数据成员在类内部被初始化了，通常情况下也应该在类的外部定义一下该成员。
## 静态成员特性
静态成员独立于任何对象。因此，在某些非静态数据成员可能非法的场合，静态成员却可以正常地使用。
举个例子，静态数据成员可以是不完全类型
![https://cdn.llfc.club/1640766187%281%29.jpg](https://cdn.llfc.club/1640766187%281%29.jpg)
静态数据成员的类型可以就是它所属的类类型。而非静态数据成员则受到限制，只能声明成它所属类的指针或引用

静态成员和普通成员的另外一个区别是我们可以使用静态成员作为默认实参
![https://cdn.llfc.club/1640766291%281%29.jpg](https://cdn.llfc.club/1640766291%281%29.jpg)

源码链接[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
我的公众号，谢谢关注
![https://cdn.llfc.club/gzh.jpg](https://cdn.llfc.club/gzh.jpg)
