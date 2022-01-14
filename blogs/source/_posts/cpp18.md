---
title: 泛型算法定制操作
date: 2022-01-10 10:03:12
tags: C++
categories: C++
---
## 向算法传递函数
默认情况下，泛型算法还实现了另外一个版本，就是接受一个额外的参数。比如sort函数，接受第三个参数，第三个参数是一个谓词。
谓词就是一个可调用的表达式，其返回值结果是一个能用作条件的值。
标准库算法所使用的谓词分为两类：
一元谓词（unary predicate，意味着它们只接受单一参数）和二元谓词（binary predicate，意味着它们有两个参数）。
接受谓词参数的算法对输入序列中的元素调用谓词。因此，元素类型必须能转换为谓词的参数类型。
我们可以暂且将谓词理解为函数
<!--more-->
我们利用谓词，修改sort的排序规则
``` cpp
bool isShort(const string &s1, const string &s2)
{
    return s1.size() < s2.size();
}
```
上述代码将规则修改为按长度有小到大排序
接下来我们实现一个函数调用sort并传递参数isShort
``` cpp
void use_predicate()
{
    vector<string> words = {"hello", "za", "zack", "no matter", "what"};
    sort(words.begin(), words.end(), isShort);
    for (auto it = words.begin(); it != words.end(); it++)
    {
        cout << *it << endl;
    }
}
```
上面的函数输出
``` cmd
za
zack
what
hello
no matter
```
## lambda表达式
lambda表达式提供了类似函数的功能，可以理解为一个匿名函数，通过传递参数和捕获外部变量的引用，值等方式完成一些逻辑处理。
一个lambda表达式表示一个可调用的代码单元。
我们可以将其理解为一个未命名的内联函数。
与任何函数类似，一个lambda具有一个返回类型、一个参数列表和一个函数体。
但与函数不同，lambda可能定义在函数内部。
一个lambda表达式具有如下形式
``` cpp
[capture list](parameter list) -> return type {function body}
```
capture list表示捕获列表，如果lambda表达式定义在函数内部，可以通过capture list 捕获该函数的局部变量的引用或者值。
return type、parameter list和function body与任何普通函数一样，分别表示返回类型、参数列表和函数体。
我们可以忽略返回类型，lambda可以根据返回值自己推导返回类型。
``` cpp
   auto f = []()
    { return 42; };
```
此例中，我们定义了一个可调用对象f，它不接受参数，返回42。lambda的调用方式与普通函数的调用方式相同，都是使用调用运算符：
``` cpp
 cout << " f is " << f() << endl;
```
如果lambda的函数体包含任何单一return语句之外的内容，且未指定返回类型，则返回void。
我们将isShorter函数定义为lambda表达式
``` cpp
  [](const string &s1, const string &s2) -> bool
    { return s1.size() < s2.size(); };
```
我们可以通过调用stable_sort进行排序，长度相同的单词维持原序列.
``` cpp
vector<string> words = {"hello", "za", "zack", "no matter", "what"};
    stable_sort(words.begin(), words.end(), [](const string &s1, const string &s2) -> bool
                { return s1.size() < s2.size(); });
```
我们打印words
``` cpp
for (auto it = words.begin(); it != words.end(); it++)
    {
        cout << *it << endl;
    }
```
输出如下
``` cmd
za
zack
what
hello
no matter
```
我们用lambda表达式的捕获功能，实现一个函数，查找长度大于指定数值的单词个数。
我们先实现一个将单词排序并去除重复单词的函数
``` cpp
void erase_dup(vector<string> &words)
{
    //先将words中的词语排序
    sort(words.begin(), words.end());
    // unique会移动元素，将不重复的元素放在前边，重复的放在后边
    // unique返回不重复的最后一个元素的位置
    const auto uniqueiter = unique(words.begin(), words.end());
    //调用erase将重复的元素删除
    words.erase(uniqueiter, words.end());
}
```
接下来我们实现biggers函数，返回大于指定长度sz的单词的个数
``` cpp
int use_bigger(int sz)
{
    vector<string> words = {"hello", "za", "zack", "no matter", "what"};
    //先排序去除重复单词
    erase_dup(words);
    //再稳定排序，按照长度有小到大
    stable_sort(words.begin(), words.end(), [](const string &s1, const string &s2) -> bool
                { return s1.size() < s2.size(); });
    auto findit = find_if(words.begin(), words.end(), [sz](const string &s)
                          { return s.size() > sz; });

    return words.end() - findit;
}
```
我们测试下
``` cpp
    cout << "count is " << use_bigger(3) << endl;
    cout << "count is " << use_bigger(5) << endl;
    cout << "count is " << use_bigger(10) << endl;
```
程序输出如下
``` cmd
count is 4
count is 1
count is 0
```
可以看出长度大于3的单词有4个，长度大于5的有1个，长度大于10的有0个。
我们通过lambda表达式[sz]的方式捕获了use_bigger的形参sz。
如果我们要将长度大于sz的单词全部打印出来,可以采用foreach函数，该函数接受三个参数，前两个是迭代器表示遍历的范围，第三个是一个表达式，表示对每个元素的操作。我们完善use_bigger函数
``` cpp
int use_bigger(int sz)
{
    vector<string> words = {"hello", "za", "zack", "no matter", "what"};
    //先排序去除重复单词
    erase_dup(words);
    //再稳定排序，按照长度有小到大
    stable_sort(words.begin(), words.end(), [](const string &s1, const string &s2) -> bool
                { return s1.size() < s2.size(); });
    auto findit = find_if(words.begin(), words.end(), [sz](const string &s)
                          { return s.size() > sz; });
    for_each(findit, words.end(), [](const string &s)
             { cout << s << " "; });
    cout << endl;
    return words.end() - findit;
}
```
## lambda捕获类型
lambda捕获分为值捕获和引用捕获。
lambda采用值捕获的方式。与传值参数类似，采用值捕获的前提是变量可以拷贝。
与参数不同，被捕获的变量的值是在lambda创建时拷贝，而不是调用时拷贝。
``` cpp
void lambda_catch()
{
    int val = 10;
    auto fn = [val]
    { return val; };
    val = 200;
    auto fv = fn();
    cout << "fv is " << fv << endl;
}
```
上述代码fv会输出10，因为fn捕获的是val的值，在lambda表达式创建时就捕获了val，此时val值为10.
如果采用引用方式捕获
``` cpp
void lambda_catch_r()
{
    int val = 10;
    auto fn = [&val]
    { return val; };
    val = 200;
    auto fv = fn();
    cout << "fv is " << fv << endl;
}
```
此时输出fv is 200， 因为fn捕获的是val的引用。
我们可以从一个函数返回lambda，此lambda不能包含引用捕获。因为如果lambda包含了函数局部变量的引用，当次局部变量被释放后，lambda调用会出现崩溃问题。

