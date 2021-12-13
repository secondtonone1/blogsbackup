---
title: 输入输出流和变量
date: 2021-12-10 17:18:31
categories: C++
tags: C++
---
## 简介
本节介绍C++输入输出流和基本的变量
## iostream
C++提供了标准的输入和输出流功能,要包含iostream头文件，就可以使用cin和cout了
cin表示输入，cout表示输出，下面是使用案例
<!--more-->
``` cpp
#include <iostream>
#include <string>
using namespace std;
void cin_func()
{
    string input;
    cout << "input your words " << endl;
    cin >> input;
    cout << "your input is " << endl;
    cout << input << endl;
}
```
程序输出
``` bash
input your words
zack
your input is
zack
```
` >>` 会获取输入写入缓存,并返回cin对象，`<<`会从缓存中读取数据写入cout并返回cout,最后endl会将cout缓存中的数据输出到终端。
## 变量
``` cpp
void var_func()
{
    //整形，4字节
    int a = 100;
    //ll整形, 8字节
    long long lla = 1000;
    //长整型， 4 字节
    long la = 1024;
    //短整型,2 字节
    short sa = 200;
    //带符号字符型,
    //字符型使用时最好指示带符号还是不带符号
    //因为在不同的机器上不指明char符号可能会有问题
    signed char sc = 'a';
    //无符号字符型
    unsigned char uc = 'm';
    //bool类型
    bool bt = true;
}
```
## 类型转换
当我们把一个非布尔类型的算术值赋给布尔类型时，初始值为0则结果为false，否则结果为true。
当我们把一个布尔值赋给非布尔类型时，初始值为false则结果为0，初始值为true则结果为1。
当我们把一个浮点数赋给整数类型时，进行了近似处理。结果值将仅保留浮点数中小数点之前的部分。
当我们把一个整数值赋给浮点类型时，小数部分记为0。如果该整数所占的空间超过了浮点类型的容量，精度可能有损失。
当我们赋给无符号类型一个超出它表示范围的值时，结果是初始值对无符号类型表示数值总数取模后的余数。例如，8比特大小的unsigned char可以表示0至255区间内的值，如果我们赋了一个区间以外的值，则实际的结果是该值对256取模后所得的余数。因此，把-1赋给8比特大小的unsigned char所得的结果是255。
当我们赋给带符号类型一个超出它表示范围的值时，结果是未定义的（undefined）。此时，程序可能继续工作、可能崩溃，也可能生成垃圾数据。
``` cpp
bool b = 42; //b为true
int i = b; //i 为1
i = 3.14; //i 为3
double pi = i; //pi为3.0
unsigned char c = -1; //
```
## 变量的声明和定义
用extern在头文件声明，在cpp中定义，可以保证变量不会被重复包含
``` cpp
//只声明a
extern int a;
```
如果extern后边做了赋值操作，则不是声明而是定义
``` cpp
extern int a= 100;
```
不带extern 直接类型+ 变量名就是定义
``` cpp
//如下都是定义
int age = 100;
int num ;
```
## 引用
引用就是变量的别名，通过修改引用达到修改变量的值的目的
``` cpp
int j = 20;
// i 是j的引用
int &i = j;
j = 200;
cout << i << " " << j << endl;
```
## 指针
指针值指针的值（即地址）应属下列4种状态之一：
1.指向一个对象。
2.指向紧邻对象所占空间的下一个位置。
3.空指针，意味着指针没有指向任何对象。
4.无效指针，也就是上述情况之外的其他值。

通过对指针的值做解引用(*)，拿到其指向的值，再修改这个值，达到修改指向对象数据的目的
``` cpp
void piont_func()
{
    int age = 18;
    int *page = &age;
    *page += 2;
    cout << age << " " << page << endl;
}
```
## 指向指针的引用
``` golang
void poinref_func()
{
    int i = 42;
    // p是一个指针
    int *p;
    // r 是一个对p的引用
    int *&r = p;
    // 令r指向了一个指针p
    //给r赋值为&i,就是p指向了i
    r = &i;
    //解引用r得到i,也就是p指向的对象，将i的值修改为0
    *r = 0;
}
```
## 常量
``` cpp
void const_func()
{
    // 常量定义一定要初始化赋值，否则编译报错
    const int bufSize = 512;
    //修改bufSize的值会报错
    //编译器提示表达式必须是可修改的左值
    // bufSize = 222;
    //运行时初始化
    const int i = get_size();
    //编译时初始化
    const int j = 43;
    //如果定义const变量不初始化也会报错
    // const int k;
    //利用一个常量初始化另一个常量
    const int cj = j;
    // const引用,引用及其对应的对象都是const
    const int &r1 = cj;
    //不可以修改r1的值
    // r1 = 42;
    //不可以用非常量引用指向一个常量对象
    // int &r2 = ci;
    int iv = 42;
    //允许将const int&绑定到一个普通int对象上
    const int &r1 = iv;
    //正确, r2是一个常量引用
    const int &r2 = 42;
    //正确, r3是一个常量引用
    const int &r3 = r1 * 2;
    //错误, r4 是一个普通非常量的引用
    // int &r4 = r1 * 2;
}
```

