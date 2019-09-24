---
title: golang接口
date: 2019-09-12 14:59:46
categories: [golang]
tags: [golang]
---
## 接口简介
golang 中接口是常用的数据结构，接口可以实现like的功能。什么叫like呢？
比如麻雀会飞，老鹰会飞，他们都是鸟，鸟有翅膀可以飞。飞机也可以飞，
飞机就是像鸟一样，like bird, 所以我们可以说飞机，气球，苍蝇都像鸟一样可以飞翔。
但他们不是鸟，那么对比继承的关系，老鹰继承自鸟类，它也会飞，但他是鸟。
先看一个接口定义
``` golang
type Bird interface {
	Fly() string
}
```
定义了一个Bird类型的interface， 内部生命了一个Fly方法，参数为空，返回值为string。
接口声明方法和struct不同，接口的方法写在interface中，并且不能包含func和具体实现。
另外interface内部不能声明成员变量。
下面去实现蝴蝶类和飞机类，实现like-bird的功能。像鸟一样飞。
<!--more-->
``` golang
type Plane struct {
	name string
}

func (p *Plane) Fly() string {
	fmt.Println(p.name, " can fly like a bird")
	return p.name
}

type Butterfly struct {
	name string
}

func (bf *Butterfly) Fly() string {
	fmt.Println(bf.name, " can fly like a bird")
	return bf.name
}
```
实现了Plane和Butterfly类，并且实现了Fly方法。那么飞机和蝴蝶就可以像鸟一样飞了。
我们在主函数中调用
``` golang
    pl := &Plane{name: "plane"}
	pl.Fly()

	bf := &Butterfly{name: "butterfly"}
    bf.Fly()
```
输出如下
``` cmd
plane  can fly like a bird
butterfly  can fly like a bird
```
有人会问，单独实现Plane和Butterfly不就好了，为什么要和Bird扯上关系呢？
因为接口作为函数形参，可以接受不同的实参类型，只要这些实参实现了接口的方法，
都可以达到动态调用不同实参的方法。
``` golang
func FlyLikeBird(bird Bird) {
    bird.Fly()
}
```
下面我们在main函数中调用上面这个函数，传入不同的实参
``` golang
    FlyLikeBird(pl)
	FlyLikeBird(bf)
```
输出如下
``` cmd
plane  can fly like a bird
butterfly  can fly like a bird
```
这样就是实现了动态调用。有点类似于C++的多态，golang又不是通过继承达到这个效果的，
只要结构体实现了接口的方法就可以转化为接口类型。
golang这种实现机制突破了Java，C++等传统静态语言显示继承的弊端。
## 接口类型转换和判断
struct类型如果实现了接口方法，可以赋值给对应的接口类型，接口类型同样可以转化为struct类型。
我们再写一个函数，通过该函数内部将bird接口转化为不同的类型，从而打印具体的传入类型。
``` golang
func GetFlyType(bird Bird) {
	_, ok := bird.(*Butterfly)
	if ok {
		fmt.Println("type is *butterfly")
		return
	}

	_, ok = bird.(*Plane)
	if ok {
		fmt.Println("type is *Plane")
		return
	}

	fmt.Println("unknown type")
}
```
main函数调用
``` golang
func main() {
	pl := &Plane{name: "plane"}
    bf := &Butterfly{name: "butterfly"}
	GetFlyType(pl)
    GetFlyType(bf)
}
```
输出如下
``` cmd
type is *Plane
type is *butterfly
```
看得出来接口也是可以转化为struct的。
结构体变量, bool类型:=接口类型.(结构体类型)
bool类型为false说明不能转化，true则能转化。

## 万能接口interface{}
golang 提供了万能接口, 类型为interface{}, 任何具体的结构体类型都能转化为该类型。我们将之前判断类型的例子
稍作修改。定义Human类和Human的Walk方法，然后实现另一个判断函数，参数为interface{}
``` golang
type Human struct {
}

func (*Human) Walk() {

}

func GetFlyType2(inter interface{}) {
	_, ok := inter.(*Butterfly)
	if ok {
		fmt.Println("type is *butterfly")
		return
	}

	_, ok = inter.(*Plane)
	if ok {
		fmt.Println("type is *Plane")
		return
	}
	_, ok = inter.(*Human)
	if ok {
		fmt.Println("type is *Human")
		return
	}
	fmt.Println("unknown type")
}
```
在main函数中调用，我们看看结果
``` golang
func main() {
	pl := &Plane{name: "plane"}
    bf := &Butterfly{name: "butterfly"}
    hu := &Human{}
	GetFlyType2(pl)
    GetFlyType2(bf)
    GetFlyType2(hu)
}
```
看到输出
``` cmd
type is *Plane
type is *butterfly
type is *Human
```
## .(type)判断具体类型
接口还提供了一个功能，通过.(type)返回具体类型，但是.(type)只能用在switch中。
我们实现另一个版本的类型判断
``` golang
func GetFlyType3(inter interface{}) {
	switch inter.(type) {
	case *Butterfly:
		fmt.Println("type is *Butterfly")
	case *Plane:
		fmt.Println("type is *Plane")
	case *Human:
		fmt.Println("type is *Human")
	default:
		fmt.Println("unknown type ")
	}
}
```
main函数中调用这个函数
``` golang
    GetFlyType3(pl)
	GetFlyType3(bf)
	GetFlyType3(hu)
```
输出结果如下
``` cmd
type is *Plane
type is *Butterfly
type is *Human
```
所以.(type)也实现了类型转换

这样接口基础都介绍完毕了，下一篇介绍接口内部实现和剖析。
感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)

