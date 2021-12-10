---
title: defer和panic
date: 2021-12-08 14:56:05
categories: [golang]
tags: [golang]
---
## 简介
今天谈谈go的两个特性，defer和panic, defer在函数return 时，将返回值压入栈，然后执行defer函数，最后返回。panic是手动触发崩溃的一种策略，可以在panic本层的函数实现defer函数，在defer里通过recover捕获该层崩溃，如果本层崩溃未被捕获，则交由上一层捕获。
<!--more-->
##  defer机制和使用
一个函数定义了多个defer函数，defer的调用顺序和栈一样，先进后出，最先调用的是最后写的defer。函数将返回值入栈，然后执行析构，在析构之前要执行defer的操作。defer使用的注意事项
#### defer常用来释放变量
我们实现一个文件copy函数，将src路径下的文件copy到dst路径下
``` golang
func CopyFile(dst, src string) (written int64, err error){
    srcF, err := os.Open(src)
    if err != nil{
        return 
    }
    defer srcF.Close()

    dstF, err := os.Create(dst)
    if err != nil{
        return 
    }
    defer dstF.Close()
    return io.Copy(dstF, srcF)
}
```
注意，如果Open失败或者Create失败，千万不要调用src.Close，因为src为nil。
#### defer被声明时，其参数是实时解析和捕获
``` golang
func DeferParam() {
	i := 0
	defer func(m int) {
		log.Println(m)
	}(i)
	i++
}
```
程序输出0，因为defer声明时捕获i的值为0，传入函数后输出也是0，不管以后i变成什么值。
如果defer是无参函数，内部引用了外部变量，就不同了，会记录i的引用
``` golang

func DeferNoParam() {
	i := 0
	defer func() {
		log.Println(i)
	}()
	i++
}
```
最后i变为什么值，defer就输出什么值。此时输出值为1
####  defer调用顺序为栈式调用
``` golang
func DeferOrder() {
	for i := 0; i < 5; i++ {
		defer func(m int) {
			log.Println(m)
		}(i)
	}
}
```
输出结果为4，3，2，1，0
####  defer可以捕获函数返回值
因为defer可以捕获函数内变量，所以可以捕获函数的返回值
``` golang
func DeferReturn() (res int) {
	defer func() {
		res++
		log.Println(res)
	}()

	return 0
}
```
defer输出为1，因为defer捕获到返回值为0，+1就输出1

####  defer链式调用
defer 只能执行一个函数，defer是栈式调用，后入先出规则。当defer执行链式操作时，前边的表达式都会优先求值，只有最后一个表达式入栈延迟执行。
``` golang
type Slice []int

func NewSlice() *Slice {
	slice := make(Slice, 0)
	return &slice
}

func (s *Slice) AddSlice(val int) *Slice {
	*s = append(*s, val)
	log.Println(val)
	return s
}

s := NewSlice()
defer s.AddSlice(1).AddSlice(3)
s.AddSlice(2)
```
s.AddSlice(1)优先被计算，然后是s.AddSlice(2)，最后是AddSlice(3)，输出值为123
## panic
我们在开发阶段可以在关键错误处使用panic触发程序崩溃，也可以在重要的逻辑不允许有异常是做逻辑判断，不符合逻辑的调用panic异常崩溃。
我们先实现两个函数
``` golang
func Funlv1() {
	defer func() {
		log.Println("Funlv1 exit ...")
	}()
	log.Println("Funlv1 begin")
	panic("sorry, Funlv1 panic")
	log.Println("Funlv1 end")
}

func Funlv2() {
	defer func() {
		log.Println("Funlv2 exit ...")
	}()
	log.Println("Funlv2 begin")
	Funlv1()
	log.Println("Funlv2 end")
}
```
在main函数中调用
``` golang
Funlv2()
```
程序输出
``` cmd
Funlv2 begin
Funlv1 begin
Funlv1 exit ...
Funlv2 exit ...
panic: sorry, Funlv1 panic

goroutine 1 [running]:
main.Funlv1()
        D:/github/go-interview-questions/defer/defer.go:103 +0xa6
main.Funlv2()
        D:/github/go-interview-questions/defer/defer.go:112 +0x8f
main.main()
        D:/github/go-interview-questions/defer/defer.go:127 +0x27
exit status 2
```
没有输出Funlv2 end 和 Funlv1 end, 因为Panic之后的逻辑不会执行，Funlv1内部panic后，不会输出Funlv1 end， 这个崩溃抛给上层Funlv2，也没有捕获，导致Funlv2也不会输出Funlv2 end，所以无论是否panic，defer函数都会执行。我们要做的是通过recover捕获panic错误，看下改进的代码
``` golang
func Funlv1Safe() {
	defer func() {
		if err := recover(); err != nil {
			log.Println("Funlv1 catch panic , err is ", err)
		}
		log.Println("Funlv1 exit ...")
	}()
	log.Println("Funlv1 begin")
	panic("sorry, Funlv1 panic")
	log.Println("Funlv1 end")
}

func Funlv2Safe() {
	defer func() {
		if err := recover(); err != nil {
			log.Println("Funlv2 catch panic , err is ", err)
		}
		log.Println("Funlv2 exit ...")
	}()
	log.Println("Funlv2 begin")
	Funlv1Safe()
	log.Println("Funlv2 end")
}
```
在main函数中调用
``` golang
Funlv2Safe()
```
输出
``` cmd
Funlv2 begin
Funlv1 begin
Funlv1 catch panic , err is  sorry, Funlv1 panic
Funlv1 exit ...
Funlv2 end
Funlv2 exit ...
```
由于Funlv1调用panic所以不会输出Funlv1 end 但是Funlv1的defer函数通过recover捕获了panic，所以Funlv2可以正常执行并结束。如果我们注释掉Funlv1的defer函数中的recover，就会由Funlv2来捕获panic
注意如果是捕获本层panic，一定要将defer写在panic之上，否则无法捕获，recover也要写在defer函数或者上层函数中。

panic要等待defer结束后才会向上传递，如果defer捕获了panic就不传递了
``` golang
func defer_call() {
	defer func() { fmt.Println("打印前") }()
	defer func() { fmt.Println("打印中") }()
	defer func() { fmt.Println("打印后") }()
	panic("触发异常")
}
```
所以上述代码调用输出为打印后，打印中，打印前，最后panic触发异常

## 总结
1 panic执行后，后续语句不再执行，会执行defer函数，如有多个defer就遵循栈式调用
2 如果某个goroutine没有捕获panic，则整个进程崩溃，不仅仅是该goroutine崩溃
3 panic被本层recover后，会影响到本层函数panic之后的语句执行，不影响当前goroutine的执行。
4 recover必须写在defer语句或者上层函数才可生效
5 recover的作用是保证捕获异常后程序可以继续稳定运行。
