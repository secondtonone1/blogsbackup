---
title: 递归反转单链表
date: 2022-01-07 16:16:28
categories: [数据结构和算法]
tags: [数据结构和算法]
---
## 递归反转
本文介绍递归反转单链表，和之前的循环遍历反转单链表方式略有不同，递归的方式要写出推到递归公式。并且在递归的同时修改指针的指向。
先定义Node节点
``` cpp
class Node
{
public:
    Node(int dt, Node *nt = nullptr) : data(dt), next(nt) {}
    Node() = default;
    int data;
    Node *next;
};
```
<!--more-->
Node节点构成的链表如下图
![https://cdn.llfc.club/1641544598%281%29.jpg](https://cdn.llfc.club/1641544598%281%29.jpg
)

基本思路是实现recursive_reverse函数

``` cpp
Node* recursive_reverse(Node* p){
    //判断p为空或者是单节点直接返回
    if(p == nullptr || p->next==nullptr){
        return ;
    }

    //递归查找，直到返回队尾元素作为头节点
    //否则继续递归处理下一个节点
    auto nextnode = recursive_reverse(p->next);
    return nextnode;
}
```
上述代码最终会将链表的尾节点返回。但是我们没有完成链表的逆转，需要改变链表节点Node的next指针。
假设有两个节点
![https://cdn.llfc.club/1641545121%281%29.jpg](https://cdn.llfc.club/1641545121%281%29.jpg)
node1节点为p节点
`p->next`指向的就是node2节点
`p->next->next`指向的就是末尾的空指针。
所以当有两个节点的时候我们可以如下操作完成p和`p->next`两个节点指向的修改，也就是node1和node2两个节点的修改。
我们将`p->next->next = p->next` 就是将node2的next指向了node1.
我们将`p->next = nullptr `就是将`node1->next`指向了空地址。所以node1此时变为尾节点。
而之前的递归保证了最后返回的是node2节点，此时node2节点就作为头节点，从而完成了逆转。
图解为下图
![https://cdn.llfc.club/1641545895.jpg](https://cdn.llfc.club/1641545895.jpg)
这是两个节点的情况，如果是三个节点呢，那就依次类推，依次完成`node3->next`指向node2，`node2->next`指向node1，`node1->next`指向空地址。
所以n>=2的情况都是和两个节点类似的，那么我们补全recursive_reverse函数
``` cpp
Node *recursive_reverse(Node *p)
{
    //如果链表为空或者为单节点，直接返回
    if (p == nullptr || p->next == nullptr)
    {
        return p;
    }
    //否则继续递归处理下一个节点
    auto nextnode = recursive_reverse(p->next);
    //改变p的下一个节点next指向，链表逆转
    p->next->next = p;
    //改变p的next指向
    p->next = nullptr;
    return nextnode;
}
```
接下来我们实现一个创建链表的函数用来测试
``` cpp
Node *createList()
{
    auto node1 = new Node(1);
    auto node2 = new Node(2);
    auto node3 = new Node(3);
    node1->next = node2;
    node2->next = node3;
    return node1;
}
```
然后实现销毁链表回收内存的函数
``` cpp
void delocateList(Node *p)
{
    while (p != nullptr)
    {
        auto temp = p;
        p = p->next;
        delete temp;
    }
}
```
然后实现打印节点的函数
``` cpp
void printList(Node *p)
{
    while (p != nullptr)
    {
        cout << p->data << " -> ";
        p = p->next;
    }

    cout << " nullptr " << endl;
}
```
最后我们在main函数中测试
``` cpp
    auto list = createList();
    printList(list);
    list = recursive_reverse(list);
    printList(list);
    delocateList(list);
```
程序输出如下
``` cmd
1 -> 2 -> 3 ->  nullptr
3 -> 2 -> 1 ->  nullptr
```
可以看到链表被逆转了。
源码链接[https://gitee.com/secondtonone1/algorithms](https://gitee.com/secondtonone1/algorithms)
我的公众号，谢谢关注
![https://cdn.llfc.club/gzh.jpg](https://cdn.llfc.club/gzh.jpg)