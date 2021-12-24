---
title: 迭代器
date: 2021-12-16 11:18:50
categories: C++
tags: C++
---
## 迭代器
当我们要遍历容器如vector,map等复杂结构时，可以通过迭代器进行遍历，依次取出容器中的值。通过容器类的begin()和end()函数获取指向第一个元素位置的迭代器和指向最后一个元素下一个位置的迭代器。
迭代器初步使用
<!--more-->
``` cpp
void iterator_func()
{
    string s("some string");
    if (s.begin() != s.end())
    {
        auto it = s.begin();
        *it = toupper(*it);
    }
}
```
上面代码修改了字符串第一个字母为大写。只有当字符串为空时s.begin()==s.end()
## 迭代器运算
迭代器支持加减运算，支持比较运算
``` cpp
*iter 返回iter所指对象得引用
iter->mem 解引用返回iter所指对象得mem成员
++iter 迭代器位置后移，指向下一个元素
--iter 迭代器位置前移，指向上一个元素
iter1 == iter2 判断iter1和iter2是否相等
iter1 != iter2 判断iter1和iter2不相等
iter = iter + n 迭代器iter向后偏移n个元素
iter = iter -n 迭代器iter 向前偏移n个元素
iter1 >= iter2 迭代器iter1指向的元素是否在iter2之后
```
## 迭代器遍历
通过迭代器修改第一个单词为大写，遇到空格或者字符串末尾结束
``` cpp
    string s("some string");
    for (auto iter = s.begin(); iter != s.end() && !isspace(*iter); iter++)
    {
        *iter = toupper(*iter);
    }

    cout << "str is " << s << endl;
```
通过iter++依次访问s中得每个字符，*iter返回的是每个字符的引用
## 泛型编程
关键概念：泛型编程原来使用C或Java的程序员在转而使用C++语言之后，会对for循环中使用！=而非<进行判断有点儿奇怪，。C++程序员习惯性地使用！=，其原因和他们更愿意使用迭代器而非下标的原因一样：因为这种编程风格在标准库提供的所有容器上都有效。之前已经说过，只有string和vector等一些标准库类型有下标运算符，而并非全都如此。与之类似，所有标准库容器的迭代器都定义了==和！=，但是它们中的大多数都没有定义<运算符。因此，只要我们养成使用迭代器和！=的习惯，就不用太在意用的到底是哪种容器类型。
## 迭代器类型
那些拥有迭代器的标准库类型使用iterator和const_iterator来表示迭代器的类型：const_iterator和常量指针差不多，能读取但不能修改它所指的元素值。相反，iterator的对象可读可写。如果vector对象或string对象是一个常量，只能使用const_iterator；如果vector对象或string对象不是常量，那么既能使用iterator也能使用const_iterator。
``` cpp
  // it能读写vector<int>的元素
    vector<int>::iterator it;
    // it2能读写string对象中的字符
    vector<string>::iterator it2;
    // it3 只能读元素,不能写元素
    vector<int>::const_iterator it3;
    // it4 只能读字符，不能写字符
    vector<string>::const_iterator it4;

    vector<int> v;
    const vector<int> cv;
    // vit1的类型是vector<int>::iterator
    auto vit1 = v.begin();
    // vit2的类型是vector<int>::const_iterator
    auto vit2 = cv.begin();
    //通过cbegin和cend可以获取常量迭代器
    // cvit 类型为vector<int>::const_iterator
    auto cvit = v.cbegin();
```
## 解引用
`迭代器解引用要注意将*和迭代器括起来，因为*的优先级比.低，假设iter是vector<string>::iterator类型`
`判断迭代器所指向的字符串是否为空应该用(*iter).empty()`
`如果用*iter.empty()会被编译器理解为对迭代器先进行empty()函数运算再解引用，会报错，因为迭代器没有empty()操作`
为了方便可以通过->解引用取出元素的成员或者成员函数，如下我们通过遍历，直到遇到空字符串就退出遍历
``` cpp
    vector<string> text = {"zack",
                           "vivo",
                           "",
                           "lisus"};
    for (auto it = text.begin(); it != text.end() && !it->empty(); ++it)
    {
        cout << *it << endl;
    }
```
## 迭代器失效
在通过迭代器遍历vector,string ,map等容器时，如果遍历的循环中添加元素或者删除元素会导致迭代器失效，因为添加元素或者删除元素会影响迭代器的值，可以通过如下方式在遍历的同时删除元素
``` cpp
    auto itdel = text.begin();
    while (itdel != text.end())
    {
        if (itdel->empty())
        {
            itdel = text.erase(itdel);
            continue;
        }
        itdel++;
    }
```
text.erase(itdel)返回的时下一个元素的迭代器，所以直接跳出本次循环继续遍历即可。
## 二分查找
迭代器可以做加减操作，所以我们用迭代器实现一个二分查找, orderv是一个vector,里面的数字是有序的，我们查找9
``` cpp
    vector<int> orderv = {1,
                          2,
                          3,
                          5,
                          6,
                          8,
                          9,
                          10};
    bool bfind = false;
    auto findit = orderv.begin();
    auto beginit = orderv.begin();
    auto endit = orderv.end();

    while (beginit != endit)
    {
        auto midit = beginit + (endit - beginit) / 2;
        if (*midit == 9)
        {
            findit = midit;
            bfind = true;
            break;
        }

        if (*midit > 9)
        {
            endit = midit - 1;
        }

        if (*midit < 9)
        {
            beginit = midit + 1;
        }
    }

    if (bfind)
    {
        cout << "find success, iter val is " << *findit << endl;
    }
```
