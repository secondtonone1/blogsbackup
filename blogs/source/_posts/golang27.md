---
title: Go项目实战：打造高并发日志采集系统（八）
date: 2020-01-15 11:09:49
categories: [golang]
tags: [golang]
---
## 前情回顾
前文我们完成了日志采集系统基本功能，包括日志监控，日志采集，配置热更新，协程动态启动和关闭，同时扩充支持了etcd管理文件路径。
## 本节目标
本节新增日志查询和检索功能。基本思路是将日志信息从kafka中读取，然后放到elasticsearch中，elasticsearch是一个分布式多用户能力
的全文搜索引擎，我们可以通过它提供的web接口访问和查询指定数据。另外，为了更方便的检索和查询，可以利用kibana配合elastic可视化
查询。Kibana 是为 Elasticsearch设计的开源分析和可视化平台。
<!--more-->
## 源码实现
将日志从kafka中读取并解析写入elastic这部分功能，我们将其提炼到另外一个进程中，单独启动监控并处理kafka数据。
``` golang
package main
import (
	"fmt"
	kafconsumer "golang-/logcatchsys/kafconsumer"
	"golang-/logcatchsys/logconfig"
)

func main() {
	v := logconfig.InitVipper()
	if v == nil {
		fmt.Println("vipper init failed!")
		return
	}

	kafconsumer.GetMsgFromKafka()
}
```
主函数调用了我封装的kafconsumer包的读取消息函数GetMsgFromKafka。
``` golang
func GetMsgFromKafka() {
	fmt.Println("kafka consumer begin ...")
	config := sarama.NewConfig()
	config.Consumer.Return.Errors = true
	var kafkaddr = "localhost:9092"
	kafkaconf, _ := logconfig.ReadConfig(logconfig.InitVipper(), "kafkaconfig.kafkaaddr")
	if kafkaconf != nil {
		kafkaddr = kafkaconf.(string)
	}
	//创建消费者
	consumer, err := sarama.NewConsumer([]string{kafkaddr}, config)
	if err != nil {
		fmt.Println("consumer create failed, error is ", err.Error())
		return
	}
	defer func(consumer sarama.Consumer) {
		if err := recover(); err != nil {
			fmt.Println("consumer panic error ", err)
		}
		consumer.Close()
		topicSet = nil
		//回收所有协程
		for _, val := range topicMap {
			for _, valt := range val {
				valt.Cancel()
			}
		}

		topicMap = nil
	}(consumer)
	topicSetTmp := ConstructTopicSet()
	if topicSetTmp == nil {
		fmt.Println("construct topic set error ")
		return
	}
	topicSet = topicSetTmp
	ConsumeTopic(consumer)
}
```
GetMsgFromKafka中创建了kafka消费者，然后根据配置调用ConstructTopicSet构造topic集合，topicSet集合其实是一个map，
保证了集合中的topic不重复。然后调用ConsumeTopic函数根据topic从kafka取出数据。
``` golang
func ConstructTopicSet() map[string]bool {
	topicSetTmp := make(map[string]bool)
	configtopics, _ := logconfig.ReadConfig(logconfig.InitVipper(), "collectlogs")
	if configtopics == nil {
		goto CONFTOPIC
	}
	for _, configtopic := range configtopics.([]interface{}) {
		confmap := configtopic.(map[interface{}]interface{})
		for key, val := range confmap {
			if key.(string) == "logtopic" {
				topicSetTmp[val.(string)] = true
			}
		}
	}
CONFTOPIC:
	return topicSetTmp
}
```
ConstructTopicSet读取配置中的topic列表，然后将这些topic放到map中返回。
``` golang
func ConsumeTopic(consumer sarama.Consumer) {

	for key, _ := range topicSet {
		partitionList, err := consumer.Partitions(key)
		if err != nil {
			fmt.Println("get consumer partitions failed")
			fmt.Println("error is ", err.Error())
			continue
		}

		for partition := range partitionList {
			pc, err := consumer.ConsumePartition(key, int32(partition), sarama.OffsetNewest)
			if err != nil {
				fmt.Println("consume partition error is ", err.Error())
				continue
			}
			defer pc.AsyncClose()

			topicData := new(TopicData)
			topicData.Ctx, topicData.Cancel = context.WithCancel(context.Background())
			topicData.KafConsumer = pc
			topicData.TPartition = new(TopicPart)
			topicData.TPartition.Partition = int32(partition)
			topicData.TPartition.Topic = key
			_, okm := topicMap[key]
			if !okm {
				topicMap[key] = make(map[int32]*TopicData)
			}
			topicMap[key][int32(partition)] = topicData
			go ReadFromEtcd(topicData)

		}
	}
	for {
		select {
		case topicpart := <-topicChan:
			fmt.Printf("receive goroutine exited, topic is %s, partition is %d\n",
				topicpart.Topic, topicpart.Partition)
			//重启消费者读取数据的协程
			val, ok := topicMap[topicpart.Topic]
			if !ok {
				continue
			}
			tp, ok := val[topicpart.Partition]
			if !ok {
				continue
			}
			tp.Ctx, tp.Cancel = context.WithCancel(context.Background())
			go ReadFromEtcd(tp)
		}

	}
}
```
ConsumeTopic实际是将topic集合中的topic遍历放到map中，然后启动协程调用ReadFromEtcd函数读取消息。
``` golang
func ReadFromEtcd(topicData *TopicData) {

	fmt.Printf("kafka consumer begin to read message, topic is %s, part is %d\n", topicData.TPartition.Topic,
		topicData.TPartition.Partition)

	logger := log.New(os.Stdout, "LOGCAT", log.LstdFlags|log.Lshortfile)
	elastiaddr, _ := logconfig.ReadConfig(logconfig.InitVipper(), "elasticconfig.elasticaddr")
	if elastiaddr == nil {
		elastiaddr = "localhost:9200"
	}

	esClient, err := elastic.NewClient(elastic.SetURL("http://"+elastiaddr.(string)),
		elastic.SetErrorLog(logger))
	if err != nil {
		// Handle error
		logger.Println("create elestic client error ", err.Error())
		return
	}

	info, code, err := esClient.Ping("http://" + elastiaddr.(string)).Do(context.Background())
	if err != nil {
		logger.Println("elestic search ping error, ", err.Error())
		esClient.Stop()
		esClient = nil
		return
	}
	fmt.Printf("Elasticsearch returned with code %d and version %s\n", code, info.Version.Number)

	esversion, err := esClient.ElasticsearchVersion("http://" + elastiaddr.(string))
	if err != nil {
		fmt.Println("elestic search version get failed, ", err.Error())
		esClient.Stop()
		esClient = nil
		return
	}
	fmt.Printf("Elasticsearch version %s\n", esversion)

	defer func(esClient *elastic.Client) {
		if err := recover(); err != nil {
			fmt.Printf("consumer message panic %s, topic is %s, part is %d\n", err,
				topicData.TPartition.Topic, topicData.TPartition.Partition)
			topicChan <- topicData.TPartition
		}

	}(esClient)

	var typestr = "catlog"
	typeconf, _ := logconfig.ReadConfig(logconfig.InitVipper(), "elasticconfig.typestr")
	if typeconf != nil {
		typestr = typeconf.(string)
	}

	for {
		select {
		case msg, ok := <-topicData.KafConsumer.Messages():
			if !ok {
				fmt.Println("etcd message chan closed ")
				return
			}
			fmt.Printf("%s---Partition:%d, Offset:%d, Key:%s, Value:%s\n",
				msg.Topic, msg.Partition, msg.Offset, string(msg.Key), string(msg.Value))
			idstr := strconv.FormatInt(int64(msg.Partition), 10) + strconv.FormatInt(msg.Offset, 10)
			logdata := &LogData{Topic: msg.Topic, Log: string(msg.Value), Id: idstr}
			createIndex, err := esClient.Index().Index(msg.Topic).Type(typestr).Id(idstr).BodyJson(logdata).Do(context.Background())

			if err != nil {
				logger.Println("create index failed, ", err.Error())
				continue
			}
			fmt.Println("create index success, ", createIndex)

		case <-topicData.Ctx.Done():
			fmt.Println("receive exited from parent goroutine !")
			return
		}
	}
}
```
ReadFromEtcd函数将kafka中读取的数据写入elastic中，同时如果协程崩溃向父协程发送通知，重启该协程。
## 效果展示
我们启动之前的日志监控程序，然后启动现在设计的信息处理程序。
可以看到日志不断被写入时，监控程序将日志的变化信息写入kafka。
同时，信息处理程序不断的从kafka中读取数据写入elastic。
![1.png](1.png)
我们通过kibana查询数据
![2.png](2.png)
## 源码下载
[https://github.com/secondtonone1/golang-/tree/master/logcatchsys](https://github.com/secondtonone1/golang-/tree/master/logcatchsys)
感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)