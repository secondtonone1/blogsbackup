---
title: python学习(十六)写爬虫获取糗事百科段子
date: 2017-12-19 19:04:33
categories: 技术开发
tags: [python]
---
利用前面学到的文件、正则表达式、urllib的知识，综合运用，爬取糗事百科的段子
先用urllib库获取糗事百科热帖第一页的数据。并打开文件进行保存，正好可以熟悉一下之前学过的文件知识。

``` python
from urllib import request, parse
from urllib import error
page = 1
url = 'https://www.qiushibaike.com/hot/page/'+str(page)
user_agent = 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.221 Safari/537.36 SE 2.X MetaSr 1.0'
try:
	req = request.Request(url)
	req.add_header('User-Agent', user_agent)
	response = request.urlopen(req)
	#bytes变为字符串
	content = response.read().decode('utf-8')
	print(type(content))
	#uf-8编码方式打开
	file = open('file.txt', 'w',encoding='utf-8')

	file.write(content)
	
except error.URLError as e:
	if hasattr(e,'code'):
		print (e.code)
	if hasattr(e,'reason'):
		print (e.reason)
finally:
	file.close()
```
<!--more-->
打开文件可以看到如下内容：
![1.png](1.png)
div class="article block untagged mb15 typs_long" id='qiushi_tag_119848276'表示一个文章的开始，id为文章对应的id，
h2 之间的是发布者的姓名‘高老庄福帅猪刚鬣’，span与/span之间的是正文， i class="number"与/i，635表示赞的个数，
同样也可以获取评论的个数。

下面要用到学过的正则表达式的知识，过滤掉没有用的信息，只获取评论数，作者，正文，以及点赞的数量。
``` python
import re

with open('file.txt','r', encoding='utf-8') as f:
	data = f.read()

pattern = re.compile(r'<div.*?<h2>(.*?)</h2>.*?<span>(.*?)</span>.*?number">(.*?)</i>.*?'+
	r'"number">(.*?)</i>', re.S )
result = re.search(pattern, data)
#print(result)
#print(result.group())
print(result.group(1))
print(result.group(2))
print(result.group(3))
print(result.group(4))
```
re.compile(),参数re.S表示将.的作用扩充为任意字符，因为前几篇文章讲述过.在一般情况下匹配除/n之外的所有字符。
正则表达式中`.*?`连起来匹配任意字符，且为非贪婪模式。因为.表示任意字符，`*`表示匹配前一个字符0个或多个，
`.*`表示匹配任意多个字符，而加上？表示非贪婪模式。
re.search是搜索匹配正则表达式规则的条目，search讲述过可以从内容的任意位置查找。这样就可以找到一个符合这种规则
的段子。如果找到所有符合规则的段子可以用re.findall进行查找。

下面一气呵成，将网站上的段子按照正则表达式匹配，并将匹配后的段子写入文件，同时在终端显示。
``` python
from urllib import request, parse
from urllib import error
page = 1
url = 'https://www.qiushibaike.com/hot/page/'+str(page)
user_agent = 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.221 Safari/537.36 SE 2.X MetaSr 1.0'
try:
	req = request.Request(url)
	req.add_header('User-Agent', user_agent)
	response = request.urlopen(req)
	#bytes变为字符串
	content = response.read().decode('utf-8')
	pattern = re.compile(r'<div.*?<h2>(.*?)</h2>.*?<span>(.*?)</span>.*?number">(.*?)</i>.*?'+
	r'"number">(.*?)</i>', re.S )
	result = re.findall(pattern, content)
	files = open('findfile.txt','w+', encoding='utf-8')
	for item in result:
		author =  item[0]
		contant = item[1]
		vote = '赞：'+item[2]
		commit = '评论数：'+item[3]
		infos = vote +' '+commit+' '+'\n\n'
		print(author)
		print(contant)
		print(infos)
		files.write(author)
		files.write(contant)
		files.write(infos)
		

except error.URLError as e:
	if hasattr(e,'code'):
		print (e.code)
	if hasattr(e,'reason'):
		print (e.reason)
finally:
	files.close()
```
效果如下：
![2.png](2.png)


