---
title: 'python学习(24) 使用Xpath解析并抓取美女图片'
date: 2018-11-18 19:42:13
categories: 技术开发
tags: [python]
---
Xpath最初用来处理XML解析，同样适用于HTML文档处理。相比正则表达式更方便一些
## Xpath基本规则
``` python
nodename   表示选取nodename 节点的所有子节点
/          表示当前节点的直接子节点
//         表示当前节点的子节点和孙子节点
.          表示当前节点
..         当前节点的父节点
@          选取属性
```
下面举例使用下
<!--more-->
``` python
text = '''
<div class="bus_vtem">
		<a href="https://www.aisinei.org/thread-17826-1-1.html" title="XINGYAN星颜社 2018.11.09 VOL.096 唐思琪 [47+1P]" class="preview"  target="_blank">
		<img src="https://i.asnpic.win/block/74/74eab64cfa4229d58c19a64970368178.jpg" width="250" height="375" alt="XINGYAN星颜社 2018.11.09 VOL.096 唐思琪 [47+1P]"/>
                <span class="bus_listag">XINGYAN星颜社</span>
		</a>
		<a href="https://www.aisinei.org/thread-17826-1-1.html" title="XINGYAN星颜社 2018.11.09 VOL.096 唐思琪 [47+1P]"  target="_blank">
			<div class="lv-face"><img src="https://www.aisinei.org/uc_server/avatar.php?uid=2&size=small" alt="发布组小乐"/></div>
			<div class="t">XINGYAN星颜社 2018.11.09 VOL.096 唐思琪 </div>
			<div class="i"><span><i class="bus_showicon bus_showicon_v"></i>5401</span><span><i class="bus_showicon bus_showicon_r"></i>1</span></div>
		</a>
	</div>
'''
from lxml import etree
html = etree.HTML(text)
result = etree.tostring(html)
#打印lxml生成的字符串，如果html格式不全，会自动补全
print(result.decode('utf-8'))
# 打印根节点下所有子孙节点
result2 = html.xpath('//*')
print(result2)
result3 = html.xpath('//a[@class="preview"]')
print(result3)
```
result.decode('utf-8') 可以补全缺失的html格式字符串
html.xpath('//*')查找根节点下所有子孙节点
html.xpath('//a[@class="preview"]') 在根节点所有子孙节点中找到属性class为preview的a节点。
## lxml同样可以读取文件
``` python
from lxml import etree
html = etree.parse('./test.html',etree.HTMLParser())
```
## lxml 操作子节点
``` python
from lxml import etree
html = etree.HTML(text)
result = html.xpath('//bus/a')
```
text 还是上面的字符串，这样可以取到bus节点下的所有子节点a
## 操作父节点
``` python
from lxml import etree
html = etree.HTML(text)
result = html.xpath('//a[@class="preview"]/../@class')
print(result)
```
先找到class属性为preview的a节点，然后找到其父节点，接着筛选父节点的class属性，打印结果为['bus_vtem']
## 属性匹配
上面已经写过了格式为： 节点名[@属性名="属性值"]
## 属性获取
上面已经谢过了，格式为: 节点名/@属性名，注意这里没有[]
## 多属性值匹配
上面的节点bus 属性class 只有一个值bus_vtem,如果新增一个值mtest，那么属性匹配要更换为contains，不然会报错
``` python
from lxml import etree
text2 = '''
        <div class="bus_vtem  mtest"> hurricane!
        </div>
    '''
html2 = etree.HTML(text2)    
result5 = html2.xpath('//*[contains(@class, "mtest")]')
# 错误用法
#result5 = html.xpath('//*[@class="mtest"]')
print(result5)
```
## 多属性匹配
多属性匹配用于筛选一个节点时非常方便，各个属性的判断可以用 and or != == 等操作
``` python
from lxml import etree
text3 = '''
        <div class="bus_vtem mtest" name="hurricane"> hurricane!
        </div>
        <div class="bus_vtem mtest" name = "tornado"> tornado!
        </div>
    '''
html3 = etree.HTML(text3)    
result6 = html3.xpath('//*[contains(@class, "mtest") and @name="hurricane"]/text()')
print(result6)
```
## 文本获取
在节点后加/text()即可，如
result6 = html3.xpath('//*[contains(@class, "mtest") and @name="hurricane"]/text()')

