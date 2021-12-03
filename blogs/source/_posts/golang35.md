---
title: golang使用redis
date: 2021-11-30 10:00:58
categories: [golang]
tags: [golang]
---
## 简介
redis是高并发场景下常用的缓存型数据库，支持持久化，常用来缓存访问频率较高的数据，本文通过几个例子列举go如何使用redis，go 操作redis的库很多，我们选择常用的go-redis来操作就可以了，当然官方也提供了redis操作的库。这几个例子出自个人博客系统的源码。源码地址[https://github.com/secondtonone1/bstgo-blog](https://github.com/secondtonone1/bstgo-blog)
<!--more-->
## 连接redis
我们创建一个redis连接池，设置最大空闲连接数，池内总共连接数，以及redis连接密码，地址，选择的分区等
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
	log.Println("redis init success!!!")
}
```
## 根据key设置value
redis是基于key-value存储的，这个例子我们为STRING_ToTAL_VISIT_NUM_KEY设置值为num，过期时间为0表示永不过期
``` golang
func SetVisitNum(num int64) (string, error) {
	val, err := rediscli.Set(STRING_ToTAL_VISIT_NUM_KEY, num, 0).Result()
	if err != nil {
		return "", err
	}

	return val, nil
}
```
## 根据key获取值
这个例子获取key为STRING_ToTAL_VISIT_NUM_KEY的value值
``` golang
func GetVisitNum() (string, error) {
	val, err := rediscli.Get(STRING_ToTAL_VISIT_NUM_KEY).Result()

	if err == redis.Nil {
		return "key not exists in redis", err
	}

	if err != nil {
		return "", err
	}

	return val, nil
}
```
## 根据key删除值
``` golang
rediscli.Del(STRING_ToTAL_VISIT_NUM_KEY).Result()
```
## HGet获取hash表中指定字段
HGet操作可以获取hash表中指定字段的值，下个例子通过获取HOME_SESSION的key找到hash表，进而找到sessioId字段的值
``` golang
func GetHomeSession(sessionId string) (string, error) {
	sessionData, err := rediscli.HGet(HOME_SESSION, sessionId).Result()
	if err != nil {
		return "", err
	}

	return sessionData, err
}
```
## HSet添加值
HSet设置hash表中指定字段的值，下个例子设置了Key为HOME_SESSION的hash表，将sessionId字段的值设置为sessionData
``` golang
func AddHomeSession(sessionId string, sessionData string) error {
	_, err := rediscli.HSet(HOME_SESSION, sessionId, sessionData).Result()

	rediscli.Expire(HOME_SESSION, time.Hour*24)
	return err
}
```
设置值的同时设置了key为HOME_SESSION的hash表中所有值的过期时间为24小时，如果要设置过期时间，每次设置值后都要重设过期时间，如不设置就会变成永久存在内存中
## HDel删除值
HDel删除hash表中指定的字段，如下例子删除了key为HOME_SESSION的hash表中字段为sessionId的值
``` golang
func DelHomeSession(sessionId string) error {
	_, err := rediscli.HDel(HOME_SESSION, sessionId).Result()
	return err
}    
```
## 增加操作
可以通过Incr命令为某个key的value自增加一，如下这个例子为key为STRING_ToTAL_VISIT_NUM_KEY的value值加一
``` golang
func AddVisitNum() (int64, error) {
	visit, err := rediscli.Incr(STRING_ToTAL_VISIT_NUM_KEY).Result()
	if err != nil {
		return 0, err
	}

	return visit, nil
}
```
类似的操作还有Decr，DecrBy等
## 获取hash表中所有key和value
HGetAll可以获取指定hash表中所有的key和value，如下例子获取了key为HSET_LV1_MENU_KEY的hash表中所有的key，value
``` golang
func GetLv1Menus() ([]*model.CatMenu, error) {
	menulist := []*model.CatMenu{}
	menus, err := rediscli.HGetAll(HSET_LV1_MENU_KEY).Result()
	if err != nil {
		return menulist, err
	}

	for _, val := range menus {
		menu := &model.CatMenu{}
		err := json.Unmarshal([]byte(val), menu)
		if err != nil {
			log.Println("json unmarshal failed, err is ", err)
			continue
		}
		menulist = append(menulist, menu)
	}
	return menulist, nil
}
```
redis中key为HSET_LV1_MENU_KEY的hash表存储的是所有一级目录的key和value，通过HGetAll获取后进行遍历和反序列化获取具体的目录结构。
以上就是redis常用的操作，通过go操作redis达到缓存高并发访问数据的目的。
