---
title: python学习(22)访问数据库
date: 2018-01-11 18:23:02
categories: 技术开发
tags: [python]
---
本文介绍python如何使用数据库方面的知识。
## SQLite
SQLite是一种嵌入式数据库，本身是*.db的文件。通过python操作数据库的步骤：
1 连接数据库返回connection连接
2 通过connection连接获取cursor，cursor即游标
3 通过cursor执行语句
4 通过cursor查询结果，如fetchall
5 关闭游标cursor
6 提交事务
7 关闭连接
<!--more-->
下边的例子说明了如何使用SQLite
``` python
import sqlite3
#连接数据库test.db文件，如果不存在则创建
conn = sqlite3.connect('test.db')
#获取游标
cursor = conn.cursor()
#通过游标创建表
cursor.execute('create table user (id varchar(20) primary key, name varchar(20) )')
#通过游标插入数据
cursor.execute('insert into user(id, name) values(\'1\',\'Bob\')')
#打印新增的行数
print(cursor.rowcount)
#关闭cursor
cursor.close()
#提交事务
conn.commit()
#关闭连接
conn.close()
```
## MYSQL
Mysql是使用广泛的数据库，通过python同样可以访问mysql数据库。步骤和之前的一样。但是要提前安装mysql数据库，
同时推荐安装navicat可视化处理工具。安装完数据库后，需要安装python访问mysql的库，推荐安装2.1.4版本，其他
版本可能会需要安装其他支持的库。命令行下执行：pip install mysql-connector==2.1.4
这些安装好后，可以写代码访问数据。
``` python
import mysql.connector
conn = mysql.connector.connect(user='root',password='123456',database='test')
cursor = conn.cursor()
#创建user表
cursor.execute('create table user(id varchar(20) primary key, name varchar(20))')
cursor.execute('insert into user(id, name) values(%s,%s)',['1','Michael'])
print(cursor.rowcount)
cursor.close()
conn.commit()
cursor = conn.cursor()
cursor.execute('select * from user where id = %s',('1',))
#获取结果集
values = cursor.fetchall()
print(values)

# 关闭Cursor和Connection:
cursor.close()

conn.close()
```

## SQLAlchemy
SQLAlchemy是python的一个库，可以支持把关系型数据库的表结构映射到类的对象上，通过操作对象从而达到操作数据库
的目的，这就是ORM技术，Object-Relational Mapping
在使用SQLAlchemy之前，需要先安装pip install sqlalchemy
下面的例子说明了如何使用sqlalchemy
``` python
from sqlalchemy import Column, String, create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base
# 创建对象的基类:
Base = declarative_base()

# 定义User对象:
class User(Base):
	#表的名字：
	__tablename__ = 'user'
	#表的结构：
	id = Column(String(20), primary_key = True)
	name = Column(String(20))


# 初始化数据库连接:
engine = create_engine('mysql+mysqlconnector://root:123456@localhost:3306/test')
# 创建DBSession类型:
DBSession = sessionmaker(bind=engine)

# 创建session对象:
session = DBSession()

# 创建新User对象:
new_user = User(id='5', name='Bob')

# 添加到session:
session.add(new_user)
# 提交即保存到数据库:
session.commit()
# 关闭session:
session.close()

# 创建Session:
session = DBSession()
# 创建Query查询，
user = session.query(User).filter(User.id=='5').one()
# 打印类型和对象的name属性:
print('type:', type(user))
print('name:', user.name)
# 关闭Session:
session.close()
```
使用sqlalchemy分几个步骤：
1 用sqlalchemy创建一个基类Base
2 基于Base类实现自己定义的表结构类
3 通过create_engine初始化数据库连接返回引擎engine，格式为
  ‘数据库类型+数据库驱动名称://用户名:口令@机器地址:端口号/数据库名’
  所以我的写成'mysql+mysqlconnector://root:123456@localhost:3306/test'
4 通过engine创建DBSession类，并通过DBSession类创建session(会话控制)对象。
5 接下来可以定义一些对象，把对象加入到session中，通过commit到数据库
6 也可以通过session查询用户，返回一个或几个结果。
7 操作完记得关闭session。

以上就是python使用数据库的基本知识。