下面结合前边讲述的request，cookie，以及今天的lxml知识，实战爬取艾丝新发布的美女图片地址，代码如下
``` python
import requests
import re
import time
from lxml import etree

USER_AGENT = 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.221 Safari/537.36 SE 2.X MetaSr 1.0'
COOKIES = r'__cfduid=d78f862232687ba4aae00f617c0fd1ca81537854419; bg5D_2132_saltkey=jh7xllgK; bg5D_2132_lastvisit=1540536781; bg5D_2132_auth=479fTpQgthFjwwD6V1Xq8ky8wI2dzxJkPeJHEZyv3eqJqdTQOQWE74ttW1HchIUZpgsyN5Y9r1jtby9AwfRN1R89; bg5D_2132_lastcheckfeed=7469%7C1541145866; bg5D_2132_st_p=7469%7C1541642338%7Cda8e3f530a609251e7b04bfc94edecec; bg5D_2132_visitedfid=52; bg5D_2132_viewid=tid_14993; bg5D_2132_ulastactivity=caf0lAnOBNI8j%2FNwQJnPGXdw6EH%2Fj6DrvJqB%2Fvv6bVWR7kjOuehk; bg5D_2132_smile=1D1; bg5D_2132_seccode=22485.58c095bd095d57b101; bg5D_2132_lip=36.102.208.214%2C1541659184; bg5D_2132_sid=mElHBZ; Hm_lvt_b8d70b1e8d60fba1e9c8bd5d6b035f4c=1540540375,1540955353,1541145834,1541562930; Hm_lpvt_b8d70b1e8d60fba1e9c8bd5d6b035f4c=1541659189; bg5D_2132_sendmail=1; bg5D_2132_checkpm=1; bg5D_2132_lastact=1541659204%09misc.php%09patch'
class AsScrapy(object):
    def __init__(self,pages=1):
        try:
            self.m_session = requests.Session()
            self.m_headers = {'User-Agent':USER_AGENT,
                        #'referer':'https://www.aisinei.org/',
                        }
           
            self.m_cookiejar = requests.cookies.RequestsCookieJar()
            for cookie in COOKIES.split(';'):
                key,value = cookie.split('=',1)
                self.m_cookiejar.set(key,value)
        except:
            print('init error!!!')
    def getOverView(self):
        try:
            req = self.m_session.get('https://www.aisinei.org/portal.php',headers=self.m_headers, cookies=self.m_cookiejar, timeout=5)
            html = etree.HTML(req.content.decode('utf-8'))
            #result=html.xpath('//div[@class="bus_vtem"]/a[@title!="紧急通知！紧急通知！紧急通知！"]/attribute::*')
            #print(result)
            htmllist = html.xpath('//div[@class="bus_vtem"]/a[@title!="紧急通知！紧急通知！紧急通知！" and @class="preview"]/@href')
            titlelist = html.xpath('//div[@class="bus_vtem"]/a[@title!="紧急通知！紧急通知！紧急通知！" and @class="preview"]/@title')
            print(htmllist)
            print(titlelist)
            print(len(htmllist))
            print(len(titlelist))            
            time.sleep(1)
            pass
        except:
            print('get over view error')

if __name__ == "__main__":
    asscrapy = AsScrapy()
    asscrapy.getOverView()
```
通过lxml分析，可以摘取资源地址
![1.png](1.png)
接下来爬取图片，读者可以发送request请求即可，留作课后题吧。
源码下载地址
[https://github.com/secondtonone1/python-](https://github.com/secondtonone1/python-)
谢谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)

