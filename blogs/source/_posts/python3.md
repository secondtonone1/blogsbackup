---
title: python学习笔记(三)高级特性
date: 2017-08-17 16:15:12
categories: [python]
tags: [python]
---
## 一、`切片`

list、tuple常常截取某一段元素，截取某一段元素的操作很常用 ，所以python提供了切片功能。

``` python
L=['a','b','c','d','e','f']
#取索引0，到索引3的元素，不包括索引3
print(L[0:3])
#开始索引为0可以省略
print(L[:3])
#下标1到3
print(L[1:3])
#取最后一个元素
print(L[-1])
#取倒数后两个元素
print(L[-2:])
#取前四个数，每两个取一个
print(L[:4:2])
#所有数，每两个取一个
print(L[::2])
```
<!--more-->
## 二、`迭代`

除了list、tuple可以迭代外，python中的dict类型变量也可以迭代。

``` python
dictor = {'name':'Jul','age':17,'femail':1}
#迭代key
for key in dictor:
    print(key)    
#迭代value
for value in dictor.values():
    print(value)
#迭代key,value
for k,v in dictor.items():
    print(k,v)
```

 可以将list变为索引元素对的形式
``` python
for x,y in [(1,2),(3,4),(5,6)]:
    print(x,y)
#变为索引元素对
for i,value in enumerate(['A','B','C']):
    print(i,value)
 

同时可以判断一个对象是否可以迭代

for x,y in [(1,2),(3,4),(5,6)]:
    print(x,y)
#变为索引元素对
for i,value in enumerate(['A','B','C']):
    print(i,value)
```
## 三、列表生成式

list函数可以将一组对象组合为列表，[]操作也可以。[]操作的方式称作列表生成式

print([x for x in range(1,11)])
print(list(range(1,11)))
在列表生成式中可以加入一些运算规则，使生成的列表具备运算规则。

``` python
#变为索引元素对
for i,value in enumerate(['A','B','C']):
    print(i,value)
#平方
print([x*x for x in range(1,11)])
#偶数平方
print([x*x for x in range(1,11) if x%2 ])
#k:v形式的列表
strdic={'a':'a1','b':'b1','c':'c1'}
print([k+':'+v for k,v in strdic.items()])
#将列表中字符串换为小写
L = ['Hello', 'World', 18, 'Apple', None]
print([s.lower() for s in L if(isinstance(s,str)) ])
```

## 四、生成器

python提供生成器的功能，生成器根据函数或运算规则产生一系列数据，

通过对返回值g调用next(g)可以依次取出生成的数据。
``` python
g = (x*2 for x in range(1,11))
print(g)
print(next(g))
```

可以一直调用next(g)，直到产生StopIteration异常。

当然也可以通过函数构造生成器，将函数return的关键字换为yield即可。

``` python
#菲波那切数列
def fib(max):
    a,b,n = 0,1,0
    while n < max:
        yield b
        a,b=b,a+b
        n = n+ 1
    return "exit"
```

通过下面方式next取出数列中的元素，第三次调用会抛出StopIteration异常。

g=fib(2)
print(g)
print(next(g))
print(next(g))
#print(next(g))
上面代码中g为迭代器，通过对g不断调用next取出数列中元素。

可以通过检测异常的方式完成遍历，避免程序崩溃。

``` python
g2 = fib(6)

while True:
    try:
        value = next(g2)
        print("value: ", value)
    except StopIteration as e:
        print("Generator return value is: ", e)
        break
```

可以用生成器实现杨辉三角，生成器函数为triangles()。

![1.png](1.png)

生成器函数triangles()实现如下：

``` python
def triangles():
    yield [1]
    yield [1,1]
    lists = [1,1]
    while True:
        i = 1
        n = len(lists)
        newlists = [1]
        while i < n:
            newlists.append(lists[i-1] + lists[i])
            i = i+1
        newlists.append(1)
        lists = newlists
        yield newlists    

```

## 五、迭代器

python提供生成器的功能，生成器根据函数或运算规则产生一系列数据，

通过对返回值g调用next(g)可以依次取出生成的数据。g就是迭代器。

有的对象可以迭代但是不是迭代器，只有可以被next调用的对象才是迭代器。

同样可以通过isinstance函数判断迭代器。

``` python
from collections import Iterable
from collections import Iterator

b1 = isinstance([], Iterable)
b2 = isinstance([], Iterator)
print('[] is Iteralbe', b1)
print('[] is Iterator', b2)

b1 = isinstance({},Iterable)
b2 = isinstance({},Iterator)

print('[] is Iteralbe', b1)
print('[] is Iterator', b2)

b1 = isinstance((x*x for x in range(10)), Iterable)
b2 = isinstance((x*x for x in range(10)), Iterator)
print('x*x for x in range(10) isIterable', b1)
print('x*x for x in range(10) isIterator', b2)

#可以被next()函数调用并不断返回下一个值的对象称为迭代器：Iterator

b1 = isinstance(triangles(),Iterable)
b2 = isinstance(triangles(),Iterator)
print('triangles()', b1)
print('triangles()', b2)
```
