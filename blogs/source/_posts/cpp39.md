---
title: 模拟实现vector
date: 2022-06-21 21:18:14
tags: C++
categories: C++
---
## 模拟vector
我们可以通过模板实现类似vector的类。我们实现一个StrVecTemp类，其内部通过allocator开辟空间，存储的类型用T来表示，T是模板类型。
<!--more-->
``` cpp
template <typename T>
class StrVecTemp
{
public:
    StrVecTemp() : elements(nullptr), first_free(nullptr),
                   cap(nullptr) {}
    //拷贝构造函数
    StrVecTemp(const StrVecTemp &);
    //拷贝赋值运算符
    StrVecTemp &operator=(const StrVecTemp &);
    //移动构造函数
    StrVecTemp(StrVecTemp &&src) noexcept : elements(src.elements),
                                            first_free(src.first_free), cap(src.cap)
    {
        //将源数据置空
        src.elements = src.first_free = src.cap = nullptr;
    }

    template <class... Args>
    void emplace_back(Args &&...args);

    //析构函数
    ~StrVecTemp();
    //拷贝元素
    void push_back(const T &);
    //抛出元素
    void pop_back(T &s);
    //返回元素个数
    size_t size() const { return first_free - elements; }
    //返回capacity返回容量
    size_t capacity() const { return cap - elements; }
    //返回首元素的指针
    T *begin() const
    {
        return elements;
    }

    //返回第一个空闲元素指针
    T *end() const
    {
        return first_free;
    }

private:
    //判断容量不足靠皮新空间
    void chk_n_alloc()
    {
        if (size() == capacity())
        {
            reallocate();
        }
    }

    //重新开辟空间
    void reallocate();
    // copy指定范围的元素到新的内存中
    std::pair<T *, T *> alloc_n_copy(const T *, const T *);
    //释放空间
    void free();
    //数组首元素的指针
    T *elements;
    //指向数组第一个空闲元素的指针
    T *first_free;
    //指向数组尾后位置的指针
    T *cap;
    //初始化alloc用来分配空间
    static std::allocator<T> alloc;
};
template <typename T>
std::allocator<T> StrVecTemp<T>::alloc;
```
alloc在使用前要在类外初始化，因为是模板类，所以放在.h中初始化即可。
接下来我们要实现根据迭代器开始和结束的区间copy旧元素到新的空间里
``` cpp
//实现区间copy
template <typename T>
std::pair<T *, T *> StrVecTemp<T>::alloc_n_copy(const T *b, const T *e)
{
    auto newdata = alloc.allocate(e - b);
    //用旧的数据初始化新的空间
    auto first_free = uninitialized_copy(b, e, newdata);
    return {newdata, first_free};
}
```
实现copy构造
``` cpp
//实现拷贝构造函数
template <class T>
StrVecTemp<T>::StrVecTemp(const StrVecTemp &strVec)
{
    auto rsp = alloc_n_copy(strVec.begin(), strVec.end());
    //利用pair类型更新elements, cap, first_free
    elements = rsp.first;
    first_free = rsp.second;
    cap = rsp.second;
}
```
实现copy赋值
``` cpp
//拷贝赋值运算符
template <class T>
StrVecTemp<T> &StrVecTemp<T>::operator=(const StrVecTemp &strVec)
{
    if (this == &strVec)
    {
        return *this;
    }
    //如果不是自赋值，就将形参copy给自己
    auto rsp = alloc_n_copy(strVec.begin(), strVec.end());
    elements = rsp.first;
    first_free = rsp.second;
    cap = rsp.second;
}
```
析构函数要先销毁数据再回收内存
``` cpp
//析构函数
template <class T>
StrVecTemp<T>::~StrVecTemp()
{
    //判断elements是否为空
    if (elements == nullptr)
    {
        return;
    }
    //缓存第一个有效元素的地址
    auto dest = elements;
    //循环析构
    for (size_t i = 0; i < size(); i++)
    {
        //析构每一个元素
        alloc.destroy(dest++);
    }
    //再回收内存
    alloc.deallocate(elements, cap - elements);
    elements = nullptr;
    cap = nullptr;
    first_free = nullptr;
}
```
重新开辟空间
``` cpp
template <class T>
void StrVecTemp<T>::reallocate()
{
    T *newdata = nullptr;
    //数组为空的情况
    if (elements == nullptr || cap == nullptr || first_free == nullptr)
    {
        newdata = alloc.allocate(1);
        elements = newdata;
        first_free = newdata;
        // cap指向数组尾元素的下一个位置
        cap = newdata + 1;
        return;
    }

    //原数据不为空，则扩充size两倍大小
    newdata = alloc.allocate(size() * 2);
    //新内存空闲位置
    auto dest = newdata;
    //就内存的有效位置
    auto src = elements;
    //通过移动操作将旧数据放到新内存中
    for (size_t i = 0; i != size(); ++i)
    {
        alloc.construct(dest++, std::move(*src++));
    }
    //移动完旧数据后一定要删除
    free();
    //更新数据位置
    elements = newdata;
    first_free = dest;
    cap = newdata + size() * 2;
}
```
上面的函数用到了free函数，我们自己实现一个free
``` cpp
template <typename T>
void StrVecTemp<T>::free()
{
    //先判断elements是否为空
    if (elements == nullptr)
    {
        return;
    }

    auto dest = elements;
    //遍历析构每一个对象
    for (size_t i = 0; i < size(); i++)
    {
        // destroy 会析构每一个元素
        alloc.destroy(dest++);
    }
    //再整体回收内存
    alloc.deallocate(elements, cap - elements);
    elements = nullptr;
    cap = nullptr;
    first_free = nullptr;
}
```
压入元素和弹出元素
``` cpp
//拷贝元素
template <class T>
void StrVecTemp<T>::push_back(const T &t)
{
    chk_n_alloc();
    alloc.construct(first_free++, t);
}
//抛出元素
template <class T>
void StrVecTemp<T>::pop_back(T &s)
{
    //先判断是否为空
    if (first_free == nullptr)
    {
        return;
    }

    //判断size为1
    if (size() == 1)
    {
        s = *elements;
        alloc.destroy(elements);
        first_free = nullptr;
        elements = nullptr;
        return;
    }

    s = *(--first_free);
    alloc.destroy(first_free);
}
```
接下来要实现emplace_back，因为emplace_back支持多种构造函数的参数，所以要用模板参数列表的方式定义该函数。
模板参数列表和形参列表都要用参数包的方式
``` cpp
template <class T>
template <class... Args>
void StrVecTemp<T>::emplace_back(Args &&...args)
{
    chk_n_alloc();
    alloc.construct(first_free++, forward<Args>(args)...);
}
```
Args是模板参数包，args是参数列表。因为construct的参数可能为右值引用，所以要用forward<Args>将原参数列表类型原样转发。
``` cpp
// forward既扩展了模板参数包Args，又扩展了函数参数包args
// std::forward<Args>(args)... 等价于std::forward<Ti>(ti)

//比如传递给emplace_back(10,'c');
//相当于调用 alloc.construct(first_free++, forward<int>(10), forward<char>('c'))
//调用的就是插入cccccccccc
```
## 总结
本文模拟实现了vector的功能。
视频链接[https://www.bilibili.com/video/BV1Et4y1p73a/?vd_source=8be9e83424c2ed2c9b2a3ed1d01385e9](https://www.bilibili.com/video/BV1ES4y187Yc/?vd_source=8be9e83424c2ed2c9b2a3ed1d01385e9)
源码链接 [https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)