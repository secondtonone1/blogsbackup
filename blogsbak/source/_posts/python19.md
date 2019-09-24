---
title: python学习(十九)常见的第三方库
date: 2017-12-28 11:00:37
categories: 技术开发
tags: [python]
---
介绍几个python中常见的第三方库.
## Pillow
Pillow简称PIL，是python中常用的图形图像处理模块。写一个简单的例子
``` python
from PIL import Image, ImageFilter
# 打开一个jpg图像文件，注意是当前路径:
im = Image.open('test.jpg')
#获取图片大小
w,h = im.size
print('Original image size : width:%d height: %d' %(w,h))

#图片缩放
im.thumbnail((w//2, h//2))

print('Resize image to: %dx%d' % (w//2, h//2))
# 把缩放后的图像用jpeg格式保存:
im.save('test2.jpg', 'jpeg')


# 打开一个jpg图像文件，注意是当前路径:
im = Image.open('test.jpg')
# 应用模糊滤镜:
im2 = im.filter(ImageFilter.BLUR)
im2.save('blur.jpg', 'jpeg')

im2 = im.filter(ImageFilter.CONTOUR)
im2.save('contour.jpg','jpeg')

```
<!--more-->
Image.open函数打开一张图片，然后调用thumbnail进行缩放，调用save进行存储。filter函数
为滤镜函数，可以匹配不同的滤镜模式，如模糊，边界效果等等。
原图：
![test.jpg](test.jpg)
通过滤镜模糊模式：
![blur.jpg](blur.jpg)
通过滤镜边界模式：
![contour.jpg](contour.jpg)

下面利用PIL库实现一个生成验证码的小程序
``` python
from PIL import Image, ImageDraw, ImageFont, ImageFilter
import random

#随机大写字母：
def rndChar():
	return chr(random.randint(65,90))
#随机颜色1:
def rndColor():
	return(random.randint(64,255), random.randint(64,255), random.randint(64,255))
#随机颜色2：
def rndColor2():
	return(random.randint(32,127), random.randint(32,127), random.randint(32,127))


#240*60
width = 60*4
height = 60
#Image.new(mode, size, color=None)
image = Image.new('RGB',(width,height), (255,255,255))
#创建Font对象
font = ImageFont.truetype('C:\\WINDOWS\\Fonts\\SIMYOU.TTF',36)
# 创建draw对象并和image绑定
#用于以后绘制像素点和文本
draw = ImageDraw.Draw(image)
#通过像素点绘制填充图片
for x in range(width):
	for y in range(height):
		draw.point((x,y),fill=rndColor())

#绘制字母
for t in range(4):
	draw.text((60*t+10,10),rndChar(),font=font, fill=rndColor2())
#模糊处理
#image = image.filter(ImageFilter.BLUR)
image.save('code.jpg','jpeg')
```
## chardet检测编码
``` python
import chardet
rs = chardet.detect(b'Hello, world!')
print(rs)

data = '江船火独明'.encode('gb2312')
rs = chardet.detect(data)
print(rs)

data2 = '此情可待成追忆'.encode('utf-8')
rs2 = chardet.detect(data2)
print(rs2)

```
用chardet可以判断编码方式，在不知道字节是按照什么格式编码时可以采用chardet。
## tkinter 制作GUI界面
``` python
from tkinter import *

class Application(Frame):
	def __init__(self, master = None):
		Frame.__init__(self,master)
		self.pack()
		self.createWidgets()

	def createWidgets(self):
		self.helloLabel = Label(self, text='Hello, world!')
		self.helloLabel.pack()
		self.quitButton = Button(self, text = 'Quit', command=self.quit)
		self.quitButton.pack()

app = Application()
# 设置窗口标题:
app.master.title('Hello World')
# 主消息循环:
app.mainloop()
```
pack()方法是将Widgets对象加载到父容器中。
具体的API读者可以查看手册。这些第三方库用到的时候再具体学习即可。





