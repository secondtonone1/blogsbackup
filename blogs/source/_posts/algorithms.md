---
title: 双链表实现LRU算法
date: 2021-12-29 17:50:00
categories: [数据结构和算法]
tags: [数据结构和算法]
---
## 链表
所谓链表就是一个节点指向另一个节点的数据结构，像一条链子把每个节点连接起来。
如果一个节点既指向了后面的节点，也指向了前面的节点，这些节点就构成了双向链表。
我们先定义这个节点结构
``` cpp
class NodeLRU
{
public:
    NodeLRU(int v) : val(v), prev(nullptr), next(nullptr) {}

    int val;
    NodeLRU *prev;
    NodeLRU *next;
};
```
val为节点的数据域表示节点存储的数值
prev为节点的前一个节点
next为节点的后一个节点
然后我们把这些节点串连起来，定义一个链表的结构
<!--more-->
一个双向链表的结构图
![https://cdn.llfc.club/1640830682%281%29.png](https://cdn.llfc.club/1640830682%281%29.png)

现在我们实现LRU算法:
1  插入一个节点数据域为val，如果链表中有和val匹配的节点，则将该节点移动到链表头,如果没有该节点则直接插入头部。
2  在执行1操作时，如果链表长度已经达到最大，则删除尾节点，在头部插入。

接下来我们实现LRU类，通过插入节点实现LRU算法。
``` cpp
class LRU
{
public:
    LRU(int sz) : max_size(sz), cur_size(0), head(nullptr), tail(nullptr) {}
    LRU() = default;
    void insert(int val)
    {
        if (max_size == 0)
        {
            return;
        }

        if (head == tail && head == nullptr)
        {
            auto node = new NodeLRU(val);
            head = node;
            tail = node;
            cur_size++;
            return;
        }

        auto node_find = find(val);
        if (node_find == nullptr)
        {
            //节点数量达到上限
            if (cur_size >= max_size)
            {
                //删除尾节点
                auto deltail = tail;
                if (tail->prev != nullptr)
                {
                    tail->prev->next = nullptr;
                }

                tail = tail->prev;
                delete (deltail);
                cur_size--;
            }

            //插入头部节点
            auto node = new NodeLRU(val);
            node->next = head;
            head->prev = node;
            head = node;
            cur_size++;
            return;
        }

        //头部节点就不处理
        if (node_find == head)
        {
            return;
        }

        //如果是尾部节点
        if (node_find == tail)
        {
            node_find->prev->next = nullptr;
            tail = node_find->prev;
        }
        else
        {
            //先将node_find节点前后节点连接起来
            node_find->prev->next = node_find->next;
            node_find->next->prev = node_find->prev;
        }

        //在将node_find节点插入头部
        node_find->prev = nullptr;
        node_find->next = head;
        head = node_find;
    }

    NodeLRU *find(int sz)
    {
        auto node = head;
        while (node != nullptr)
        {
            if (node->val == sz)
            {
                return node;
            }
            node = node->next;
        }

        return nullptr;
    }

    void print()
    {
        for (auto node = head; node != nullptr; node = node->next)
        {
            cout << node->val << " -> ";
        }

        cout << "null" << endl;
    }

    ~LRU()
    {
        for (auto node = head; node != nullptr;)
        {
            auto nodedel = node;
            node = node->next;
            delete (nodedel);
        }
        head = nullptr;
        tail = nullptr;
    }

private:
    NodeLRU *head;
    NodeLRU *tail;
    int max_size;
    int cur_size;
};
```

max_size 表示链表最大长度，cur_size表示链表当前长度，head表示链表的头结点，tail表示链表的尾节点。

接下来我们测试LRU类
``` cpp
    auto lru = LRU(3);
    lru.insert(4);
    lru.insert(3);
    lru.insert(2);
    lru.print();
    lru.insert(100);
    lru.print();
    lru.insert(10);
    lru.print();
```
程序输出
``` cpp
2 -> 3 -> 4 -> null
100 -> 2 -> 3 -> null
10 -> 100 -> 2 -> null
```
源码链接[https://gitee.com/secondtonone1/algorithms](https://gitee.com/secondtonone1/algorithms)
我的公众号，谢谢关注
![https://cdn.llfc.club/gzh.jpg](https://cdn.llfc.club/gzh.jpg)