## 指向常量的指针
指向常量的指针不可以通过指针修改指向内容的数据
``` cpp
void pconst_func()
{
    //指向常量的指针
    const double pi = 3.14;
    //不可以用普通指针指向常量
    // double *ptr = &pi;
    //用常量指针指向常量
    const double *cptr = &pi;
    //不能给*cptr赋值因为cptr指向的是常量
    // *cptr = 42;
    //指向常量的指针指向非常量
    double dval = 3.14;
    cptr = &dval;
}
```
## 常量指针
指针是对象而引用不是，因此就像其他对象类型一样，允许把指针本身定为常量。
常量指针（const pointer）必须初始化，而且一旦初始化完成，则它的值（也就是存放在指针中的那个地址）就不能再改变了。
把＊放在const关键字之前用以说明指针是一个常量，这样的书写形式隐含着一层意味，即不变的是指针本身的值而非指向的那个值
``` cpp
//常量指针
//常量指针的值初始化后就不允许修改
int errNumb = 0;
// curErr将一直指向errNumb
int *const curErr = &errNumb;
//不允许修改curErr的指向
int rightNumb = 1;
//编译报错，提示=左侧必须为可修改的左值
// curErr = &rightNumb;
const double pi = 3.14159;
// pip是一个指向常量对象的常量指针
const double *const pip = &pi;
```
## 顶层const
指针本身是一个对象，它又可以指向另外一个对象。因此，指针本身是不是常量以及指针所指的是不是一个常量就是两个相互独立的问题。用名词顶层const（top-levelconst）表示指针本身是个常量，而用名词底层const（low-level const）表示指针所指的对象是一个常量。顶层const可以表示任意的对象是常量，这一点对任何数据类型都适用，如算术类型、类、指针等。底层const则与指针和引用等复合类型的基本类型部分有关。比较特殊的是，指针类型既可以是顶层const也可以是底层const

## constexper变量
在一个复杂系统中，很难（几乎肯定不能）分辨一个初始值到底是不是常量表达式。当然可以定义一个const变量并把它的初始值设为我们认为的某个常量表达式，但在实际使用时，尽管要求如此却常常发现初始值并非常量表达式的情况。可以这么说，在此种情况下，对象的定义和使用根本就是两回事儿

C++11新标准规定，允许将变量声明为constexpr类型以便由编译器来验证变量的值是否是一个常量表达式。声明为constexpr的变量一定是一个常量，而且必须用常量表达式初始化：
``` cpp
void constexpr_func()
{
    // 20是一个常量表达式
    constexpr int mf = 20;
    // mf + 1是一个常量表达式
    constexpr int limit = mf + 1;
    // size是一个constexpr函数
    constexpr int sizen = size();
}
```
尽管不能使用普通函数作为constexpr变量的初始值，新标准允许定义一种特殊的constexpr函数。这种函数应该足够简单以使得编译时就可以计算其结果，这样就能用constexpr函数去初始化constexpr变量了。

常量表达式的值需要在编译时就得到计算，因此对声明constexpr时用到的类型必须有所限制。因为这些类型一般比较简单，值也显而易见、容易得到，就把它们称为“字面值类型”（literal type）。到目前为止接触过的数据类型中，算术类型、引用和指针都属于字面值类型。
尽管指针和引用都能定义成constexpr，但它们的初始值却受到严格限制。一个constexpr指针的初始值必须是nullptr或者0，或者是存储于某个固定地址中的对象。

## 指针和constexpr
在constexpr声明中如果定义了一个指针，限定符constexpr仅对指针有效，与指针所指的对象无关。
``` cpp
void pointer_constexpr()
{
    // p是一个指向整形常量的指针,p可以修改，但是*P不可修改
    const int *p = nullptr;
    // q是一个指向整形变量的常量指针,q不可修改,但是*q可以修改
    constexpr int *q = nullptr;
}
```
p和q的类型相差甚远，p是一个指向常量的指针，而q是一个常量指针，其中的关键在于constexpr把它所定义的对象置为了顶层const。
与其他常量指针类似，constexpr指针既可以指向常量也可以指向一个非常量：
``` cpp
int j = 0;
// i 的类型是整型常量
constexpr int i = 42;

void pointer_constexpr()
{
    // p是一个指向整形常量的指针,p可以修改，但是*P不可修改
    const int *p = nullptr;
    // q是一个指向整形变量的常量指针,q不可修改,但是*q可以修改
    constexpr int *q = nullptr;
    // np是一个指向整数的常量指针，其中为空
    constexpr int *np = nullptr;
    // i和j必须定义在函数体之外，否则报错，提示p访问运行时存储
    //因为constexpr要求表达式为常量，在编译时展开
    //  p是常量指针，指向整形常量i
    constexpr const int *p2 = &i;
    // p1是常量指针，指向整数j
    constexpr int *p1 = &j;
}
```
## 类型别名

