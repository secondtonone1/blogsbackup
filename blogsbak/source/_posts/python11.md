---
title: python学习(十一)元类学习和应用
date: 2017-09-20 16:35:11
categories: 技术开发
tags: [python]
---
最近学习了python的`错误处理`和几种`测试`方法
## `try except`
可以通过`try except`方式捕捉异常
``` python
try:
    print('try...')
    r = 10/0
    print('result is :', r)
except ZeroDiversionError as e:
    print('except is :', e)
finally:
    print('finally ...')
print('END')
```
可以捕捉不同类型的错误，编写多个except
<!--more-->
``` python
try:
    print('try...')
    r = 10/int('a')
    print('result is: ', r)
except ValueError as e:
    print('ValueError : ', e)
except ZeroDiversionError as e:
    print('ZeroDivisionError is : ', e)
finally:
    print('finally ...')
print('END')
```
try except同样支持else结构
``` python
try:
    print('try... ')
    r = 10/int('2')
    print('result is : ', r)
except ValueError as e:
    print('ValueError : ', e)
except ZeroDivisionError as e:
    print('ZeroDivisionError is : ', e)
else:
    print('no error')
finally:
    print('finally ...')
print('END')
```
某个函数调用出现异常，在上层同样可以捕获到

``` python
def foo(s):
    return 10/int(s)
def bar(s):
    return  foo(s) * 2

def main():
    try:
        bar('0')
    except Exception as e:
        print('Exception is : ', e)
    finally:
        print('finally...')
main()
```
## logging
python 提供打日志方式输出异常，并且不会因为出现异常导致程序中断
``` python
import logging
def foo(s):
    return 10/int(s)
def bar(s):
    return foo(s) * 2

def main():
    try:
        bar('0')
    except Exception as e:
        logging.exception(e)
main()
print('END')
```
如果想要将异常处理的更细致，可以自定义一个异常类，继承于几种错误类，如ValueError等，在可能出现问题的地方将错误抛出来
``` python
class FooError(ValueError):
    pass
def foo(s):
    n = int(s)
    if n == 0:
        raise FooError('invalid error is : %s' %s)
    return 10/n
foo('0')
```
错误可以一层一层向上抛，直到有上层能处理这个错误为止
``` python
def foo(s):
    n = int(s)
    if n==0:
        raise ValueError('invalid error is: %s' %s)
    return 10/n

def bar():
    try:
        foo('0')
    except ValueError as e:
        print('ValueError')
        raise

bar()
```
logging可以设置不同的级别,通过basicConfig可以设置
``` python
import logging 
logging.basicConfig(level=logging.INFO)

def foo(s):
    n = int(s)
    return 10/n

def main():
    m = foo('0')
    logging.info('n is : %d' %m)
main()
```

## 断言assert
大部分语言都支持assert，python也一样，在可能出错的地方写assert，会在异常出现时使程序终止
``` python
def foo(s):
    n = int(s)
    assert n != 0 ,'n is zero'
    return 10/n

def main():
    foo('0')

main()
```
## pdb调试和set_trace
pdb调试用 python -m pdb 文件名.py, 单步执行敲n,退出为q
python 可以在代码里设置断点，在程序自动执行到断点位置暂停,暂停在`set_trace`的代码行
``` python
import pdb
def foo(s):
    n = int(s)
    pdb.set_trace()
    return 10/n
def main():
    m = foo('0')
main()
```
## 单元测试
先实现一个自己定义的Dict类，将文件保存为mydict.py
``` python
class Dict(dict):
    def __init__(self, **kw):
        super(Dict, self).__init__(**kw)
    def __getattr__(self, key):
        try:
            return self[key]
        except Exception as e:
            raise AttributeError('AttributeError is :%s', e)
    def __setattr__(self, key, value):
        self[key] =  value
```
python 提供了单元测试的类，开发者可以继承`unittest.Test`实现特定的测试类,下面实现Dict的单元测试类，保存为`unittestdict.py`
``` python
import unittest
from mydict import Dict

class TestDict(unittest.TestCase):
	def setUp(self):
		print('setUp...')
	def tearDown(self):
		print('tear Down...')

	def test_init(self):
		d = Dict(a='testa', b = 1)
		self.assertEqual(d.a, 'testa')
		self.assertEqual(d.b, 1)
		self.assertTrue(isinstance(d, dict))

	def test_key(self):
		d = Dict()
		d['name'] = 'hmm'
		self.assertEqual(d.name, 'hmm')

	def test_attr(self):
		d = Dict()
		d.name = 'hmm'
		self.assertEqual(d['name'], 'hmm')
		self.assertTrue('name' in d)

	def test_attrerror(self):
		d = Dict()
		with self.assertRaises(AttributeError):
			value = d.empty
	
	def test_keyerror(self):
		d = Dict()
		with self.assertRaises(AttributeError):
			value = d['empty']
	
if __name__ == '__main__':
	unittest.main()
```
运行unittest.py可以检测mydict中Dict类是否有错误
## `文档测试`
文档测试在代码中按照特定格式编写python输入和期待的输出，通过python提供的文档测试类，实现测试代码的目的
``` python
class Dict(dict):
	'''
	>>> d1 = Dict()
	>>> d1['x'] = 100
	>>> d1.x
	100
	>>> d1.y = 200
	>>> d1['y']
	200
	>>> d2=Dict(a=1,b=2,c='m')
	>>> d2.c
	'm'
	
	'''
	def __init__(self, **kw):
		super(Dict,self).__init__(**kw)
	def __getattr__(self,key):
		try:
			return self[key]
		except KeyError:
			raise AttributeError('AttributeError key is %s' %key)
	def __setattr__(self,key,value):
		self[key] = value


if __name__ == '__main__':
	import doctest
	doctest.testmod()

```