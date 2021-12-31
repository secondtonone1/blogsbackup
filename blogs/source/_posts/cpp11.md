---
title: 类的访问控制
date: 2021-12-27 10:09:56
tags: C++
categories: C++
---
## 私有和公有
一个类里有方法和成员变量，public关键字标识后，public下的方法和变量都变为公有函数。private关键字标识后，private关键字下的方法和成员变量都变为私有。默认情况下，如果不声明public，class中所有的方法和成员都是私有的。如果不声明private, struct中所有的方法和成员都是公有的。
<!--more-->
## 友元
上一篇，我们将print，read等非Sales_data类的全局函数声明为Sales_data类的友元函数，所以print，read可以访问Sales_data类的私有成员。我们再次回忆一下Sales_data类。
``` cpp
class Sales_data
{
public:
    //通过default实现默认构造
    // Sales_data() = default;
    //显示实现默认构造
    Sales_data() : bookNo(""), units_sold(0), revenue(0.0) {}
    // copy构造，根据Sales_data类型对象构造一个新对象
    Sales_data(const Sales_data &sa);
    Sales_data(const std::string &s) : bookNo(s) {}
    Sales_data(const std::string &s, unsigned n, double p)
        : bookNo(s), units_sold(n), revenue(p * n) {}
    Sales_data(std::istream &is);
    //返回图书号
    std::string isbn() const { return bookNo; }
    //获取平均单价
    double avg_price() const;
    //将一个Sales_data对象合并到当前类对象
    Sales_data &combine(const Sales_data &);
    friend std::ostream &print(std::ostream &, const Sales_data &);
    friend std::istream &read(std::istream &, Sales_data &);

private:
    //图书编号
    std::string bookNo;
    //销量
    unsigned units_sold = 0;
    //收入
    double revenue = 0.0;
};
```
封装有两个重要的优点：
· 确保用户代码不会无意间破坏封装对象的状态。
· 被封装的类的具体实现细节可以随时改变，而无须调整用户级别的代码。
## 友元的声明
友元的声明友元的声明仅仅指定了访问的权限，而非一个通常意义上的函数声明。如果我们希望类的用户能够调用某个友元函数，那么我们就必须在友元声明之外再专门对函数进行一次声明。为了使友元对类的用户可见，我们通常把友元的声明与类本身放置在同一个头文件中（类的外部）。因此，我们的Sales_data头文件应该为read、print和add提供独立的声明（除了类内部的友元声明之外）。所以我在Sales_data类的头文件里声明了这些函数
``` cpp
// Sales_data的非成员接口
extern Sales_data add(const Sales_data &, const Sales_data &);
extern std::ostream &print(std::ostream &, const Sales_data &);
extern std::istream &read(std::istream &, Sales_data &);
```
## 隐藏类型定义
我们可以在类中定义一种新的类型，这种类型对于外部是隐藏内部实现的，外部不知道该类型是什么。
``` cpp
class Screen
{
public:
    typedef std::string::size_type pos;

private:
    pos cursor = 0;
    pos height = 0, width = 0;
    std::string contents;
};
```
用来定义类型的成员必须先定义后使用，这一点与普通成员有所区别，具体原因将在7.4.1节（第254页）解释。因此，类型成员通常出现在类开始的地方。
## inline成员函数
所谓内联函数就是在编译时展开，减少运行时开销的一种手段，可以通过inline关键字声明，也可以在类的cpp文件里定义函数时前面指明inline，当然一个类的成员函数在类的头文件实现了，那它也是内联函数，我们完善Screen类，用以上三种方式实现内联函数
``` cpp
class Screen
{
public:
    typedef std::string::size_type pos;
    //因为Screen有另一个构造函数
    //所以要实现一个默认构造函数
    Screen() = default;
    // cursor被初始化为0
    Screen(pos ht, pos wd, char c) : height(ht), width(wd), contents(ht * wd, c) {}
    //读取光标处字符
    char get() const
    {
        //隐式内联
        return contents[cursor];
    }

    //显示内联
    inline char get(pos ht, pos wd) const;
    //能在之后被设为内联
    Screen &move(pos r, pos c);

private:
    pos cursor = 0;
    pos height = 0, width = 0;
    std::string contents;
};
```
我们可以在类的内部把inline作为声明的一部分显式地声明成员函数，同样的，也能在类的外部用inline关键字修饰函数的定义：虽然我们无须在声明和定义的地方同时说明inline，但这么做其实是合法的。不过，最好只在类外部定义的地方说明inline，这样可以使类更容易理解。
## 重载成员函数
类的成员函数同样支持重载。只要函数名相同，参数列表不同即可。
## mutable属性
如果一个成员变量被指明mutable属性，则无论对象是否为const，无论成员函数是否为const，该成员变量都可以被修改。
我们给Screen定义一个mutable成员变量access_ctr，以及一个const成员函数some_member，并在该函数中修改access_ctr变量。
``` cpp
class Screen
{
public:
    typedef std::string::size_type pos;
    //因为Screen有另一个构造函数
    //所以要实现一个默认构造函数
    Screen() = default;
    // cursor被初始化为0
    Screen(pos ht, pos wd, char c) : height(ht), width(wd), contents(ht * wd, c) {}
    //读取光标处字符
    char get() const
    {
        //隐式内联
        return contents[cursor];
    }

    //显示内联
    inline char get(pos ht, pos wd) const;
    //能在之后被设为内联
    Screen &move(pos r, pos c);
    void some_member() const;

private:
    pos cursor = 0;
    pos height = 0, width = 0;
    std::string contents;
    //即使在一个const对象里access_ctr也可被修改
    mutable size_t access_ctr;
};
```
实现some_member的一个成员变量
``` cpp
void Screen::some_member() const
{ //在const函数中也可以修改access_ctr
    ++access_ctr;
}
```
## 链式调用
当我们通过成员函数内部返回*this，也就是类对象本身，则可以继续链式调用其内部的成员函数，比如我们通过重载实现两个set函数
``` cpp
Screen &Screen::set(char c)
{
    contents[cursor] = c;
    return *this;
}
Screen &Screen::set(pos r, pos col, char ch)
{
    //给定位置设置新值
    contents[r * width + col] = ch;
    return *this;
}
```
链式调用
``` cpp
    Screen::pos row = 3;
    Screen::pos col = 4;
    Screen screen(3, 4, 'c');
    screen.move(2, 3).set('#');
```
从const成员返回的*this是常量指针,如果我们实现一个display的const函数，
``` cpp
const Screen &Screen::display(ostream &os) const
{
    os << "width is " << width << " "
       << "height is " << height << endl;
    return *this;
}

```
如果按照如下链式调用编译器将报错,因为display返回const Screen& 类型
``` cpp
 screen.display(cout).move(2, 3).set('#');
```
所以我们可以通过重载实现链式调用，实现一个返回const Screen &类型的display函数和一个Screen &类型的display函数,
这两个display函数内部调用do_display函数，因为const函数只能调用const函数，所以我们先实现display函数，他是一个const函数
``` cpp
void Screen::do_display(ostream &os) const
{
    os << "width is " << width << " "
       << "height is " << height << endl;
}
```
再实现两个重载的display函数
``` cpp
const Screen &Screen::display(ostream &os) const
{
    do_display(os);
    return *this;
}

Screen &Screen::display(ostream &os)
{
    do_display(os);
    return *this;
}
```
这样编译器就会根据类型动态选择display的版本
``` cpp
    Screen screen(3, 4, 'c');
    screen.display(cout).move(2, 3).set('#');
    const Screen cscreen(2, 1, ' ');
    cscreen.display(cout);
```
## 类类型和声明
``` cpp
class First
{
    int memi;
    int getMem();
};

struct Second
{
    int memi;
    int getMem();
};
```
如上我们定义了两个类型，下面的赋值会报错，因为类内的成员虽然一致，但是不同的类就是不同的类型
``` cpp
    First obj1;
    //编译报错，obj1和obj2不是一个类型
     Second obj2 = obj1;
```
我们可以不定义类，先进行类的声明
``` cpp
class Bags;
```
这种声明有时被称作前向声明（forward declaration），它向程序中引入了名字Screen并且指明Bags是一种类类型。对于类型Bags来说，在它声明之后定义之前是一个不完全类型（incomplete type），也就是说，此时我们已知Bags是一个类类型，但是不清楚它到底包含哪些成员。