捕获一个普通变量，如int、string或其他非指针类型，通常可以采用简单的值捕获方式。
在此情况下，只需关注变量在捕获时是否有我们所需的值就可以了。
如果我们捕获一个指针或迭代器，或采用引用捕获方式，就必须确保在lambda执行时，绑定到迭代器、指针或引用的对象仍然存在。
而且，需要保证对象具有预期的值。
在lambda从创建到它执行的这段时间内，可能有代码改变绑定的对象的值。
也就是说，在指针（或引用）被捕获的时刻，绑定的对象的值是我们所期望的，但在lambda执行时，该对象的值可能已经完全不同了。
一般来说，我们应该尽量减少捕获的数据量，来避免潜在的捕获导致的问题。而且，如果可能的话，应该避免捕获指针或引用。
## 隐式捕获
为了指示编译器推断捕获列表，应在捕获列表中写一个&或=。&告诉编译器采用捕获引用方式，=则表示采用值捕获方式。
比如我们修改use_bigger函数，参数增加一个ostream和char的分隔符，在use_bigger内部利用for_each调用
``` cpp
int use_bigger2(ostream &os, char c, int sz)
{
    vector<string> words = {"hello", "za", "zack", "no matter", "what"};
    //先排序去除重复单词
    erase_dup(words);
    //再稳定排序，按照长度有小到大
    stable_sort(words.begin(), words.end(), [](const string &s1, const string &s2) -> bool
                { return s1.size() < s2.size(); });
    auto findit = find_if(words.begin(), words.end(), [sz](const string &s)
                          { return s.size() > sz; });
    // os 按照引用方式捕获，其余变量c 通过= 值方式隐士捕获。
    for_each(findit, words.end(), [=, &os](const string &s)
             { os << s << c; });

    // c 按照值的方式捕获，其余按照引用方式捕获。
    for_each(findit, words.end(), [&, c](const string &s)
             { os << s << c; });
    cout << endl;
    return words.end() - findit;
}
```
上述代码两个for_each通过不同的隐式方式捕获局部变量。
## mutable改变值
默认情况下，值捕获的变量，lambda不会改变其值。lambda可以声明mutable，这样可以修改捕获的变量值。
``` cpp
void mutalble_lam()
{
    int val = 100;
    auto fn = [val]() mutable
    {
        return ++val;
    };

    cout << "val is " << val << endl;

    cout << "fn val is " << fn() << endl;

    val = 200;
    cout << "val is " << val << endl;

    cout << "fn val is " << fn() << endl;
}
```
程序输出
``` cmd
val is 100
fn val is 101
val is 200
fn val is 102
```
fn捕获val的值，因为fn是mutable所以可以修改val，但不会影响外界的val。
## lambda返回类型
我们要做一个返回序列中数值的绝对值的函数
``` cpp
void rt_lambda()
{
    vector<int> nums = {-1, 2, 3, -5, 6, 7, -9};
    transform(nums.begin(), nums.end(), nums.begin(), [](int a)
              { return a < 0 ? -a : a; });
    for_each(nums.begin(), nums.end(), [](int a)
             { cout << a << endl; });
}
```
通过transform将nums中的数值全部变为其绝对值。transform前两个参数表示输入序列，第三个参数表示写入的目的序列，如果目的序列迭代器和输入序列开始的迭代器相同，则表示transform序列全部元素。lambda表达式并没有写返回值类型，但是是一个三目运算符的表达式，所以lambda可以推断返回类型。如果将lambda表达式写成如下会报错。
``` cpp
[](int a){ if(a<0) return -a; else return a; }
```
此时我们修改上面的lambda表达式，明确写出返回类型为int
``` cpp
 transform(nums.begin(), nums.end(), nums.begin(), [](int a) -> int
              { if (a < 0)  return -a; else return a; });
```
## bind绑定参数
bind的形式为
``` cpp
auto newCallable = bind(callable, arg_list);
```
newCallable本身是一个可调用对象，arg_list是一个逗号分隔的参数列表，对应给定的callable的参数。
即，当我们调用newCallable时，newCallable会调用callable，并传递给它arg_list中的参数。

