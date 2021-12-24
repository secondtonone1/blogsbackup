---
title: 函数
date: 2021-12-22 10:06:38
categories: C++
tags: C++
---
## 函数
一个典型的函数（function）定义包括以下部分：返回类型（return type）、函数名字、由0个或多个形参（parameter）组成的列表以及函数体。其中，形参以逗号隔开，形参的列表位于一对圆括号之内，如下就是一个函数的定义
<!--more-->
``` cpp
void funca(){
    cout << "hello world!!!" << endl;
}
```
## 局部变量
在C++语言中，名字有作用域，对象有生命周期（lifetime）。理解这两个概念非常重要。
· 名字的作用域是程序文本的一部分，名字在其中可见。
· 对象的生命周期是程序执行过程中该对象存在的一段时间。
如我们所知，函数体是一个语句块。块构成一个新的作用域，我们可以在其中定义变量。形参和函数体内部定义的变量统称为局部变量（local variable）。它们对函数而言是“局部”的，仅在函数的作用域内可见，同时局部变量还会隐藏（hide）在外层作用域中同名的其他所有声明中。
### 自动对象
对于普通局部变量对应的对象来说，当函数的控制路径经过变量定义语句时创建该对象，当到达定义所在的块末尾时销毁它。我们把只存在于块执行期间的对象称为自动对象（automatic object）。函数形参和函数内部定义的普通变量都是自动对象。
### 局部静态对象
某些时候，有必要令局部变量的生命周期贯穿函数调用及之后的时间。可以将局部变量定义成static类型从而获得这样的对象。局部静态对象（local static object）在程序的执行路径第一次经过对象定义语句时初始化，并且直到程序终止才被销毁，在此期间即使对象所在的函数结束执行也不会对它有影响。
``` cpp
size_t count_calls()
{
    //调用结束后，这个值仍然有效
    static size_t ctr = 0;
    return ++ctr;
}

for (size_t i = 0; i != 10; ++i)
{
    cout << count_calls() << endl;
}
```
这段程序将输出从1到10（包括10在内）的数字。在控制流第一次经过ctr的定义之前，ctr被创建并初始化为0。每次调用将ctr加1并返回新值。每次执行count_calls函数时，变量ctr的值都已经存在并且等于函数上一次退出时ctr的值。因此，第二次调用时ctr的值是1，第三次调用时ctr的值是2，以此类推。
局部静态变量赋初值只在第一次执行时赋初值的操作，以后再执行都不会进行赋初值的操作。而且生命周期随着程序结束才结束。
## 参数传递
### 值传递
函数的形参如果是非引用类型则是值传递,函数内部修改形参不会影响到外部实参的值
``` cpp
void nochange(int a)
{
    a--;
    cout << a << endl;
}
    int m = 6;
    nochange(m);
    cout << m << endl;
```
程序输出5和6，在函数内部输出的是5，在函数外部输出的是6，可见值传递不会改变实参的值，如果要改变实参的值可以通过引用或者指针操作。
### 指针形参
指针的行为和其他非引用类型一样。当执行指针拷贝操作时，拷贝的是指针的值。拷贝之后，两个指针是不同的指针。因为指针使我们可以间接地访问它所指的对象，所以通过指针可以修改它所指对象的值
``` cpp
void change(int *p)
{
    (*p)--;
    cout << *p << endl;
}
int m = 6;
change(&m);
cout << m << endl;
```
输出5，5
p指向了m的地址，所以*p取到的是m的空间数据，这样就达到修改m的效果。
### 传引用参数
函数参数为引用类型可以达到通过函数内部修改外部实参的效果，也可以减少传递参数造成的copy开销，拷贝大的类类型对象或者容器对象比较低效，甚至有的类类型（包括IO类型在内）根本就不支持拷贝操作。当某种类型不支持拷贝操作时，函数只能通过引用形参访问该类型的对象。
``` cpp
void change(int &ra)
{
    ra--;
    cout << ra << endl;
}
int m = 6;
change(m);
cout << m << endl;
```
输出两个5，参数为引用类型，可以通过函数内部修改外部实参的值。
我们准备编写一个函数比较两个string对象的长度。因为string对象可能会非常长，所以应该尽量避免直接拷贝它们，这时使用引用形参是比较明智的选择。又因为比较长度无须改变string对象的内容，所以把形参定义成对常量的引用
``` cpp
bool isShorter(const string &s1, const string &s2)
{
    return s1.size() < s2.size();
}
```
### 使用引用形参返回额外信息
一个函数只能返回一个值，然而有时函数需要同时返回多个值，引用形参为我们一次返回多个结果提供了有效的途径。举个例子，我们定义一个名为find_char的函数，它返回在string对象中某个指定字符第一次出现的位置。同时，我们也希望函数能返回该字符出现的总次数。
``` cpp
string ::size_type find_char(const string &s, char c, string::size_type &occurs)
{
    //第一次出现的位置(如果有的话)
    auto ret = s.size();
    //设置表示出现次数的形参的值
    occurs = 0;
    for (decltype(ret) i = 0; i != s.size(); ++i)
    {
        if (s[i] == c)
        {
            if (ret == s.size())
                //记录c第一次出现的位置
                ret = i;

            //出现的次数+1
            ++occurs;
        }
    }
    return ret;
}
```
### 参数为数组
当函数的参数为数组时，一般都显示传递一个数组的大小参数
``` cpp
// const int ia[]等价于const int * ia
// size表示数组的大小，将它显示地传给函数用于控制对ia元素的访问
void print_array(const int ia[], size_t size)
{
    for (size_t i = 0; i != size; ++i)
    {
        cout << ia[i] << endl;
    }
}
```
主函数可以这样调用
``` cpp
   int j[] = {0, 1};
    print_array(j, end(j) - begin(j));
```
### 数组引用形参
``` cpp
// arr是数组的引用，维度是类型的一部分
void print_arrayref(int (&arr)[10])
{
    for (auto elem : arr)
    {
        cout << elem << endl;
    }
}
```
### initializer_list形参
如果函数的实参数量未知但是全部实参的类型都相同，我们可以使用initializer_list类型的形参。initializer_list是一种标准库类型，用于表示某种特定类型的值的数组
``` cpp
void error_msg(initializer_list<string> il)
{
    for (auto beg = il.begin(); beg != il.end(); beg++)
    {
        cout << *beg << " ";
    }
    cout << endl;
}
```
## 返回值
函数可以是void类型不返回数据，也可以是有返回值类型，但是不要返回局部变量的指针或者引用。如前所述，返回局部对象的引用是错误的；同样，返回局部对象的指针也是错误的。一旦函数完成，局部对象被释放，指针将指向一个不存在的对象。
也可以返回引用类型，这样返回值就可以作为左值使用
``` cpp
char &get_val(string &str, string::size_type ix)
{
    return str[ix];
}
  string s("a value");
    cout << s << endl; //输出a value
      //将s的第一个字母修改为A
    get_val(s, 0) = 'A';
```
## 返回值为数组的指针或引用
因为数组不能被拷贝，所以函数不能返回数组。不过，函数可以返回数组的指针或引用。
``` cpp
// arrT是一个类型别名，他表示的类型含有10个整数数组
typedef int arrT[10];
// arrT的等价声明
using arrT2 = int[10];
// func返回一个指向含有10个整数的数组的指针
arrT *func(int);
```
## 声明一个返回数组指针的函数
如果我们想定义一个返回数组指针的函数，则数组的维度必须跟在函数名字之后。然而，函数的形参列表也跟在函数名字后面且形参列表应该先于数组的维度。因此，返回数组指针的函数形式如下所示：
``` cpp
Type (*function(parameter_list))[dimension]
```
举个具体点的例子，下面这个func函数的声明没有使用类型别名：
``` cpp
int (*func(int i))[10];
```
· func（int i）表示调用func函数时需要一个int类型的实参。
· （＊func（int i））意味着我们可以对函数调用的结果执行解引用操作。
· （＊func（int i））[10]表示解引用func的调用将得到一个大小是10的数组。
· int （＊func（int i））[10]表示数组中的元素是int类型。
### 尾置类型
在C++11新标准中还有一种可以简化上述func声明的方法，就是使用尾置返回类型（trailing return type）。任何函数的定义都能使用尾置返回，但是这种形式对于返回类型比较复杂的函数最有效，比如返回类型是数组的指针或者数组的引用。尾置返回类型跟在形参列表后面并以一个->符号开头。为了表示函数真正的返回类型跟在形参列表之后，我们在本应该出现返回类型的地方放置一个auto：
``` cpp
// func接受一个int类型的实参，返回值为一个指针
//该指针指向含有10个整数的数组
auto func(int i) -> int (*)[10];
```
### 使用decltype
如果我们知道函数返回的指针将指向哪个数组，就可以使用decltype关键字声明返回类型。例如，下面的函数返回一个指针，该指针根据参数i的不同指向两个已知数组中的某一个：
``` cpp

int odd[] = {1, 3, 5, 7, 9};
int even[] = {0, 2, 4, 6, 8};
//返回一个指针，该指针指向含有5个整数的数组
decltype(odd) *arrPtr(int i)
{
    return (i % 2) ? &odd : &even;
}
```
## 函数重载
如果同一作用域内的几个函数名字相同但形参列表不同，我们称之为重载（overloaded）函数。
``` cpp
void print(const char *cp);
void print(const int *beg, const int *end);
void print(const int ia[], size_t size);
```
利用const_cast实现两个返回最小字符串的函数
``` cpp
const string &shorterString(const string &s1, const string &s2)
{
    return s1.size() <= s2.size() ? s1 : s2;
}

string &shorterString(string &s1, string &s2)
{
    auto &r = shorterString(const_cast<const string &>(s1), const_cast<const string &>(s2));
    return const_cast<string &>(r);
}
```
## 默认实参
我们可以对函数形参设置默认值，如果不传实参，则用形参默认值
``` cpp
typedef string::size_type sz;
void screen(sz ht = 24, sz wh = 80, char back = ' ');
```
我们可以为一个或多个形参定义默认值，不过需要注意的是，一旦某个形参被赋予了默认值，它后面的所有形参都必须有默认值。
使用时可以
``` cpp
//函数形参分别为100,200,'a'
screen(100,200,'a');
//函数形参分别为100,200,' '
screen(100,200);
//函数形参分别为24,80,' '
screen();
```
## 内联函数
内联函数可避免函数调用的开销将函数指定为内联函数（inline），通常就是将它在每个调用点上“内联地”展开
constexpr函数（constexpr function）是指能用于常量表达式的函数。定义constexpr函数的方法与其他函数类似，不过要遵循几项约定：函数的返回类型及所有形参的类型都得是字面值类型，而且函数体中必须有且只有一条return语句：
``` cpp
constexpr int new_sz() { return 42; }
constexpr int foo = new_sz();
```
执行该初始化任务时，编译器把对constexpr函数的调用替换成其结果值。为了能在编译过程中随时展开，constexpr函数被隐式地指定为内联函数。
我们允许constexpr函数的返回值并非一个常量：
``` cpp
constexpr size_t scale(size_t cnt) { return new_sz() * cnt; }
```
当scale的实参是常量表达式时，它的返回值也是常量表达式；反之则不然：
``` cpp
//正确，scale(2)返回的是常量
int arr[scale(2)];
//i不是常量，scale返回的不是常量
int i = 2;
//编译器报错
int arr2[scale(i)];
```
## 函数指针
函数指针指向的是函数而非对象。和其他指针一样，函数指针指向某种特定类型。函数的类型由它的返回类型和形参类型共同决定，与函数名无关。
``` cpp
// pf指向一个函数,该函数的参数是两个const string 的引用，返回bool类型
bool (*pf)(const string &, const string &);
```
从我们声明的名字开始观察，pf前面有个＊，因此pf是指针；右侧是形参列表，表示pf指向的是函数；再观察左侧，发现函数的返回类型是布尔值。因此，pf就是一个指向函数的指针，其中该函数的参数是两个const string的引用，返回值是bool类型。
＊pf两端的括号必不可少。如果不写这对括号，则pf2是一个返回值为bool指针的函数：
``` cpp
//声明一个名为pf2的函数返回值类型为bool*
bool *pf2(const string &, const string &);
```
虽然不能返回一个函数，但是能返回指向函数类型的指针。然而，我们必须把返回类型写成指针形式，编译器不会自动地将函数返回类型当成对应的指针类型处理。与往常一样，要想声明一个返回函数指针的函数，最简单的办法是使用类型别名
``` cpp
// F是函数类型，不是指针
using F = int(int *, int);
// PF是指针类型
using PF = int (*)(int *, int);
```
f1,f2,f3都是返回函数指针的函数
``` cpp
F *f1(int);
PF *f2(int);
int (*f3(int))(int *, int);
```
对于f3的声明，按照由内向外的顺序阅读这条声明语句：我们看到f3有形参列表，所以f3是个函数；f3前面有*，所以f3返回一个指针；进一步观察发现，指针的类型本身也包含形参列表，因此指针指向函数，该函数的返回类型是int。
我们可以使用尾置声明
``` cpp
auto f4(int) -> int (*)(int *, int);
string::size_type sumLength(const string &, const string &);
//根据形参取值，getFcn函数返回值为指向sumLength的指针
decltype(sumLength) *getFcn(const string &);
```