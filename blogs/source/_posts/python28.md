---
title: python学习(28) 浅谈可变对象的单例模式设计
date: 2019-03-13 10:42:07
categories: 技术开发
tags: [python]
---
python开发，有时候需要设计单例模式保证操作的唯一性和安全性。理论上python语言底层实现和C/C++不同，python采取的是引用模式，当一个对象是可变对象，对其修改不会更改引用的指向，当一个对象是不可修改对象，对其修改会改变引用指向。
## 可变对象和不可变对象
### 不可变对象
该对象所指向的内存中的值不能被改变。当改变某个变量时候，由于其所指的值不能被改变，相当于把原来的值复制一份后再改变，这会开辟一个新的地址，变量再指向这个新的地址。
### 可变对象
该对象所指向的内存中的值可以被改变。变量（准确的说是引用）改变后，实际上是其所指的值直接发生改变，并没有发生复制行为，也没有开辟新的出地址，通俗点说就是原地改变。
### python 中的可变对象和不可变对象
Python中，数值类型（int和float）、字符串str、元组tuple都是不可变类型。
而列表list、字典dict、集合set,以及开发人员自己定义的类是可变类型
<!--more-->
### python 和C/C++ 内存分配差异
``` python
a = 2
b = 2
c = a + 0 
c += 0
print(id(a), id(b), id(2))  # id都相同
print(c is b) #True
```
python 中变量a,b,c都是常量2的引用，所以他们的地址空间都相同。在C/C++中,a,b,c是三个变量，每个变量地址都不一样，这一点大家在学习语言时要注意，这算是python的特性吧。同样的道理，字符串也是一样的
``` python
astr = 'good'
bstr = 'good'
cstr = astr + ''
print(cstr is bstr) # True
print(id(astr), id(bstr), id('good'))  # 三个id相同
```
字符串也是不可变对象，所以astr,bstr,cstr指向的都'good'所在空间
如果修改astr,则astr指向改变了
``` python
astr = 'good'
print(id(astr))
astr += 'aa'
print(id(astr)) # id和上面的不一样
```
因为str是不可变对象，当修改它的值后变为'goodaa'，那么astr指向的地址也改变为'goodaa'所在地址。id(astr)和之前的不一样了。
``` python
lis = [1, 2, 3]
lis2 = [1, 2, 3]
# 虽然它们的内容一样，但是它们指向的是不同的内存地址
print(lis is lis2)
print(id(lis), id(lis2), id([1, 2, 3]))  # 三个id都不同
```
虽然lis和lis2内容一样，但是可变对象都会单独开辟空间，所以上边三个id打印结果都不一样。
``` python
alist = [1, 2, 3]
# alist实际上是对对象的引用，blist = alist即引用的传递，现在两个引用都指向了同一个对象（地址）
blist = alist
print(id(alist), id(blist))  # id一样
# 所以其中一个变化，会影响到另外一个
blist.append(4)
print(alist)  # 改变blist, alist也变成了[1 ,2 ,3 4]
print(id(alist), id(blist))  # id一样，和上面值没有改变时候的id也一样
```
blist赋值为alist，这两个指向同一个地址，当修改blist时，alist也改变了，所以打印两个id都是一样的。一般我们自己定义的类也是可变对象，我们想做的是通过设计一个单例类实现统一的控制，这样便于管理，比如网络模块，比如数据库处理模块等等。下面浅谈三种单例模式设计
## python 单例模式设计
### 方法一：使用装饰器
装饰器维护一个字典对象instances，缓存了所有单例类，只要单例不存在则创建，已经存在直接返回该实例对象。
``` python
def singleton(cls):
    instances = {}
    def wrapper(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return wrapper

@singleton
class Foo(object):
    pass
foo1 = Foo()
foo2 = Foo()
print foo1 is foo2
```
singleton函数中定义了instances字典，当使用它作为装饰器装饰Foo后instances也会被缓存在闭包环境中，第一次使用Foo()后，instances就回被设置为instances[Foo]=Foo(),这样根据类名就可以区分是否被初始化过，从而实现单例模式
### 方法二：使用基类
__new__是真正创建实例对象的方法，所以重写基类的__new__方法，以此来保证创建对象的时候只生成一个实例
``` python
class Singleton(object):
    def __new__(cls, *args, **kwargs):
        if not hasattr(cls, '_instance'):
            cls._instance = super(Singleton, cls).__new__(cls, *args, **kwargs)
        return cls._instance


class Foo(Singleton):
    pass

foo1 = Foo()
foo2 = Foo()

print foo1 is foo2  # True
```
__new__在一个类构造实例对象时会调用，所以通过判断hasattr，是否含有某个属性，即可实现单例模式。
super(Singleton,cls)调用的是Singleton的基类。我目前用的是这种方式实现的单例，用作网络和数据库管理。
### 方法三：使用元类

元类（参考：深刻理解Python中的元类）是用于创建类对象的类，类对象创建实例对象时一定会调用__call__方法，因此在调用__call__时候保证始终只创建一个实例即可，type是python中的一个元类
``` python
class Singleton(type):
    def __call__(cls, *args, **kwargs):
        if not hasattr(cls, '_instance'):
            cls._instance = super(Singleton, cls).__call__(*args, **kwargs)
        return cls._instance


class Foo(object):
    __metaclass__ = Singleton


foo1 = Foo()
foo2 = Foo()

print foo1 is foo2  # True
```
这种方式和new类似，都是通过系统级的函数__call__进行控制。通过在类中设置元类从而实现单例控制。

到目前为止，单例模式介绍完毕，感谢关注我的公众号

![wxgzh.jpg](wxgzh.jpg)

