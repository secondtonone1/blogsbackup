---
title: Go进阶篇(01) interface应用和复习
date: 2019-10-11 15:10:37
categories: [golang]
tags: [golang]
---
## interface 意义？
golang 为什么要创造interface这种机制呢？我个人认为最主要的就是做约束，定义一种规范，大家可以按照同一种规范实现各自的功能，从而实现多态。
同时当interface做函数形参，可以很好地限制传入参数，并且根据不同的实参调用达到多态的效果。多态的意思就是多种多样的功能，比如我们定义了一
个接口
``` golang
type IOInter interface{
    write()int
    read()int
}
```
定义了一个IOInter的接口，只要别人实现了write和read方法，都可以转化为这个接口。至于具体怎么读，读什么，网络IO还是文件IO取决于具体的实现，
这就形成了多样化的功能，从而实现多态。同时IOInter做函数的形参，
``` golang
func WriteFunc(io IOInter){
    io.Write()
}
```
<!--more-->
还达到了安全限制的功能。比如没有实现读写功能的类实例无法传给WriteFunc,WriteFunc内部调用io的Write函数会根据实参具体的实现完成特定的读写。
我们通过一个小例子实战下接口做形参的意义。golang中sort包提供了几个排序api，我们先实现一个int类型slice排序功能。
``` golang
    arrayint := []int{6, 1, 0, 5, 2, 7}
	sort.Ints(arrayint)
	fmt.Println(arrayint)
```
sort.Ints可以完成对整形slice的排序，同样sort.Strings可以完成对string类型slice的排序
``` golang
    arraystring := []string{"hello", "world", "Alis", "and", "Bob"}
	sort.Strings(arraystring)
	fmt.Println(arraystring)
```
如果有这样一个需求，游戏中有很多个英雄，每个英雄有四个属性，攻击，防御，名字，出生时间，对这些英雄排序，从小到大，优先按照攻击排序，其次攻击相等的按
照防御排序，如果防御相等的按照出生时间排序。这是比较规则，我们根据sort包的Sort函数可以实现这个功能。
首先我们先定义英雄的结构
``` golang
type Hero struct {
	Name    string
	Attack  int
	Defence int
	GenTime int64
}
```
其次我们再定义一个英雄列表
``` golang
type HeroList []*Hero
```
接下来，我们看看golang的sort包实现的Sort源码
``` golang
func Sort(data Interface) {
	n := data.Len()
	quickSort(data, 0, n, maxDepth(n))
}
```
可以看出，Sort函数有一个Interface类型的形参，我们继续查看Interface的类型
``` golang
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int
	// Less reports whether the element with
	// index i should sort before the element with index j.
	Less(i, j int) bool
	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```
