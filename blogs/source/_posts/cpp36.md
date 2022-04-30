---
title: C++ 函数模板和类模板
date: 2022-04-30 11:13:01
tags: C++
categories: C++
---
## 函数模板
当我们想要定义一个可以支持泛型的函数时，就要采用函数模板的方式了。所谓泛型就是可以支持多种类型的操作，比如我们定义一个compare操作，他可以根据传递给他的参数类型动态调用对应的函数版本，实现多种类型的比较。
``` cpp
template <typename T>
int compare(const T &v1, const T &v2)
{
    if (v1 < v2)
        return -1;
    if (v2 < v1)
        return 1;
    return 0;
}
```
<!--more-->
比较函数是一个模板函数，它支持T类型的对象比较，模板函数定义的规则是用template <typename T>声明模板的类型为T，然后用T做参数即可。
调用的规则传递实参就可以了，前提是实参的类型要支持比较大小，如果是类的类型我们可以重载比较运算符。
``` cpp
 int res = compare(3, 4);
    cout << "compare(3,4) res is " << res << endl;

    vector<int> v1 = {1, 3, 5};
    vector<int> v2 = {2, 4};
    res = compare(v1, v2);
    cout << "compare(v1, v2) res is " << res << endl;
```
我们分别传递了int类型和vector<int>类型的参数作为compare比较的参数。模板函数也支持多个类型，我们可以再定义一个支持多个参数类型的模板函数
``` cpp
template <typename T, typename U>
int printData(const T &t, const U &u)
{
    cout << "t is " << t << endl;
    cout << "u is " << u << endl;
}
```
调用规则和上边类似，传递两个不同类型即可
``` cpp
 printData(3.4, "hello world");
```
模板函数也支持非参数类型，用已知类型定义变量
``` cpp
template <unsigned N, unsigned M>
int compareArray(const char (&p1)[N], const char (&p2)[M])
{
    return strcmp(p1, p2);
}
```
compareArray的模板里用了已知类型unsigned定义了两个变量N和M。
调用的时候N和M会自动根据实参获取值
``` cpp
 res = compareArray("hello zack", "nice to meet u");
    cout << "compareArray("
         << "hello zack "
         << ", nice to meet u"
         << ") res is " << res << endl;
```
M和N就是传递的两个数组的长度。
## 类模板
我们实现一个模板类，使其支持类似vector的操作，包括push_back， empty， back, 以及pop_back，取索引[]操作等。
``` cpp
//定义模板类型的blob
template <typename T>
class Blob
{
public:
    typedef T value_type;
    typedef typename std::vector<T>::size_type size_type;
    //构造函数
    Blob()
    {
        data = make_shared<std::vector<T>>();
    }
    Blob(std::initializer_list<T> il)
    {
        data = make_shared<std::vector<T>>(il);
        // for (const T &m : il)
        // {
        //     data->push_back(m);
        // }
    }
    // Blob 中元素数目
    size_type size() const { return data->size(); }
    bool empty() const { return data->empty(); }
    //添加和删除元素
    void push_back(const T &t) { data->push_back(t); }
    //移动版本的push_back
    void push_back(const T &&t) { data->push_back(std::move(t)); }
    //删除元素
    void pop_back();
    //元素访问
    T &back();
    T &operator[](size_type i);

private:
    std::shared_ptr<std::vector<T>> data;
    //校验数据是否有效
    void check(size_type i, const std::string &msg) const;
};
```
我们在类外实现check， pop_back, back, 以及[]操作。
``` cpp
template <typename T>
void Blob<T>::check(size_type i, const std::string &msg) const
{
    if (i >= data->size())
        throw std::out_of_range(msg);
}

template <typename T>
void Blob<T>::pop_back()
{
    if (data->empty())
    {
        return;
    }
    data->pop_back();
}

template <typename T>
T &Blob<T>::back()
{
    return data->back();
}

template <typename T>
T &Blob<T>::operator[](size_type i)
{
    check(i, "index out of range");
    return (*data)[i];
}
```
每一个类的成员函数在类外实现时都要声明template<typename T>。
类模板的使用如下
``` cpp
void use_classtemp()
{
    Blob<int> ia;
    Blob<int> ia2 = {0, 1, 2, 3, 5};
    Blob<string> ia3 = {"hello ", "zack", "nice"};
    for (size_t i = 0; i < ia2.size(); i++)
    {
        ia2[i] = i * i;
    }

    for (size_t i = 0; i < ia2.size(); i++)
    {
        cout << ia2[i] << endl;
    }

    for (size_t i = 0; i < ia3.size(); i++)
    {
        string_upper(ia3[i]);
    }

    for (size_t i = 0; i < ia3.size(); i++)
    {
        cout << ia3[i] << endl;
    }

    const auto &data = ia3.back();
    cout << data << endl;

    ia3.pop_back();
    const auto &data2 = ia3.back();
    cout << data2 << endl;
}
```
## 总结
源码链接 [https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
视频链接 [https://www.bilibili.com/video/BV1yY4y187SC?spm_id_from=333.999.0.0](https://www.bilibili.com/video/BV1yY4y187SC?spm_id_from=333.999.0.0)