---
title: python学习(十五)内建模块
date: 2017-12-19 10:42:20
categories: 技术开发
tags: [python]
---

介绍python的几个內建模块
## 1 python的时间模块datetime
取现在时间
``` python
from datetime import datetime
now = datetime.now()
print(now)
print(type(now))
```
<!--more-->
将指定日期转化为时间戳
``` python
from datetime import datetime
dt = datetime(2017,12,13,13,7)
# 把datetime转换为timestamp
print( dt.timestamp() )
```
将时间戳转化为日期
``` python
from datetime import datetime
t = 1429417200.0
print(datetime.fromtimestamp(t))
```
根据时间戳转化为本地时间和utc时间
``` python
from datetime import datetime
t = 1429417200.0
# 本地时间
print(datetime.fromtimestamp(t))
# UTC时间 
print(datetime.utcfromtimestamp(t))
```
将字符串转化为时间
``` python
from datetime import datetime
cday = datetime.strptime('2017-6-1 18:22:22','%Y-%m-%d %H:%M:%S')
print(day)
```
将时间戳转化为字符串
``` python
from datetime import datetime
now = datetime.now()
print(now.strftime('%a,%b %d %H:%M'))
```
时间加减
``` python
from datetime import datetime , timedelta
now = datetime.now()
print( now )
print(now + timedelta(hours = 10))
print(now + timedelta(days = 1))
print(now + timedelta(days = 2, hours = 12))
```
设置时区
``` python
from datetime import datetime, timedelta, timezone
# 创建时区UTC+8:00
timezone_8 = timezone(timedelta(hours = 8) )
now = datetime.now()
print(now)
# 强制设置为UTC+8:00
dt = now.replace(tzinfo=timezone_8)
print(dt)

```

获取utc时区和时间，并且转化为别的时区的时间
``` python
from datetime import datetime, timedelta, timezone

# 拿到UTC时间，并强制设置时区为UTC+0:00:
utc_dt = datetime.utcnow().replace(tzinfo=timezone.utc)
print(utc_dt)
bj_dt = utc_dt.astimezone(timezone(timedelta(hours = 8) ))
print(bj_dt)

tokyo_dt = utc_dt.astimezone(timezone(timedelta(hours = 9) ) )
print(tokyo_dt)

tokyo_dt2 = bj_dt.astimezone(timezone(timedelta(hours = 9) ) )
print(tokyo_dt2)
```

## 2命名tuple
``` python
#可命名tuple
from collections import namedtuple
Point = namedtuple('Point', ['x','y'])
p = Point(1,2)
print(p.x)

from collections import deque
q = deque(['a','b','c'])
q.append('x')
q.appendleft('y')
print(q)


from collections import defaultdict
dd = defaultdict(lambda:'N/A')
dd['key1']='abc'
print(dd['key1'])
print(dd['key2'])
```

## 3顺序字典
``` python
from collections import OrderedDict
d = dict([ ['a',1], ['b',2],['c',3]])
print(d)
od = OrderedDict([('a',1),('b',2),('c',3)])
print(od)

od2 = OrderedDict([['Bob',90],['Jim',20],['Seve',22]])
print(od2)
```
## 4计数器
``` python
from collections import Counter
c = Counter()

for ch in 'programming':
	c[ch]=c[ch]+1
```
## 5 itertools
从一开始生成自然数
``` python
#itertools.count(start , step)
import itertools
natuals = itertools.count(1)
for n in natuals:
	print(n)
```

在生成的可迭代序列中按规则筛选
``` python
natuals = itertools.count(1)
ns = itertools.takewhile(lambda x: x <= 10, natuals)
print(ns)
print(list(ns) )
```

将两个字符串生成一个序列
``` python
for c in itertools.chain('ABC','XYZ'):
	print(c)

print(list(itertools.chain('ABC','XYZ')) )
```

迭代器把连续的字母放在一起分组
``` python
for key, group in itertools.groupby('AAABBBCCAAA'):
	print(key, list(group))
	print(key, group)

for key, group in itertools.groupby('AaaBBbcCAAa', lambda c: c.upper() ):
	print(key,list(group))
```

## 6 contextmanager

open 返回的对象才可用with，或者在类中实现__enter__和__exit__可以使该类对象支持with用法

``` python
class Query(object):
	def __init__(self, name):
		self.name = name
	
	def __enter__(self):
		print('Begin')
		return self

	def __exit__(self, exc_type, exc_value, traceback):
		if exc_type:
			print('Error')
		else:
			print('End')

	def query(self):
		print('Query info about %s...' %self.name)		


with Query('BBBB') as q:
	if q:
		q.query()
```
简单介绍下原理

``` 
with EXPR as VAR:
实现原理:
在with语句中, EXPR必须是一个包含__enter__()和__exit__()方法的对象(Context Manager)。
调用EXPR的__enter__()方法申请资源并将结果赋值给VAR变量。
通过try/except确保代码块BLOCK正确调用，否则调用EXPR的__exit__()方法退出并释放资源。
在代码块BLOCK正确执行后，最终执行EXPR的__exit__()方法释放资源。
```

