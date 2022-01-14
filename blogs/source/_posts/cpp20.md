---
title: 关联容器
date: 2022-01-11 16:20:06
tags: C++
categories: C++
---
之前我们介绍过顺序容器，list, vector, queue等，这一篇介绍关联容器, C++关联容器主要有两大类，map和set。
## map
map主要是用来管理key,value类型的结构的。
``` cpp
void use_map()
{
    map<string, size_t> word_count;
    string word;
    while (cin >> word)
    {
        word_count[word]++;
    }

    for (const auto &val : word_count)
    {
        cout << "word is " << val.first << " counts are " << val.second << endl;
    }
}
```
<!--more-->
用map管理输入的单词，当单词第一次出现时对应的value为0，再执行++操作变为1， 多次出现多次累加。
## set
set和map不同，set用来存储key，重复的key只会存储一份。
我们用set和map配合，统计不在set中出现的单词的出现次数
``` cpp
void use_set()
{
    map<string, size_t> word_count;
    set<string> excludes = {"zack", "joyce", "mongo"};
    string word;
    while (cin >> word)
    {
        if (excludes.find(word) == excludes.end())
        {
            word_count[word]++;
        }
    }

    for (const auto &val : word_count)
    {
        cout << "word is " << val.first << " counts are " << val.second << endl;
    }
}
```
## multiset与multimap
一个map或者set中的key是唯一的，multiset和multimap的key不是唯一的，也就是说同一个key可以对应多个value。
``` cpp
void use_multiset()
{
    vector<int> ivec;
    for (int i = 0; i < 10; i++)
    {
        ivec.push_back(i);
        //数据重复插入一次
        ivec.push_back(i);
    }

    set<int> iset(ivec.begin(), ivec.end());
    multiset<int> imulset(ivec.begin(), ivec.end());
    for_each(iset.begin(), iset.end(), [](const int &i)
             { cout << i << " "; });
    cout << endl;

    for_each(imulset.begin(), imulset.end(), [](const int &i)
             { cout << i << " "; });
    cout << endl;
}
```
上面的例子用ivec初始化iset和imulset，因为imulset允许key相同，所以imulset大小为20，iset大小为10。
## 关键字key为复合类型
对于关联容器,如果key为自定义的复合类型，需要为该类重载比较运算符或者定义比较函数.
类重载运算符我们放到之后的章节介绍，这里介绍定义比较函数的方式。先定义一个我们自己的图书信息类
``` cpp
class BookData
{
public:
    BookData() = default;
    BookData(string nm, string au, string is) : name(nm), author(au), isbn(is) {}
    string name;
    string author;
    string isbn;
};
//定义比较函数

bool compareIsbn(const BookData &b1, const BookData &b2)
{
    return b1.isbn < b2.isbn;
}
```
接下来我们定义一个multiset,其key为BookData类型
``` cpp
  multiset<BookData, decltype(compareIsbn) *> bookmap(compareIsbn);
```
定义了一个bookmap，类型为multiset，尖括号内部一个是key值，为BookData，另一个是比较函数指针，通过decltype自动推导函数指针类型。构造函数要传递函数对象给bookmap。
这种方式并不常用，谨在此做演示，以后学习类重载运算符才是常用作法。
## pair类型

