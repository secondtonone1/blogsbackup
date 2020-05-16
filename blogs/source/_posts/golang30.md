---
title: 谈谈高并发秒拍系统架构设计
date: 2020-05-15 12:36:07
categories: [golang]
tags: [golang]
---
## 缓冲系统结构
今天谈谈电商秒杀抢购或者高并发集中访问情况下，如何设计稳定高效的缓冲系统。常用的做法是采取逻辑分离，将秒杀功能分化为不同的逻辑进行设计，降低耦合度同时增加缓冲队列降低访问压力。
<!--more-->
可以将秒杀抢购功能分为接入层和逻辑层，接入层主要负责基本的判断如token检测，用户检测，请求是否合法等，逻辑层则做主要的逻辑处理和判断。如下图
![1.png](1.png)
1 秒杀接入层主协程启动后，启动多个接入层的读协程和写协程，当有请求到来时接入层主协程判断是否合理，将合理的请求写入chan缓冲队列。
2 然后多个接入层的读协程从chan中读取待处理的消息，每个协程操作redis，将待处理的请求判断无误后写入redis待处理请求队列中。
3 逻辑层主协程启动多个逻辑层读协程和写协程，逻辑层读协程从redis的待处理请求队列中读取待处理请求，进行并发处理，然后写入逻辑层chan队列中
4 逻辑层写协程从逻辑层chan取出处理结果写入redis的处理结果队列
5 接入层的读协程并发从redis处理结果队列中取出处理结果，将处理结果写入主协程，从而完成整个消息流程。
消息处理通过接入层和逻辑层分离解耦，压力降低，同时每层带有多个读写协程和自己的chan缓冲队列，实现了异步处理。redis的加入也让高并发处理更稳定和安全。

## 代码实现
接入层主协程逻辑判断和消息写入chan中，同时监听读协程返回的处理结果
``` golang
func SecKill(req *config.SecRequest) (data map[string]interface{}, err error) {

	data = make(map[string]interface{}, components.INIT_INFO_SIZE)
    //。。。。省略判断逻辑
	msgtoredis := &MsgReqToRedis{
		ProductId: req.ProductId,
		UserId:    req.UserId,
	}
	writetimer := time.NewTimer(time.Second * 5)
	defer writetimer.Stop()
	select {
	case <-writetimer.C:
		logs.Debug("msg write to redis chan timeout, maybe chan has beeen closed")
		data["status"] = config.MSG_CHAN_CLOSED
		data["message"] = "msg chan to redis closed"
		return
	case MsgRdMgr.MsgChanToRedis <- msgtoredis:
		logs.Debug("msg chan to redis success")
	}

	//设置定时器，超时检测，防止请求阻塞
	ticker := time.NewTicker(time.Duration(10) * time.Second)
	defer func() {
		ticker.Stop()
	}()
	select {
	case msgrsp, ok := <-MsgRdMgr.MsgChanFromRedis:
		if !ok {
			logs.Debug("msg rsp from redis chan closed ")
			data["status"] = config.MSG_CHAN_CLOSED
			data["message"] = "msg chan from redis closed"
			return
		}

		if msgrsp.Status != config.STATUS_SEC_SUCCESS {
			logs.Debug(msgrsp.Message)
			data["status"] = msgrsp.Status
			data["message"] = msgrsp.Message
			return
		}
		datamap := make(map[string]interface{}, components.INIT_INFO_SIZE)
		datamap["productid"] = msgrsp.ProductId
		datamap["userid"] = msgrsp.UserId
		datamap["token"] = msgrsp.Token
		data["data"] = datamap
		data["status"] = config.STATUS_SEC_SUCCESS
		data["message"] = "seckill success"

		//更新product 信息
		components.SKConfData.SecInfoRWLock.Lock()
		defer components.SKConfData.SecInfoRWLock.Unlock()
		components.SKConfData.SecInfoData[msgrsp.ProductId].Left = msgrsp.Left
		return
	case <-ticker.C:
		data["status"] = config.STATUS_REQ_TIMEOUT
		data["message"] = "seckill timeout"
		return
	case <-MsgRdMgr.FromRedisGrClose:
		data["status"] = config.FROM_REDIS_GR_CLOSED
		data["message"] = "from redis chan group closed"
		return
	case <-MsgRdMgr.ToRedisGrClose:
		data["status"] = config.TO_REDIS_GR_CLOSED
		data["message"] = "to redis chan group closed"
		return
	}

}
```

接入层读协程从redis处理结果队列中读取结果
``` golang
func ReadFromRedis(wg *sync.WaitGroup) {
	conn := components.MsgReqPool.Get()
	defer func() {
		wg.Done()
		conn.Close()
	}()
	for {
		reply, err := conn.Do("blpop", "msgfromredis", 0)
		if err != nil {
			logs.Debug("pop from msgfromredis failed ...%s", err.Error())
			continue
		}

		if reply == nil {
			logs.Debug("msg read from redis ,data is nil")
			continue
		}
		kvarray, err := redis.Strings(reply, err)
		if err != nil {
			logs.Debug("msgfromredis string convert failed, %v", err.Error())
			continue
		}
		logs.Debug("read from redis msgfromredis , ip is %v", kvarray)
		msgfromrd := new(MsgRspFromRedis)
		err = json.Unmarshal([]byte(kvarray[1]), msgfromrd)
		if err != nil {
			logs.Warn("json unmarshal failed , err is : %v", err.Error())
			continue
		}

		select {
		case MsgRdMgr.MsgChanFromRedis <- msgfromrd:
			logs.Debug("read from redis success, put data intto read redis chan")
			continue
		}
	}
}
```
接入层写协程向redis待处理请求队列中写入请求
``` golang
//proxy向redis中写
func WriteToRedis(wg *sync.WaitGroup) {
	defer func() {
		wg.Done()
	}()
	for {
		select {
		case msgtoredis, ok := <-MsgRdMgr.MsgChanToRedis:
			if !ok {
				logs.Debug("msg chan to redis closed")
				return
			}
			jsmal, err := json.Marshal(msgtoredis)
			if err != nil {
				logs.Debug("json marshal failed")
				continue
			}
			conn := components.MsgReqPool.Get()
			defer conn.Close()
			_, err = conn.Do("rpush", "msgtoredis", string(jsmal))

			if err != nil {
				logs.Debug("rpush to msgtoredis failed  ...%s", err.Error())
				continue
			}

		}
	}
}
```
同样的道理，逻辑层也实现了读写协程，和上面类似不做赘述
我们可以看看效果，分别启动接入层和逻辑层进程，然后再浏览器启动输入请求http://localhost:9091/seckill?product_id=12&user_id=14
可以看到两个进程打印的日志信息
![2.png](2.png)
![3.png](3.png)
浏览器返回处理结果
![4.png](4.png)
具体源码地址
[https://github.com/secondtonone1/golang-/tree/master/seckill](https://github.com/secondtonone1/golang-/tree/master/seckill)
感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)
