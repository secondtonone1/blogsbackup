---
title: Go项目实战：打造高并发日志采集系统（四)
date: 2019-12-03 14:01:29
categories: [golang]
tags: [golang]
---
## 前情回顾
前文我们完成了如下目标
1 项目架构整体编写
2 使框架支持热更新
## 本节目标
在前文的框架基础上，我们
1 将之前实现的日志监控功能整合到框架中。
2 一个日志对应一个监控协程，当配置热更新后根据新配置动态关闭和启动协程。
3 编写测试代码，模拟向文件中不断写入日志，并备份日志，观察监控功能是否健壮。
<!--more-->
## 增加协程监控日志文件
我们将之前实现的日志监控功能整合到现有框架，文件结构如下
![5.png](5.png)
logdir为存储日志的文件夹，模拟不同系统记录的日志。实际生产中不同系统会自己记录日志并保存在指定文件夹中，logdir模拟的就是这些指定文件。
logtailf实现日志文件的监控功能。
writefile模拟不同系统向指定文件夹写入日志，用来测试日志写入和备份时，我们的采集系统是否健壮。
我们先看下logtailf.go
``` golang
func WatchLogFile(datapath string, ctx context.Context) {
	fmt.Println("begin goroutine watch log file ", datapath)
	tailFile, err := tail.TailFile(datapath, tail.Config{
		//文件被移除或被打包，需要重新打开
		ReOpen: true,
		//实时跟踪
		Follow: true,
		//如果程序出现异常，保存上次读取的位置，避免重新读取
		Location: &tail.SeekInfo{Offset: 0, Whence: 2},
		//支持文件不存在
		MustExist: false,
		Poll:      true,
	})

	if err != nil {
		fmt.Println("tail file err:", err)
		return
	}

	for true {
		select {
		case msg, ok := <-tailFile.Lines:
			if !ok {
				fmt.Printf("tail file close reopen, filename: %s\n", tailFile.Filename)
				time.Sleep(100 * time.Millisecond)
				continue
			}
			//fmt.Println("msg:", msg)
			//只打印text
			fmt.Println("msg:", msg.Text)
		case <-ctx.Done():
			fmt.Println("receive main gouroutine exit msg")
			fmt.Println("watch log file ", datapath, " goroutine exited")
			return
		}

	}
}
```
只有一个函数WatchLogFile用来监控指定路径的日志文件，两个参数分别为日志路径和上下文开关，开关用来关闭这个协程，整体功能和前几篇讲述的一样，不做赘述。
接下来我们在main.go中填加协程启动逻辑监控日志文件
``` golang
func ConstructMgr(configPaths interface{}) {
	configDatas := configPaths.(map[string]interface{})
	for conkey, confval := range configDatas {
		configData := new(logconfig.ConfigData)
		configData.ConfigKey = conkey
		configData.ConfigValue = confval.(string)
		ctx, cancel := context.WithCancel(context.Background())
		configData.ConfigCancel = cancel
		configMgr[conkey] = configData
		go logtailf.WatchLogFile(configData.ConfigValue,
			ctx)
	}
}
```
ConstructMgr新增了协程启动逻辑，并且将协程的ctx保存在map中，这样主协程可以根据热更新启动和关闭这个协程。然后我完善了main函数中的析构函数
``` golang
defer func() {
		mainOnce.Do(func() {
			if err := recover(); err != nil {
				fmt.Println("main goroutine panic ", err) // 这里的err其实就是panic传入的内容
			}
			cancel()
			for _, oldval := range configMgr {
				oldval.ConfigCancel()
			}
			configMgr = nil
		})
	}()
```
析构函数里遍历map，关闭所有监控协程，并且将map设置为nil回收资源。在main函数中添加热更新控制协程开关。
``` golang
	for oldkey, oldval := range configMgr {
		_, ok := pathDataNew[oldkey]
		if ok {
				continue
			}
		oldval.ConfigCancel()
		delete(configMgr, oldkey)
	}

	for conkey, conval := range pathDataNew {
		oldval, ok := configMgr[conkey]
		if !ok {
			configData := new(logconfig.ConfigData)
			configData.ConfigKey = conkey
			configData.ConfigValue = conval.(string)
			ctx, cancel := context.WithCancel(context.Background())
			configData.ConfigCancel = cancel
			configMgr[conkey] = configData
			fmt.Println(conval.(string))
			go logtailf.WatchLogFile(configData.ConfigValue,
						ctx)
					continue
		}

		if oldval.ConfigValue != conval.(string) {
			oldval.ConfigValue = conval.(string)
			oldval.ConfigCancel()
			ctx, cancel := context.WithCancel(context.Background())
			oldval.ConfigCancel = cancel
			go logtailf.WatchLogFile(conval.(string),
						ctx)
			continue
		}
	}
```
对比新旧配置，如果路径取消，则停止监控其的协程，如果路径新增或修改，则启动新的协程。
这样我们启动程序看到如下效果
![1.png](1.png)
## 编写测试文件，模拟日志写入和备份
我们在writefile.go中实现日志写入和备份，这部分内容前几篇有讲解，我只是将原来实现的功能搬过来
``` golang
func writeLog(datapath string, wg *sync.WaitGroup) {
	filew, err := os.OpenFile(datapath, os.O_APPEND|os.O_CREATE|os.O_RDWR, 0644)
	if err != nil {
		fmt.Println("open file error ", err.Error())
		return
	}
	defer func() {
		wg.Done()
	}()
	w := bufio.NewWriter(filew)
	for i := 0; i < 20; i++ {
		timeStr := time.Now().Format("2006-01-02 15:04:05")
		fmt.Fprintln(w, "Hello current time is "+timeStr)
		time.Sleep(time.Millisecond * 100)
		w.Flush()
	}
	logBak := time.Now().Format("20060102150405") + ".txt"
	logBak = path.Join(path.Dir(datapath), logBak)
	filew.Close()
	err = os.Rename(datapath, logBak)
	if err != nil {
		fmt.Println("Rename error ", err.Error())
		return
	}
}

func main() {
	v := viper.New()
	configPaths, confres := logconfig.ReadConfig(v)
	if !confres {
		fmt.Println("config read failed")
		return
	}
	wg := &sync.WaitGroup{}

	for _, confval := range configPaths.(map[string]interface{}) {
		wg.Add(1)
		go writeLog(confval.(string), wg)
	}
	wg.Wait()
}
```
根据配置文件启动多个协程，向日志文件中不断写入日志，写入20条后备份日志，我们启动日志收集程序和这个测试程序，可以看到日志不断被写入，日志收集程序不断打印日志新增的内容
![2.png](2.png)
日志备份为不同的文件
![3.png](3.png)
可以看到日志收集系统在日志不断写入时，可以健壮运行
## 热更新控制监控协程打开和关闭
现在config.yaml文件中路径记录如下
``` yaml
configpath: 
  logdir1: "D:/golangwork/src/golang-/logcatchsys/logdir1/log.txt"
  logdir2: "D:/golangwork/src/golang-/logcatchsys/logdir2/log.txt"
  logdir3: "D:/golangwork/src/golang-/logcatchsys/logdir3/log.txt"
```
我们启动日志收集程序，模拟热更新配置，将logdir3这条数据修改,配置变为如下
``` yaml
configpath: 
  logdir1: "D:/golangwork/src/golang-/logcatchsys/logdir1/log.txt"
  logdir2: "D:/golangwork/src/golang-/logcatchsys/logdir2/log.txt"
  logdir4: "D:/golangwork/src/golang-/logcatchsys/logdir4/log.txt"
```
我们可以看到程序作出检测，并关闭了曾经监控logdir3的协程，并启动了新的协程监控logdir4，而其他的日志监控协程不受影响，正常运行。
![4.png](4.png)

源码下载地址
[https://github.com/secondtonone1/golang-/tree/master/logcatchsys](https://github.com/secondtonone1/golang-/tree/master/logcatchsys)
我的公众号
![wxgzh.jpg](wxgzh.jpg)
