---
title: python学习笔记(六) 函数式编程
date: 2017-08-22 10:45:54
categories: [python]
tags: [python]
---
## 一 `函数对象`

函数同样可以作为对象复制给一个变量，如下：

``` python
f = abs;
print(f(-10))
f = 'abs';
print(f)

def add(a,b,f):
    return f(a) + f(b)

print(add(-1,2,f))
```
<!--more-->
`map 函数`，map函数接受`一个函数变量`，第二个参数为一个`可迭代对象`，最后返回一个迭代器，由于迭代器的惰性，`需要用list()函数返回所有元素`。

``` python
def squart(n):
    return n* n;

print(map(squart,range(1,11) ) )
print(list(map(squart,range(1,11) ) ))
```

`reduce函数`， reduce函数接受两个参数，第一个参数同样是`函数对象f`，`f必须接受两个参数`，并且`返回和参数同类型的数据`。`第二个参数为一个可迭代序列`。

``` python
def func(a, b):
    return a + b

print(reduce(func, range(1,11)))
 
reduce和map函数不一样，reduce返回的是一个最终值

reduce(f,[x1, x2, x3, x4]) = f(f(f(x1,x2),x3),x4)
```

可以通过reduce和map函数搭配，将一个字符串转化为整数

``` python
def str2int(str):
    def char2int(c):
        return {'0':0,'1':1,'2':2,'3':3,'4':4,'5':5,'6':6,'7':7,'8':8,'9':9}
    def convertnum(a,b):
        return a*10 + b
    return reduce(convertnum, map(char2int, str))

print(str2int("123456789"))
```

 `filter 函数`，filter函数同样有两个参数，`第一个参数为函数对象，返回值为bool类型`，`第二个参数为可迭代序列，返回值为迭代器`，

同样需要list()转化为序列。下面用filter和生成器实现一个素数生成器函数

``` python
def odd_generater():
    n = 1
    while True:
        n = n+1
        yield n

def primer_generater():
    yield 2
    it = odd_generater()
    while(True):
        n = next(it)
        yield n
        it = filter(lambda x:x%n > 0, it)

```

打印测试：
``` python
for i in primer_generater():
    if(i < 100):
        print (i)
    else:
        break
```

sorted 函数，第一个接受一个list，第二个为比较的规则，可以不写
``` python
print(sorted(["Abert","cn","broom","Dog"]) )

print(sorted(["Abert","cn","broom","Dog"], key = str.lower))

print(sorted(["Abert","cn","broom","Dog"], key = str.lower, reverse = True))
```

## 二  函数封装和返回

``` python
def lazy_sum(*arg):
    def sum():
        x = 0
        for i in arg:
            x = x +i
        return x
    return sum

f = lazy_sum(2,3,1,6,8)
print(f())
```

定义了一个lazy_sum函数，函数返回内部定义的sum函数。可以在函数A内部定义函数B，调用A返回函数B，从而达到函数B延时调用。

`闭包`：

在`函数A`内部定义函数B，`函数B`内使用了函数A定义的局部变量或参数，这种情况就是`闭包`。

使用闭包需要注意，在函数B中修改了函数A 定义的局部变量，那么需要使用`nonlocal关键字`。如果在`函数B中修改了全局变量`，那么`需要使用global关键字`。
``` python
def lazy_sum(*arg):
    sums = 0
    def sum():
        for i in arg:
            nonlocal sums
            sums = sums + i
        return sums
    return sum

f1 = lazy_sum(1,3,5,7,9)
f2 = lazy_sum(1,3,5,7,9)
print(f1 == f2)
print(f1() )
print(f2() )
```

`匿名函数`：lambda， `lambda后面跟函数的参数`，`然后用：隔开`，`写运算规则作为返回值`

``` python
it = map(lambda x:x*x, (1,3,5,7,9))
print(list(it))

def lazy_squart():
    return lambda x:x*x
f = lazy_squart()
print(f(3) )
```

`装饰器`： `装饰器实际就是函数A中定义了函数B，并且返回函数B`，为了实现特殊功能，如写日志，计算时间等等。

先看个返回函数，并且调用的例子

``` python
def decoratorfunc(func):
    def wrapperfunc():
        print('func name is: %s'%(func.__name__))
        func()
    return wrapperfunc


def helloworld():
    print('Helloworld !!!')

helloworld = decoratorfunc(helloworld)
helloworld() 
```

以后每次调用helloword，不仅会打印Helloworld，还会打印函数名字。

python提供装饰器的功能，可以简化上面代码，并且实现每次调用helloworld函数都会打印函数名字。

``` python
def decoratorfunc(func):
    def wrapperfunc(*args, **kw):
        time1 = time.time()
        func(*args, **kw)
        time2 = time.time()
        print('cost %d secondes'%(time2-time1))
    return wrapperfunc

@decoratorfunc
def output(str):
    print(str)
    time.sleep(2)

output('hello world!!!')
```

如果函数带参数，实现装饰器可以内部定义万能参数的函数

``` python
def decoratorfunc(func):
    def wrapperfunc(*args, **kw):
        time1 = time.time()
        func(*args, **kw)
        time2 = time.time()
        print('cost %d secondes'%(time2-time1))
    return wrapperfunc

@decoratorfunc
def output(str):
    print(str)
    time.sleep(2)

output('hello world!!!')
```

装饰器执行@decoratorfunc相当于

output = decoratorfunc(output)
output('hello world!!!')

如果装饰器需要传入参数，那么可以增加多一层的函数定义，完成装饰器参数传入和调用。

``` python
def decoratorfunc(param):
    def decoratorfunc(func):
        def wrapperfunc(*arg, **kw):
            print('%s %s' %(param, func.__name__))
            func(*arg, **kw)
        return wrapperfunc
    return decoratorfunc

@decoratorfunc('execute')
def output(str):
    print(str)

output('nice to meet u')
print(output.__name__)
```

#实际执行过程
``` python
decorator = decoratorfunc('execute')
output = decorator(now)

output('nice to meet u')
```

执行print(output.__name__)发现打印出的函数名字不是output而是wrapperfunc，这对以后的代码会有影响。

可以通过python提供的装饰器@functools.wraps(func) 完成函数名称的绑定 

``` python
def decoratorfunc(param):
    def decoratorfunc(func):
        @functools.wraps(func)
        def wrapperfunc(*arg, **kw):
            print('%s %s' %(param, func.__name__))
            func(*arg, **kw)
        return wrapperfunc
    return decoratorfunc

@decoratorfunc('execute')
def output(str):
    print(str)

print(output.__name__)
```

print(output.__name__)显示为output，这符合我们需要的逻辑。

# 三  `偏函数`

如函数 `int(a, base = 2)` 可以实现一个字符串根据base提供的进制，转化成对应进制的数字。

`可以通过偏函数，实现指定参数的固定，并且生成新的函数`
``` python
intnew = functools.partial(int, base = 2)
print(intnew('100'))
```

也可以自己定义函数：

``` python
def add(a,b):
    return a+b
print(add(3,7))

addnew = functools.partial(add, 3)
print(addnew(7))

addnew2 = functools.partial(add, b = 7)
print(addnew2(3))
```

函数部分介绍到此为止，我的公众号，谢谢关注：
![1.jpg](1.jpg)
