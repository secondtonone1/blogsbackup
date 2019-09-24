---
title: golang结构体
date: 2019-09-11 15:37:30
categories: [golang]
tags: [golang]
---
golang支持面向对象的设计，一般支持面向对象的语言都会有class的设计，但是golang没有class关键字，只有struct结构体。通过结构体达到类的效果，这叫做大成若缺，其用不弊。
## struct简介
在使用struct之前，先介绍golang的一个特性，golang允许用户将类型A定义为另一种类型B，并为类型B添加方法。
``` golang
type Integer int 
func (a Integer) Less (b Integer) bool{
    return a < b
}
```
<!--more-->
我们将int定义为一种新的类型Integer，Integer就和int不是一个类型了，这和C++不一样。然后为Integer添加了方法Less，所有Integer对象都可以使用Less方法。类似于C++的成员函数。下面我们使用一下
``` golang
func main() {
	var varint1 Integer = 100
	var varint2 Integer = 200
	fmt.Println(varint1.Less(varint2))
}
```
定义了两个变量varint1和varint2，调用了varint1的Less方法，输出true
下面介绍struct的基本定义
``` golang
//构造函数和初始化
type Rect struct{
    x,y float64
    width, height float64
}
```
定义了一个结构体Rect，Rect包含四个成员，x,y为float64类型，width, height为float64类型。
下面为Rect定义方法，计算矩形面积
``` golang
func (r* Rect) Area() float64{
    return r.width* r.height
}
```
golang结构体没有public,private等字段，是通过成员的大小写区分权限的。大写的结构体成员，别的包可以访问，小写的成员不可被别的包访问。
Rect的成员都为小写，所以别的包无法访问，但是可以通过定义大写的方法，提供给别的包访问
``` golang
func (r *Rect) GetX() float64 {
	return r.x
}

func (r *Rect) GetY() float64 {
	return r.y
}

func (r *Rect) Area() float64 {
	return r.width * r.height
}

func (r *Rect) GetWidth() float64 {
	return r.width
}

func (r *Rect) GetHeight() float64 {
	return r.height
}
```
这样其他的包就可以通过Rect的方法访问Rect内部变量值了。
## 结构体方法和函数的区别
结构体方法就好比是C++的成员函数，是类对象调用的方法。函数和结构体对象无关，可以自由编写。
二者定义也有区别
``` golang
func (this* 结构体类型) 方法名(方法参数列表) 方法返回值{
    //方法内部实现
}

func 函数名(函数参数列表) 函数返回值{
    //函数内部实现
}
```
## golang的精髓是组合
``` golang
type Inner struct {
	Name string
	Num  int
}

type Wrappers struct {
	inner Inner
	Name  string
}
```
Wrappers 包含了Inner结构体，golang中叫做组合。下面写代码打印信息,我们先为Wrappers添加方法
``` golang
func (wp *Wrappers) PrintInfo() {
	fmt.Println(wp.Name)
	fmt.Println(wp.inner.Name)
	fmt.Println(wp.inner.Num)
}
```
定义变量调用方法
``` golang
func main() {
	wp := &Wrappers{Name: "wrapper", inner: Inner{Name: "inner", Num: 100}}
	wp.PrintInfo()
}
```
打印结果如下
``` cmd
wrapper
inner
100
```
组合后，打印内部成员inner的Name需要显示指定wp.inner.Name,因为默认打印wp.Name是Wrappers的。
## 匿名组合实现继承(派生)
``` golang
//匿名组合和派生
type Base struct {
	Name string
}

func (base *Base) Foo() {
	fmt.Println("this is Base Foo")
}

func (base *Base) Bar() {
	fmt.Println("this is Base Bar")
}

type Foo struct {
	//匿名组合
	Base
}

func (foo *Foo) Foo() {
	foo.Base.Foo()
	fmt.Println("this is Foo Foo")
}
```
Foo内部组合了Base,但是并没有显示指定成员名，Foo内部只写了Base类型，这叫做匿名组合，
匿名组合会在Foo内部自动生成Base同名的成员变量Base，golang根据匿名组合，会认为Foo继承自Base，
从而Foo拥有Base的方法和成员。下面写代码看看效果
``` golang
func main() {
	foo := &Foo{}
	//Foo继承Base，所以拥有Name属性
	foo.Name = "foobase"
	//Foo 重写(覆盖)了Base的Foo
	foo.Foo()
	//Foo继承了Base的Bar函数
	foo.Bar()
	//显示调用基类Base的Foo
	foo.Base.Foo()
}
```
由于Foo继承Base后重写了Foo方法，所以想要调用Base的Foo方法，需要显示调用。
## 匿名指针组合
匿名组合如果是指针类型，在子类对象定义时需要显示初始化基类指针，否则会出问题。
先定义匿名组合结构体
``` golang
//匿名指针组合
type DerivePoint struct {
	*Base
}

func (derivep *DerivePoint) Foo() bool {
	fmt.Println("this is DerivePoint Foo")
	fmt.Println("inherit base ,name is ", derivep.Name)
	return true
}
```
定义了DerivePoint类，和方法Foo，在Foo内部打印了derivep的Name，该Name继承自*Base
下面调用
``` golang
	dr := &DerivePoint{Base: &Base{Name: "base"}}
	dr.Foo()
```
输出如下
``` cmd
this is DerivePoint Foo
inherit base ,name is  base
```
可见输出了Name值。
## 匿名组合造成命名冲突
``` golang
//重复定义，因为匿名组合默认用类型做变量名
type MyJob struct {
	*Logger
	Name string
	*log.Logger // duplicate field Logger
}
```
MyJob匿名组合了Logger类和log.Logger类，由于匿名组合默认用类型做变量名，所以编译器会认为定义了两个Logger名的成员，
从而报错，所以匿名组合一定要注意这一点。
## 多重继承
golang 支持多重继承，实现多重继承只需要多个匿名组合即可。

golang结构体介绍完毕，关注我的公众号领取学习资料
![wxgzh.jpg](wxgzh.jpg)