类型别名（type alias）是一个名字，它是某种类型的同义词。使用类型别名有很多好处，它让复杂的类型名字变得简单明了、易于理解和使用，还有助于程序员清楚地知道使用该类型的真实目的。有两种方法可用于定义类型别名。传统的方法是使用关键字typedef：
1 typedef 
``` cpp
void typedef_func()
{
    // wages是double的同义词
    typedef double wages;
    // base是double的同义词， p 是double*的同义词
    typedef wages base, *p;
    // C11用法
    using newd = double;
    newd dd = 3.14;
}
```
新标准规定了一种新的方法，使用别名声明（aliasdeclaration）来定义类型的别名, using newd = double
就是通过using定义newd类型和double是相同的。

如果某个类型别名指代的是复合类型或常量，那么把它用到声明语句里就会产生意想不到的后果。例如下面的声明语句用到了类型pstring，它实际上是类型char＊的别名

``` cpp
void typedef_func()
{
    typedef char *pstring;
    // pstring是一个指向char的常量指针
    const pstring cstr = 0;
    // ps 是一个指针，其对象是指向char的常量指针
    const pstring *ps;
    char b = 'H';
    //不可修改
    // cstr = &b;
    ps = &cstr;
    const pstring cstr2 = &b;
    ps = &cstr2;
    //不可修改*ps的值
    // *ps = cstr;
}
```
## auto 推导
编程时常常需要把表达式的值赋给变量，这就要求在声明变量的时候清楚地知道表达式的类型。然而要做到这一点并非那么容易，有时甚至根本做不到。为了解决这个问题，C++11新标准引入了auto类型说明符，用它就能让编译器替我们去分析表达式所属的类型。和原来那些只对应一种特定类型的说明符（比如double）不同，auto让编译器通过初始值来推算变量的类型。显然，auto定义的变量必须有初始值.
使用auto也能在一条语句中声明多个变量。因为一条声明语句只能有一个基本数据类型，所以该语句中所有变量的初始基本数据类型都必须一样.
``` cpp
void auto_func()
{
    int a = 100;
    int b = 1024;
    // c被推导为int类型
    auto c = a + b;
    auto i = 0, *p = &i;
    //一条声明语句只能有一个基本数据类型
    //不同类型编译器会报错
    // auto sz = 0, pi = 3.14;
    const int ma = 1;
    // auto会忽略顶层const
    //可以通过const明确指出，此时f为const int类型
    const auto f = ma;
    // auto配合引用类型
    auto &g = a;
    // 不能为非常量引用绑定字面值
    // auto &h = 42;
    //指明const 引用绑定字面值
    const auto &j = 42;
}
```
auto一般会忽略掉顶层const（参见2.4.3节，第57页），同时底层const则会保留下来
要在一条语句中定义多个变量，切记，符号&和＊只从属于某个声明符，而非基本数据类型的一部分，因此初始值必须是同一种类型：
``` cpp
// k是int类型，l是int的引用
// auto 忽略了顶层const
auto k = ci, &l = i;
// m是int常量的引用，p是指向int常量的指针
// auto保留了底层const
auto &m = ci, *p = &ci;
// 错误 i的类型是int， ci的类型是 const int
// auto &n = i, *p2 = &ci;
```
## decltype类型指示符
有时会遇到这种情况：希望从表达式的类型推断出要定义的变量的类型，但是不想用该表达式的值初始化变量。为了满足这一要求，C++11新标准引入了第二种类型说明符decltype，它的作用是选择并返回操作数的数据类型。在此过程中，编译器分析表达式并得到它的类型，却不实际计算表达式的值：
``` cpp
decltype(size()) sum;
```
编译器并不实际调用函数size，而是使用当调用发生时size的返回值类型作为sum的类型。换句话说，编译器为sum指定的类型是什么呢？就是假如size被调用的话将会返回的那个类型。decltype处理顶层const和引用的方式与auto有些许不同。如果decltype使用的表达式是一个变量，则decltype返回该变量的类型（包括顶层const和引用在内）：
``` cpp
void decltype_func()
{
    decltype(size()) sum;
    const int ci = 0, &cj = ci;
    // x的类型是const int
    decltype(ci) x = 0;
    // y的类型是 const int& , y绑定到变量x
    decltype(cj) y = x;
    //错误，z是一个引用，必须初始化
    // decltype(cj) z;
}
```
因为cj是一个引用，decltype（cj）的结果就是引用类型，因此作为引用的z必须被初始化。需要指出的是，引用从来都作为其所指对象的同义词出现，只有用在decltype处是一个例外。如果decltype使用的表达式不是一个变量，则decltype返回表达式结果对应的类型.
``` cpp
int i = 42, *p = &i, &r = i;
// b1 是一个int类型的引用
decltype(r) b1 = i;
// r+0 通过decltype返回int类型
decltype(r + 0) b2;
//错误，必须初始化,c是int&类型
// decltype(*p) c;
```