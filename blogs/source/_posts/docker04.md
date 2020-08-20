---
title: Dockerfile入门
date: 2020-08-03 10:08:29
categories: [docker]
tags: [docker]
---
今天介绍下Dockerfile的基本命令和使用案例
## Dockerfile基本命令

``` cmd
FROM ：基础镜像，该镜像基于哪个镜像生成
MAINTAINER ：镜像维护者的姓名和邮箱
RUN ：构建容器时需要运行的命令
EXPOSE ：容器对外暴露的端口
WORKDIR ： 指定在创建容器后，终端默认登录进来的工作目录
ENV ：用来在构建镜像过程中设置环境变量
ADD : 将宿主机目录下的文件拷贝进镜像且ADD命令会自动处理URL和解压tar压缩包
COPY : 类似ADD，拷贝文件和目录到镜像中。将从构建上下文目录中
CMD : 指定容器启动时要运行的命令，如果有多个CMD命令，只有最后一个生效，CMD会被docker run 之后的参数替换。
ENTRYPOINT ：指定一个容器启动时要运行的命令，ENTRYPOINT的目的和CMD一样，都是指定容器启动程序及参数
ONBUILD ：当构建一个被继承的Dockerfile时运行命令，父镜像在被子继承后父镜像的onbuild被触发。
```
<!--more-->
## Dockerfile使用示例
``` Dockerfile
FROM centos
MAINTAINER zack<zack@163.com>

ENV MYPATH /usr/local
WORKDIR $MYPATH
RUN yum -y install vim
RUN yum -y install net-tools
EXPOSE 80

CMD echo $MYPATH
CMD echo "success---------ok"
CMD /bin/bash
```
## 根据Dockerfile生辰镜像
docker build -t mycentos:v1.0 .
## 启动容器
docker run -it --name mycentosdk  2081f3e41884

## ENTRYPOINT 案例
entrypoint可以接收 命令行参数，从而达到组合命令的效果
现在我们制作一个请求网页的镜像
``` Dockerfile
FROM centos
RUN yum install -y curl
ENTRYPOINT ["curl", "-s", "www.baidu.com" ]
```
接下来生成镜像
``` cmd
docker build -f ./Dockerfile -t ipcheck .
``
之后分别运行两个命令做对比
``` cmd
docker run --name ipcheck01 --rm ipcheck 
```
这个是进阶版本
``` cmd
docker run --name ipcheck02 --rm ipcheck -i
```
会分别看到不同的效果，-i 的启动方式会额外打印请求的头部信息。

## ONBUILD 案例
当子类镜像继承父类镜像时，ONBUILD会被执行,我们将上边的Dockerfile改进下
``` Dockerfile
FROM centos
RUN yum install -y curl
ENTRYPOINT ["curl", "-s", "www.baidu.com"]
ONBUILD RUN echo "father images onbuild ..."
```
然后我们生成镜像
``` cmd
docker build -f ./Dockerfile01 -t fatherdk .
```
我们再写一个子类Dockerfile
``` Dockerfile
FROM fatherdk
RUN yum install -y curl
ENTRYPOINT ["curl", "-s", "www.baidu.com"]
```
我们生成一个子类镜像
``` cmd
docker build -f ./Dockerfile02 -t sondk .
```
可以看到构建子类镜像同时会触发父类镜像
## 综合运用上述命令构造tomcat镜像
先写一个Dockerfile 安装tomcat以及jdk
``` Dockerfile
FROM centos
MAINTAINER zack<zack@126.com>
#把宿主机当前上下文的c.txt拷贝到容器/usr/local/路径下
COPY c.txt /usr/local/cincontainer.txt
#把java与tomcat添加到容器中
ADD jdk-8u144-linux-x64.tar.gz  /usr/local
ADD  apache-tomcat-9.0.10.tar.gz /usr/local
#安装vim 编辑器
RUN yum -y install vim
#设置工作访问时的WORKDIR路径，登录落脚点
ENV MYPATH /usr/local
WORKDIR $MYPATH
#配置java与tomcat环境变量
ENV JAVA_HOME   /usr/local/jdk1.8.0_144
ENV CLASSPATH   $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV CATALINA_HOME  /usr/local/apache-tomcat-9.0.10
ENV CATALINA_BASE  /usr/local/apache-tomcat-9.0.10
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
#容器运行时监听的端口
EXPOSE 8080
#启动时运行tomcat
# ENTRYPOIINT ["/usr/local/apache-tomcat-9.0.10/bin/startup.sh" ]
# CMD ["/usr/local/apache-tomcat-9.0.10/bin/catalina.sh", "run" ]
CMD /usr/local/apache-tomcat-9.0.10/bin/startup.sh && tail -F /usr/local/apache-tomcat-9.0.10/logs/catalina.out
```
生成镜像
``` Dockerfile
docker build -t zacktomcat .
```

运行容器
``` Dockerfile
docker run -d -p 9080:8080 --name myt9 -v /home/zack/dockerwork/tomcat9/test:/usr/local/apache-tomcat-9.0.10/webapps/test -v /home/zack/dockerwork/tomcat9/tomcat9logs/:/usr/local/apache-tomcat-9.0.10/logs --privileged=true zacktomcat
```
在浏览器输入地址和端口9080就可以看到tomcat的首页了。
## 感谢关注公众号
今天的笔记就这些吧，感谢关注公众号
![wxgzh.jpg](wxgzh.jpg)