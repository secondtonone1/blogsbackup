---
title: python学习笔记(四) 思考和准备
date: 2017-08-21 10:46:53
categories: 技术开发
tags: [python]
---
## 一、zip的坑

zip()函数接收多个可迭代数列，将数列中的元素重新组合，在3.0中返回迭代器指向

数列首地址，在3.0以下版本返回List类型的列表数列。我用的是3.5版本python，

所以zip返回的是指向地址。

先看几个例子
![1.png](1.png)
<!--more-->
结果：
![2.png](2.png)
可见，在3.0以上版本，对zip函数返回的结果采用list函数可以转化为列表。

通过列表生成式同样可以将zip结果化为列表。
![3.png](3.png)
结果：
![4.png](4.png)

当zip操作的对象为一个列表，那么生成的列表中每个元素(元祖)中为(n,)形式。当zip操作的多个列表长度不一样，那么zip返回生成的列表中元素个数为最短列表长度。

![5.png](5.png)

list函数可以将一个元祖转化为列表。下面可以将zip返回的数据转化为我们方便操作的列表元素.
![6.png](6.png)
结果：
![7.png](7.png)
这样将zip函数返回的数据通过list和迭代，生成了二维List，方便以后操作。下边这段代码在3.0版本以上和3.0版本以下会有不同结果。

![8.png](8.png)
2.7版本结果
![9.png](9.png)
3.0版本结果
![10.png](10.png)

之前提起过zip在3.0以上版本返回迭代器指向内存地址。3.0以下版本返回的为列表，所以在3.0版本一下输出是符合最初目的。

但是3.0版本python最后一行输出却为空列表[]。这个原因主要是迭代器在被循环迭代或者访问后，会自动移动指针，指向下一个要迭代的元素。这和C++是不同的，

C++/C语言需要用户自己控制迭代器移位。那么肯定有人会说第一句和第三句打印的list1的值相同，是不是list1迭代器指向的空间没有移动呢？

不是的，只要list1被循环迭代，内部指向空间的地址就会变化，只是调用print打印list1时，python只返回迭代器指向空间的首地址，而不会告诉具体指向的地址空间。

修改下代码，看看是不是上文所述那样：
![11.png](11.png)
结果：
![12.png](12.png)

可见，输出list1指向地址内容的时候出现StopIteration异常，这个异常前几篇介绍过，是因为迭代器已经指向空间的末尾了，再调用就会出现该异常，所以对于迭代器当遍历迭

代后一定要注意迭代器指向地址变化的问题。

## 二、迭代器的坑

迭代器的问题就在于被迭代使用后，内部指向的地址空间变化了，但是打印迭代器，返回的是迭代器最初指向的内存空间首地址。
![13.png](13.png)
结果：
![14.png](14.png)
每次打印g返回结果都一样，但是g指向的位置确实变了。说实话，这种隐藏性的问题应该让别人知道。

## 三、矩阵的转置和左右逆置

通过zip函数可以实现矩阵的转置和逆置，
![15.png](15.png)
将矩阵按照每一行存储在一个list中，这些list再组合成一个大的list，构成二维list表示矩阵。

矩阵的转置：

下边的例子可以看效果：
![16.png](16.png)
结果：
![17.png](17.png)

同样的道理，矩阵的左右逆置

![18.png](18.png)
结果：
![19.png](19.png)

## 四、format函数介绍

format函数通过{}替代%，实现字符串的动态匹配。

![20.png](20.png)
结果：
![21.png](21.png)

## 五、defaultdict函数介绍

实现一个统计字符串个数的功能。
``` python
strings = ('puppy', 'kitten', 'puppy', 'puppy','weasel', 'puppy', 'kitten', 'puppy')
```
如果用下边的代码实现统计功能
![22.png](22.png)
当counts中不存在某个单词，第一次调用counts[kw]+=1会提示报错。所以有几种方式实现该功能。
![23.png](23.png)
这种方式先判断counts中是否含有关键字，没有就先赋值。
![24.png](24.png)
这种方式通过设置counts中关键字对应的默认值，当counts中不存在某个关键字，

那么该关键字对应的value为0，然后该值+1，表示第一次统计。

如果counts中存在该关键字，那么就不执行setdefault函数，关键字对应的value值加1。

![25.png](25.png)
这种方式引用了collections的defaultdict，difaultdict接收一个函数，该函数

返回dict中元素的默认值。

## 六、any函数介绍

any函数接收一个可迭代对象，一般为list或者tuple，list或者tuple中有一个

元素满足条件any函数就返回true，当所有元素都不满足条件，any返回false

![26.png](26.png)
结果：
![27.png](27.png)
这是后一篇制作2048游戏的准备，下一篇制作2048小游戏

谢谢关注我的公众号
![1.jpg](1.jpg)