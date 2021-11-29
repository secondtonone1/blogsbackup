---
title: golang使用mongo
date: 2021-11-29 16:16:21
categories: [golang]
tags: [golang]
---
## 简介
mongodb是著名的非关系型数据库，常用来存储大量关联性不大的数据。golang操作mongo数据库可选的库很多，目前主流的使用为"go.mongodb.org/mongo-driver/mongo"，本文通过代码demo的方式介绍go如何操作mongo，实现增删改查，以及多条更新，分组查询，分页查询等复杂查询，代码demo选自个人博客系统的源码。源码地址[https://github.com/secondtonone1/bstgo-blog](https://github.com/secondtonone1/bstgo-blog)
## 初始化连接和断开连接
初始化连接,包含必要的mongo-driver库即可
``` golang
import(
    "go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
	"go.mongodb.org/mongo-driver/mongo/readpref"
)

var MongoClient *mongo.Client = nil
var MongoDb *mongo.Database = nil

func MongoInit() (e error) {
 ctx, _ := context.WithTimeout(context.Background(), 10*time.Second)
	// 连接uri
 uri := "mongodb://" + config.TotalCfgData.Mongo.User + ":" + config.TotalCfgData.Mongo.Passwd +
		"@" + config.TotalCfgData.Mongo.Host + "/?authSource=admin"
	log.Println("uri is ", uri)
	// 构建mongo连接可选属性配置
	opt := new(options.ClientOptions)
	// 设置最大连接的数量
	opt = opt.SetMaxPoolSize(uint64(config.TotalCfgData.Mongo.MaxPoolSize))
	// 设置连接超时时间 5000 毫秒
	du, _ := time.ParseDuration(config.TotalCfgData.Mongo.ConTimeOut)
	opt = opt.SetConnectTimeout(du)
	// 设置连接的空闲时间 毫秒
	mt, _ := time.ParseDuration(config.TotalCfgData.Mongo.MaxConIdle)
	opt = opt.SetMaxConnIdleTime(mt)
	// 开启驱动
	MongoClient, e = mongo.Connect(ctx, options.Client().ApplyURI(uri), opt)
	if e != nil {
		log.Println("err is ", e)
		return
	}
	// 注意，在这一步才开始正式连接mongo
	e = MongoClient.Ping(ctx, readpref.Primary())
	if e != nil {
		log.Println("err is ", e)
	}

	log.Println("mongo init success!!!")
	//连接数据库
	MongoDb = MongoClient.Database(config.TotalCfgData.Mongo.Database)
	return
}
```
当服务器关闭时要回收mongo的资源，这里简单关闭即可
``` golang
func MongoRelease() {
	MongoClient.Disconnect(context.TODO())
}
```
## 文章表结构
在go文件中定义ArticleInfo类
``` golang
//文章信息
type ArticleInfo struct {
	Id    string `bson:"id" json:"infoid"`
	Cat   string `bson: "cat" json: "cat"`
	Title string `bson: "title" json: "title"`

	Subcat   string `bson: "subcat" json: "subcat"`
	Subtitle string `bson: "subtitle" json: "subtitle"`
	ScanNum  int    `bson:"scannum" json:"scannum"`
	LoveNum  int    `bson:"lovenum json:"lovenum`
	CreateAt int64  `bson:"createdAt" json:"createdAt"`
	LastEdit int64  `bson:"lastedit" json:"lastedit"`
	Author   string `bson:"author" json:"author"`
	Index    int    `bson:"index" json:"index"`
}
```
字段的tag中一定要写bson标识，用来通知mongo以该bson指定的名字存储该字段,json可以不写，我这里写json是为了数据传输
接下来看一下mongo数据库表中的文章信息结构表
``` json
{
    "id": "20RgXj2UbtwIYbhQIScpEyoeera",
    "cat": "Go",
    "title": "Linux环境搭建和编码",
    "subcat": "安装和使用",
    "subtitle": "Linux环境搭建",
    "scannum": 1246,
    "lovenum json:": 594,
    "createdAt": {
        "$numberLong": "1636012855"
    },
    "lastedit": 1636594645,
    "author": "恋恋风辰",
    "index": 1,
    "lovenum": 2
}
```
数据库表中的数据字段和go程序定义的结构体bson命名的字段是相符合。
## 查找一条数据
我们根据文章id获取文章信息，代码如下
``` golang
//通过文章获取文章概要信息
func GetArticleInfo(id string) (*model.ArticleInfo, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	filter := bson.M{"id": id}
	//log.Println("filter is ", filter)
	info := &model.ArticleInfo{}
	err := MongoDb.Collection("articles").FindOne(ctx, filter).Decode(info)
	if err != nil {
		log.Println("get article failed, error is ", err)
		return nil, err
	}

	return info, nil
}
```
## 查找多条数据
查询未分组的文章列表，这里用到了Find函数，返回的是一个cursor，通过不断的cursor.Next获取每条记录。而且Find设置了查询选项，用了or或查询，这个或查询的条件就是cat为default或者subcat为default，同时对查询结果设置了排序
``` golang
//获取未分类的文章
func GetDefaultArts() ([]*model.ArticleInfo, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	catfilter := bson.M{}
	catfilter["cat"] = "default"

	subfilter := bson.M{}
	subfilter["subcat"] = "default"

	filterarry := []bson.M{catfilter, subfilter}
	//或查询
	orfilter := bson.M{}
	orfilter["$or"] = filterarry

	//log.Println("filter is ", filter)
	sort := bson.D{{"lastedit", -1}}
	findOptions := options.Find().SetSort(sort)
	articles := []*model.ArticleInfo{}
	cursor, err := MongoDb.Collection("articles").Find(ctx, orfilter, findOptions)
	if err != nil {
		return articles, err
	}

	defer cursor.Close(context.TODO())

	for cursor.Next(context.TODO()) {
		article := &model.ArticleInfo{}
		if err := cursor.Decode(article); err != nil {
			continue
		}

		articles = append(articles, article)
	}

	return articles, nil
}
```
## 插入一条数据
这里直接调用insertone传入我们定义的结构体，mongo-driver会自动根据bson命名写入mongo
``` golang
func SaveArtInfo(article *model.ArticleInfo) error {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	_, err := MongoDb.Collection("articles").InsertOne(ctx, article)
	return err
}
```
## 更新一条数据
根据文章id更新浏览量
``` golang
//更新文章浏览量
func AddArticleScan(id string) error {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	filter := bson.M{"id": id}
	value := bson.M{"$inc": bson.M{"scannum": 1}}
	_, err := MongoDb.Collection("articles").UpdateOne(ctx, filter, value)
	if err != nil {
		return err
	}

	return nil
}
```
更新时设置filter为更新的条件，value为更新的字段值，这里实现的是浏览量自增运算，如果要实现覆盖式更新也很简单
``` golang
//更新文章
func UpdateArticle(req *model.UpdateArticleReq) error {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	filter := bson.M{"id": req.Id}
	value := bson.M{}
	value["title"] = req.Title
	value["subtitle"] = req.SubTitle
	value["cat"] = req.Cat
	value["subcat"] = req.SubCat
	value["lastedit"] = req.LastEdit
	value["author"] = req.Author
	//value["content"] = req.Content

	upvalue := bson.M{"$set": value}
	_, err := MongoDb.Collection("articles").UpdateOne(ctx, filter, upvalue)
	if err != nil {
		return err
	}
    return nil
}
更新指定id的文章，更新字段值包括title，subtitle，cat，subcat等，此时用$set选项可以实现覆盖式更新
```
## 更新多条
批量更新满足条件的多条记录，比如更新一个序列的文章列表，将其分类和子分类都更新为指定字段
``` golang
//批量更新默认文章的分类
func UpDefaultArtsCtg(cat string, subcat string, arts []string) error {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	//根据不同的条件，更新相同的值
	filter := bson.M{"id": bson.M{"$in": arts}}
	//将该子分类的文章的子分类设置为新的分类
	artupval := bson.M{"subcat": subcat, "cat": cat}
	artupcmd := bson.M{"$set": artupval}
	_, err := MongoDb.Collection("articles").UpdateMany(ctx, filter, artupcmd)
	if err != nil {
		return err
	}
	return nil
}
```
上述代码将arts数组中的文章列表，统一更新了分类为cat，子分类为subcat
如果需要将多条记录，更新成多个不同的值怎么处理呢？这里要用到bulkwrite
``` golang
//批量更新文章列表序列
func UpdateArticleSort(sortArt *model.ArticleSortReq) error {
	if len(sortArt.ArticleList) == 0 {
		log.Println("sort article list is empty")
		return nil
	}
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	models := []mongo.WriteModel{}
	for _, article := range sortArt.ArticleList {
		filter := bson.M{"id": article.Id, "title": article.Title}
		updatecmd := bson.D{{"$set", bson.D{{"index", article.Index}}}}
		model := mongo.NewUpdateOneModel().SetFilter(filter).SetUpdate(updatecmd).SetUpsert(false)
		models = append(models, model)
	}

	//log.Println("models are ", models)
	opts := options.BulkWrite().SetOrdered(false)
	_, err := MongoDb.Collection("articles").BulkWrite(ctx, models, opts)
	return err
}
```
上述代码将文章列表的多个文章的index更新为不同的值，index代表文章的排序索引，将不同文章的index值更新为不同的index，达到的效果就是id为1的文章index索引更新为index1，id为2的文章index更新为index2
## 分页查询
可以对查询选项设置排序，并且设置每次获取多少条记录从而达到分页查询的效果。比如将skipTmp设置为5,10,15，将limitTmp设置为5就是每页获取五条记录，将sort设置为按照lastedit排序，就达到了根据最后编辑日期排序，并分页查询，获取每页五条数据的功能。
``` golang
//获取文章列表
func GetArticlesByPage(page int) ([]*model.ArticleInfo, error) {
	articles := []*model.ArticleInfo{}
	if page < 1 {
		return articles, nil
	}
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	sort := bson.D{{"lastedit", -1}}
	findOptions := options.Find().SetSort(sort)

	//从第1页获取，每次获取5条
	skipTmp := int64((page - 1) * 5)
	limitTmp := int64(5)
	findOptions.Skip = &skipTmp
	findOptions.Limit = &limitTmp

	filter := bson.D{}
	cursor, err := MongoDb.Collection("articles").Find(ctx, filter, findOptions)

	if err != nil {
		return articles, err
	}

	defer cursor.Close(context.TODO())

	for cursor.Next(context.TODO()) {
		article := &model.ArticleInfo{}
		if err := cursor.Decode(article); err != nil {
			continue
		}

		articles = append(articles, article)
	}

	return articles, nil
}
```
## 获取文档总记录数量
有时我们需要获取一个文档的所有记录数，比如分页查询后也要返回总的页数，这其实就是需要查询出总的条数计算返回总页数即可。
``` golang
//获取文章总数
func ArticleTotalCount() (int, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	filter := bson.D{}
	count, err := MongoDb.Collection("articles").CountDocuments(ctx, filter)

	if err != nil {
		return 0, err
	}

	return int(count), nil
}
```
CountDocuments返回的是文档总的记录条数
## 模糊查询
我们可以根据年月日以及分类查询，返回文章列表，当然还可以通过输入keywords关键字进行模糊查询。这里做一个较为复杂的查询，查询条件为某年某月某日创建的文章，分类为cat，并根据keywords模糊查询，如果文章内容或标题中有符合keywords的，返回该文章列表。
``` golang
//搜索文章
func SearchArticle(condition *model.SearchArticleReq) ([]*model.ArticleInfo, int, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	filter := bson.M{}
	if condition.Year != "" && condition.Year != "不限" {
		var stamp int64 = 0
		valStr := condition.Year
		if condition.Month != "" && condition.Month != "不限" {
			tempStr := "2006-01月"
			valStr = valStr + "-" + condition.Month
			localTime, err := time.ParseInLocation(tempStr, valStr, time.Local)
			if err != nil {
				log.Println("time parse failed, err is ", err)
			} else {
				stamp = localTime.Unix()
				log.Println("query time stamp is ", stamp)
			}
		} else {
			localTime, err := time.ParseInLocation("2006", valStr, time.Local)
			if err != nil {
				log.Println("time parse failed, err is ", err)
			} else {
				stamp = localTime.Unix()
				log.Println("query time stamp is ", stamp)
			}
		}

		filter["createdAt"] = bson.M{"$gte": stamp}
	}

	if condition.Cat != "" && condition.Cat != "不限" {
		filter["cat"] = condition.Cat
	}

	if condition.Keywords != "" && condition.Keywords != "不限" {

		filter["$or"] = []bson.M{
			bson.M{
				"content": bson.M{"$regex": condition.Keywords, "$options": "$i"},
			},

			bson.M{
				"title": bson.M{"$regex": condition.Keywords, "$options": "$i"},
			},
		}
	}

	//log.Println("filter is ", filter)
	sort := bson.D{{"lastedit", -1}}
	findOptions := options.Find().SetSort(sort)

	//从第1页获取，每次获取5条
	skipTmp := int64((condition.Page - 1) * 5)
	limitTmp := int64(5)
	findOptions.Skip = &skipTmp
	findOptions.Limit = &limitTmp

	articles := []*model.ArticleInfo{}
	cursor, err := MongoDb.Collection("articles").Find(ctx, filter, findOptions)
	if err != nil {
		return articles, 0, err
	}

	defer cursor.Close(context.TODO())

	for cursor.Next(context.TODO()) {
		article := &model.ArticleInfo{}
		if err := cursor.Decode(article); err != nil {
			continue
		}

		articles = append(articles, article)
	}

	count, err := MongoDb.Collection("articles").CountDocuments(ctx, filter)

	if err != nil {
		return articles, 0, err
	}

	return articles, int(count), nil
}
```
## 分组查询
我们常遇到这种情况，查询每个班级成绩最好的学生，或者查询每个类别销量最好的产品品牌等，这里我也用到了分组查询，根据不同分类返回每个分类下最大index值，这样做主要是统计每个分类下文章最大索引。
``` golang
//获取子分类下最大index
func GetSubCatMaxIndex(subcat string) (int, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	pipeline := bson.A{
		bson.M{
			"$match": bson.M{"subcat": subcat},
		},

		bson.M{
			"$group": bson.M{
				"_id":      bson.M{"subcat_": "$subcat"},
				"maxIndex": bson.M{"$max": "$index"}},
		},
	}
	cursor, err := MongoDb.Collection("articles").Aggregate(ctx, pipeline)
	if err != nil {
		log.Println("aggrete failed, error is ", err)
		return 0, err
	}

	maxIndex := 0
	for cursor.Next(context.Background()) {
		doc := cursor.Current
		maxindex_, err := doc.LookupErr("maxIndex")
		if err != nil {
			log.Println("LookupErr failed, error is ", err)
			return maxIndex, err
		}
		log.Println("maxindex_ is ", maxindex_)
		maxIndex = int(maxindex_.Int32())
		log.Println("maxindex is ", maxIndex)
	}
	log.Println("get max index is ", maxIndex)
	return maxIndex, nil
}
```
通过match匹配subcat，然后根据匹配结果进行分组，分组的区分的条件为subcat，分组条件的字段要用_id(只能用这个名字)表示，然后用maxIndex(可以自己命名)表示获取分组的最大索引。
当然我们还可以做一些分组运算的其他操作
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
先用birthday做查询，匹配三月份出生的人，然后根据月份进行分组，totalCount用来计算分组下人的总数，nameG表示最小名字等。
## 文档内查询
有时候mongo的文档中的记录形式为一条记录，该记录有多个字段，某个字段为一个数组列表，查询记录中数组列表某个值满足条件，返回该记录。我有这样一个文档，表示文章的目录菜单
``` json
{ 
   "_id":{"$oid":"61138b43094825c520604e56"},
  "catmenus":[
                { "catid":"1wZexxzy9FsQOtdfKSro5EM6Zzv",
                  "name":"C++",
                  "subcatmenus": [
                         {"subcatid":"1wZf3EsGe5UdNxXTcH71iEfxgpb","name":"变量"},   
                         {"subcatid":"1wZfFxSHDw5gdOF0g39lb72n8r9","name":"aa"},
                         {"subcatid":"1wZldzHUziSZyTXvfhO9FNfRWnD","name":""}]
                }, 
                { "catid":"1wZezd7c961MNGZ0s0U8aUef4hq",
                   "name":"Go",
                  "subcatmenus":[
                        {"subcatid":"1wZfHzBeJmDOtNt3XdMysAQ1bhh","name":"aaa"}
                      ]
                }
             ]
 }
```
当想查询文档内catid为1wZezd7c961MNGZ0s0U8aUef4hq的片段，并且更新其subcatmenus字段为新的数组
db.menu.find({"catmenus.catid":"1wZezd7c961MNGZ0s0U8aUef4hq"})是可以查询到该记录的，
但是这种查询只限于单个条件，
如果有多个条件如下db.menu.find({"catmenus.catid":"1wZezd7c961MNGZ0s0U8aUef4hq","catmenus.name":"Go"})
如果有多条记录分别满足条件，查询的就有可能是多条，而不是交集，mongo返回满足以上条件任意一条即可。
为了要实现交集选择器，则需要用elemMatch
db.menu.find({"catmenus":  {"$elemMatch":{"catid":"1wZezd7c961MNGZ0s0U8aUef4hq", "name":"Go"}}})
转化为go代码实现查询
``` golang
func UpdateSortMenu(submenu *model.SortMenuReq) error {

    ctx, _ := context.WithTimeout(context.Background(), 5*time.Second)
    //指定连接集合
    col := MongoDb.Collection("menu")
    //设定更新filter
    //filter := bson.D{{"catmenus.catid", submenu.ParentId}}
    filter := bson.D{{"catmenus",
        bson.D{{"$elemMatch",
            bson.D{{"catid",
                submenu.ParentId}},
        }},
    }}
    update := bson.D{{"$set",
        bson.D{
            {"catmenus.$.subcatmenus", submenu.Menu},
        }}}

    _, err := col.UpdateOne(ctx, filter, update)
    return err
}
```
目前收录了几种常见的go操作mongo的方法，都是基于mongo原生支持的操作实现的。