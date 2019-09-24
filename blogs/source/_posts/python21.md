---
title: python学习(21) smtp发送邮件
date: 2018-01-06 10:07:23
categories: [python]
tags: [python]
---
本文介绍python发送邮件模块smtplib以及相关MIME模块。
smtplib用于生成邮件发送的代理，发送邮件前需要通过MIMEText构造邮件内容。
## 发送纯文本邮件
下面是个发送纯文本邮件的例子。
``` python
import smtplib
from email.mime.text import MIMEText
msg_from='XXXXX@163.com'                                 
passwd='XXXXX'                                  
msg_to='XXXXX@qq.com'                                  
                            
subject="python邮件测试"                                       
content="这是我使用python smtplib及email模块发送的邮件"

msg = MIMEText(content)
msg['Subject'] = subject
msg['From'] = msg_from
msg['To'] = msg_to
try:
    #s = smtplib.SMTP_SSL("smtp.163.com",465)
    s = smtplib.SMTP("smtp.163.com",25)
    s.login(msg_from, passwd)
    s.sendmail(msg_from, msg_to, msg.as_string())
    print ("发送成功")
except smtplib.SMTPException as e:
    print ("发送失败")
finally:
    s.quit()
```
<!--more-->
MIMEText实例化一个邮件对象，内容为content，对于邮件标题Subject，发件人From，以及收件人To
需要以字典形式指出，或者通过add_header(下文会给出)添加，否则对方看不到这些信息。
想要通过smtp发送邮件，需要打开指定邮箱的smtp协议，以及设置smtp授权密码。我设置的是163邮箱的。
![1.png](1.png)
![2.png](2.png)
设置好密码后，将上述代码中的passwd改为你的密码，msg_from改为你的邮箱。smtplib可以通过SMTP_SSL
发送，也可以采用普通形式直接初始化，对应的两个参数分别是授权的smtp服务器地址和端口号，因为我
设置的是163的，所以使用smtp.163.com服务器地址，端口号和服务器地址读者可以自己去查。通过生成的
smtp实例，一次调用login，sendemail就可以发送了。最后记得调用quit退出。
发送一封纯文本邮件，看一下效果
![3.png](3.png)
我们发现发件人标题显示的只有邮箱地址，没有昵称，可以采用parseaddr和formataddr对发件人信息完善。
``` python
def _format_addr(s):
    name, addr = parseaddr(s)
    return formataddr((Header(name, 'utf-8').encode(), addr))

import smtplib
from email.header import Header
from email.mime.text import MIMEText
from email.utils import parseaddr, formataddr

msg_from='XXXXX@163.com'                                 
passwd='XXXXX'                                  
msg_to='XXXXX@qq.com'
receivers = ['XXXXX@qq.com']                                
                            
subject="python邮件测试"                                       
content="这是我使用python smtplib及email模块发送的邮件"

msg = MIMEText(content,'plain','utf-8')
msg['Subject'] = Header(subject,'utf-8').encode()
msg['From'] = _format_addr('恋恋风辰 <%s>' %msg_from)
msg['To'] = msg_to

try:
    #s = smtplib.SMTP_SSL("smtp.163.com",465)
    s = smtplib.SMTP("smtp.163.com",25)
    s.login(msg_from, passwd)
    s.sendmail(msg_from, receivers, msg.as_string())
    print ("发送成功")
except smtplib.SMTPException as e:
    print ("发送失败")
finally:
    s.quit()
```
这样可以看到发件人的昵称了。我设置的是恋恋风辰。Header函数的作用是防止中文乱码。
Header对字符串按照utf-8方式编码。MIMEText中参数plain表示纯文本，utf-8表示纯文本
的编码方式。
![4.png](4.png)
## 发送html邮件
发送html邮件和之前发送纯文本类似，只需要将plain变为html，即可。
``` python
def _format_addr(s):
    name, addr = parseaddr(s)
    return formataddr((Header(name, 'utf-8').encode(), addr))

import smtplib
from email.header import Header
from email.mime.text import MIMEText
from email.utils import parseaddr, formataddr
msg_from = 'XXXXXX@163.com'
passwd = 'XXXXX'
msg_to='XXXXXX@qq.com'
receivers = ['XXXXXX@qq.com']
subject = 'python邮件测试html'
content = '<html><body><h1>Hello</h1>' +\
    '<p>send by <a href="http://www.python.org">Python</a>...</p>'

msg = MIMEText(content, 'html', 'utf-8')
msg['Subject'] = Header(subject, 'utf-8').encode()
msg['From'] = _format_addr('恋恋风辰 <%s>' %msg_from)
msg['To'] = msg_to

try:
    s = smtplib.SMTP("smtp.163.com",25)
    s.login(msg_from, passwd)
    s.sendmail(msg_from, receivers, msg.as_string())
    print('发送成功')
except smtplib.SMTPException as e:
    print('发送失败')
finally:
    s.quit()
```
看看效果：
![5.png](5.png)
## 发送带附件的邮件
发送带附件的邮件，和之前不同，需要通过MIMEMultipart创建邮件实例，
然后将文本，附件等通过attach方法绑定到邮件实例上，然后一起发送。
``` python
import smtplib
import email
from email.header import Header
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart   
from email.utils import parseaddr, formataddr
from email.mime.base import MIMEBase


def _format_addr(s):
    name, addr = parseaddr(s)
    return formataddr((Header(name, 'utf-8').encode(), addr))

msg_from = 'XXXXX@163.com'
passwd = 'XXXXX'
msg_to='XXXXXX@qq.com'
receivers = ['XXXX@qq.com']
subject = 'python邮件测试附件'
content = '<html><body><h1>Hello</h1>' +\
    '<p>send by <a href="http://www.python.org">Python</a>...</p>'

#附件邮件对象
msg = MIMEMultipart()
msg['From'] = _format_addr('恋恋风辰 <%s>' %msg_from)
msg['To'] = msg_to
msg['Subject'] = Header(subject, 'utf-8').encode()
#添加正文
text = MIMEText(content, 'html','utf-8')
msg.attach(text)
#添加附件就是创建一个MIMEBase对象，然后attach到msg上。
with open('./email.jpg','rb') as f:
    #设置附件名字
    mime = MIMEBase('image', 'jpg', filename='text.jpg')
    #加上头信息
    mime.add_header('Content-Disposition','attachment',filename='test.jpg')
    mime.add_header('Content-ID','<0>')
    mime.add_header('X-Attachment-Id','0')
    #读取内容放入附件
    mime.set_payload(f.read())
    #用Base64编码
    email.encoders.encode_base64(mime)
    #添加到MIMEMultipart中
    msg.attach(mime)

try:
    s = smtplib.SMTP("smtp.163.com",25)
    s.login(msg_from, passwd)
    s.sendmail(msg_from, receivers, msg.as_string())
    print('发送成功')
except smtplib.SMTPException as e:
    print('发送失败')
finally:
    s.quit()
```
MIMEMultipart创建邮件实例msg，将收件人，发件人，主题设置到msg上。
然后通过MIMEText创建html文本内容，调用msg.attach方法将文本内容绑定
到邮件上。同样的道理，打开一个图片，通过MIMEBase创建一个附件实例，
设置文件名，文件类型，绑定的id等等，最后通过set_payload加载到附件，
然后msg.attach绑定到邮件实例上。后面的发送流程和之前一样。
看看效果：
![6.png](6.png)
## 发送带图片的html邮件
想要在html中添加图片，并且在邮件正文中显示，只需要在html文本中引用
图片id即可。
``` python
import smtplib
from email.header import Header
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart   
from email.utils import parseaddr, formataddr
from email.mime.base import MIMEBase
from email.mime.image import MIMEImage


def _format_addr(s):
    name, addr = parseaddr(s)
    return formataddr((Header(name, 'utf-8').encode(), addr))

msg_from = 'XXXXXXXXXXXX@163.com'
passwd = 'XXXXX'
msg_to='XXXXXXXXX@qq.com'
receivers = ['XXXXXXXXXX@qq.com']
subject = 'python邮件测试附件'
content = '<b>Some <i>HTML</i> text</b> and an image.<br><img src="cid:image1"><br>good!'

#附件邮件对象
msg = MIMEMultipart()
msg['From'] = _format_addr('恋恋风辰 <%s>' %msg_from)
msg['To'] = msg_to
msg['Subject'] = Header(subject, 'utf-8').encode()
#添加正文
text = MIMEText(content, 'html','utf-8')
msg.attach(text)

#添加附件就是创建一个MIMEBase对象，然后attach到msg上。
with open('./email.jpg','rb') as f:
    #设置附件名字
    mime = MIMEImage(f.read())
    #加上头信息
    mime.add_header('Content-Disposition','attachment',filename='test.jpg')
    mime.add_header('Content-ID','`<image1>`')
   
    #添加到MIMEMultipart中
    msg.attach(mime)

try:
    s = smtplib.SMTP("smtp.163.com",25)
    s.login(msg_from, passwd)
    s.sendmail(msg_from, receivers, msg.as_string())
    print('发送成功')
except smtplib.SMTPException as e:
    print('发送失败')
finally:
    s.quit()
```
 mime.add_header('Content-ID',`'<image1>'`) 设置图片id为image1，
 在html中引用image1就可以在邮件中文中显示图片了。
 通过`<img src="cid:image1">`方式进行引用。
 ## Messge类的继承和派生关系
 ``` python
 Message
+- MIMEBase
   +- MIMEMultipart
   +- MIMENonMultipart
      +- MIMEMessage
      +- MIMEText
      +- MIMEImage
 ```
