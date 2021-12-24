---
title: string类
date: 2021-12-13 16:35:43
categories: C++
tags: C++
---
## 简介
今天介绍string类的使用
<!--more-->
## 初始化和定义
``` cpp
//默认初始化,s1是一个空字符串
string s1;
//赋值初始化,s2是s1的副本
string s2 = s1;
//直接初始化 字面值初始化
string s3 = "hiya";
//直接初始化 构造函数初始化
string s4(10, 'c');
string s5("hello zack");
```
## string操作
``` cpp
void opstr_func()
{
    //定义空字符串
    string s;
    //从输入流写入s
    cin >> s;
    //将s写入输出流
    cout << s << endl;
    //循环读取，直到遇到换行符或者非法输入
    string world;
    while (cin >> world)
        cout << world << endl;
    //读取一整行
    string linestr;
    while (getline(cin, linestr))
    {
        cout << linestr << endl;
    }

    //每次读入一整行，遇到空行跳过
    while (getline(cin, linestr))
    {
        if (!linestr.empty())
        {
            cout << linestr << endl;
            //打印字符串长度
            cout << linestr.size() << endl;
            // size()返回string::size_type类型的数据
            string::size_type size = linestr.size();
        }
    }

    // 比较
    string str1 = "Hello";
    string str2 = "Hello W";
    string str3 = "Za";
    //依次比较每个字符，字符大的字符串就大
    auto b2 = str3 > str1;
    cout << b2 << endl;
    //前面字符相同，长度长的字符串大
    auto b = str2 > str1;
    cout << b << endl;
}
```
string 类重载了 比较运算符，也重载了+运算符等,所以string支持+运算
``` cpp
    // string类对象相加
    string s1 = "Hello", s2 = "Zack";
    string s3 = s1 + "," + s2 + '\n';
    cout << s3 << endl;
    //加号两侧至少有一个是string类型，否则报错
    // string s4 = "Hello" + "Zack";
```
## 对C语言的兼容
建议：使用C++版本的C标准库头文件
C++标准库中除了定义C++语言特有的功能外，也兼容了C语言的标准库。C语言的头文件形如name.h，C++则将这些文件命名为cname。也就是去掉了.h后缀，而在文件名name之前添加了字母c，这里的c表示这是一个属于C语言标准库的头文件。因此，cctype头文件和ctype.h头文件的内容是一样的，只不过从命名规范上来讲更符合C++语言的要求。特别的，在名为cname的头文件中定义的名字从属于命名空间std，而定义在名为.h的头文件中的则不然。一般来说，C++程序应该使用名为cname的头文件而不使用name.h的形式，标准库中的名字总能在命名空间std中找到。如果使用.h形式的头文件，程序员就不得不时刻牢记哪些是从C语言那儿继承过来的，哪些又是C++语言所独有的。
## C11用法
如果想对string对象中的每个字符做点儿什么操作，目前最好的办法是使用C++11新标准提供的一种语句：范围for（rangefor）语句。这种语句遍历给定序列中的每个元素并对序列中的每个值执行某种操作，其语法形式是：
``` cpp
for(declaration:expression)
    statement
```
其中，expression部分是一个对象，用于表示一个序列。declaration部分负责定义一个变量，该变量将被用于访问序列中的基础元素。每次迭代，declaration部分的变量会被初始化为expression部分的下一个元素值。一个string对象表示一个字符的序列，因此string对象可以作为范围for语句中的expression部分。举一个简单的例子，我们可以使用范围for语句把string对象中的字符每行一个输出出来：
``` cpp
string str("hello zack");
    //遍历输出str中的每个字符
for (auto c : str)
{
    cout << c << endl;
}
```
统计字符串中标点符号的数量
``` cpp
string s("Hello World!!!");
decltype(s.size()) punct_cnt = 0;
//统计s中标点符号的数量
for (auto c : s)
{
    if (ispunct(c))
        punct_cnt++;
}
cout << punct_cnt
     << " punctuation characters in "
    << s << endl;
```
将字符串变为大写
``` cpp
//将字符串变为大写
string s3("Hello Vivo");
for (auto &c : s3)
{
    //通过引用string中的字符，然后修改字符
    c = toupper(c);
}
cout << s << endl;
```
将第一个单词变为大写
``` cpp
//通过下标索引修改字符串
//把第一个单词变为大写
string sind("some string");
for (decltype(sind.size()) index = 0; index != sind.size() && isspace(sind[index]); ++index)
{
    sind[index] = toupper(sind[index]);
}
```