---
title: 泛型算法
date: 2022-01-06 17:10:44
tags: C++
categories: C++
---
## 泛型算法
泛型算法是STL库里面定义的一些算法,这些算法可以用一个接口操作各种数据类型,因此成为泛型算法。大多算法定义在头文件algorithm和numeric中。意思就是可以用一个接口操作各种类型的算法就是泛型算法。
<!--more-->
泛型算法分为两类，一类是只读算法，一类是修改原有容器的算法。
只读算法包括find(查找),accumulate(累加)等。
修改算法包括replace(替换),fill(填充)等。
## accumulate
``` cpp
    vector<int> nvec = {1, 2, 3, 4, 5, 6, 7};
    //调用accumulate累加，sum的初始值为0，累加结果写入sum
    auto sum = accumulate(nvec.begin(), nvec.end(), 0);
    cout << "sum is " << sum << endl;
    list<string> strlist = {"hello", "zack", "good", "idea"};
    string stradd = accumulate(strlist.begin(), strlist.end(), string(""));
    cout << "str add result is " << stradd << endl;
```
accumulate将第三个参数作为求和起点，这蕴含着一个编程假定：将元素类型加到和的类型上的操作必须是可行的。即，序列中元素的类型必须与第三个参数匹配，或者能够转换为第三个参数的类型。在上例中，vec中的元素可以是int，或者是double、long long或任何其他可以加到int上的类型。由于string定义了+运算符，所以我们可以通过调用accumulate来将vector中所有string元素连接起来.
程序输出
``` cpp
sum is 28
str add result is hellozackgoodidea
```
## equal
泛型算法中有操作两个序列的算法，比如equal就是比较两个序列中元素是否有相等的值，如果第一个序列中每个元素与第二个序列中的元素都相等，则返回true，否则返回false。
``` cpp
 bool bequa = equal(strlist.begin(), strlist.end(), strlist2.begin());
    if (bequa)
    {
        cout << "strlist is equal to strlist2" << endl;
    }
    else
    {
        cout << "strlist is not equal to strlist2" << endl;
    }
```
上述代码比较了strlist和strlist2，切记strlist2的长度要大于等于strlist，否则程序会出现问题。那些只接受一个单一迭代器来表示第二个序列的算法，都假定第二个序列至少与第一个序列一样长。
## fill
可以通过fill算法修改容器的值,算法fill接受一对迭代器表示一个范围，还接受一个值作为第三个参数。fill将给定的这个值赋予输入序列中的每个元素。
``` cpp
    vector<int> nvec2 = {1, 2, 3, 4};
    //将nvec2中所有元素设置为0
    fill(nvec2.begin(), nvec2.end(), 0);
    //将nvec2中前半部分设置为10
    fill(nvec2.begin(), nvec2.begin() + nvec2.size() / 2, 10);
```
类似的还有fill_n函数，该函数接受一个单迭代器，一个计数值和一个值。
``` cpp
    vector<int> vec;
    fill_n(vec.begin(), vec.size(), 0);
```
如下调用fill会导致程序崩溃，因为vec3大小为0，而fill要向vec3写入10个0，会造成越界崩溃。
``` cpp
//空向量
vector<int> vec3;
// 灾难，修改vec3中的10个不存在元素
fill_n(vec3.begin(),10,0);
```
## back_inserter
fill_n如果传递的个数大于容器的大小会造成崩溃，为了防止类似的问题，stl引入了back_inserter。back_inserter接受一个指向容器的引用，返回一个与该容器绑定的插入迭代器。当我们通过此迭代器赋值时，赋值运算符会调用push_back将一个具有给定值的元素添加到容器中：
``` cpp
    //空vector
    vector<int> nvec4;
    // back_inserter绑定nvec4并返回迭代器
    auto iter = back_inserter(nvec4);
    //对迭代器的赋值就是对nvec插入元素
    *iter = 2;
    *iter = 4;
    for (auto it = nvec4.begin(); it != nvec4.end(); it++)
    {
        cout << *it << " ";
    }
```
程序依次打印输出2, 4
我们常常使用back_inserter来创建一个迭代器，作为算法的目的位置来使用。例如：
``` cpp
    //空vector
    vector<int> nvec5;
    // back_inserter绑定nvec5并返回迭代器
    auto iter5 = back_inserter(nvec5);
    //添加10个元素写入nvec5
    fill_n(iter5, 10, 0);
```
## copy
拷贝（copy）算法是另一个向目的位置迭代器指向的输出序列中的元素写入数据的算法。此算法接受三个迭代器，前两个表示一个输入范围，第三个表示目的序列的起始位置。此算法将输入范围中的元素拷贝到目的序列中。传递给copy的目的序列至少要包含与输入序列一样多的元素。
``` cpp
    int a1[] = {0, 1, 2, 3, 4, 5, 6};
    constexpr int nszie = sizeof(a1) / sizeof(int);
    int a2[nszie];
    //将a1内容copy到a2中
    copy(begin(a1), end(a1), a2);
```
## replace
我们可以通过replace替换原容器中的某个值为设定的新值
``` cpp
    vector<int> nvec6 = {1, 2, 3, 4};
    //将nvec6中所有元素为3的设置为32
    replace(nvec6.begin(), nvec6.end(), 3, 32);
```
如果保留原容器数据，可以通过back_inserter绑定一个新的容器，然后用replace_copy完成拷贝和替换操作
``` cpp
    //原始数据列表
    list<int> ilist = {0, 1, 2, 3, 4, 5};
    //空向量
    vector<int> rcpvec;
    //将ilist中的数据copy到rcpvec里，但是将其中的0替换为42
    replace_copy(ilist.begin(), ilist.end(), back_inserter(rcpvec), 0, 42);
```
## unique和sort
我们实现一个功能，将vector中的单词排序并且去除其中重复的单词。
我们可以用sort函数先将vector中的单词排序，然后用unique去除重复的单词，unique返回不重复的最后一个元素的迭代器，unique保证容器中前n个元素是不重复的，n+1个开始就是重复的，所以我们用erase再删除n+1个元素以后的内容就可以了。
``` cpp
    vector<string> words = {"good", "idea", "zack", "lucy", "good", "idea"};
    //先将words中的词语排序
    sort(words.begin(), words.end());
    // unique会移动元素，将不重复的元素放在前边，重复的放在后边
    // unique返回不重复的最后一个元素的位置
    const auto uniqueiter = unique(words.begin(), words.end());
    //调用erase将重复的元素删除
    words.erase(uniqueiter, words.end());
```
打印输出words可以看到words变为{good , idea , lucy , zack}