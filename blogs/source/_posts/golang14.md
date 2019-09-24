---
title: golang 接口内部实现
date: 2019-09-24 14:16:53
categories: [golang]
tags: [golang]
---
前文介绍过golang interface用法，本文详细剖析interface内部实现和细节。
## empty interface实现细节
interface底层使用两种类型实现的，一个是eface，一个是iface。当interface中没有方法的时候，底层是通过eface实现的。
当interface包含方法时，那么它的底层是通过iface实现的。
对于iface和eface具体实现在go源码runtime2.go中，我们看下源码
<!--more-->
``` golang
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```
可以看到eface包含两个结构，一个是_type类型指针，一个是unsafe包的Pointer变量
继续追踪Pointer
``` golang
type Pointer *ArbitraryType
type ArbitraryType int
```
可以看出Pointer实际上是int类型的指针。我们再看看_type类型
``` golang
type _type struct {
	size       uintptr
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32
	tflag      tflag
	align      uint8
	fieldalign uint8
	kind       uint8
	alg        *typeAlg
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff
	ptrToThis typeOff
}
```
size 为该类型所占用的字节数量。
kind 表示类型的种类，如 bool、int、float、string、struct、interface 等。
str 表示类型的名字信息，它是一个 nameOff(int32) 类型，通过这个 nameOff，可以找到类型的名字字符串

eface结构总结图
![1.jgp](1.jpg)
eface 分两个部分， *_type 类型为实际类型转化为type类型的指针，data为实际数据。
## eface证明

