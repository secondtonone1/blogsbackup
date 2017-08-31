---
title: python自学笔记(二)
date: 2017-08-17 15:07:01
categories: 技术开发
tags: [python]
---
通过前文介绍，大体上可以用学过的知识做一些东西了。

这里简单介绍下python参数解析`argparse命令`。

使用`argparse`需要引用

 `import argparse`

 然后调用

`parser = argparse.ArgumentParser()`

`ArgumentParser()函数可以传一些参数 `

`parser = argparse.ArgumentParser(description='This is a PyMOTW sample program')`

参数有很多种类型，读者自己查阅，参考资料的链接：

[https://blog.ixxoo.me/argparse.html](https://blog.ixxoo.me/argparse.html)
<!--more-->
 
接下来添加参数

`parser.add_argument('file')`

`parser.add_argument('-o', '--output')`

添加参数 `-表示可选参数`,`用于一个字符，表示缩写`

`--也是可选参数`，`用于两个或以上的字符`

最后是参数解析  

`parser.parse_args(['-o', 'output.txt'])`

`parse_args()运行时，会用'-'来认证可选参数，剩下的即为位置参数`。

位置参数不可缺少，可选参数可提供默认值

`如果python程序运行，parse_args()会依次处理传入参数`，`第一个参数为该python程序的文件名`，其余的依次为传入参数。

这些文字看不懂不要紧，试着看看下边的程序和运行结果

``` python
#-*-coding:utf-8-*-

import argparse

#命令行输入参数处理
parser = argparse.ArgumentParser()

parser.add_argument('file')     #输入文件
parser.add_argument('-o', '--output')   #输出文件
parser.add_argument('--width', type = int, default = 50) #输出字符画宽
parser.add_argument('--height', type = int, default = 30) #输出字符画高

#获取参数
args = parser.parse_args()

IMG = args.file
WIDTH = args.width
HEIGHT = args.height
OUTPUT = args.output

print("IMG is %s" %(IMG))
print("WIDTH is %d" %(WIDTH))
print("HEIGHT is %d" %(HEIGHT))
print("OUTPUT is %s" %(OUTPUT))

```
这个程序就是解析命令行参数，然后将输入的参数打印出来

如果不输入参数直接python test.py 试试？
![1.png](1.png)
提示缺少位置参数file

试试python test.py test.png
四个命令行参数打印出来了 

--height 可选参数为默认值30

--width 可选参数为默认值50

file 位置参数为test.png

-o 为--output的缩写为None，因为没提供默认值。我也没输入-o参数

所以为None

输入python test.py test.png --width 30 -- height 50 -o output.txt
![2.png](2.png)
用全称--output录入也可以

python test.py test.png --width 30 -- height 50 --output output.txt
![3.png](3.png)
这个过了就可以往下做了，下面安装PIL库，PIL为python处理图形图像的基本库

windows安装的方式为：

[http://jingyan.baidu.com/article/ff42efa929e6c8c19f220254.html](http://jingyan.baidu.com/article/ff42efa929e6c8c19f220254.html)

Linux安装方式为：

[安装方式](https://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/00140767171357714f87a053a824ffd811d98a83b58ec13000)

 下面编写图片转字符画程序：

将文件命名为print.py

1 `包含库和函数，定义基本的字符序列`
``` python
from PIL import Image
import argparse

#命令行输入参数处理
parser = argparse.ArgumentParser()

parser.add_argument('file')     #输入文件
parser.add_argument('-o', '--output')   #输出文件
parser.add_argument('--width', type = int, default = 50) #输出字符画宽
parser.add_argument('--height', type = int, default = 30) #输出字符画高

#获取参数
args = parser.parse_args()

IMG = args.file
WIDTH = args.width
HEIGHT = args.height
OUTPUT = args.output

ascii_char = list("$@B%8&WM#*oahkbdpqwmZO0QLCJUYXzcvunxrjft/\|()1{}[]?-_+~<>i!lI;:,\"^`'. ")
```
2 `编写转换函数`
``` python
# 将256灰度映射到70个字符上
def get_char(r,g,b,alpha = 256):
    if alpha == 0:
        return ' '
    length = len(ascii_char)
    gray = int(0.2126 * r + 0.7152 * g + 0.0722 * b)

    unit = (256.0 + 1)/length
    return ascii_char[int(gray/unit)]
```
0.2126 * r + 0.7152 * g + 0.0722 * b为官方提供的

灰度计算公式，unit表示一个单元占多少灰度，

gray/unit可以找到对应的单元，从而转换为字符。


3 `调用get_char完成转换`

``` python
if __name__ == '__main__':

    im = Image.open(IMG)
    im = im.resize((WIDTH,HEIGHT), Image.NEAREST)

    txt = ""

    for i in range(HEIGHT):
        for j in range(WIDTH):
            txt += get_char(*im.getpixel((j,i)))
        txt += '\n'

    print (txt)

    #字符画输出到文件
    if OUTPUT:
        with open(OUTPUT,'w') as f:
            f.write(txt)
    else:
        with open("output.txt",'w') as f:
            f.write(txt)
```

 `Image`为`PIL`提供的类，可以看看

[Image基本功能](http://www.cnblogs.com/way_testlife/archive/2011/04/17/2019013.html)

im.getpixel((j,i)) 通过传入横纵坐标，返回tuple

tuple中数据依次为r,g,b,alpha

之前讲过可以通过*() 或*[]实现逐个元素传入。

get_char(*im.getpixel((j,i)))将参数传入返回字符。

之后分别将字符打印出来，并写入文件。
源码下载地址： [python图片转字符画](http://download.csdn.net/detail/secondtonone1/9834692)

效果如下：
![4.png](4.png)
![5.png](5.png)
谢谢关注，我的公众号：
![6.jpg](6.jpg)