通过python提供的装饰器contextmanager，作用在生成器函数，可以达到with操作的目的
``` python
from contextlib import contextmanager
class Query(object):
	def __init__(self, name):
		self.name = name

	def query(self):
		print('Query info about %s ...' %self.name)

@contextmanager
def create_query(name):
	print('Begin')
	q = Query(name)
	yield q
	print('End')

with create_query('aaa') as q:
	if q:
		print(q.query())
```
可以看看contextmanager源码，了解下
``` python
class GeneratorContextManager(object):
	def __init__(self, gen):
        self.gen = gen

    def __enter__(self):
        try:
            return self.gen.next()
        except StopIteration:
            raise RuntimeError("generator didn't yield")
​
    def __exit__(self, type, value, traceback):
        if type is None:
            try:
                self.gen.next()
            except StopIteration:
                return
            else:
                raise RuntimeError("generator didn't stop")
        else:
            try:
                self.gen.throw(type, value, traceback)
                raise RuntimeError("generator didn't stop after throw()")
            except StopIteration:
                return True
            except:
                # only re-raise if it's *not* the exception that was
                # passed to throw(), because __exit__() must not raise
                # an exception unless __exit__() itself failed.  But
                # throw() has to raise the exception to signal
                # propagation, so this fixes the impedance mismatch 
                # between the throw() protocol and the __exit__()
                # protocol.
                #
                if sys.exc_info()[1] is not value:
                    raise
​
def contextmanager(func):
    def helper(*args, **kwds):
        return GeneratorContextManager(func(*args, **kwds))
    return helper
```

也可以采用closing用法作用在一个对象上支持with open操作

``` python
from contextlib import closing
from urllib.request import urlopen

with closing(urlopen('https://www.python.org')) as page:
    for line in page:
        print(line)
```

介绍下closing 实现原理
``` python
@contextmanager
def closing(thing):
    try:
        yield thing
    finally:
        thing.close()
```
同样可以用contextmanager实现打印指定标签的上下文对象
``` python
@contextmanager
def tag(name):
    print("<%s>" % name)
    yield
    print("</%s>" % name)

with tag("h1"):
    print("hello")
    print("world")
```

上述代码执行结果为：
``` 
<h1>
hello
world
</h1>
代码的执行顺序是：

with语句首先执行yield之前的语句，因此打印出<h1>；
yield调用会执行with语句内部的所有语句，因此打印出hello和world；
最后执行yield之后的语句，打印出</h1>。
```


## 7 urllib库
这是个非常重要的库，做爬虫会用到

采用urllib get网页信息

``` python
from urllib import request 
with request.urlopen('http://www.limerence2017.com/') as f:
	data = f.read()
	print('Status:', f.status, f.reason)
	for k, v in f.getheaders():
		print('%s: %s' %(k,v))
	print('Data:', data.decode('utf-8') )
```

在request中添加信息头模拟浏览器发送请求
``` python
from urllib import request
req = request.Request('http://www.douban.com/')
req.add_header('User-Agent', 'Mozilla/6.0 (iPhone; CPU iPhone OS 8_0 like Mac OS X) AppleWebKit/536.26 (KHTML, like Gecko) Version/8.0 Mobile/10A5376e Safari/8536.25')
with request.urlopen(req) as f:

	print('Status:', f.status, f.reason)
	for k, v in f.getheaders():
		print('%s: %s' %(k,v))
	print('Data:', f.read().decode('utf-8'))


from urllib import request, parse
print('Login to weibo.cn...')
email = input('Email: ')
passwd = input('Password: ')
login_data = parse.urlencode([
    ('username', email),
    ('password', passwd),
    ('entry', 'mweibo'),
    ('client_id', ''),
    ('savestate', '1'),
    ('ec', ''),
    ('pagerefer', 'https://passport.weibo.cn/signin/welcome?entry=mweibo&r=http%3A%2F%2Fm.weibo.cn%2F')
])
```

采用post方式获取信息, request.urlopen()，参数可以携带data发送给网址

``` python

print('Login to weibo.cn...')
email = input('Email: ')
passwd = input('Password: ')
login_data = parse.urlencode([
    ('username', email),
    ('password', passwd),
    ('entry', 'mweibo'),
    ('client_id', ''),
    ('savestate', '1'),
    ('ec', ''),
    ('pagerefer', 'https://passport.weibo.cn/signin/welcome?entry=mweibo&r=http%3A%2F%2Fm.weibo.cn%2F')
])

req = request.Request('https://passport.weibo.cn/sso/login')
req.add_header('Origin', 'https://passport.weibo.cn')
req.add_header('User-Agent', 'Mozilla/6.0 (iPhone; CPU iPhone OS 8_0 like Mac OS X) AppleWebKit/536.26 (KHTML, like Gecko) Version/8.0 Mobile/10A5376e Safari/8536.25')
req.add_header('Referer', 'https://passport.weibo.cn/signin/login?entry=mweibo&res=wel&wm=3349&r=http%3A%2F%2Fm.weibo.cn%2F')

with request.urlopen(req, data=login_data.encode('utf-8')) as f:
	print('Status:', f.status, f.reason)
	for k, v in f.getheaders():
		print('%s:%s' %(k,v))
	print('Data: ', f.read().decode('utf-8'))
```

采用代理方式获取网页信息
``` python
proxy_handler = urllib.request.ProxyHandler({'http': 'http://www.example.com:3128/'})
proxy_auth_handler = urllib.request.ProxyBasicAuthHandler()
proxy_auth_handler.add_password('realm', 'host', 'username', 'password')
opener = urllib.request.bulid_opener(proxy_handler, proxy_auth_handler)
with opener.open('http://www.example.com/login.html') as f:
	pass


proxy_auth_handler = urllib.request.ProxyBasicAuthHandler()
proxy_auth_handler.add_password('realm', 'host', 'username', 'password')
opener = urllib.request.build_opener(proxy_handler, proxy_auth_handler)
with opener.open('http://www.example.com/login.html') as f:
    pass
```



