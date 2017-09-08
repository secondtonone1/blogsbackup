---
title: python学习(十)元类学习和应用
date: 2017-09-07 15:33:37
categories: 技术开发
tags: [python]
---
python 可以通过`type`函数创建类,也可通过type判断数据类型
``` python
import socket
from io import StringIO
import sys
class TypeClass(object):
    def typeprint(self, name = 'typeclass'):
        print('class name is %s' %(name))

typeclasses = TypeClass()
print(type(TypeClass))
print(type(typeclasses))

def printHW(self):
    print('Hello world!!!')
TypeClass2 = type('TypeClass2', (object,), dict(typeprinthw = printHW))
typeclass2 = TypeClass2()
typeclass2.typeprinthw()
```
type创建类格式为type('类名',(基类1,基类2...), dict(成员函数名=函数名))
第一个参数为类名，第二个参数为一个tuple,如果继承的基类只有一个，要注意tuple写法(基类,),第三个参数为dict构造的类成员函数
除了可以用type创建类之外，可以用`metaclass`限制类的行为
<!--more-->
使用metaclass需要先定义一个特定功能的元类，这个元类一般按照功能名字命名，末尾以Metaclass结束，表示一个特定功能的元类,这个元类必须继承于type类,内部必须实现
`__new__`借口完成类的限定。`__new__`的调用先于`__init__`，在对象构造之前调用。
``` python
class DictMetaclass(type):
	def __new__(cls, name, bases, attrs):
		def insertfunc(self, key, value):
			self[key]=value
		attrs['insert'] = insertfunc
		return type.__new__(cls,name,bases,attrs)

class MyDict(dict, metaclass = DictMetaclass):
    pass

mydict = MyDict()
mydict.insert('nice', 3)
print(mydict['nice'])

```
定义了一个DictMetaclass元类，继承于type类，内部实现了__new__方法，__new__方法四个参数分别为准备创建的类对象，类名字，该类的基类集合，类的方法集合。并且__new__内部调用了type类的__new__函数，完成了新功能属性的绑定。下面同样实现一个list类的扩展
``` python
class ListMetaclass(type):
    def __new__(cls, name, bases, attrs):
        attrs['insert'] = lambda self, value: self.append(value)
        return type.__new__(cls,name,bases,attrs)

class Mylist(list,metaclass = ListMetaclass):
    pass

mylist = Mylist()
mylist.insert('name')
mylist.insert('age')
print(mylist)
```
下面为在廖雪峰官网学习的ORM框架例子
ORM全称“Object Relational Mapping”，即对象-关系映射，就是把关系数据库的一行映射为一个对象，也就是一个类对应一个表，这样，写代码更简单，不用直接操作SQL语句。
要编写一个ORM框架，所有的类都只能动态定义，因为只有使用者才能根据表的结构定义出对应的类来。
让我们来尝试编写一个ORM框架。
先实现Field类和其派生类
``` python
class Field(object):
    def __init__(self, name, column_type):
        self.name = name
        self.column_type = column_type
    def __str__(self):
        return '<%s%s>' %(self.__class__.__name__, self.name)

class IntegerField(Field):
    def __init__(self, name):
        super(IntegerField,self).__init__(name, 'bigint')

class StringField(Field):
    def __init__(self, name):
        super(StringField, self).__init__(name,'varchar(100)')
```
Field类用来管理数据库的字段和字段类型
实现ModelMetaclass这个元类，用于限制User类
``` python
class ModelMetaclass(type):
    def __new__(cls,name,bases,attrs):
        if  name == 'Model':
            return type.__new__(cls, name, bases, attrs)
        print('Founc model: %s' % name)
        mappings = dict()
        for k, v in attrs.items():
            if isinstance(v,Field):
                print('Found mapping: %s ==> %s' %(k,v))
                mappings[k] = v
        for k in mappings.keys():
            attrs.pop(k)
        attrs['__mappings__'] = mappings
        attrs['__table__'] = name
        return type.__new__(cls, name, bases, attrs)
```
ModelMetaclass类中过滤了Model类的处理，将其他类的属性删除，将映射关系存储至__mappings__属性字段，并且__table__字段用来存储类名。
下面实现Model类
``` python
class Model(dict , metaclass=ModelMetaclass):
    def __init__(self, **kw):
        super(Model, self).__init__(**kw)
    def __getattr__(self, key):
        try:
            return self[key]
        except KeyError:
            raise AttributeError(r"'Model' object has no attribute '%s' " %key)
    def __setattr__(self, key, value):
        self[key]=value

    def save(self):
        fields = []
        params = []
        args = []
        for k, v in self.__mappings__.items():
            fields.append(v.name)
            params.append('?')
            args.append(getattr(self, k, None))
        sql = 'insert into %s (%s) values (%s)' % (self.__table__, ','.join(fields), ','.join(params))
        print('SQL: %s' % sql)
        print('ARGS: %s' % str(args))
```
Model 类继承了dict, 实现save函数，save函数将__mappings__属性值内容取出来，就是在ModelMetaclass中构造的mappings,其他类继承Model，调用save函数可以动态调用sql语句。
定义User类并且调用save
``` python
class User(Model):
    # 定义类的属性到列的映射：
    id = IntegerField('id')
    name = StringField('username')
    email = StringField('email')
    password = StringField('password')

u = User(id=12345, name='Michael', email='test@orm.org', password='my-pwd')
u.save()
```
输出如下：
![1.png](1.png)
