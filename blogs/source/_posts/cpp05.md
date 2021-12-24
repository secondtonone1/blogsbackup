---
title: vector类
date: 2021-12-15 15:51:06
categories: C++
tags: C++
---
## 简介
本文介绍vector的使用方法，vector是一种高效访问和修改的容器，支持遍历，索引访问。
## 初始化
1 用花括号进行列表初始化
2 可以用()指定初始值和个数初始化
``` cpp
void vector_init()
{
    //列表初始化
    vector<string> v1{"a", "b", "c"};
    //错误用法
    // vector<string> v2("a", "b", "c");
    //初始化vector大小为10，每个元素为-1
    vector<int> ivec(10, -1);
    // 10个string类型的元素,每个都是hi
    vector<string> svec(10, "hi!");
    // 10个元素，每个都初始化为0
    vector<int> ivec2(10);
    // 10个元素，每个都初始化为空string
    vector<string> svec2(10);
}
```
<!--more-->
## 添加元素
``` cpp
//利用push_back将元素添加到vector末尾
vector<int> v2;
for (int i = 0; i != 100; ++i)
{
    v2.push_back(i);
}
```
## 遍历访问
``` cpp
 // 求vector 每个元素平方值
vector<int> v3{1, 2, 3, 4, 5, 6, 7, 8, 9};
for (auto &i : v3)
{
    i *= i;
}
for (auto i : v3)
{
    cout << i << " ";
}
cout << endl;
```
## 下标访问
``` cpp
//索引访问
// 11个分数段，全部初始化为0
vector<unsigned> scores(11, 0);
unsigned grade;
//读取成绩
while (cin >> grade)
{
    //只处理有效成绩，小于等于100的成绩
    if (grade <= 100)
    //对应的分数段+1，修改索引对应的元素值
        ++scores[grade / 10];
}
```