pair的标准库类型，它定义在头文件utility中。一个pair保存两个数据成员。类似容器，pair是一个用来生成特定类型的模板。
pair主要用来map的插入操作。
``` cpp
void use_pair()
{
    //可以定义多种类型的pair
    pair<string, string> str_pair;
    pair<string, vector<int>> vec_pair;
    // pair的初始化
    //用{}初始化
    pair<string, string> author = {"zack", "fair"};
    //用make_pair初始化
    auto mkpair = make_pair("zack", 1);
    //用pair模板生成pair对象
    auto pairinit = pair<string, size_t>("zack", 1024);
    //用map的value_type
    auto vtypepair = map<string, size_t>::value_type("zack", 1023);
}
```
上述代码介绍了几种pair的初始化方式，记住一种即可。
## 关联容器的迭代器
当解引用一个关联容器迭代器时，我们会得到一个类型为容器的value_type的值的引用。对map而言，value_type是一个pair类型，其first成员保存const的关键字，second成员保存值
``` cpp
void map_iter()
{
    map<string, size_t> word_count;
    word_count["zack"] = 1024;
    auto mapit = word_count.begin();
    //*mapit是指向一个<const string, size_t>pair对象的引用
    //打印第一个元素的key值
    cout << mapit->first << endl;
    //打印第一个元素的value值
    cout << mapit->second << endl;
    //不允许修改key值，因为key值为const不允许修改
    // mapit->first = "rolin";
    //可以修改value值
    mapit->second = 222;
}
```
必须记住，一个map的value_type是一个pair，我们可以改变pair的值，但不能改变key的值。
set的迭代器是const的，因为set仅有一个key值，而key是不允许更改的，所以set获取到迭代器后无法修改其值。
``` cpp
void set_iter()
{
    set<int> iset = {1, 3, 5, 7, 9, 0, 2, 4, 6, 11};
    for (auto it = iset.begin(); it != iset.end(); it++)
    {
        //解引用获得key，key无法修改
        //(*it)++;
        cout << *it << " ";
    }
}
```
关联容器也支持遍历操作，如map的遍历
``` cpp
void map_iteration()
{
    map<string, size_t> word_count;
    word_count["zack"] = 1;
    word_count["vivo"] = 2;
    word_count["fair"] = 3;
    for (auto it = word_count.begin(); it != word_count.end(); it++)
    {
        cout << "key is " << it->first << " ,value is " << it->second << endl;
    }
}
```
## 容器操作
添加元素可以使用insert函数
可以通过insert向关联容器中插入一定范围的元素
``` cpp
void insert_set()
{
    set<int> iset;
    vector<int> ivec = {1, 3, 5, 7, 9, 1, 3, 5, 7, 9};
    iset.insert(ivec.begin(), ivec.end());
    // iset内容为1, 3, 5, 7, 9
    for_each(iset.begin(), iset.end(), [](const int &i)
             { cout << i << " "; });
    iset.insert({2, 4, 6, 8, 10, 2, 4, 6, 8, 10});
    // iset内容为1,2,3,4,5,6,7,8,9,10
    for_each(iset.begin(), iset.end(), [](const int &i)
             { cout << i << " "; });
}
```
上述代码中iset插入重复的元素会自动过滤，保证set中有且仅有不重复的key。
如下向map中添加元素
``` cpp
void insert_map()
{
    map<string, size_t> word_count;
    //通过make_pair插入
    word_count.insert(make_pair("zack", 1024));
    //通过{}插入
    word_count.insert({"rolin", 1234});
    //通过pair构造出入
    word_count.insert(pair<string, size_t>("lucas", 112));
    //通过map::value_type插入
    word_count.insert(map<string, size_t>::value_type("Lily", 2333));
}
```
insert（或emplace）返回的值依赖于容器类型和参数。对于不包含重复关键字的容器set和map等，添加单一元素的insert和emplace版本返回一个pair，告诉我们插入操作是否成功。pair的first成员是一个迭代器，指向具有给定关键字的元素；second成员是一个bool值，指出元素是插入成功还是已经存在于容器中。如果关键字已在容器中，则insert什么事情也不做，且返回值中的bool部分为false。如果关键字不存在，元素被插入容器中，且bool值为true。
``` cpp
void insert_map()
{
    map<string, size_t> word_count;
    //通过make_pair插入
    word_count.insert(make_pair("zack", 1024));
    auto res = word_count.insert(make_pair("zack", 2234));
    // insert返回一个pair，pair的first是指向给定key的迭代器
    // pair的second返回的是bool值，如果给定的key已经存在，bool值为false
    if (!res.second)
    {
        cout << res.first->first << " has been inserted into map" << endl;
    }
}
```
可以看到key为zack被插入两次，所以第二次res的second为false，res的第一个元素为key为zack的元素的迭代器，然后我们再次取迭代器的first，就是key值zack。
删除元素
``` cpp
void insert_map()
{
    map<string, size_t> word_count;
    auto delres = word_count.erase("zack");
    if (delres)
    {
        cout << "del zack success, res is " << delres << endl;
    }
    else
    {
        cout << "zack is not in map" << endl;
    }
}
```
对于保存不重复关键字的容器，erase的返回值总是0或1。若返回值为0，则表明想要删除的元素并不在容器中.
对允许重复关键字的容器，删除元素的数量可能大于1
对于map的下标操作一定要小心，当不存在key时，通过下标访问key会将key和初始的空value自动插入到map中。
我们可以通过find操作替代下标操作，这样保证不存在的key不会写入map
``` cpp
void find_map()
{
    map<string, size_t> word_count;
    auto findit = word_count.find("zack");
    if (findit != word_count.end())
    {
        cout << "find key is " << findit->first << "value is " << findit->second << endl;
    }
    else
    {
        cout << "key zack not found" << endl;
    }
}
```
对于multimap查找元素也可以通过find
``` cpp
void find_map()
{
    // authors为multimap允许key重复
    multimap<string, string> authors;
    authors.insert(make_pair("zack", "i see i know"));
    authors.insert(make_pair("zack", "who am i"));
    //对于map遍历，lambda表达式要用pair做参数
    for_each(authors.begin(), authors.end(), [](pair<string, string> a)
             { cout << a.first << " " << a.second << endl; });
    //返回包含zack的元素个数
    auto count = authors.count(string("zack"));
    //查找第一个zack出现的迭代器
    auto auit = authors.find(string("zack"));
    while (count > 0)
    {
        cout << auit->second << endl;
        auit++;
        count--;
    }
}
```
authors为multimap，map以及multimap的for_each函数要传递的lambda表达式参数为pair。