arg_list中的参数可能包含形如_n的名字，其中n是一个整数。这些参数是“占位符”，表示newCallable的参数，它们占据了传递给newCallable的参数的“位置”。
数值n表示生成的可调用对象中参数的位置：_1为newCallable的第一个参数，_2为第二个参数，依此类推。
我们先实现一个判断字符串长度的函数
``` cpp
bool check_size(const string &str, int sz)
{
    if (str.size() > sz)
    {
        return true;
    }

    return false;
}
```
check_size如果字符串str的长度大于sz就返回true，否则就返回false。
接下来用bind操作生成一个新的函数，只接受一个sz参数。
使用bind函数要包含头文件functional，也需要使用using namespace std::placeholders
``` cpp
void calsize_count()
{
    string str = "hello";
    //将check_size第一个参数绑定给bind_check
    auto bind_check = bind(check_size, _1, 6);
    //相当于调用check_size(str,6)
    bool bck = bind_check(str);
    if (bck)
    {
        cout << "check res is true" << endl;
    }
    else
    {
        cout << "check res is false" << endl;
    }
}
```
通过bind将check_size第一个参数绑定给bind_check，第二个参数为6
所以调用bind_check(str)相当于调用check_size(str,6)。
我们可以用bind方式实现find_if的查找，因为find_if接受的谓词只能有一个参数，所以通过bind将check_size生成为单参数函数。
``` cpp
int use_bigger3(ostream &os, char c, int sz)
{
    vector<string> words = {"hello", "za", "zack", "no matter", "what"};
    //先排序去除重复单词
    erase_dup(words);
    //再稳定排序，按照长度有小到大
    stable_sort(words.begin(), words.end(), [](const string &s1, const string &s2) -> bool
                { return s1.size() < s2.size(); });
    auto findit = find_if(words.begin(), words.end(), bind(check_size, _1, sz));

    // c 按照值的方式捕获，其余按照引用方式捕获。
    for_each(findit, words.end(), [&, c](const string &s)
             { os << s << c; });
    cout << endl;
    return words.end() - findit;
}
```
通过bind生成新的函数传递给find_if就可以使用了。
bind极大地方便了泛型编程的可扩展性。

