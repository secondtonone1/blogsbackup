---
title: golang操作mongo
date: 2020-08-20 08:58:56
categories: [golang]
tags: [golang]
---
本文采用mongo-driver/mongo驱动操作数据库
## 设计mongo插件结构
将代码分为如下结构
model : odm模型，主要是映射为数据库存储的表结构
constants : 存储一些常量
config : mongo的配置信息，比如空闲时长，连接数，超时时间等
mongodb : 实现了mongo的连接和关闭等功能。
<!--more-->
目录结构如下
![1.png](1.png)

## mongo的连接和断开
在mongodb.go中实现了连接和断开操作
初始化
``` golang
var (
	DB *Database
)

type Database struct {
	Mongo *mongo.Client
}

//初始化
func Init() {
	DB = &Database{
		Mongo: SetConnect(),
	}
}
```
连接数据库
``` golang
func SetConnect() *mongo.Client {

	var retryWrites bool = false

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	clientOptions := options.Client().SetHosts(config.GetConf().MongoConf.Hosts).
		SetMaxPoolSize(config.GetConf().MongoConf.MaxPoolSize).
		SetHeartbeatInterval(constants.HEART_BEAT_INTERVAL).
		SetConnectTimeout(constants.CONNECT_TIMEOUT).
		SetMaxConnIdleTime(constants.MAX_CONNIDLE_TIME).
		SetRetryWrites(retryWrites)

	//设置用户名和密码
	username := config.GetConf().MongoConf.Username
	password := config.GetConf().MongoConf.Password

	if len(username) > 0 && len(password) > 0 {
		clientOptions.SetAuth(options.Credential{Username: username, Password: password})
	}

	client, err := mongo.Connect(ctx, clientOptions)
	if err != nil {
		fmt.Println(err)
	}

	// Check the connection
	err = client.Ping(context.TODO(), nil)

	if err != nil {
		fmt.Println(err)
	}

	fmt.Println("Connected to MongoDB!")

	return client
}
```
关闭数据库
``` golang
func Close() {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	DB.Mongo.Disconnect(ctx)
}
```
## 在model中定义模型
定义的模型加上bson标签，这样可以映射到数据库里了
``` golang
type UserData struct {
	Id         string `bson:"_id,omitempty" json:"id"`
	Name       string `bson:"name" json:"name"`
	Number     int    `bson:"number" json:"number"`
	Age        int    `bson:"age" json:"age"`
	BirthMonth int    `bson:"birthMonth" json:"birthMonth"`
}
```
## 在main函数中实现一些demo
在main函数中实现一些demo，包括单条插入，多条插入，单条更新，多条更新，分组查询，分页查询
1 单条插入
``` golang
//单条插入
func insertOne() {
	client := mongodb.DB.Mongo
	// 获取数据库和集合
	collection := client.Database(constants.DB_DATABASES).Collection(constants.DB_COLLECTION)
	userdata := model.UserData{}
	userdata.Age = 13
	userdata.BirthMonth = 11
	userdata.Number = 3
	userdata.Name = "zack"
	ctx, cancel := context.WithTimeout(context.Background(), constants.QUERY_TIME_OUT)
	defer cancel()
	// 插入一条数据
	insertOneResult, err := collection.InsertOne(ctx, &userdata)
	if err != nil {
		fmt.Println("insert one error is ", err)
		return
	}
	log.Println("collection.InsertOne: ", insertOneResult.InsertedID)
	//将objectid转换为string
	docId := insertOneResult.InsertedID.(primitive.ObjectID)
	recordId := docId.Hex()
	fmt.Println("insert one ID str is :", recordId)
}
```
2 多条插入
``` golang
//多条插入
func insertMany() {
	ctx, cancel := context.WithTimeout(context.Background(), constants.QUERY_TIME_OUT)
	defer cancel()
	client := mongodb.DB.Mongo
	// 获取数据库和集合
	collection := client.Database(constants.DB_DATABASES).Collection(constants.DB_COLLECTION)
	userdata1 := model.UserData{}
	userdata1.Age = 20
	userdata1.BirthMonth = 11
	userdata1.Number = 4
	userdata1.Name = "Lilei"

	userdata2 := model.UserData{}
	userdata2.Age = 20
	userdata2.BirthMonth = 12
	userdata2.Number = 5
	userdata2.Name = "HanMeiMei"

	var list []interface{}
	list = append(list, &userdata1)
	list = append(list, &userdata2)
	result, err := collection.InsertMany(ctx, list)
	if err != nil {
		fmt.Println("insert many error is ", err)
	}
	fmt.Println("insert many success, res is ", result.InsertedIDs)
}
```
3 查找单条
``` golang
//查找单个
func findOne() {

	client := mongodb.DB.Mongo
	// 获取数据库和集合
	collection := client.Database(constants.DB_DATABASES).Collection(constants.DB_COLLECTION)
	filter := bson.M{"name": "HanMeiMei"}

	ctx, cancel := context.WithTimeout(context.Background(), constants.QUERY_TIME_OUT)
	defer cancel()
	singleResult := collection.FindOne(ctx, filter)
	if singleResult == nil || singleResult.Err() != nil {
		fmt.Println("find one error is ", singleResult.Err().Error())
		return
	}

	userData := &model.UserData{}
	err := singleResult.Decode(userData)
	if err != nil {
		fmt.Println("find one failed error is ", err)
		return
	}

	fmt.Println("find one success, res is ", userData)
}
```
4 查找多条
``` golang
//查询多个结果集，用cursor
func findMany() {
	client := mongodb.DB.Mongo
	collection := client.Database(constants.DB_DATABASES).Collection(constants.DB_COLLECTION)

	ctx, cancel := context.WithTimeout(context.Background(), constants.QUERY_TIME_OUT)
	defer cancel()
	filter := bson.M{"birthMonth": bson.M{"$lte": 12}}

	cursor, err := collection.Find(ctx, filter)
	if err != nil {
		fmt.Println("find res failed , error is ", err)
		return
	}
	defer cursor.Close(context.Background())

	result := make(map[string]*model.UserData)
	for cursor.Next(context.Background()) {
		ud := &model.UserData{}
		err := cursor.Decode(ud)
		if err != nil {
			fmt.Println("decode error is ", err)
			continue
		}
		result[ud.Name] = ud
	}

	fmt.Println("success is ", result)
	return
}
```
5 更新一条
``` golang
//更新
func updateOne() {
	client := mongodb.DB.Mongo
	collection, _ := client.Database(constants.DB_DATABASES).Collection(constants.DB_COLLECTION).Clone()
	ctx, cancel := context.WithTimeout(context.Background(), constants.QUERY_TIME_OUT)
	defer cancel()
	/*
		oid, err := primitive.ObjectIDFromHex(obj.RecordId)
		if err != nil {
			logging.Logger.Info("convert string from object failed")
			return err
		}
		filter := bson.M{"_id": oid}
	*/
	filter := bson.M{"name": "zack"}
	value := bson.M{"$set": bson.M{
		"number": 1024}}

	_, err := collection.UpdateOne(ctx, filter, value)
	if err != nil {
		fmt.Println("update user data failed, err is ", err)
		return
	}
	fmt.Println("update success !")
	return
}
```
6 更新多条
``` golang
//更新多条记录
func updateMany() {
	client := mongodb.DB.Mongo
	collection := client.Database(constants.DB_DATABASES).Collection(constants.DB_COLLECTION)

	ctx, cancel := context.WithTimeout(context.Background(), constants.QUERY_TIME_OUT)
	defer cancel()
	var names = []string{"zack", "HanMeiMei"}
	filter := bson.M{"name": bson.M{"$in": names}}

	value := bson.M{"$set": bson.M{"birthMonth": 3}}
	result, err := collection.UpdateMany(ctx, filter, value)
	if err != nil {
		fmt.Println("update many failed error is ", err)
		return
	}
	fmt.Println("update many success !, result is ", result)
	return
}
```
7 分组查询
``` golang
//分组查询
func findGroup() {
	client := mongodb.DB.Mongo
	collection := client.Database(constants.DB_DATABASES).Collection(constants.DB_COLLECTION)

	ctx, cancel := context.WithTimeout(context.Background(), constants.QUERY_TIME_OUT)
	defer cancel()
	//复杂查询，先匹配后分组
	pipeline := bson.A{
		bson.M{
			"$match": bson.M{"birthMonth": 3},
		},
		bson.M{"$group": bson.M{
			"_id":        bson.M{"birthMonthUid": "$birthMonth"},
			"totalCount": bson.M{"$sum": 1},
			"nameG":      bson.M{"$min": "$name"},
			"ageG":       bson.M{"$min": "$age"},
		},
		},

		//bson.M{"$sort": bson.M{"time": 1}},
	}
	fmt.Println("pipeline is ", pipeline)

	cursor, err := collection.Aggregate(ctx, pipeline)
	fmt.Println("findGroup cursor is ", cursor)
	if err != nil {
		fmt.Printf("dao.findGroup collection.Aggregate() error=[%s]\n", err)
		return
	}

	for cursor.Next(context.Background()) {
		doc := cursor.Current

		totalCount, err_2 := doc.LookupErr("totalCount")
		if err_2 != nil {
			fmt.Printf("dao.findGroup totalCount err_2=[%s]\n", err_2)
			return
		}

		nameData, err_4 := doc.LookupErr("nameG")
		if err_4 != nil {
			fmt.Printf("dao.findGroup insertDateG err_4=[%s]\n", err_4)
			return
		}

		ageData, err_5 := doc.LookupErr("ageG")
		if err_5 != nil {
			fmt.Printf("dao.findGroup ageG err_5=[%s]\n", err_5)
			continue
		}
		fmt.Println("totalCount is ", totalCount)
		fmt.Println("nameData is ", nameData)
		fmt.Println("ageData is ", ageData)
	}
}
```
8 分页查询
``` golang
//分页查询
func limitPage() {
	client := mongodb.DB.Mongo
	collection := client.Database(constants.DB_DATABASES).Collection(constants.DB_COLLECTION)

	ctx, cancel := context.WithTimeout(context.Background(), constants.QUERY_TIME_OUT)
	defer cancel()

	filter := bson.M{"age": bson.M{"$gte": 0}}

	SORT := bson.D{{"number", -1}}
	findOptions := options.Find().SetSort(SORT)
	//从第1页获取，每次获取10条
	skipTmp := int64((1 - 1) * 10)
	limitTmp := int64(10)
	findOptions.Skip = &skipTmp
	findOptions.Limit = &limitTmp
	cursor, err := collection.Find(ctx, filter, findOptions)
	defer cursor.Close(context.Background())
	if err != nil {
		fmt.Println("limit page error is ", err)
		return
	}

	for cursor.Next(context.Background()) {
		ud := &model.UserData{}
		err := cursor.Decode(ud)
		if err != nil {
			fmt.Println("user data decode error is ", err)
			continue
		}

		fmt.Println("user data is ", ud)

	}
}
```
## 源码地址
可以下载源码，二次封装放到项目中直接使用
[https://github.com/secondtonone1/golang-/tree/master/gomongo](https://github.com/secondtonone1/golang-/tree/master/gomongo)
## 感谢关注公众号
![wxgzh.jpg](wxgzh.jpg)
