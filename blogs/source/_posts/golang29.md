---
title: Go项目实战：打造高并发日志采集系统（十）
date: 2020-04-02 13:42:06
categories: [golang]
tags: [golang]
---
## 前情回顾
前文我们完成了日志管理系统后台开发。
## 本节目标
这次为日志管理搭建一个web管理平台，可以通过web端录入项目和配置信息，以及项目对应的日志路径和采集信息，并且写入etcd，
这样通过之前编写的日志采集系统可以根据etcd采集对应的日志。
<!--more-->
## 选择beego作为web后台开发
web端采用beego框架进行开发，beego是一个采用mvc三层架构设计的web框架。这里阐述下web管理平台的架构和功能。
![1.png](1.png)
components里包含web平台用到的组件，包括beego日志插件，etcd插件，mysql插件。
conf里包含了web的配置信息，配置在app.conf这个文件中。
logs存储了web平台生成的日志
controllers为mvc架构中的c，也就是控制层，将models数据映射到view中。
models为mvc架构中的m，也就是数据层，负责从mysql中读取数据以及写入数据
views为mvc架构中的v，也就是视图层，这里为web端展示的前端界面。
routers保存了路由回应的回调函数，这样根据对应的路由调用不同的函数，从而调用不同的controller。
statics存储了css和js文件，这个主要是网页端用到的。
## 代码流程
main函数中初始化插件，并且启动beego
``` golang
func init_components() bool {
	err := components.InitLogger() //调用logger初始化
	if err != nil {
		logs.Warn("initDb failed, err :%v", err)
		return false
	}

	err = components.InitDb()
	if err != nil {
		logs.Warn("initDb failed, err:%v", err)
		return false
	}

	err = components.InitEtcd()
	if err != nil {
		logs.Warn("init etcd failed, err:%v", err)
		return false
	}

	return true
}

func main() {
	if init_components() == false {
		return
	}
	beego.Run()
}
```
插件初始化具体可以查看我的源码，之后在最下方我给出源码链接。
接下来我们看看routers中路由规则的注册
``` golang
func init() {
	beego.Router("/index", &AppController.AppController{}, "*:AppList")
	beego.Router("/app/list", &AppController.AppController{}, "*:AppList")
	beego.Router("/app/apply", &AppController.AppController{}, "*:AppApply")
	beego.Router("/app/create", &AppController.AppController{}, "*:AppCreate")
	beego.Router("/log/apply", &LogController.LogController{}, "*:LogApply")
	beego.Router("/log/list", &LogController.LogController{}, "*:LogList")
	beego.Router("/log/create", &LogController.LogController{}, "*:LogCreate")
}
```
/index为首页展示
/app/list为项目列表
/app/apply为创建项目
/app/create为创建项目后跳转的路由
/log/apply为创建日志
/log/list为为日志列表展示
/log/create为日志创建成功后跳转的路由
接下来通过一个路由说说逻辑流程。当用户在浏览器输入http://localhost:8080/index, 
web后端通过路由调用Controller中的AppList函数
``` golang
func (p *AppController) AppList() {

	logs.Debug("enter index controller")

	p.Layout = "layout/layout.html"
	appList, err := model.GetAllAppInfo()
	if err != nil {
		p.Data["Error"] = fmt.Sprintf("服务器繁忙")
		p.TplName = "app/error.html"

		logs.Warn("get app list failed, err:%v", err)
		return
	}

	logs.Debug("get app list succ, data:%v", appList)
	p.Data["applist"] = appList

	p.TplName = "app/index.html"
}
```
可以看到我们设置了布局文件和模板文件，并且调用models获取所有项目的信息，然后设置到data中，通过模板返回。
前端可以通过网页展示项目列表。GetAllAppInfo获取项目信息的实现放在model层。
``` golang
func GetAllAppInfo() (appList []AppInfo, err error) {
    err = Db.Select(&appList, "select app_id, app_name, app_type, 
    create_time, develop_path from tbl_app_info")
	if err != nil {
		logs.Warn("Get All App Info failed, err:%v", err)
		return
	}
	return
}
```
model通过查询数据库将项目信息返回。这些数据存储在mysql表中。这里其实是通过orm映射，
将数据库表的数据存储在我们定义的结构体
``` golang
type AppInfo struct {
	AppId       int    `db:"app_id"`
	AppName     string `db:"app_name"`
	AppType     string `db:"app_type"`
	CreateTime  string `db:"create_time"`
	DevelopPath string `db:"develop_path"`
	IP          []string
}
```
数据库表如下
![2.png](2.png)
## 测试web管理平台
我们点击项目申请,填写项目信息
![3.png](3.png)
提交后可以看到项目列表
![4.png](4.png)
同样我们点击日志申请，填写信息
![5.png](5.png)
查看日志信息
![6.png](6.png)
数据库表也存储成功了
![7.png](7.png)
## 源码下载
[https://github.com/secondtonone1/golang-/tree/master/logcatchsys](https://github.com/secondtonone1/golang-/tree/master/logcatchsys)
感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)
个人微信号1017234088, 添加请注明来源!