MIMEBase继承于Message,MIMEMultipart继承于MIMEBase。
## 用MIMEText发送多种附件
``` python
import smtplib
from email.mime.text import MIMEText
from email.header import Header
from email.mime.multipart import MIMEMultipart
from email.utils import parseaddr, formataddr
import os

def _format_addr(s):
    name, addr = parseaddr(s)
    return formataddr((Header(name, 'utf-8').encode(), addr))

msg_from = 'XXXXXXXXX@163.com'
passwd = 'XXXXXXXXXXX'
msg_to='XXXXXXXXX@qq.com'
receivers = ['XXXXXXXXXXX@qq.com']
subject = 'python邮件测试附件'
content = '多种附件'

#附件邮件对象
msg = MIMEMultipart()
msg['From'] = _format_addr('恋恋风辰 <%s>' %msg_from)
msg['To'] = msg_to
msg['Subject'] = Header(subject, 'utf-8').encode()
#添加正文
text = MIMEText(content, 'html','utf-8')
msg.attach(text)

os.chdir('./res')    
dir = os.getcwd()

for fn in os.listdir(dir):
    print(fn)
    with open(fn,'rb') as f:
        mime = MIMEText(f.read(), 'base64', 'utf-8')
        mime.add_header('Content-Disposition','attachment',filename = fn)
        mime.add_header('Content-Type', 'application/octet-stream')
        msg.attach(mime)

try:
    s = smtplib.SMTP("smtp.163.com",25)
    s.login(msg_from, passwd)
    s.sendmail(msg_from, receivers, msg.as_string())
    print('发送成功')
except smtplib.SMTPException as e:
    print('发送失败')
finally:
    s.quit()
```
大体原理和之前一样，通过MIMEText可以实现多种附件的发送。
注意格式改为base64，编码用utf-8，可以实现多种附件发送。
效果如下：
![7.png](7.png)
## 通过MIMEApplication发送多种附件
同样可以通过MIMEApplication发送多种附件。
``` python
import smtplib
from email.mime.text import MIMEText
from email.mime.application import MIMEApplication
from email.header import Header
from email.mime.multipart import MIMEMultipart
from email.utils import parseaddr, formataddr
import os

def _format_addr(s):
    name, addr = parseaddr(s)
    return formataddr((Header(name, 'utf-8').encode(), addr))

msg_from = 'xxxxxxxxx@163.com'
passwd = 'xxxxxxxxxx'
msg_to='xxxxxxxxxxx@qq.com'
receivers = ['xxxxxxxxx@qq.com']
subject = 'python邮件测试附件'
content = '多种附件'

#附件邮件对象
msg = MIMEMultipart()
msg['From'] = _format_addr('恋恋风辰 <%s>' %msg_from)
msg['To'] = msg_to
msg['Subject'] = Header(subject, 'utf-8').encode()
#添加正文
text = MIMEText(content, 'html','utf-8')
msg.attach(text)

os.chdir('./res')    
dir = os.getcwd()

for fn in os.listdir(dir):
    print(fn)
    with open(fn,'rb') as f:
        mime = MIMEApplication(f.read())
        mime.add_header('Content-Disposition','attachment',filename = fn)
        mime.add_header('Content-Type', 'application/octet-stream')
        msg.attach(mime)

try:
    s = smtplib.SMTP("smtp.163.com",25)
    s.login(msg_from, passwd)
    s.sendmail(msg_from, receivers, msg.as_string())
    print('发送成功')
except smtplib.SMTPException as e:
    print('发送失败')
finally:
    s.quit()
```
效果和之前的一样，这就是python中利用smtplib和MIME构造邮件发送的案例。
我的公众号：
![wxgzh.jpg](wxgzh.jpg)
