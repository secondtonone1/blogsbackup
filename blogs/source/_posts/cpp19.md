---
title: 迭代器分类
date: 2022-01-11 10:29:49
tags: C++
categories: C++
---
除了容器自定义的迭代器之外，标准库还提供了其他几种迭代器，包括插入迭代器，流迭代器，反向迭代器，移动迭代器。
## 插入迭代器
迭代器被绑定到一个容器上，可用来向容器插入元素。插入迭代器包括back_inserter, front_inserter,  inserter三种。
back_inserter绑定到容器后，对该迭代器赋值，就执行了类似于push_back的操作，前提是该容器要支持push_back。
front_inserter绑定到容器后，对该迭代器赋值，就执行了类似于push_front的操作，前提是该容器支持push_front。
inserter创建一个使用insert的迭代器，此函数接受第二个参数必须是一个指向给定容器的迭代器，元素被插入到给定迭代器所表示的元素之前。
<!--more-->
``` cpp
void use_inserter()
{
    list<int> list1 = {1, 2, 3, 4};
    list<int> list2, list3, list4;
    copy(list1.begin(), list1.end(), front_inserter(list2));
    copy(list1.begin(), list1.end(), back_inserter(list3));
    copy(list1.begin(), list1.end(), inserter(list4, list4.begin()));

    for_each(list2.begin(), list2.end(), [](const int &i)
             { cout << i << " "; });
    cout << endl;

    for_each(list3.begin(), list3.end(), [](const int &i)
             { cout << i << " "; });
    cout << endl;

    for_each(list4.begin(), list4.end(), [](const int &i)
             { cout << i << " "; });
    cout << endl;
}
```
程序输出
``` cmd
4 3 2 1
1 2 3 4
1 2 3 4
```
## iostream迭代器
istream_iterator 读取输入流，ostream_iterator 向一个输出流写数据。
这些迭代器将它们对应的流当作一个特定类型的元素序列来处理。
通过使用流迭代器，我们可以用泛型算法从流对象读取数据以及向其写入数据。
我们先看看输入流的操作:
``` cpp
void use_istreamiter()
{
    //输入流迭代器
    istream_iterator<int> in_int(cin);
    //迭代器终止标记
    istream_iterator<int> in_eof;
    vector<int> in_vec;
    while (in_int != in_eof)
    {
        in_vec.push_back(*in_int++);
    }
    for_each(in_vec.begin(), in_vec.end(), [](const int &i)
             { cout << i << " "; });
    cout << endl;
}
```
上述代码创建了输入流迭代器in_int，绑定了cin。
同时生成了一个输入流的结尾迭代器in_eof，in_eof未绑定任何输入流，所以是输入流的终止。
通过循环将输入流数据写入in_vec中。
再看看输出流的操作:
``` cpp
void use_ostreamiter()
{
    vector<int> in_vec = {1, 3, 4, 2, 5, 6, 7, 9};
    ostream_iterator<int> out_in(cout, " ");
    for (auto data : in_vec)
    {
        *out_in++ = data;
    }
    cout << endl;
}
```
输出流迭代器out_in和cout绑定，并且为每一个输出的元素设置了空格间隔。
通过向*out_in赋值达到向cout写入数据的目的，同时out_in++保证了迭代器的后移。
## 反向迭代器
反向迭代器就是在容器中从尾元素向首元素反向移动的迭代器。对于反向迭代器，递增（以及递减）操作的含义会颠倒过来。递增一个反向迭代器（++it）会移动到前一个元素；递减一个迭代器（--it）会移动到下一个元素
cbegin和cend表示正向迭代器，crbegin和crend表示反向迭代器,如下图
![https://cdn.llfc.club/1641885505%281%29.jpg](https://cdn.llfc.club/1641885505%281%29.jpg)
我们通过反向迭代器逆序打印原容器中的数据
``` cpp
void use_reverseiter()
{
    vector<int> in_vec = {1, 3, 4, 2, 5, 6, 7, 9};
    for (auto rit = in_vec.crbegin(); rit != in_vec.crend(); rit++)
    {
        cout << *rit << " ";
    }
    cout << endl;
}
```
程序输出9 7 6 5 2 4 3 1
我们知道sort默认规则是从小到大，如果我们想实现从大到小，可以利用反向迭代器完成
``` cpp
sort(vec.rbegin(), vec.rend());
```
反向迭代器遍历是从后往前，这一点也会造成一些不必要的问题
``` cpp
    string line = "FIRST,MIDDLE,LAST";
    auto rcomma = find(line.crbegin(), line.crend(), ',');
    cout << string(line.crbegin(), rcomma) << endl;
```
上述代码会找到最后一个逗号，获取crbegin和rcomma之间的数据实际是TSAL,也就是说反向迭代器遍历是反向的。
标准库提供了一个将反向迭代器转化为正向迭代器的方法base()
``` cpp
    //通过base将反向迭代器转化为正向的
    cout << string(rcomma.base(), line.cend()) << endl;
```
程序输出LAST
大家可以看一下正向迭代器和反向迭代器的关系图
![https://cdn.llfc.club/1641887218.jpg](https://cdn.llfc.club/1641887218.jpg)
