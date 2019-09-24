---
title: golang 函数介绍
date: 2019-09-11 14:50:45
categories: 技术开发
tags: [golang]
---
## 函数简介
函数是编程语言中不可缺少的部分，在golang这门语言中函数是一等公民。也是使用好golang的必备技能。
看下golang函数的格式
``` golang
func 函数名(函数参数)返回值类型{

}
```
一个简单的函数
``` golang
func HelloFunc(str string) string{
    return str
}
```
该函数返回传入的字符串,函数调用如下
``` golang
fmt.Println(Hello("Nice to meet you!"))
```
<!--more-->
## 返回多个返回值
golang 允许函数返回多个返回值
``` golang
func Add(a int, b int) (ret int, err error) {
	if a < 0 || b < 0 {
		//不允许负数相加
		err = errors.New("should be non-negative numbers")
		return
	}
	return a + b, nil
}
```
函数返回两个数相加的结果和错误，下面调用这个函数
``` golang
 res, err:=Add(100, 200) 
 if err != nil{
     fmt.Println("add error !")
     return
 }
 fmt.Println("add result is ", res)
```
## 函数内部可以使用标签
``` golang
func myfunc() {
	i := 0
HERE:
	fmt.Println(i)
	i++
	if i < 10 {
		goto HERE
	}
}
```
goto会跳转到fmt.Println(n)处继续执行，达到i循环自增并输出的效果
## 函数可变参数
golang 中允许函数使用可变参数
``` golang
//不定参数
func myfuncv(args ...int) {
	for _, arg := range args {
		fmt.Println(arg)
	}
}
```
可变参数的写法是参数名后加上...类型，我们调用一下这个函数
``` golang
myfuncv(1,2,4,7,8)
```
可变参数可以通过切片获取元素。再写一个函数
``` golang
func myfuncv3(args ...int) {
	//按原样传递
	myfuncv(args...)
	//按切片传递
	myfuncv(args[1:]...)
}
```
这个函数内部将参数用展开传递给myfuncv,args是一个切片，展开用...，这样myfuncv就可以继续处理了。
下面来个复杂点的，带interface参数的函数
``` golang
func MyPrintf(args ...interface{}) {
	for _, arg := range args {
		//interface 任意类型，
		//arg.(type)只能用于switch结构
		switch arg.(type) {
		case int:
			fmt.Println(arg, "is an int value.")
		case string:
			fmt.Println(arg, "is a string value.")
		case int64:
			fmt.Println(arg, "is an int64 value")
		default:
			fmt.Println(arg, "is an unknown type")
		}
	}
}
```
MyPrintf函数将参数args遍历后根据类型判断，做出相应输出。interface类型的变量后边加上.(type)就可以返回他的类型。接口的相关知识之后讲解。
目前就介绍到此，我的公众号
![wxgzh.jpg](wxgzh.jpg)