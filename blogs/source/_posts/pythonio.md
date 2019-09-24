---
title: python文件读写和序列化
date: 2017-11-14 15:47:51
categories: [python]
tags: [python]
---
python 文件读写和序列化学习。
## python文件读写
`1 打开并且读取文件`
``` python
f = open('openfile.txt','r')
print(f.read())
f.close()
```
<!--more-->
`2 打开并且读取一行文件`
``` python
f = open('openfile.txt','r')
print(f.readline())
f.close()
```
`3 打开并以二进制形式读取文件`
``` python
f = open('./openfile.txt','rb')
print(f.read())
f.close()
```
`4 打开并自动关闭文件`
``` python
with open('openfile.txt','r') as f:
    print(f.read())
```
`5 读取所有行`
``` python
f = open('openfile.txt','r')
for line in f.readlines():
    print(line.strip())
f.close()
```

`6 以gbk方式读取文件`
``` python
f = open('openfiles.txt','r',encoding='gbk' )
print(f.read())
f.close()
```

`7 以追加方式写`
``` python
with open('openfile.txt', 'a') as f:
    f.write('\n')
    f.write('Hello World!!!')
```

## python IO操作
`1 StringIO 写字符串`

``` python
from io import StringIO
f = StringIO()
f.write('hello')
f.write(' ')
f.write('world !')
print(f.getvalue() )
```
`2 StringIO 读取字符串`
``` python
from io import StringIO
f = StringIO("Hello\nWorld\nGoodBye!!")
while True:
    s = f.readline()
    if(s==''):
        break
    print(s.strip())
```

`3 BytesIO 读写二进制字符`
``` python
from io import BytesIO
f = BytesIO()
f.write('中文'.encode('utf-8') )
print(f.getvalue())

from io import BytesIO
f = BytesIO(b'\xe4\xb8\xad\xe6\x96\x87')
f.read()
```

## python环境变量和目录

`1 打印系统的名字和环境变量`
``` python
import os
print(os.name)
print(os.environ)
```
`2 获取指定key值得环境变量`
``` python
print(os.environ.get('PATH'))
```
`3 相对路径转化绝对路径`
``` python
print(os.path.abspath('.'))
```
`4 在某个目录下创建一个新的目录`
``` python
#首先把新目录的完整路径表示出来
print(os.path.join('/Users/michael','testdir') )
# 然后创建一个目录:
#print(os.mkdir('/Users/michael/testdir') )
# 删掉一个目录:
#print(os.rmdir('/Users/michael/testdir') )
```

`5 路径切割`
``` python
print(os.path.split('/path/to/file.txt') )
print(os.path.splitext('/path/to/file.txt') )
```
`6 文件重命名和删除`
``` python
#print(os.rename('test.txt', 'test.py') )
#print(os.remove('test.py'))
```
`7 列举当前目录下所有目录和py文件`
``` python
print([x for x in os.listdir('.') if  os.path.isdir(x) ])
print([x for x in os.listdir('.') if os.path.isfile(x) and os.path.splitext(x)[1] == '.py'])
```

## python序列化

`1 序列化为二进制`
``` python
import pickle
d = dict(name='Bob', age=20, score=88)
print(pickle.dumps(d))
```
`2 序列化写入文件`
``` python
f = open('openfile3.txt','wb')
print(pickle.dump(d, f) )
f.close()
```
`3 反序列化读取文件`
``` python
f = open('openfile3.txt','rb')
d = pickle.load(f)
f.close()
print(d)
```
`4 序列化为json`
``` python
import json

class Student(object):
    def __init__(self, name, age, score):
        self.name = name
        self.age = age
        self.score = score

def convertFunc(std):
    return {'name':std.name,
    'age':std.age,
    'score':std.score}


s = Student('Bob', 20, 88)
print(json.dumps(s,default=convertFunc))
print(json.dumps(s,default=lambda obj:obj.__dict__))

```
`5 反序列化`

``` python
def revert(std):
    return Student(std['name'], std['age'], std['score'])

json_str = '{"age": 20, "score": 88, "name": "Bob"}'
print(json.loads(json_str, object_hook=revert ) )

```


