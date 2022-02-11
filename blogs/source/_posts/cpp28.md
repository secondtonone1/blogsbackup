---
title: C++类的拷贝控制demo
date: 2022-02-07 10:18:25
tags: C++
categories: C++
---
## 拷贝控制
有时候我们需要两个类对象互相关联，当其中一个对象修改后也要关联修改另一个，用这个例子说明拷贝控制的案例。我们有两个类，Message类表示信息类，Folder类表示文件夹类，Message类里有成员folders表示其所属于哪些文件夹。Folder类有成员messages表示其包含哪些messages，所以Folder和Message之间是互相包含，多对多的关系。
同时我们要考虑Message类的拷贝，赋值，销毁等操作，如何同步处理其关联的Folder类。
<!--more-->
其关系图是这样的
![https://cdn.llfc.club/1644226908%281%29.jpg](https://cdn.llfc.club/1644226908%281%29.jpg)
``` cpp
class Message
{
    friend class Folder;

public:
    // folder被隐式初始化为空集合
    explicit Message(const std::string &str = "") : contents(str) {}
    // 拷贝控制成员，用来管理指向本Message的指针
    Message(const Message &);
    // 拷贝赋值运算符
    Message &operator=(const Message &);
    //析构函数
    ~Message();
    //将Message保存在指定Folder中
    void save(Folder &);
    //从Folder中删除Message
    void remove(Folder &);

private:
    //  消息内容
    std::string contents;
    // 消息所属文件夹
    std::set<Folder *> folders;
    //将本Message添加到参数msg的folder中
    void add_to_Folders(const Message &msg);
    //从folders中的每个Folder删除本Message
    void remove_from_Folders();
};
```
Message类定义了构造函数，默认将本Message所属的Folder集合设置为空。同时提供了save和remove操作，将本Message保存给指定Folder以及从指定Folder中删除。两个私有函数在很多地方通用，所以提出来作为私有函数。
同样我们声明Folder类
``` cpp
class Folder
{
    friend class Message;

public:
    explicit Folder(const std::string &nm = "") : name(nm) {}
    //拷贝控制成员
    Folder(const Folder &);
    //拷贝赋值运算符
    Folder &operator=(const Folder &);
    //析构函数
    ~Folder();
    //保存指定的msg
    void addMsg(Message &);
    //移除指定的msg
    void remMsg(Message &);

private:
    //文件夹名字
    std::string name;
    //包含的消息列表
    std::unordered_map<std::string, Message *> msgs;
};
```
接下来我们实现Message类的添加和删除操作
``` cpp
//将Message保存在指定Folder中
void Message::save(Folder &f)
{
    //将文件夹f添加到msg的folders里
    this->folders.insert(&f);
    //将本msg添加到folder中
    f.addMsg(*this);
}
//从Folder中删除Message
void Message::remove(Folder &f)
{
    //将文件夹从msg的folders里删除
    this->folders.erase(&f);
    f.remMsg(*this);
}
```
接下来我们实现folder的addMsg和remMsg操作
``` cpp
//保存指定的msg
void Folder::addMsg(Message &msg)
{
    this->msgs.insert(make_pair(msg.contents, &msg));
}
//移除指定的msg
void Folder::remMsg(Message &msg)
{
    this->msgs.erase(msg.contents);
}
```
上述代码完成了msg插入folder后两个类之间的关联逻辑，当msg之间拷贝构造时需要完成folder的拷贝
``` cpp
//将本Message添加到参数msg的folder中
void Message::add_to_Folders(const Message &msg)
{
    for (auto f : msg.folders)
    {
        f->addMsg(*this);
    }
}

//拷贝构造函数将m的folders拷贝给自己
Message::Message(const Message &m)
{
    contents = m.contents;
    folders = m.folders;
    add_to_Folders(m);
}
```
拷贝构造函数就是将参数message的成员拷贝给新生成的对象，然后通过add_to_Folders函数将本消息添加到m的folders中。
接下来实现析构函数，message析构时将folders遍历移除本message
``` cpp
//从folders中的每个Folder删除本Message
void Message::remove_from_Folders()
{
    for (auto f : folders)
    {
        f->remMsg(*this);
    }
}

//析构函数
Message::~Message()
{
    remove_from_Folders();
}
```
拷贝构造函数需要两个操作，将自身的folders中删除本msg，然后将=右侧的message的folders赋值给本msg，并且将本message添加到folders中。其实是融合了析构和拷贝构造的两个操作。
``` cpp
// 拷贝赋值运算符
Message &Message::operator=(const Message &msg)
{
    remove_from_Folders();
    contents = msg.contents;
    folders = msg.folders;
    add_to_Folders(msg);
    return *this;
}
```
在有些时候会用到swap操作，比如sort排序等，我们也实现一个Message版本的swap
``` cpp
void swap(Message &lhs, Message &rhs)
{
    //将lhs从关联的folders中移除
    lhs.remove_from_Folders();
    //将rhs从关联的folders中移除
    rhs.remove_from_Folders();
    //交换两个成员
    swap(lhs.contents, rhs.contents);
    swap(lhs.folders, rhs.folders);
    //重新将lhs添加到关联的folders中
    lhs.add_to_Folders(lhs);
    //重新将rhs添加到关联的folders中
    rhs.add_to_Folders(rhs);
}
```
同样的道理，为实现folders的拷贝构造我单独实现一个私有函数add_msgs，将其
``` cpp
// 将f中的msgs添加到本folder中
void Folder::add_msgs(const Folder &f)
{
    //将f的msgs添加到本folder
    for (auto mp : f.msgs)
    {
        //将消息保存在本folder
        mp.second->save(*this);
    }
}

Folder::Folder(const Folder &f)
{
    this->msgs = f.msgs;
    this->name = f.name;
    add_msgs(f);
}
```
实现folders的析构函数,同样我也实现了一个私有函数remove_msgs
``` cpp
//删除folder中的所有msgs
void Folder::remove_msgs()
{
    for (auto msgit = this->msgs.begin(); msgit != this->msgs.end();)
    {
        msgit->second->folders.erase(this);
        msgit = this->msgs.erase(msgit);
    }
}

//析构函数
Folder::~Folder()
{
    remove_msgs();
    msgs.clear();
}
```
同样道理拷贝赋值运算符的重载逻辑是拷贝构造和析构的综合
``` cpp
//拷贝赋值运算符
Folder &Folder::operator=(const Folder &f)
{
    // 先从本folder的msg解除和本folder的关联
    remove_msgs();
    //再将f的参数赋值给本folder
    this->name = f.name;
    this->msgs = f.msgs;
    //将msgs添加到本folder
    add_msgs(f);
}
```
同样我们实现Folder的swap函数
``` cpp
void swap(Folder &lf, Folder &rf)
{
    //先解除lf的msg和其的联系
    lf.remove_msgs();
    //再解除rf的msg和其的联系
    rf.remove_msgs();
    //交换数据结构
    swap(lf.msgs, rf.msgs);
    swap(lf.name, rf.name);
    //最后各自绑定msg和其folder的联系
    lf.add_msgs(lf);
    rf.add_msgs(rf);
}
```
接下来我们写一个函数测试上述赋值，析构，构造是否存在问题
``` cpp
void test_msgfolder()
{
    auto f1 = new Folder("folder1");
    auto msg1 = new Message("msg1");
    msg1->save(*f1);
    auto f2 = new Folder("folder2");
    auto msg2 = new Message("msg2");
    msg2->save(*f2);
    auto f3 = new Folder(*f1);
    *f3 = *f2;
    swap(f1, f2);
    delete (f1);
    delete (msg1);
    delete (f2);
    delete (msg2);
    delete (f3);
}
```
上面的测试函数，创建了三个Folder对象和两个Message对象，调用了对象的拷贝构造，拷贝赋值以及析构等操作，在主函数中调用测试函数，程序运行稳定。
## 总结
本文主要通过Folder和Message的示例，演示了拷贝构造和拷贝赋值的控制逻辑，在两个类互相引用的情况下如何保证代码高效稳定运行。
源码链接
[https://gitee.com/secondtonone1/cpplearn](https://gitee.com/secondtonone1/cpplearn)
想系统学习更多C++知识,可点击下方链接。
[C++基础](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/24H7mZgZw57zCqGoULERZ2mQWOM)