对于key为zack的元素有两个，count返回key为zack的元素个数2，authors.find(string("zack"))返回key为zack的第一个元素的迭代器位置。然后根据count控制迭代器移动依次打印就可以了。

也可以通过lower_bound和upper_bound打印一个key出现的多个元素。
lower_bound返回大于等于key值的第一个元素的迭代器，upper_bound返回大于key的迭代器，通过左闭右开的区间去遍历打印元素就可以了
``` cpp
void find_map()
{
    // authors为multimap允许key重复
    multimap<string, string> authors;
    authors.insert(make_pair("zack", "i see i know"));
    authors.insert(make_pair("zack", "who am i"));
  
    for (auto lowit = authors.lower_bound("zack"); 
    lowit != authors.upper_bound("zack"); lowit++)
    {
        cout << "key is " << lowit->first << " val is " 
        << lowit->second << endl;
    }
}
```
还有一种方法遍历multiset中查找的指定key，通过通过equal_range函数。
此函数接受一个关键字，返回一个迭代器pair。
若关键字存在，则第一个迭代器指向第一个与关键字匹配的元素，第二个迭代器指向最后一个匹配元素之后的位置。
若未找到匹配元素，则两个迭代器都指向关键字可以插入的位置。
``` cpp
void find_map()
{
    // authors为multimap允许key重复
    multimap<string, string> authors;
    authors.insert(make_pair("zack", "i see i know"));
    authors.insert(make_pair("zack", "who am i"));
  
    auto equal_range = authors.equal_range("zack");
    for (; equal_range.first != equal_range.second; equal_range.first++)
    {
        cout << "key is " << equal_range.first->first << " val is " 
        << equal_range.first->second << endl;
    }
}
```
## 无序容器
新标准定义了4个无序关联容器（unordered associative container）。这些容器不是使用比较运算符来组织元素，而是使用一个哈希函数（hash function）和关键字类型的==运算符。在关键字类型的元素没有明显的序关系的情况下，无序容器是非常有用的。在某些应用中，维护元素的序代价非常高昂，此时无序容器也很有用。
最常用的无序容器是unordered_map
可以用unordered_map重写最初的单词计数程序
``` cpp
void use_unorderd_map()
{
    unordered_map<string, size_t> word_count;
    string word;
    while (cin >> word)
    {
        word_count[word]++;
    }

    for (const auto &word : word_count)
    {
        cout << "key is " << word.first << " value is " << word.second << endl;
    }
}
```
可以看到打印的key是无序的。

无序容器在存储上组织为一组桶，每个桶保存零个或多个元素。
无序容器使用一个哈希函数将元素映射到桶。为了访问一个元素，容器首先计算元素的哈希值，它指出应该搜索哪个桶。
容器将具有一个特定哈希值的所有元素都保存在相同的桶中。
如果容器允许重复关键字，所有具有相同关键字的元素也都会在同一个桶中。
因此，无序容器的性能依赖于哈希函数的质量和桶的数量和大小。

源码连接
[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)