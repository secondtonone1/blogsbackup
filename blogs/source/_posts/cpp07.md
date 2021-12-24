---
title: 数组
date: 2021-12-16 16:18:58
categories: C++
tags: C++
---
## 数组
数组是一种类似于标准库类型vector的数据结构，但是在性能和灵活性的权衡上又与vector有所不同。与vector相似的地方是，数组也是存放类型相同的对象的容器，这些对象本身没有名字，需要通过其所在位置访问。与vector不同的地方是，数组的大小确定不变，不能随意向数组中增加元素。因为数组的大小固定，因此对某些特殊的应用来说程序的运行时性能较好，但是相应地也损失了一些灵活性。
<!--more-->
## 数组初始化
初始化数组要指定大小，如果不指定维度系统会根据初始化列表自动设置数组大小，但是不要将数组数组的大小小于列表长度，否则编译器会报错。
``` cpp
void arrary_init()
{
    //不是常量表达式
    unsigned cnt = 42;
    //常量表达式
    constexpr unsigned sz = 42;
    //常量表达式
    const unsigned usz = 42;
    //包含10个整数
    int arr[10];
    //含有42个整形指针的数组
    int *parr[sz];
    //含有42个string的数组
    string strvec[usz];
    //含有42个int的数组
    string invec[get_size()];
    // 编译报错,因为cnt不是常量表达式
    // string bad[cnt];
    const unsigned msz = 3;
    //含有3个元素的数组，元素值分别为0,1,2
    int ia1[msz] = {0, 1, 2};
    //维度是3的数组
    int a2[] = {0, 1, 2};
    //等价于a3[] = {0,1,2,0,0}
    int a3[5] = {0, 1, 2};
    //等价于a4[] = {"hi","bye",""}
    string a4[3] = {"hi", "bye"};
    //错误，初始值过多
    // string a5[2] = {0, 1, 2};
}
```
## 错误操作
不允许拷贝和赋值不能将数组的内容拷贝给其他数组作为其初始值，也不能用数组为其他数组赋值：
``` cpp
 //含有3个整数的数组
  int a[] = {0,1,2};
  //不允许用一个数组初始化另一个数组
  int a2[] = a;
  //不能把一个数组直接赋值给另一个数组
   a2 = a;
```
## 复杂声明
和vector一样，数组能存放大多数类型的对象。例如，可以定义一个存放指针的数组。又因为数组本身就是对象，所以允许定义数组的指针及数组的引用。在这几种情况中，定义存放指针的数组比较简单和直接，但是定义数组的指针或数组的引用就稍微复杂一点了：
``` cpp
void dif_array()
{
    // ptrs是含有10个整形指针的数组
    int *ptrs[10];
    //不存在引用的数组
    // int &refs[10] = /*?*/;
    int arr[10] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
    // 数组的引用 arrRef是arr的引用
    int(&arrRef)[10] = arr;
    // Parray指向一个含有10个整数的数组
    int(*Parray)[10] = &arr;
}
```
## 数组访问
和vector一样，数组也支持下标访问和遍历访问
``` cpp
void visit_array()
{
    //以10分为一个分数段统计成绩，0~9，10~19...，90~99,100
    // 11 个分数段，全部初始化为0
    unsigned scores[11] = {};
    unsigned grade;
    while (cin >> grade)
    {
        if (grade <= 100)
        {
            ++scores[grade / 10];
        }
    }

    //通过for range 遍历打印
    for (auto i : scores)
    {
        cout << i << " ";
    }
    cout << endl;
}
```
## 指针和数组
在C++语言中，指针和数组有非常紧密的联系。使用数组的时候编译器一般会把它转换成指针。
通常情况下，使用取地址符来获取指向某个对象的指针，取地址符可以用于任何对象。
数组的元素也是对象，对数组使用下标运算符得到该数组指定位置的元素。
因此像其他对象一样，对数组的元素使用取地址符就能得到指向该元素的指针：
``` cpp
   //数组的元素是string元素
    string nums[] = {"one", "two", "three"};
    // p指向nums的第一个元素
    string *p = &nums[0];
```
在一些情况下数组的操作实际上是指针的操作，这一结论有很多隐含的意思。其中一层意思是当使用数组作为一个auto变量的初始值时，推断得到的类型是指针而非数组：
``` cpp
    // ia是一个含有10个整数的数组
    int ia[] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
    // ia2是一个整数型指针，指向ia第一个元素
    auto ia2(ia);
    //错误：ia2是一个指针，不能用int值给指针赋值
    // ia2 = 42;
```
尽管ia是由10个整数构成的数组，但当使用ia作为初始值时，编译器实际执行的初始化过程类似于下面的形式：
``` cpp
ia2是int*类型
auto ia2(&ia[0]);
```
必须指出的是，当使用decltype关键字时上述转换不会发生，decltype（ia）返回的类型是由10个整数构成的数组：
``` cpp
// ia3是一个含有10个整数的数组
    decltype(ia) ia3 = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
    int *pint = nullptr;
    //错误，不能用整形指针给数组赋值
    // ia3 = pint;
    //正确，可以对数组的元素赋值
    ia3[4] = 1024;
```
## 指针也是迭代器
就像使用迭代器遍历vector对象中的元素一样，使用指针也能遍历数组中的元素。当然，这样做的前提是先得获取到指向数组第一个元素的指针和指向数组尾元素的下一位置的指针。之前已经介绍过，通过数组名字或者数组中首元素的地址都能得到指向首元素的指针；不过获取尾后指针就要用到数组的另外一个特殊性质了。
``` cpp
    int arr[] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
    int *p = arr; // p指向arr的第一个元素
    ++p;          // p指向arr[1]
```
我们可以设法获取数组尾元素之后的那个并不存在的元素的地址：
``` cpp
 int *e = &arr[10]; //指向arr尾元素的下一个位置的指针
```
这里显然使用下标运算符索引了一个不存在的元素，arr有10个元素，尾元素所在位置的索引是9，接下来那个不存在的元素唯一的用处就是提供其地址用于初始化e。就像尾后迭代器一样，尾后指针也不指向具体的元素。因此，不能对尾后指针执行解引用或递增的操作。
所以我们利用指针的末尾元素可以实现另一种方式的遍历
``` cpp
 for (int *b = arr; b != e; ++b)
        cout << *b << endl;
```
数组也支持sizeof操作sizeof计算的是数组所占用的空间,除以sizeof(int)，得到的就是数组的长度，所以数组的遍历可以这样
``` cpp
    for (int i = 0; i < sizeof(arr) / sizeof(int); i++)
    {
        cout << arr[i] << endl;
    }
```
## 标准库函数begin
尽管能计算得到尾后指针，但这种用法极易出错。为了让指针的使用更简单、更安全，C++11新标准引入了两个名为begin和end的函数。这两个函数与容器中的两个同名成员功能类似，不过数组毕竟不是类类型，因此这两个函数不是成员函数。正确的使用形式是将数组作为它们的参数：
``` cpp
// beg指向arr第一个元素
    int *beg = begin(arr);
    // last指向arr最后一个元素的下一个位置
    int *last = end(arr);
    while (beg != last)
    {
        cout << *beg << endl;
        beg++;
    }
```
通过begin和end函数获取数组第一个元素地址和最后一个元素的下一个位置，然后实现遍历，非常安全
## 指针运算
指向数组元素的指针包括解引用、递增、比较、与整数相加、两个指针相减等，用在指针和用在迭代器上意义完全一致。给（从）一个指针加上（减去）某整数值，结果仍是指针。新指针指向的元素与原来的指针相比前进了（后退了）该整数值个位置：
``` cpp
    int arr[] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
    //等价于int *p = &arr[0]
    int *ip = arr;
    //等价于ip2指向arr的第四个元素
    int *ip2 = ip + 4;
```
另外一种计算数组元素个数的方式
``` cpp
    int arr[] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
 // beg指向arr第一个元素
    int *beg = begin(arr);
    // last指向arr最后一个元素的下一个位置
    int *last = end(arr);
    int length = last - beg;
```
## C风格字符串
C风格字符串被C++包含在cstring头文件里,包括strcmp字符串比较，strcpy字符串copy，strcat字符串连接
比较字符串
``` cpp
    const char cal1[] = "A string example";
    const char cal2[] = "A different string";
    if (strcmp(cal1, cal2) < 0)
    {
        cout << "cal1 is less than cal2" << endl;
    }
    else
    {
        cout << "cal2 is less than cal1" << endl;
    }
```
字符串连接
字符串的连接用到了memset清空操作，以及strcpy, strcat等操作，大家看看就好不用深入理解，这是C语言的方式
``` cpp
void c_string()
{
    const char cal1[] = "A string example";
    const char cal2[] = "A different string";
    if (strcmp(cal1, cal2) < 0)
    {
        cout << "cal1 is less than cal2" << endl;
    }
    else
    {
        cout << "cal2 is less than cal1" << endl;
    }

    const int total_len = strlen(cal1) + strlen(cal2) + 1;
    //开辟total_len字节的空间
    char *total_str = new char(total_len);
    //将空间清空为0
    memset(total_str, 0, total_len);
    //将cal1 copy 到 total_str
    strcpy(total_str, cal1);
    //将total_str和cal2连接
    strcat(total_str, cal2);
    //输出total_str 的值
    cout << "total_str is " << total_str << endl;
    //最后释放内存
    if (total_str != nullptr)
    {
        delete total_str;
        total_str = nullptr;
    }
}
```
习惯使用C语言的同学可以通过c_str()函数将string转化为const char*类型的字符串
``` cpp
    string strcpp = "CPP";
    const char *strc = strcpp.c_str();
```
## 使用数组初始化vector对象
vector除了可以通过初始化列表，指定初始值和大小等方式外，还可以通过数组和vector初始化
通过vector初始化
``` cpp
    vector<int> v1 = {1, 3, 5, 7, 9};
    vector<int> v2(v1);
    for (auto v : v2)
    {
        cout << v << endl;
    }
```
通过数组初始化
``` cpp
void vector_init2()
{
    int a[] = {2, 4, 6, 8, 10};
    vector<int> v3(begin(a), end(a));

    for (auto v : v3)
    {
        cout << v << endl;
    }
}
```

