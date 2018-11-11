---
title: python学习(23)requests库爬取猫眼电影
date: 2018-11-11 20:26:35
categories: 技术开发
tags: [python]
---
本文介绍如何结合前面讲解的基本知识，采用requests，正则表达式，cookies结合起来，做一次实战，抓取猫眼电影排名信息。
## 用requests写一个基本的爬虫
排行信息大致如下图
<!--more-->
![1.jpg](1.jpg)
网址链接为[http://maoyan.com/board/4?offset=0](http://maoyan.com/board/4?offset=0)
我们通过点击查看源文件，可以看到网页信息
![2.png](2.png)
每一个电影的html信息都是下边的这种结构
``` python
  <i class="board-index board-index-3">3</i>
    <a href="/films/2641" title="罗马假日" class="image-link" data-act="boarditem-click" data-val="{movieId:2641}">
      <img src="//ms0.meituan.net/mywww/image/loading_2.e3d934bf.png" alt="" class="poster-default" />
      <img data-src="http://p0.meituan.net/movie/54617769d96807e4d81804284ffe2a27239007.jpg@160w_220h_1e_1c" alt="罗马假日" class="board-img" />
    </a>
    <div class="board-item-main">
      <div class="board-item-content">
              <div class="movie-item-info">
        <p class="name"><a href="/films/2641" title="罗马假日" data-act="boarditem-click" data-val="{movieId:2641}">罗马假日</a></p>
        <p class="star">
                主演：格利高里·派克,奥黛丽·赫本,埃迪·艾伯特
        </p>
```
其实对我们有用的就是 img src(图片地址) title 电影名 star 主演。
所以根据前边介绍过的正则表达式写法，可以推导出正则表达式
``` python
compilestr = r'''<dd>.*?<i class="board-index.*?<img data-src="(.*?)@.*?title="(.*?)".*?<p class="star">
(.*?)</p>.*?<p class="releasetime">.*?(.*?)</p'''
```
'.'表示匹配任意字符，如果正则表达式用re.S模式，.还可以匹配换行符，'*'表示匹配前一个字符0到n个，'？'表示非贪婪匹配，
所以'.*?'可以理解为匹配任意字符。接下来写代码打印我们匹配的条目
``` python
#-*-coding:utf-8-*-
import requests
import re
USER_AGENT = 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.221 Safari/537.36 SE 2.X MetaSr 1.0'

if __name__ == "__main__":
    headers={'User-Agent':USER_AGENT,
           }
    session = requests.Session()
    req = session.get('http://maoyan.com/board/4?offset=0',headers = headers, timeout = 5)
    compilestr = r'<dd>.*?<i class="board-index.*?<img data-src="(.*?)@.*?title="(.*?)".*?<p class="star">(.*?)</p>.*?<p class="releasetime">.*?(.*?)</p'
    #print(req.content)
    pattern = re.compile(compilestr,re.S)
    #print(req.content.decode('utf-8'))
    lists = re.findall(pattern,req.content.decode('utf-8'))
    for item in lists:
        #print(item)
        print(item[0].strip())
        print(item[1].strip())
        print(item[2].strip())
        print(item[3].strip())
        print('\n')
    
```
运行一下，结果如下
![3.png](3.png)
看来我们抓取到数据了，我们只爬取了这一页的信息，接下来我们分析第二页，第三页的规律，点击第二页，网址变为'http://maoyan.com/board/4?offset=10',点击第三页网址变为'http://maoyan.com/board/4?offset=20'，所以每一页的offset偏移量为20，这样我们可以计算偏移量达到抓取不同页码的数据，将上一个程序稍作修改，变为可以爬取n页数据的程序
``` python
#-*-coding:utf-8-*-
import requests
import re
import time
USER_AGENT = 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.221 Safari/537.36 SE 2.X MetaSr 1.0'

class MaoYanScrapy(object):
    def __init__(self,pages=1):
        self.m_session = requests.Session()
        self.m_headers = {'User-Agent':USER_AGENT,}
        self.m_compilestr = r'<dd>.*?<i class="board-index.*?<img data-src="(.*?)@.*?title="(.*?)".*?<p class="star">(.*?)</p>.*?<p class="releasetime">.*?(.*?)</p'
        self.m_pattern = re.compile(self.m_compilestr,re.S)
        self.m_pages = pages
    
    def getPageData(self):
        try:
            for i in range(self.m_pages):
                httpstr = 'http://maoyan.com/board/4?offset='+str(i)
                req = self.m_session.get(httpstr,headers=self.m_headers,timeout=5)
                lists = re.findall(self.m_pattern,req.content.decode('utf-8'))
                time.sleep(1)
                for item in lists:
                    img = item[0]
                    print(img.strip()+'\n')
                    name = item[1]
                    print(name.strip()+'\n')
                    actor = item[2]
                    print(actor.strip()+'\n')
                    fiemtime = item[3]
                    print(fiemtime.strip()+'\n')
                

        except:
            print('get error')

if __name__ == "__main__":
    maoyanscrapy = MaoYanScrapy()
    maoyanscrapy.getPageData()
```
运行下，效果和之前一样，只是支持了页码的传参了。
下面继续完善下程序，把每个电影的图片抓取并保存下来，这里面用到了创建文件夹，路径拼接，文件保存的基础知识，综合运用如下
``` python
#-*-coding:utf-8-*-
import requests
import re
import time
import os
USER_AGENT = 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.221 Safari/537.36 SE 2.X MetaSr 1.0'

class MaoYanScrapy(object):
    def __init__(self,pages=1):
        self.m_session = requests.Session()
        self.m_headers = {'User-Agent':USER_AGENT,}
        self.m_compilestr = r'<dd>.*?<i class="board-index.*?<img data-src="(.*?)@.*?title="(.*?)".*?<p class="star">(.*?)</p>.*?<p class="releasetime">.*?(.*?)</p'
        self.m_pattern = re.compile(self.m_compilestr,re.S)
        self.m_pages = pages
        self.dirpath = os.path.split(os.path.abspath(__file__))[0]
        
    
    def getPageData(self):
        try:
            for i in range(self.m_pages):
                httpstr = 'http://maoyan.com/board/4?offset='+str(i)
                req = self.m_session.get(httpstr,headers=self.m_headers,timeout=5)
                lists = re.findall(self.m_pattern,req.content.decode('utf-8'))
                time.sleep(1)
                for item in lists:
                    img = item[0]
                    print(img.strip()+'\n')
                    name = item[1]
                    dirpath = os.path.join(self.dirpath,name)
                    if(os.path.exists(dirpath)==False):
                        os.makedirs(dirpath)
                    print(name.strip()+'\n')
                    actor = item[2]
                    print(actor.strip()+'\n')
                    fiemtime = item[3]
                    print(fiemtime.strip()+'\n')
                    txtname = name+'.txt'
                    txtname = os.path.join(dirpath,txtname)
                    if(os.path.exists(txtname)==True):
                        os.remove(txtname)
                    with open (txtname,'w') as f:
                        f.write(img.strip()+'\n')
                        f.write(name.strip()+'\n')
                        f.write(actor.strip()+'\n')
                        f.write(fiemtime.strip()+'\n')
                    picname=os.path.join(dirpath,name+'.'+img.split('.')[-1])
                    if(os.path.exists(picname)):
                        os.remove(picname)
                    req=self.m_session.get(img,headers=self.m_headers,timeout=5)
                    time.sleep(1)
                    with open(picname,'wb') as f:
                        f.write(req.content)
        except:
            print('get error')

if __name__ == "__main__":
    maoyanscrapy = MaoYanScrapy()
    maoyanscrapy.getPageData()
```
运行一下，可以看到在文件的目录里多了几个文件夹
![4.png](4.png)
点击一个文件夹，看到里边有我们保存的图片和信息
![5.png](5.png)
好了，到此为止，正则表达式和requests结合，做的爬虫实战完成。
谢谢关注我的公众号：
![wxgzh.jpg](wxgzh.jpg)


