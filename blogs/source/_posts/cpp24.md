---
title: 文本查询程序
date: 2022-01-21 14:30:17
tags: C++
categories: C++
---
## 简介
本篇利用之前介绍的智能指针，关联容器，顺序容器等知识，做一个文本查询程序。该程序读取文本中的内容，然后根据输入的单词，判断该单词出现多少次，出现在哪些行，每行的句子是什么。比如我们输入elment，输出如下
``` cmd
element occurs 112 times
(line 36) A set element contains only a key;
(line 158) operator creates a new element
(line 160) Regardless of whether the element
(line 168) When we fetch an element from a map,we
(line 214) If the element is not found,find returns
```
<!--more-->
## 需求分析
1  我们需要实现一个TextQuery类，将文件的内容按照一行一行放入vector中，vector每个元素就是这一行的内容。
2  将每一行的内容按照单词分割，构造一个map，单词作为key，行号放入set，set作为value 。这样可以根据单词找到行号的set。
3  根据输入单词查找其出现的次数和行号，将结果作为一个类QueryResult返回。
所以当我们实现完成后其调用逻辑类似如下
``` cpp
void runQueries(ifstream &infile)
{
    //根据输入文件构建一个TextQuery对象
    TextQuery tq(infile);
    //根据输入的单词返回查询结果
    while (true)
    {
        cout << "enter word to look for, or q to quit" << endl;
        //输入单词查询结果，或者输入q退出循环,或者遇到终止退出。
        string s;
        if (!(cin >> s) || s == "q")
            break;
        print(cout, tq.query(s)) << endl;
    }
}
```
## 实现TextQuery类
``` cpp
class QueryResult;
class TextQuery
{
public:
    using line_no = std::vector<string>::size_type;
    TextQuery(ifstream &ifile);
    TextQuery() = default;
    QueryResult query(const string &str) const;

private:
    shared_ptr<vector<string>> file;
    map<string, shared_ptr<set<line_no>>> wm;
};
```
TextQuery声明了基于ifstream的构造函数和一个默认构造函数,同时定义了query查询函数，将查询的结果写入QueryResult类对象。
file是一个智能指针，指向一个vector，vector保存了读取文件后每行的内容。
wm是一个map，key为文件中出现的单词，value是一个智能指针，指向该单词出现的行号set。
接下来我们声明QueryResult类
``` cpp
class QueryResult
{
public:
    friend std::ostream &print(std::ostream &os, const QueryResult &qr);
    QueryResult(std::string s, shared_ptr<set<TextQuery::line_no>> p,
                shared_ptr<vector<string>> f) : sought(s), lines(p), file(f) {}

private:
    //查询的单词
    std::string sought;
    //出现的行号
    shared_ptr<set<TextQuery::line_no>> lines;
    //输入的文件
    shared_ptr<vector<string>> file;
};
```
QueryResult构造函数接受三个参数，分别是查询的单词，单词出现的行号set的指针，以及我们读取文件保存内容的vector的指针。
我们现实TextQuery的构造函数
``` cpp
TextQuery::TextQuery(ifstream &infile) : file(make_shared<vector<string>>())
{
    string text;
    //按行读取文件内容写入text
    while (getline(infile, text))
    {
        //将一行的内容写入vector file中
        file->push_back(text);
        //统计行号，vector的大小-1就是行号
        int n = file->size() - 1;
        //将字符串构造为string流
        istringstream line(text);
        string word;
        //将字符串中的单词依次写入word
        while (line >> word)
        {
            auto &lineset = wm[word];
            //如果单词不在wm中，返回的就是空指针
            if (!lineset)
            {
                //重新构造一个set，让智能指针绑定他
                lineset.reset(new set<line_no>());
            }
            lineset->insert(n);
        }
    }
}
```
构造函数读取文件，将内容写入file中，file是一个vector指针，保存了文件中每行的内容。然后分割每行字符串，将单词和其出现的行号写入wm中。wm是一个map，key就是统计的单词，value就是set的指针，set保存了该单词出现的行号序列。
接下来我们实现query查询函数
``` cpp
QueryResult TextQuery::query(const string &str) const
{
    //根据单词去map中查找
    auto findval = wm.find(str);
    //构造一个指向空的set的指针
    static auto nodata = make_shared<set<line_no>>();
    if (findval == wm.end())
    {
        return QueryResult(str, nodata, file);
    }

    return QueryResult(str, findval->second, file);
}
```
查询函数返回QueryResult对象，根据查找结果，如果找到就返回对应的行号列表等信息，否则就返回一个空行号列表。
最后我们实现print函数，打印查询结果
``` cpp
std::ostream &print(std::ostream &os, const QueryResult &qr)
{
    //依次输出单词，出现次数
    os << qr.sought << " occurs " << qr.lines->size() << " times" << endl;
    //打印出现的每一行
    for (auto num : *(qr.lines))
    {
        os << "\t line( " << num + 1 << " ) " << *(qr.file->begin() + num) << endl;
    }
    return os;
}
```
## 测试
我们新建一个文本text_query.txt，内容如下
``` bash
hello zack !!
zack , no matter where are u
i am here waiting for you 
good luck zack
```
然后我们在主函数中调用之前写好的runQueries函数
``` cpp
    std::ifstream file("./config/text_query.txt");
    runQueries(file);
```
程序运行后会提示我们输入单词，我们输入zack，会看到输出信息如下
``` cmd
enter word to look for, or q to quit
zack
zack occurs 3 times
         line( 1 ) hello zack !!
         line( 2 ) zack , no matter where are u
         line( 4 ) good luck zack

enter word to look for, or q to quit
```
## 总结
这一篇demo主要是复习之前学习的智能指针和容器的知识，通过综合运用，实现了一个文本查询功能的程序。接下来的内容更精彩。
源码链接
[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
想系统学习更多C++知识,可点击下方链接。
[C++基础](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/23uSgIjfVfmwfhNGMprDFxL0uKL)
