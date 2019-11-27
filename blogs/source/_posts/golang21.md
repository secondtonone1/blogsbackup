---
title: Go项目实战：打造高并发日志采集系统（二）
date: 2019-11-27 16:55:16
categories: [golang]
tags: [golang]
---
日志统计系统的整体思路就是监控各个文件夹下的日志，实时获取日志写入内容并写入kafka队列，写入kafka队列可以在高并发时排队，而且达到了逻辑解耦合的目的。然后从kafka队列中读出数据，根据实际需求显示网页上或者控制台等。
## 前情提要
上一节我们完成了如下目标
1 配置kafka，并启动消息队列。
2 编写代码向kafka录入消息，并且从kafka读取消息。
## 本节目标
1 写代码从kafka中读取消息，保证kafka消息读写功能无误。
2 借助tailf实现文件监控，并模拟测试事实写文件以及文件备份时功能无误。
3 本系列文章开发语言使用Go
## 从kafka中读取消息
<!--more-->
``` golang
func main(){
	fmt.Println("consumer begin...")
	config := sarama.NewConfig()
	config.Consumer.Return.Errors = true
	wg  :=sync.WaitGroup{}
	//创建消费者
	consumer, err := sarama.NewConsumer([]string{"localhost:9092"},config)
	if err != nil {
		fmt.Println("consumer create failed, error is ", err.Error())
		return
	}
	defer consumer.Close()
	
	//Partitions(topic):该方法返回了该topic的所有分区id
    partitionList, err := consumer.Partitions("test")
    if err != nil {
		fmt.Println("get consumer partitions failed")
		fmt.Println("error is ", err.Error())
		return
    }

	for partition := range partitionList {
		//ConsumePartition方法根据主题，
		//分区和给定的偏移量创建创建了相应的分区消费者
		//如果该分区消费者已经消费了该信息将会返回error
		//OffsetNewest消费最新数据
        pc, err := consumer.ConsumePartition("test", int32(partition), sarama.OffsetNewest)
        if err != nil {
            panic(err)
		}
		//异步关闭，保证数据落盘
        defer pc.AsyncClose()
        wg.Add(1)
        go func(sarama.PartitionConsumer) {
            defer wg.Done()
            //Messages()该方法返回一个消费消息类型的只读通道，由代理产生
            for msg := range pc.Messages() {
				fmt.Printf("%s---Partition:%d, Offset:%d, Key:%s, Value:%s\n", 
				msg.Topic,msg.Partition, msg.Offset, string(msg.Key), string(msg.Value))
            }
        }(pc)
    }
    wg.Wait()
    consumer.Close()
	
}
```
这样我们启动zookeeper和kafka后，分别运行前文实现的向kafka中写入数据的代码，以及现在的从kafka中消费的代码，看到如下效果
![1.jpg](1.jpg)
## 实现文件监控
实现文件监控，主要是在文件中有内容写入时，程序可以及时获取写入的内容，类似于Linux命令中的tailf -f 某个文件的功能。
golang 中提供了tail库，我们借助这个库完成指定文件的监控，我的文件组织如下
![4.jpg](4.jpg)
logdir文件夹下的log.txt记录的是不断增加的日志文件
tailf文件夹下logtailf.go实现log.txt监控功能。
writefile文件夹下writefile.go实现的是向log.txt文件写日志并备份的功能。
``` golang
func main() {
	logrelative := `../logdir/log.txt`
	_, filename, _, _ := runtime.Caller(0)
	fmt.Println(filename)
	datapath := path.Join(path.Dir(filename), logrelative)
	fmt.Println(datapath)
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
		msg, ok := <-tailFile.Lines
		if !ok {
			fmt.Printf("tail file close reopen, filename: %s\n", tailFile.Filename)
			time.Sleep(100 * time.Millisecond)
			continue
		}
		//fmt.Println("msg:", msg)
		//只打印text
		fmt.Println("msg:", msg.Text)
	}
}
```
为了测试监控的功能。我们实现向log.txt中每隔0.1s写入一行"Hello+时间戳"的日志。当写入20条内容后我们将log.txt备份重命名。
然后创建新的log.txt继续写入。
在writefile.go实现一个函数定时写入，并且备份功能
``` golang
func writeLog(datapath string) {
	filew, err := os.OpenFile(datapath, os.O_APPEND|os.O_CREATE|os.O_RDWR, 0644)
	if err != nil {
		fmt.Println("open file error ", err.Error())
		return
	}

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
```
然后我们实现main函数，调用三次writeLog，这样会产生三个备份文件
``` golang
func main() {
	logrelative := `../logdir/log.txt`
	_, filename, _, _ := runtime.Caller(0)
	fmt.Println(filename)
	datapath := path.Join(path.Dir(filename), logrelative)
	for i := 0; i < 3; i++ {
		writeLog(datapath)
	}
}
```
我们分别启动文件监控和文件写入程序，效果如下
![2.jpg](2.jpg)
可以看到,当log.txt有内容写入时，logtailf.go实现了动态监控，而且当文件备份时，logtailf.go提示了文件被重命名备份。
最终我们看到产生三个备份文件
![3.jpg](3.jpg)
## 总结
目前我们已经完成了kafka消息读写，文件监控，动态写入和备份等功能，接下来我们实现项目的配置化和统筹代码。
源码下载
[https://github.com/secondtonone1/golang-](https://github.com/secondtonone1/golang-)
感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)
