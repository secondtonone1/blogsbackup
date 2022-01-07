---
title: 容器
date: 2022-01-04 14:56:09
tags: C++
categories: C++
---
## 常用容器
C++ 常用的stl容器包括:
1 vector 可变大小的数组，支持随机访问。在尾部之外位置插入或删除元素很慢。
2 deque 双端队列，支持快速随机访问，在头尾位置插入删除速度很快。
3 list 双向链表，支持双向访问，任何位置插入和删除都很快
4 forward_list 单向链表，只支持单向访问，在列表任何位置插入和删除都很快
5 array 固定大小数组，支持快速随机访问，不能添加和删除元素。
6 string 与vector相似，专门用于处理字符串。
<!--more-->
## 基本用法
stl容器可以存储任意类型的数据，因为容器是通过模板实现的，下面列举了容器的一些用法
``` cpp
  // sales_data对象的list
    list<Sales_data> sales_list;
    // 保存double数据的deque
    deque<double> dd;
    // vector里存储了vector<string>
    vector<vector<string>> strvec_vec;
```
容器嵌套时尽量将`>>`分开，保留一个空格，有些陈旧的编译器会将`>>`当作输入流。
## 迭代器
迭代器可以理解为访问容器的通用工具，通过容器名.begin()返回指向第一个元素的迭代器，通过容器名.end()返回指向最后一个元素的下一个元素的迭代器。
``` cpp
  //通过begin和end访问容器元素
    for (auto begin = dd.begin(); begin != dd.end(); begin++)
    {
        cout << *begin << endl;
    }
```
begin和end形成了一个左闭右开的空间，所以要用begin != end 判断迭代器是否走到最后一个元素的下一个元素,因为最后一个元素的下一个位置是不合法的，所以是左闭右开空间。
迭代器的类型可以通过容器类型获取
``` cpp
    // vector<string>容器的迭代器
    vector<string>::iterator strit;
    // deque<double>容器的迭代器
    deque<double>::iterator ddit;
```
## 容器初始化
每个容器类型都定义了一个默认构造函数,如上面代码我们定义的deque<double> dd,list<Sales_data> sales_list等都执行了容器默认的构造函数。
将一个新容器创建为另一个容器的拷贝的方法有两种：
可以直接拷贝整个容器，或者（array除外）拷贝由一个迭代器对指定的元素范围。
``` cpp
    //将一个容器copy给另一个相同类型的容器
    list<string> persons2(persons);
    //通过迭代器实现copy
    list<string> persons3(persons.begin(), persons.end());
```
利用assign可以将一个容器内的数据赋值给另一个
``` cpp
 persons2.assign(persons.begin(), persons.end());
```
由于其旧元素被替换，因此传递给assign的迭代器不能指向调用assign的容器。
assign的第二个版本接受一个整型值和一个元素值。
它用指定数目且具有相同给定值的元素替换容器中原有的元素：
``` cpp
    //用指定数目和初始值
    persons2.assign(5, "zack");
```
swap可以交换两个容器的内容
``` cpp
    //交换两个容器
    vector<string> svec1(10);
    vector<string> svec2(24);
    swap(svec1, svec2);
```
调用swap后，svec1将包含24个string元素，svec2将包含10个string。除array外，交换两个容器内容的操作保证会很快——元素本身并未交换，swap只是交换了两个容器的内部数据结构。
除array外，swap不对任何元素进行拷贝、删除或插入操作，因此可以保证在常数时间内完成。
## 添加元素
push_back操作向容器末尾添加指定元素，出了array和forward_list不支持push_back，其余顺序容器都支持。
``` cpp
    // push_back添加尾元素
    vector<string> svec3;
    svec3.push_back("good idea!");
```
当我们用一个对象来初始化容器时，或将一个对象插入到容器中时，实际上放入到容器中的是对象值的一个拷贝，而不是对象本身。就像我们将一个对象传递给非引用参数。随后对容器中元素的任何改变都不会影响到原始对象，反之亦然。
push_front向容器头部插入元素,注意支持push_front的容器仅有deque, list, forwardlist等。
``` cpp
 // push_front添加头部元素
    list<int> nlist;
    for (int i = 0; i < 100; i++)
    {
        nlist.push_front(i);
    }
```
push_back和push_front操作提供了一种方便地在顺序容器尾部或头部插入单个元素的方法。
insert成员提供了更一般的添加功能，它允许我们在容器中任意位置插入0个或多个元素。
vector、deque、list和string都支持insert成员。forward_list提供了特殊版本的insert成员.
每个insert函数都接受一个迭代器作为其第一个参数。迭代器指出了在容器中什么位置放置新元素。
它可以指向容器中任何位置，包括容器尾部之后的下一个位置。
由于迭代器可能指向容器尾部之后不存在的元素的位置，而且在容器开始位置插入元素是很有用的功能，所以insert函数将元素插入到迭代器所指定的位置之前
``` cpp
   // insert 插入
    vector<string> strinvec;
    // vector不支持push_front，可以通过insert实现
    // 在vector头部插入元素会很慢，以为vector要移动所有元素
    strinvec.insert(strinvec.begin(), "zack");
    list<string> strlist;
    //相当于push_back
    strlist.insert(strlist.end(), "zack");
```
insert有很多重载版本，也可以插入指定范围内的元素。
``` cpp
    //批量插入
    strlist.insert(strlist.end(), 10, "ai");
    vector<string> names = {"zack", "rolin", "bob", "pior"};
    //通过迭代器插入
    strlist.insert(strlist.begin(), names.end() - 2, names.end());
    //通过参数列表插入
    strlist.insert(strlist.end(), {"vivo", "hua", "nokia"});
```
如果我们传递给insert一对迭代器，它们不能指向添加元素的目标容器。
insert的返回值为返回新插入元素的位置，insert是将元素插入到迭代器指向元素之前的位置，插入成功后并返回这个位置，所以insert返回的是指向插入之后新元素的位置。
## emplace操作
emplace也是插入操作，但是emplace接受的一个类的构造函数，这样和insert相比，减少了对类对象的构造开销
``` cpp
 // sales_data对象的list
    list<Sales_data> sales_list;
    //传递Sales_data的构造函数
    sales_list.emplace_back(string("love easy"), 200, 1.8);
```
## 访问元素
可以通过front和back获取队首元素和队尾元素
``` cpp
     auto first = sales_list.front();
    auto last = sales_list.back();
    auto strfirst = svec3.front();
```
线性容器支持下标访问，比如vector,string等
``` cpp
    auto strfirst = svec3.front();
    auto strfirst2 = svec3.at(0);
    auto strfrist3 = svec3[0];
```
## 删除元素
list, deque支持pop_back()弹出和最后一个元素,以及pop_front弹出第一个元素。
forward_list不支持pop_back和push_back， vector和string 不支持pop_front和push_front.
``` cpp
  vector<string> strinvec;
  strinvec.pop_back();
  forward_list<string> forlist;
  forlist.push_front("hello");
  forlist.pop_front();
```
也可以通过迭代器删除指定位置的元素
``` cpp
    //删除头部元素
    strinvec.erase(strinvec.begin());
    //范围删除，删除从头部到尾部的元素
    strinvec.erase(strinvec.begin(), strinvec.end());
    //删除所有
    strinvec.clear();
```
删除和插入会导致容器迭代器失效，所以要更新删除后的迭代器确保程序稳定
``` cpp
 for (auto iter = numlist.begin(); iter != numlist.end();)
    {
        if (*iter % 2)
        {
            //删除后返回下一个迭代器
            iter = numlist.erase(iter);
        }
        else
        {
            //没删除则移动迭代器
            iter++;
        }
    }
```
## forward_list的删除和插入
对于forward_list 提供了特殊的删除操作,erase_after 和特殊的插入操作insert_after。
以下通过一次遍历实现删除forward_list中的偶数
``` cpp
   forward_list<int> numlist2 = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 0};
    // numlist2的首前元素，第一个元素之前的位置，不能解引用，因为该位置没有元素。
    auto prev = numlist2.before_begin();
    //第一个元素
    auto curr = numlist2.begin();
    while (curr != numlist2.end())
    {
        if (*curr % 2)
        {
            //返回删除元素的下一个元素
            curr = numlist2.erase_after(prev);
        }
        else
        {
            // prev更更新为curr
            prev = curr;
            // curr向后移动
            curr++;
        }
    }
```
删除操作和插入操作会导致迭代器失效，所以要时刻更新迭代器的值保证程序运行稳定。
## 容器大小
我们可以通过resize改变容器size大小，通过reverse改变容器capacity容量。
capacity是容量，容器预先开辟一块空间，capacity比size大，所以当插入元素导致容器size增加时不用每次都开辟空间。
当size大于等于capacity时，会扩充capacity。
``` cpp
    vector<string> strvec2 = {"zack", "lucy", "vivo", "rolin"};
    cout << "strvec2.size is " << strvec2.size() << endl;
    cout << "strvec2.capacity is " << strvec2.capacity() << endl;
    //插入一个元素，导致size变化
    //当size>=capacity是，capacity也会增加
    strvec2.push_back("lirus");
    cout << "strvec2.size is " << strvec2.size() << endl;
    cout << "strvec2.capacity is " << strvec2.capacity() << endl;
    //修改大小
    strvec2.resize(20);
    //修改容量
    strvec2.reserve(23);
    for (auto it = strvec2.begin(); it != strvec2.end(); it++)
    {
        cout << "data is " << *it << endl;
    }

    cout << "strvec2.size is " << strvec2.size() << endl;
    cout << "strvec2.capacity is " << strvec2.capacity() << endl;
```
程序输出
``` cmd
strvec2.size is 4
strvec2.capacity is 4
strvec2.size is 5
strvec2.capacity is 8
data is zack
data is lucy
data is vivo
data is rolin
data is lirus
data is
data is
data is
data is
data is
data is
data is
data is
data is
data is
data is
data is
data is
data is
data is
strvec2.size is 20
strvec2.capacity is 23
```
可以看出当size >= capacity时,重新开辟capacity.
resize扩充size后，容器会填充空数据，反之resize缩小size后会删除容器尾部多余数据。
## 容器适配器
stl标准库还定义了三个顺序容器适配器, stack，queue和priority_queue。
stack只要求push_back、pop_back和back操作，因此可以使用除array和forward_list之外的任何容器类型来构造stack。queue适配器要求back、push_back、front和push_front，因此它可以构造于list或deque之上，但不能基于vector构造。priority_queue除了front、push_back和pop_back操作之外还要求随机访问能力，因此它可以构造于vector或deque之上，但不能基于list构造。
``` cpp
    //基于vector构建stack
    stack<string, vector<string>> strstack;
    //基于vector构造priority_queue
    priority_queue<string, vector<string>> strpque;
    //基于list构造一个queue
    queue<list<string>> strqueue;
```
以下示例演示stack的使用
``` cpp
    //构造一个
    stack<int> intStack;
    for (int i = 0; i < 10; i++)
    {
        intStack.push(i);
    }
    // stack不为空
    while (!intStack.empty())
    {
        //访问栈顶元素但是不取出
        auto data = intStack.top();
        cout << data << endl;
        //栈顶元素出栈
        intStack.pop();
    }
```
程序输出9,8,7,6,5,4,3,2,1,0
同样stack也支持emplace操作，节省构造函数传值的开销。
队列适配器queue和栈适配器stack都是基于双端队列deque实现,而priority_queue基于vector实现。
获取队首元素用front(),弹出队首元素用pop()。
``` cpp
    //队列
    queue<int> numque;
    for (int i = 0; i < 10; i++)
    {
        numque.push(i);
    }

    while (!numque.empty())
    {
        //访问队列首元素
        auto &first = numque.front();
        cout << first << " ";
        //弹出队首元素
        numque.pop();
    }
```
程序输出0,1,2,3,4,5,6,7,8,9