可以看出Interface是一个接口，内部声明了三个方法Len,Less,Swap.
我们需要给自己的英雄列表实现这三个方法，就可以调用Sort排序了。先实现Len方法。
``` golang
func (hl HeroList) Len() int {
	return len(hl)
}
```
Len功能很简单，就是实现了列表大小的获取。接下来实现比较函数Less,Less是按照我们之前说的英雄比较规则实现
``` golang
func (hl HeroList) Less(i, j int) bool {
	if i < 0 || j < 0 {
		return true
	}

	lenth := len(hl)
	if i >= lenth || j >= lenth {
		return true
	}

	if hl[i].Attack != hl[j].Attack {
		return hl[i].Attack < hl[j].Attack
	}

	if hl[i].Defence != hl[j].Defence {
		return hl[i].Defence < hl[j].Defence
	}

	return hl[i].GenTime < hl[j].GenTime
}
```
优先判断攻击是否相等，不相等就按照攻击力大小排序，小于返回true，从而实现从小到大排序
其次，攻击相等按照防御排序，以此类推。
接下来实现交换函数，交换英雄列表中的两个英雄
``` golang
func (hl HeroList) Swap(i, j int) {
	if i < 0 || j < 0 {
		return
	}

	lenth := len(hl)
	if i >= lenth || j >= lenth {
		return
	}

	hl[i], hl[j] = hl[j], hl[i]

}
```
这样，我们的英雄列表功能和结构都设计好了，写个main函数测试下
``` golang
//自定义类型排序用sort.Sort
	var herolists HeroList
	for i := 0; i < 10; i++ {
		generate := time.Now().Unix()
		name := fmt.Sprintf("Hero%d", generate)
		hero := Hero{
			Name:    name,
			Attack:  rand.Intn(100),
			Defence: rand.Intn(200),
			GenTime: generate,
		}
		herolists = append(herolists, &hero)
		time.Sleep(time.Duration(1) * time.Second)
	}

	sort.Sort(herolists)
	for _, value := range herolists {
		fmt.Print(value.Name, " ", value.Attack, " ", value.Defence, " ", value.GenTime, "\n")
	}
```
我们通过for循环，每一秒生成一个英雄，攻击力和防御力随机，然后调用sort排序，接下来打印下看看效果
``` cmd
Hero1570780749 11 45 1570780749
Hero1570780744 25 140 1570780744
Hero1570780748 28 74 1570780748
Hero1570780750 37 106 1570780750
Hero1570780742 47 59 1570780742
Hero1570780745 56 100 1570780745
Hero1570780747 62 89 1570780747
Hero1570780741 81 87 1570780741
Hero1570780743 81 118 1570780743
Hero1570780746 94 111 1570780746
```
看的出来，优先按照攻击力排序，攻击力相同则按照防御值排序。这样，这个排序的小功能就做好了。
## interface万能接口
interface{}空接口可以接受任何类型的变量，从而可以实现类似于泛型编程的功能。golang本身并不支持泛型，原作者说泛型编程太过复杂，
以后会更新进来。
interface{}类型的变量在使用时要转化为具体的类型，否则会报错。转化方法前文提起过，现在复习一遍
``` golang
    var ife interface{}
	ife = herolists
	val, ok := ife.(HeroList)
	if !ok {
		fmt.Println("ife can't transfer to HeroList!")
		return
	}
	fmt.Println("herolist's len is ", val.Len())
```
herolists是上个例子定义的英雄列表，将它赋值给ife后，ife为interface{}类型,不能直接使用，所以用ife.(HeroList)来转换为HeroList
类型。
当然interface{}还可以做类型判断
``` golang
func JudgeType(itf interface{}) {
	switch itf.(type) {
	case string:
		fmt.Println("type is string")
	case int:
		fmt.Println("type is int")
	case HeroList:
		fmt.Println("type is HeroList")
	default:
		fmt.Println("unknown type ")
	}
}
```
## interface实现万能类型双向链表
基于上面的知识，我们再做一个例子，实现双向链表，支持头部插入，尾部插入，指定位置插入，指定位置删除，可存储任意类型数据
我们先定义链表的基本结构
``` golang
type LinkList struct {
	Head *LinkEle
	Tail *LinkEle
}

type LinkEle struct {
	Data interface{}
	Pre  *LinkEle
	Next *LinkEle
}
```
LinkeEle是链表中的每个节点类型，包含Data数据域，其为interface{}类型，以及Pre指向前一个节点的指针，Next指向后一个节点的指针。
LinkList是链表结构，包含头和尾部节点的指针
好的，为了方便取出节点数据域，我们给LinkEle实现一个GetData方法。
``` golang
func (le *LinkEle) GetData() interface{} {
	return le.Data
}
```
接下来我们实现插入操作，实现头部插入,基本思路为在头部节点前插入新节点，然后将新节点和头部节点连接，更新新节点为头部节点
``` golang
func (ll *LinkList) InsertHead(le *LinkEle) {

	if ll.Tail == nil && ll.Head == nil {
		ll.Tail = le
		ll.Head = ll.Tail
		return
	}

	ll.Head.Pre = le
	le.Pre = nil
	le.Next = ll.Head
	ll.Head = le
}
```
上面先判断链表是否为空，如果链表为空，那么直接更新头尾信息即可，否则就需要执行连接操作。同样的道理我们实现尾部插入
尾部插入实在尾部节点的后边插入
``` golang
func (ll *LinkList) InsertTail(le *LinkEle) {
	if ll.Tail == nil && ll.Head == nil {
		ll.Tail = le
		ll.Head = ll.Tail
		return
	}
	ll.Tail.Next = le
	le.Pre = ll.Tail
	le.Next = nil
	ll.Tail = le
}
```
我们测试下实现的功能
``` golang
    ll := &LinkList{nil, nil}
	fmt.Println("insert head .....................")
	for i := 0; i < 2; i++ {
		num := rand.Intn(100)
		node1 := &LinkEle{Data: num, Next: nil, Pre: nil}
		ll.InsertHead(node1)
		fmt.Println(num)
	}
	fmt.Println("after insert head .................")
	for node := ll.Head; node != nil; node = node.Next {
		val, ok := node.GetData().(int)
		if !ok {
			fmt.Println("interface transfer error")
			break
		}
		fmt.Println(val)
	}

	fmt.Println("insert tail .....................")
	for i := 0; i < 2; i++ {
		num := rand.Intn(100)
		node1 := &LinkEle{Data: num, Next: nil, Pre: nil}
		ll.InsertTail(node1)
		fmt.Println(num)
	}

	fmt.Println("after insert tail .................")
	for node := ll.Head; node != nil; node = node.Next {
		val, ok := node.GetData().(int)
		if !ok {
			fmt.Println("interface transfer error")
			break
		}
		fmt.Println(val)
	}

```
我们初始化了一个LinkList类型的链表变量ll,然后在头部插入两个节点，在尾部插入两个节点,看看效果
``` cmd
insert head .....................
81
87
after insert head .................
87
81
insert tail .....................
47
59
after insert tail .................
87
81
47
59
```
头部插入先插入81,然后插入87，所以列表变为87,81
接着尾部插入47,59，列表变为87,81,47,59
接下来实现在指定位置的节点后插入节点
``` golang
func (ll *LinkList) InsertIndex(le *LinkEle, index int) {
	if index < 0 {
		return
	}

	if ll.Head == nil {
		ll.Head = le
		ll.Tail = ll.Head
		return
	}

	node := ll.Head
	indexfind := 0
	for ; indexfind < index; indexfind++ {
		if node.Next == nil {
			break
		}
		node = node.Next
	}

	if indexfind != index {
		fmt.Println("index is out of range")
		return
	}
	//node 后边的节点缓存起来
	nextnode := node.Next

	//node 和le连接起来
	node.Next = le
	le.Pre = node

	if node == ll.Tail {
		ll.Tail = le
		return
	}

	//le和next node 连接起来
	if nextnode != nil {
		nextnode.Pre = le
		le.Next = nextnode
	}
}
```
首先判断链表是否为空，如果为空，则直接更新节点为列表头尾节点。否则判断插入位置是否越界，如果越界则直接返回。
如果不越界，则将节点插入，并判断插入节点是否为最后位置，如果为最后位置，则更新其为尾结点。
接下来我们测试在第三个节点后边插入节点。补充代码如下，前边的插入不变，看下效果
``` golang
fmt.Println("insert after third element........")
	{
		num := rand.Intn(100)
		node1 := &LinkEle{Data: num, Next: nil, Pre: nil}
		ll.InsertIndex(node1, 2)
		fmt.Println(num)
	}

	fmt.Println("after insert index .................")
	for node := ll.Head; node != nil; node = node.Next {
		val, ok := node.GetData().(int)
		if !ok {
			fmt.Println("interface transfer error")
			break
		}
		fmt.Println(val)
	}
```
结果如下
``` cmd
insert head .....................
81
87
after insert head .................
87
81
insert tail .....................
47
59
after insert tail .................
87
81
47
59
insert after third element........
81
after insert index .................
87
81
47
81
59
```
前边是头部和尾部插入的输出，接着我们在第三个位置节点后插入81,打印看到确实插入在了47的后边。
同样我们接下来实现删除操作，删除指定位置的节点
``` golang
func (ll *LinkList) DelIndex(index int) {
	if index < 0 {
		return
	}

	if ll.Head == nil {
		return
	}

	node := ll.Head
	indexfind := 0
	for ; indexfind < index; indexfind++ {
		if node.Next == nil {
			break
		}
		node = node.Next
	}

	if indexfind != index {
		fmt.Println("index is out of range")
		return
	}

	if ll.Head == ll.Tail {
		ll.Tail = nil
		ll.Head = ll.Tail
		return
	}

	//如果是头节点
	if node == ll.Head {
		ll.Head = node.Next
		node.Next.Pre = nil
		return
	}

	//如果是尾结点
	if node == ll.Tail {
		ll.Tail = node.Pre
		ll.Tail.Next = nil
		return
	}
	//将前后连接起来
	node.Pre.Next = node.Next
	node.Next.Pre = node.Pre
}
```
和插入操作类似，判断是否为空链表，是否只有一个节点等情况，接着判断删除的是否为头结点，是否为尾节点，否则就执行删除后的连接操作。
继续上边的测试代码，我们添加如下测试代码补充测试
``` golang
fmt.Println("delete second element, its index is 1")
	ll.DelIndex(1)
	fmt.Println("after delete second element, its index is 1")
	for node := ll.Head; node != nil; node = node.Next {
		val, ok := node.GetData().(int)
		if !ok {
			fmt.Println("interface transfer error")
			break
		}
		fmt.Println(val)
	}
```
输出如下
``` cmd
87
81
47
81
59
delete second element, its index is 1
after delete second element, its index is 1
87
47
81
59
```
我们删除了index为1，也就是第二个节点81，测试成功了。
由于链表的数据域Data为空接口类型，所以可以存储各种类型的数据，只需要在GetData时做具体类型转换即可。
接下来读者可以自己考虑实现删除头部节点，删除尾部节点等。
可以下载我的源码 :
[https://github.com/secondtonone1/golang-/tree/master/day26](https://github.com/secondtonone1/golang-/tree/master/day26)

## 总结
本文通过两个小例子实战，演示了interface常用的用法，为接下来讲解反射做铺垫。
感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)

