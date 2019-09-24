---
title: Go(05)map介绍
date: 2019-06-11 17:12:28
categories: [golang]
tags: [golang]
---
## 基本用法 
map同样也是引用类型，map在使用前需要通过make进行初始化，否则会报panic错误。
### map 初始化和插入
``` golang
type PersonInfo struct {
		ID      string
		Name    string
		Address string
	}

	var personDB map[string]PersonInfo
	personDB = make(map[string]PersonInfo)
	personDB["12345"] = PersonInfo{"12345", "Tom", "Room 203"}
	personDB["1"] = PersonInfo{"1", "Jack", "Room 102"}
```
可以看到map使用前用make先构造初始化，之后进行了插入，如果key存在，则修改value
### map 查找
``` golang
//从这个map查找键为"1234"
	person, ok := personDB["1234"]
	if ok {
		fmt.Println("Found person", person.Name, "with ID 1234")
	} else {
		fmt.Println("Did not find person with ID 1234")
	}

```
查找指定key，返回值为value和bool类型结果，所以先判断bool类型值是否为true
<!--more-->
### map 进阶
map可以直接显示初始化不需要make构造。
``` golang
var data map[string]int = map[string]int{"bob": 18, "luce": 28}
```
map是引用类型，函数通过修改形参，达到修改外部实参的功能
``` golang
func modify(data map[string]int, key string, value int) {
	v, res := data[key]
	//不存在,则res是false
	if !res {
		fmt.Println("key not find")
		return
	}
	fmt.Println("key is ", key, "value is ", v)
	data[key] = value
}

func main() {
	var data map[string]int = map[string]int{"bob": 18, "luce": 28}
	modify(data, "lilei", 28)
	modify(data, "luce", 22)
    fmt.Println(data)
}
```
map 大小可以通过len函数获得，如果不采用显示初始化方式，只声明map，在使用前一定要make初始化
map遍历采用range方式，且map是无序的，切记。
``` golang
//map大小
	fmt.Println(len(data))
	//map 使用前一定要初始化，可以显示初始化，也可以用make
	var data2 map[string]int = make(map[string]int, 3)
	fmt.Println(data2)
	//当key不存在时，则会插入
	data2["sven"] = 19
	fmt.Println(data2)
	//当key存在时，则修改
	data2["sven"] = 299
	fmt.Println(data2)
	data2["Arean"] = 33
	data2["bob"] = 178
	//map是无序的,遍历输出
	for key, value := range data2 {
		fmt.Println("key: ", key, "value: ", value)
	}
```
上面的代码遍历map，打印结果为 
``` cmd
key:  bob value:  178
key:  sven value:  299
key:  Arean value:  33
```
可以实现一个函数，将map中的key存到slice中，然后排序，之后根据排好顺序的slice遍历
得到的就是排序后的结果
``` golang
func sortprintmap(data map[string]int) {
	slice := make([]string, 0)
	for k, _ := range data {
		slice = append(slice, k)
	}
	sort.Strings(slice)
	for _, s := range slice {
		d, e := data[s]
		if !e {
			continue
		}
		fmt.Println("key is ", s, "value is ", d)

	}
}
```
在main函数调用sortprintmap(data2)，结果如下
``` cmd
key is  Arean value is  33
key is  bob value is  178
key is  sven value is  299
```
### 二维map
二维map操作和之前类似，只是声明时value还是一个map
``` golang
//二维map
	var usrdata map[string]map[string]int
```
二维map同样遵循使用前先make初始化原则，并且在二层map要使用前仍然需要make
``` golang
//使用前需要初始化
	usrdata = make(map[string]map[string]int)
	usrdata["sven"] = make(map[string]int)
	usrdata["sven"]["age"] = 21
	usrdata["sven"]["id"] = 1024

	usrdata["susan"] = make(map[string]int)
	usrdata["susan"]["age"] = 19
	usrdata["susan"]["id"] = 1000
```
二维map遍历
``` golang
//二维map 遍历
	for k, v := range usrdata {
		for k2, v2 := range v {
			fmt.Println(k, " ", k2, " ", v2)
		}
	}
```
### slice 中存储map
``` golang
//slice of map
	slicem := make([]map[string]int, 5)
	for i := 0; i < len(slicem); i++ {
		slicem[i] = make(map[string]int)
	}

	fmt.Println(slicem)
```
本着golang所有引用类型,如chan，map，slice，interface，使用前都需要make初始化。
上面代码先初始化slicem，然后再遍历slice，为每个元素初始化map类型

### map 反转
``` golang
//map 反转
	rvmap := make(map[int]string)
	for k, v := range data2 {
		rvmap[v] = k
	}
	fmt.Println(rvmap)
	fmt.Println(data2)
```
其实就是构造一个和原map 的key value相反的map，然后为该map初始化并且插入元素。

上述所有源码下载地址
[源码下载地址](https://github.com/secondtonone1/golang-)
谢谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)


