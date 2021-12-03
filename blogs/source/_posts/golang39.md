---
title: 博客系统后台开发(三)完善项目并容器化
date: 2021-12-03 14:09:20
categories: [golang]
tags: [golang]
---
## 本节目标
上一节完成了模板渲染，业余时间我增加了几个页面，大家可以根据分支去查看每天做的工作，这一节增加配置文件的读取，完成redis缓存的添加，一些信息优先访问redis缓存，另外增加日志库打印日志，最后完成项目的容器化
<!--more-->
## redis缓存
之前的文章讲述过redis的增删改查，这里也和之前redis操作类似，增加文章的查询效率。
初始化redis连接池
``` golang
func InitRedis() {
	rediscli = redis.NewClient(&redis.Options{
		Addr:         config.TotalCfgData.Redis.Host,
		Password:     config.TotalCfgData.Redis.Passwd,
		DB:           config.TotalCfgData.Redis.DB,
		PoolSize:     config.TotalCfgData.Redis.PoolSize,
		MinIdleConns: config.TotalCfgData.Redis.IdleCons,
	})

	_, err := rediscli.Ping().Result()
	if err != nil {
		log.Println("ping failed, error is ", err)
		return
	}

	clearch = make(chan struct{}, 1000)
	exitch = make(chan struct{})
	log.Println("redis init success!!!")
}
```
比如我们将session保存在redis中
``` golang
func AddAdminSession(sessionId string, sessionData string) error {
	_, err := rediscli.HSet(ADMIN_SESSION, sessionId, sessionData).Result()
	rediscli.Expire(ADMIN_SESSION, time.Hour*24*30)
	return err
}
```
依此类推，增加了很多redis读写模块，不一一赘述了。
## 配置文件读取
服务器用到的配置文件我写在config/config.toml中
``` golang
[mongo]
    host = "81.68.86.123:27017"
    user = "admin"
    passwd = "12345678"
    maxpoolsize = 10
    contimeout  = "5000"
    maxconidle = "5000"
    database = "blog"

[cookie]
    host = "81.68.86.123"
    #本地启动请设置host为本地
    #host = "localhost"
    alive = 86400

[location]
    timezone = "Asia/Shanghai"

[redis]
    host = "81.68.86.123:6379"
    idlecons = 16
    poolsize = 1024
    idletimeout = 300
    passwd = "123456"
    db = 0
```
然后实现了config.go用来读取配置文件
``` golang
package config

import (
	"log"

	"flag"

	"github.com/BurntSushi/toml"
)

type MongoCfg struct {
	User        string `toml: "user"`
	Passwd      string `toml: "passwd"`
	Host        string `toml: "host"`
	MaxPoolSize int16  `toml: "maxpoolsize"`
	MaxConIdle  string `toml:"maxconidle"`
	ConTimeOut  string `toml: "contimeout"`
	Database    string `toml: "database"`
}

type CookieCfg struct {
	Host  string `toml: "host"`
	Alive int    `toml: "alive"`
}

type RedisCfg struct {
	Host        string `toml: "host"`
	PoolSize    int    `toml: "poolsize"`
	IdleCons    int    `toml: "idlecons"`
	IdleTimeout int    `toml: "idletimeout"`
	Passwd      string `toml: "passwd"`
	DB          int    `toml: "db"`
}

type TotalCfg struct {
	Mongo     MongoCfg  `toml: "mongo"`
	Cookie    CookieCfg `toml: "cookie"`
	Location_ Location  `toml:"location"`
	Redis     RedisCfg  `toml:"redis"`
}

type Location struct {
	TimeZone string `toml:"timezone"`
}

var TotalCfgData TotalCfg

func init() {
	cfgpath := flag.String("config", "./config/config.toml", "-config ./config/config.toml")
	flag.Parse()
	if _, err := toml.DecodeFile(*cfgpath, &TotalCfgData); err != nil {
		log.Println("decode file failed , error is ", err)
		panic("decode file failed")
	}
}
```
上述代码根据toml标签读取配置文件将对应字段写入结构体对象中，就完成了读取。
## 增加日志库
对于日志库的选择，我选择了uber提供的zap库，同时配合lumberjack完成日志切割
``` golang
package logger

import (
	"github.com/natefinch/lumberjack"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

var Sugar *zap.SugaredLogger = nil

func getLogWriter() zapcore.WriteSyncer {

	lumberJackLogger := &lumberjack.Logger{
		Filename:   "./log/blog.log",
		MaxSize:    10,
		MaxBackups: 5,
		MaxAge:     30,
		Compress:   false,
	}

	return zapcore.AddSync(lumberJackLogger)
}

func init() {
	// 编码器配置
	config := zap.NewProductionEncoderConfig()
	// 指定时间编码器
	config.EncodeTime = zapcore.ISO8601TimeEncoder
	// 日志级别用大写
	config.EncodeLevel = zapcore.CapitalLevelEncoder
	// 编码器
	encoder := zapcore.NewConsoleEncoder(config)
	writeSyncer := getLogWriter()
	// 创建Logger
	core := zapcore.NewCore(encoder, writeSyncer, zapcore.DebugLevel)
	logger := zap.New(core, zap.AddCaller())
	Sugar = logger.Sugar()
	// 打印日志
	Sugar.Info("logger init success")
}
```
设置日志文件最大10M，最多备份五个文件，最长时间为30天，同时可以支持打印行号等功能。
## 容器化
先实现dockerfile，然后生成镜像
``` golang
FROM golang:1.16
# 为我们的镜像设置必要的环境变量
ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
	GOPROXY="https://goproxy.cn,direct"

ENV TZ=Asia/Shanghai

# 创建代码目录
WORKDIR  /src
# copy代码放入代码目录
COPY . .
# 将代码编译为二进制可执行文件
RUN go build -o main .
# 创建运行运行环境
WORKDIR /bin
#将二进制文件从 /src 移动到/bin
RUN cp /src/main .
# 将项目所需配置和static资源copy至该目录
RUN cp -r /src/config .
RUN cp -r /src/public .
RUN cp -r /src/views .

# 暴露端口对外服务
EXPOSE 8080

# 启动容器运行命令
CMD ["/bin/main"]
```
设置ENV准备了golang的运行环境，时区设置为上海，工作目录为/src，然后将源码由宿主机拷贝进入镜像编译并运行
在项目的根目录，执行如下命令生成镜像
``` cmd
docker build -t blog .
```
启动容器即可
``` cmd
docker run --name blogds -p 8088:8088 -v /data/blog/log:/bin/log  --restart=always  -d blog
```
## 总结
到目前为止，我们完成了博客系统后台的开发，仅用三篇文章无法全部说明其中细节，只是列举了后台系统经历的几个阶段，具体代码可以去github看看，谢谢大家赏星。
源码地址：
https://github.com/secondtonone1/bstgo-blog

