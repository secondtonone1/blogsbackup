---
title: 类的作用域
date: 2021-12-28 10:14:25
tags: C++
categories: C++
---
## 访问成员
每个类都会定义它自己的作用域。在类的作用域之外，普通的数据和函数成员只能由对象、引用或者指针使用成员访问运算符来访问。对于类类型成员则使用作用域运算符访问
``` cpp
    Screen::pos row = 3;
    Screen::pos col = 4;
    Screen screen(row, col, 'c');
    screen.get();
    Screen *psc = &screen;
    psc->get();
```
<!--more-->
类的外部定义成员函数时必须同时提供类名和函数名。在类的内部定义的成员函数不需要提供类名，成员函数内部调用类定义的变量和类型，无需提供类名。举个例子
``` cpp
void Window_mgr::clear(ScreenIndex i)
{
    // s是一个Screen的引用，指向我们想清空的屏幕
    Screen &s = screens[i];
    //清空屏幕
    s.contents = string(s.height * s.width, ' ');
}
```
这是我们之前实现的clear, ScreenIndex为类内定义的类型，所以无需加Window_mgr::
反之，如果我们在类外定义一个新的函数addScreen函数，其返回值为ScreenIndex类型，就需要添加Window_mgr::
``` cpp
class Window_mgr
{
public:
    //窗口中每个屏幕的编号
    using ScreenIndex = std::vector<Screen>::size_type;
    //按照编号将指定的Screen重置为空白
    void clear(ScreenIndex);
    ScreenIndex addScreen(const Screen &);

private:
    std::vector<Screen> screens{Screen(24, 80, ' ')};
};
```
这里要注意所有用到ScreenIndex类型的函数，一定要在 using ScreenIndex 类型声明的下边，否则编译器会报错，因为编译器检测函数声明类型是按顺序的。接下来如果在类外实现addScreen，就要为ScreenIndex添加Window_mgr类名
``` cpp
Window_mgr::ScreenIndex Window_mgr::addScreen(const Screen &s)
{
    screens.push_back(s);
    return screens.size() - 1;
}
```
## 变量查找和作用域

类的定义分两步处理：
· 首先，编译成员的声明。
· 直到类全部可见后才编译函数体。
编译器处理完类中的全部声明后才会处理成员函数的定义。
所以我们可以在类中定义的函数使用类的任何成员变量而不会报错
``` cpp
 //读取光标处字符
    char get() const
    {
        //隐式内联
        return contents[cursor];
    }
```
这种两阶段的处理方式只适用于成员函数中使用的名字。
声明中使用的名字，包括返回类型或者参数列表中使用的名字，都必须在使用前确保可见。
如果某个成员的声明使用了类中尚未出现的名字，则编译器将会在定义该类的作用域中继续查找。
比如
``` cpp
typedef double Money;
class Account
{
public:
    Money balance() { return bal; }

private:
    Money bal;
};
```
当编译器看到balance函数的声明语句时，它将在Account类的范围内寻找对Money的声明。编译器只考虑Account中在使用Money前出现的声明，因为没找到匹配的成员，所以编译器会接着到Account的外层作用域中查找。在这个例子中，编译器会找到Money的typedef语句，该类型被用作balance函数的返回类型以及数据成员bal的类型.
## 成员函数中变量查找规则
成员函数中使用的名字按照如下方式解析：
· 首先，在成员函数内查找该名字的声明。和前面一样，只有在函数使用之前出现的声明才被考虑。
· 如果在成员函数内没有找到，则在类内继续查找，这时类的所有成员都可以被考虑。
· 如果类内也没找到该名字的声明，在成员函数定义之前的作用域内继续查找。
如果全局作用域定义了一个变量名和成员函数内使用的变量名一样，则优先查找类中是否有同名的成员名,如果有使用的就是类的成员变量。
如果成员函数的形参名和成员变量名相同，则会覆盖同名的成员变量，可以使用this指代成员变量避免错误
``` cpp
  void setbalance(Money bal)
    {
        this->bal = bal;
    }
```
源码链接[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
我的公众号，谢谢关注
![https://cdn.llfc.club/gzh.jpg](https://cdn.llfc.club/gzh.jpg)
