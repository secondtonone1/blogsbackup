---
title: 博客系统后台开发(一)服务结构
date: 2021-12-02 15:32:02
categories: [golang]
tags: [golang]
---
## 简介
基于gin框架搭建一个博客系统后台，返回html，json等数据与前端交互，包括登录模块，session维持，redis读写缓存，mongo读写等多种技术综合应用，意在打造一个高可用的稳定性博客后台。目前后台已经稳定运行，演示地址http://81.68.86.146:8080/, 源码地址：
https://github.com/secondtonone1/bstgo-blog
<!--more-->
## 项目结构
![https://cdn.llfc.club/1638432058%281%29.jpg](https://cdn.llfc.club/1638432058%281%29.jpg)
config: 文件夹放的是配置文件以及配置管理模块
demopic: demo图片，没什么用
dockerconfig: docker用到的配置文件
log: 日志文件
logger: 日志模块
model: 数据库模型和redis模型
mongo: mongo模块
redis: redis模块
public: 前端用到的js,lib等资源
router: 路由模块

## 本节目标
本节意在实现gin框架基本调用，启动gin服务，编写基础的路由和模板回传
## 主函数启动gin框架
在主函数中启动了gin server服务
然后添加views目录为go得模板路径，设置/static为资源路径，关联得是public文件夹下得静态资源。
然后添加了几个路由
``` golang
package main

import (
	"bstgo-blog/router/home"
	"net/http"

	"bstgo-blog/router/admin"

	"github.com/gin-gonic/gin"
)

func main() {
	router := gin.Default()
	router.LoadHTMLGlob("views/**/*")
	router.StaticFS("/static", http.Dir("./public"))
	router.GET("/home", home.Home)
	router.GET("/category", home.Category)
	router.GET("/admin", admin.Admin)
	router.POST("/admin/category", admin.Category)
	router.POST("/admin/sort", admin.Sort)
	router.POST("/admin/sortsave", admin.SortSave)
	router.POST("/admin/index", admin.IndexList)
	router.Run(":8080")
}
```
## admin和home路由
因为要实现前台展示和后台管理，admin是后台管理得模块，home是客户展示模块
admin模块目前包括两个文件admin.go和category.go
![https://cdn.llfc.club/1638435991%281%29.jpg](https://cdn.llfc.club/1638435991%281%29.jpg)

admin.go文件定义了Admin函数返回管理页面admin/index.html
``` golang
package admin

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func Admin(c *gin.Context) {
	c.HTML(http.StatusOK, "admin/index.html", nil)
}
```
同理，category.go文件定义了几个函数
``` golang
package admin

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func Category(c *gin.Context) {
	c.HTML(http.StatusOK, "admin/articlecateg.html", nil)
}

func Sort(c *gin.Context) {
	c.HTML(http.StatusOK, "admin/articlesort.html", nil)
}

func SortSave(c *gin.Context) {
	c.HTML(http.StatusOK, "admin/articlecateg.html", nil)
}

func IndexList(c *gin.Context) {
	c.HTML(http.StatusOK, "admin/indexlist.html", nil)
}
```
## html模板
我们将views目录下建立两个文件夹admin和home分别存储管理后台和前台显示的html模板文件
![https://cdn.llfc.club/1638436339%281%29.jpg](https://cdn.llfc.club/1638436339%281%29.jpg)
admin下的模板文件
![https://cdn.llfc.club/1638436547.jpg](https://cdn.llfc.club/1638436547.jpg)
具体html内容就不粘贴了，去github下载源码即可[https://github.com/secondtonone1/bstgo-blog/tree/day01](https://github.com/secondtonone1/bstgo-blog/tree/day01)
## 测试访问
执行命令
``` bash
go run ./main.go 
```
然后在控制台输入localhost:8080/home
可以看到如下效果
![https://cdn.llfc.club/1638437133%281%29.jpg](https://cdn.llfc.club/1638437133%281%29.jpg)




