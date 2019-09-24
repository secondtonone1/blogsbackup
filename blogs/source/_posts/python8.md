---
title: python学习(八)定制类和枚举
date: 2017-08-31 15:39:49
categories: [python]
tags: [python]
---

`python`定制类主要是实现特定功能，通过在类中定义特定的函数完成特定的功能。
``` python
class Student(object):
    def __init__(self, name):
        self.name =name

student = Student("lilei")
print(student)
```

`实现定制类`
``` python
class Student(object):
    def __init__(self, name):
        self.name = name
    def __str__(self):
        return ("self name is %s" %(self.name))

student2 = Student("hanmeimei")
print(student2)
```
<!--more-->

实现`__str__`函数，可以在print类对象时打印指定信息

通过实现`__iter__`和`__next__`同样可以使类对象产生可迭代序列，下面实现了`斐波那契数列`
``` python
class Fib(object):
    def __init__(self):
        self.a , self.b = 0,1
    def __iter__(self):
        return self
    def __next__(self):
        self.a, self.b = self.b, self.a+ self.b
        if self.a > 30:
            raise StopIteration()
        return self.a
```
打印输出
``` python
for n in Fib():
    print(n)
```
可以实现`__getitem__`函数,这样就可以按照索引访问类对象中迭代元素了。
``` python
class OddNum(object):
    def __init__(self):
        self.num = -1
    def __iter__(self):
        return self
    def __next__(self):
        self.num = self.num +2
        if self.num > 10:
            raise StopIteration()
        return self.num 
    
    def __getitem__(self,n):
        temp = 1
        for i in range(n):
            temp += 2
        return temp
```

``` python
for n in OddNum():
    print(n)

oddnum = OddNum()
print(oddnum[3])
```
可以进一步完善OddNum类的`__getitem__`函数，使其支持`切片处理`

``` python
def __getitem__(self, n):
    if isinstance(n ,int):
        temp =1
        for i in range(n):
            temp +=2
        return temp
    if isinstance(n, slice):
        start = n.start
        end = n.stop
        if start is None:
            start = 0
        tempList = []
        temp = 1
        for i in range(end):
            if i >= start:
                temp += 2
                tempList.append(temp)
        return tempList
```
`print(oddnum[1:4])`
通过实现`__getattr__`函数，可以在类对象中没有某个属性时，自动调用`__getattr__`函数
实现`__call__`函数，可以实现类对象的函数式调用
``` python
def __getattr__(self,attr):
    if attr == 'name':
        return 'OddNum'
    if attr == 'data':
        return lambda:self.num
    raise AttributeError('\'OddNum\' object has no attribute \'%s\'' %attr)
def __call__(self):
    return "My name is OddNum!!"
```   
    
只有在没有找到属性的情况下，才调用`__getattr__`，已有的属性不会在`__getattr__`中查找。

``` python
print(oddnum.name)
print(oddnum.data)
#没有func函数会抛出异常
#print(oddnum.func)
#可以直接通过oddnum()函数式调用
print(oddnum())
```
下面是廖雪峰官方网站上的一个链式转化例子，用到了这些特定函数

``` python
class Chain(object):
    def __init__(self, path=''):
        self.path = path
    def __getattr__(self,attr):
        return Chain('%s/%s'%(self.path, attr))
    def users(self, users):
        return Chain('%s/users/%s' %(self.path, users))
    def __str__(self):
        return self.path
    __repr__ = __str__
print(Chain().users('michael').repos)
```

``` python
class Chain(object):
    def __init__(self, path=''):
        self.path = path
    def __getattr__(self,attr):
        return Chain('%s/%s'%(self.path, attr))
    def __call__(self, param):
        return Chain('%s/%s'%(self.path, param))
    def __str__(self):
        return self.path
    __repr__ = __str__

print(Chain().get.users('michael').group('doctor').repos)
```
python同样支持`枚举`操作
``` python
from enum import Enum

Month = Enum('Month', ('Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'))

for name, member in Month.__members__.items():
    print(name, '=>', member, ',', member.value)

from enum  import Enum
Month = Enum('Month', ('Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec') )
for name, member in Month.__members__.items():
    print(name, '=>', member, ',', member.value)

from enum import  unique
@unique
class Weekday(Enum):
    Sun = 0 # Sun的value被设定为0
    Mon = 1
    Tue = 2
    Wed = 3
    Thu = 4
    Fri = 5
    Sat = 6

for name , member in Weekday.__members__.items():
    print(name, '=>', member, ',', member.value)
```


