---
title: eos节点启动源码分析
date: 2019-01-22 14:02:07
categories: 技术开发
tags: [区块链eos]
---
在eos源码目录中programs/nodeos/main.cpp文件里，为节点启动的主函数
main函数内部做了两件事
1 初始化 application
``` cpp
if(!app().initialize<chain_plugin, http_plugin, net_plugin, producer_plugin>(argc, argv))
return INITIALIZE_FAIL;
```
2  application启动和插件启动
``` cpp
app().startup();
app().exec();
```
<!--more-->
## application类
先说application的基本实现和常用接口，application 定义了注册插件的函数，获取插件，查找插件等功能，根据模板动态绑定的
![1.png](1.png)
同样定义了一些私有函数和变量
![2.png](2.png)
application 构造函数为私有的，方便实现单例模式，plugins为存储各类注册插件的map，key为插件名，value为独占的基类指针，不可copy。
io_serv为boost库提供的io_service的shared_ptr，网络通信和事件注册派发会用到。initialized_plugins为已经初始化的插件vector，
running_plugins为已经runing的插件vector。
erased_method_ptr和erased_channel_ptr分别在method.hpp和channel.hpp中实现了定义，以后用到再分析。
application类protected部分包含三个函数
![3.jpg](3.jpg)
initialize_impl 根据配置初始化插件。
plugin_initialized 将插件放入初始化vector
plugin_started 将插件放入runing vector
下面是application实现的单例模式，定义全局函数app(),内部调用application类静态函数instance()
![4.png](4.png)
application的start函数内部调用了各个插件的startup，出现异常shutdown
![5.png](5.png)
exec()内部注册了SIGINT,SIGTERM,SIGPIPE信号，当进程收到这几个信号会导致ioservice->stop()，否则io_ser->run()一直监听等待就绪事件
![6.png](6.png)
## plugin类
abstract_plugin为所有插件继承的纯虚类，不同插件会实现各自特有的功能
![7.png](7.png)
plugin.hpp中定义了
![8.png](8.png)
BOOST_PP_SEQ_FOR_EACH为boost定义的宏，按照参数PLUGINS依次展开，将lambda函数l和每个PLUGIN传入
APPBASE_PLUGIN_REQUIRES_VISIT，比如net_api_plugin展开
![9.png](9.png)
展开后
``` cpp
void plugin_requires( Lambda&& l ) { 
l(appbase::app().register_plugin<net_plugin>())
l(appbase::app().register_plugin<http_plugin>()) 
}
```
就是采用注册的lambda表达式依次调用，并且将各个插件注册到application中。
plugin 继承了abstract_plugin类
![10.png](10.png)
Impl是模板类型，不同的插件会传入不同的模板类型，initialize，startup，shutdown为虚函数，重写了基类abstract_plugin的功能。内部通过
static_cast<Impl*>(this)转化为对应的不同模板类型的plugin，进而实现特定功能的绑定，初始化，启动，停止等。举例：
![11.png](11.png)
net_plugin 继承了plugin，并且模板类型为net_plugin，这样当plugin<net_plugin>调用initialize、startup、shutdown、函数，内部展开如下：
``` cpp
static_cast<net_plugin*>(this)->plugin_requires([&](auto& plug){ plug.initialize(options); });
static_cast<net_plugin*>(this)->plugin_initialize(options);
static_cast<net_plugin*>(this)->plugin_startup();
static_cast<net_plugin*>(this)->plugin_shutdown();
```
APPBASE_PLUGIN_REQUIRES((chain_plugin))展开
``` cpp
void plugin_requires( Lambda&& l ) { 
l(appbase::app().register_plugin<chain_plugin>()
}
```
所以static_cast<net_plugin*>(this)->plugin_requires([&](auto& plug){ plug.initialize(options); });
实际继续展开内部调用appbase::app().register_plugin<chain_plugin>().initialize(options);完成网络部分的初始化。
startup，shutdown也是一样的道理。