不完全类型只能在非常有限的情景下使用：
可以定义指向这种类型的指针或引用，也可以声明（但是不能定义）以不完全类型作为参数或者返回类型的函数。

直到类被定义之后数据成员才能被声明成这种类类型。我们必须首先完成类的定义，然后编译器才能知道存储该数据成员需要多少空间。
因为只有当类全部完成后类才算被定义，所以一个类的成员类型不能是该类自己。
然而，一旦一个类的名字出现后，它就被认为是声明过了（但尚未定义），因此类允许包含指向它自身类型的引用或指针：
``` cpp
class Link_screen
{
    Screen window;
    Link_screen *next;
    Link_screen *prev;
};
```
## 友元类和成员函数
可以将一个类A声明为另一个类B的友元，则A类对象可以访问B类对象的私有成员。
``` cpp
class Screen
{
public:
    // Window_mgr可以访问Screen类的私有部分
    friend class Window_mgr;
    //Screen类的其他部分......
};
```
Window_mgr类可以访问Screen类的私有成员，通过class前向声明了Window_mgr类。
接下来我们定义Window_mgr类
``` cpp
class Window_mgr
{
public:
    //窗口中每个屏幕的编号
    using ScreenIndex = std::vector<Screen>::size_type;
    //按照编号将指定的Screen重置为空白
    void clear(ScreenIndex);

private:
    std::vector<Screen> screens{Screen(24, 80, ' ')};
};

void Window_mgr::clear(ScreenIndex i)
{
    // s是一个Screen的引用，指向我们想清空的屏幕
    Screen &s = screens[i];
    //清空屏幕
    s.contents = string(s.height * s.width, ' ');
}
```
也可以让成员函数作为友元
``` cpp
class Screen
{
public:
    // Window_mgr可以访问Screen类的私有部分
    friend void Window_mgr::clear(ScreenIndex);
    //Screen类的其他部分......
};
```
· 首先定义Window_mgr类，其中声明clear函数，但是不能定义它。在clear使用Screen的成员之前必须先声明Screen。
· 接下来定义Screen，包括对于clear的友元声明。
· 最后定义clear，此时它才可以使用Screen的成员。

类和非成员函数的声明不是必须在它们的友元声明之前。当一个名字第一次出现在一个友元声明中时，我们隐式地假定该名字在当前作用域中是可见的。然而，友元本身不一定真的声明在当前作用域中

源码链接[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
我的公众号，谢谢关注
![https://cdn.llfc.club/gzh.jpg](https://cdn.llfc.club/gzh.jpg)