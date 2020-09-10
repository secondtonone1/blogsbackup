---
title: Dockerfile实战例子
date: 2020-09-09 09:25:15
categories: [docker]
tags: [docker]
---
## 构建flask镜像
先实现一个flask的python程序app.py
``` python
from flask import Flask
app = Flask(__name__)
@app.route('/')
def index():
    return 'Hello World'
if __name__ == '__main__':
    app.debug = True # 设置调试模式，生产模式的时候要关掉debug
    app.run(host='0.0.0.0',port=5000)
```
<!--more-->
接下来实现一个Dockerfile，构造python镜像
``` Dockerfile
FROM python:3.6
RUN  pip install flask
WORKDIR /app
COPY app.py  /app/
EXPOSE 5000
CMD ["python", "app.py"]
```
创建镜像
``` cmd
docker build -t secondtonone1/python-flask .
```
启动容器
``` cmd
docker run -d --name py-flask -p 5001:5000 550aa063e1bc
```
这时候通过网页输入 服务器ip:5001即可看到输出hello world
## 容器配置stress
Dockerfile配置stress
``` Dockerfile
FROM ubuntu:18.04
RUN apt-get update && apt-get install -y stress
ENTRYPOINT ["usr/bin/stress"]
CMD ["--vm 1 --vm-bytes 128M --verbose"]
```
接下来生成镜像
``` cmd
docker build -t secondtonone1/stress .
```
启动容器
``` cmd
docker run -it --rm secondtonone1/stress
```
可以看到默认是使用ENTRYPOINT里的命令，弹出了help提示
我们重新启动一个新的容器，后边带着参数，这样可以覆盖Dockerfile的CMD
``` cmd
docker run -it --rm secondtonone1/stress --vm 1 --vm-bytes 128M --verbose
```
## 个人公众号
![wxgzh.jpg](wxgzh.jpg)






