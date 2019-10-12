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
![1.jpg](1.jpg)
eface 分两个部分， *_type 类型为实际类型转化为type类型的指针，data为实际数据。
## 具体类型如何转化为eface
我们写一段程序efacedemo.go，然后用gobuild命令生成可执行文件，再用汇编查看下源码。
``` golang
package main

import "fmt"

type EmpInter interface {
}

type EmpStruct struct {
	num int
}

func main() {

	emps := EmpStruct{num: 1}
	var empi EmpInter
	empi = emps
	fmt.Println(empi)
	fmt.Println(emps)

}
```
先用gcflags标记编译生成可执行文件efacedemo
go build -gcflags "-l" -o efacedemo  efacedemo.go
然后执行go tool objdump 将 可执行程序efacedemo中main包的main函数转为汇编代码
go tool objdump -s "main\.main" efacedemo
``` cmd
efacedemo.go:12       0x48ffa0                65488b0c2528000000      MOVQ GS:0x28, CX
  efacedemo.go:12       0x48ffa9                488b8900000000          MOVQ 0(CX), CX
  efacedemo.go:12       0x48ffb0                483b6110                CMPQ 0x10(CX), SP
  efacedemo.go:12       0x48ffb4                0f86ae000000            JBE 0x490068
  efacedemo.go:12       0x48ffba                4883ec48                SUBQ $0x48, SP
  efacedemo.go:12       0x48ffbe                48896c2440              MOVQ BP, 0x40(SP)
  efacedemo.go:12       0x48ffc3                488d6c2440              LEAQ 0x40(SP), BP
  efacedemo.go:16       0x48ffc8                48c7042401000000        MOVQ $0x1, 0(SP)
  efacedemo.go:16       0x48ffd0                e8fb89f7ff              CALL runtime.convT64(SB)  //注意这里调用runtime包的convT64函数
  efacedemo.go:16       0x48ffd5                488b442408              MOVQ 0x8(SP), AX
  efacedemo.go:17       0x48ffda                0f57c0                  XORPS X0, X0
  efacedemo.go:17       0x48ffdd                0f11442430              MOVUPS X0, 0x30(SP)
  efacedemo.go:17       0x48ffe2                488d0d17c20100          LEAQ runtime.types+111104(SB
  efacedemo.go:17       0x48ffe9                48894c2430              MOVQ CX, 0x30(SP)
  efacedemo.go:17       0x48ffee                4889442438              MOVQ AX, 0x38(SP)
  efacedemo.go:17       0x48fff3                488d442430              LEAQ 0x30(SP), AX
  efacedemo.go:17       0x48fff8                48890424                MOVQ AX, 0(SP)
  efacedemo.go:17       0x48fffc                48c744240801000000      MOVQ $0x1, 0x8(SP)
  efacedemo.go:17       0x490005                48c744241001000000      MOVQ $0x1, 0x10(SP)
  efacedemo.go:17       0x49000e                e8fd98ffff              CALL fmt.Println(SB)
  efacedemo.go:18       0x490013                48c7042401000000        MOVQ $0x1, 0(SP)
  efacedemo.go:18       0x49001b                e8b089f7ff              CALL runtime.convT64(SB)   //注意这里调用runtime包的convT64函数
  efacedemo.go:18       0x490020                488b442408              MOVQ 0x8(SP), AX
  efacedemo.go:18       0x490025                0f57c0                  XORPS X0, X0
  efacedemo.go:18       0x490028                0f11442430              MOVUPS X0, 0x30(SP)
  efacedemo.go:18       0x49002d                488d0dccc10100          LEAQ runtime.types+111104(SB
  efacedemo.go:18       0x490034                48894c2430              MOVQ CX, 0x30(SP)
  efacedemo.go:18       0x490039                4889442438              MOVQ AX, 0x38(SP)
  efacedemo.go:18       0x49003e                488d442430              LEAQ 0x30(SP), AX
  efacedemo.go:18       0x490043                48890424                MOVQ AX, 0(SP)
  efacedemo.go:18       0x490047                48c744240801000000      MOVQ $0x1, 0x8(SP)
  efacedemo.go:18       0x490050                48c744241001000000      MOVQ $0x1, 0x10(SP)
  efacedemo.go:18       0x490059                e8b298ffff              CALL fmt.Println(SB)
  efacedemo.go:20       0x49005e                488b6c2440              MOVQ 0x40(SP), BP
  efacedemo.go:20       0x490063                4883c448                ADDQ $0x48, SP
  efacedemo.go:20       0x490067                c3                      RET
  efacedemo.go:12       0x490068                e883f2fbff              CALL runtime.morestack_noctx
  efacedemo.go:12       0x49006d                e92effffff              JMP main.main(SB)
```
抛开寄存器寻址和寄存数据不谈，我们看到efacedemo.go:16行， CALL runtime.convT64(SB)语句说明调用了runtime包的convT64函数
这个函数在runtime.go 中有声明
``` golang
// Specialized type-to-interface conversion.
// These return only a data pointer.
func convT16(val any) unsafe.Pointer     // val must be uint16-like (same size and alignment as a uint16)
func convT32(val any) unsafe.Pointer     // val must be uint32-like (same size and alignment as a uint32)
func convT64(val any) unsafe.Pointer     // val must be uint64-like (same size and alignment as a uint64 and contains no pointers)
func convTstring(val any) unsafe.Pointer // val must be a string
func convTslice(val any) unsafe.Pointer  // val must be a slice
```
看注释就知道是type类型转化为interface类型，计算机将EmpStruct强制转化为type类型后，type类型进一步转化为interface，并且将EmpStruct数据
转化为unsafe.Pointer,毕竟int在64位机器中为8字节，所以采用了convT64函数。
还有一个函数convT2E,这个函数是将type转化为eface类型，大家可以读一读runtime源码。
convT2E源码
``` golang
func convT2E(t *_type, elem unsafe.Pointer) (e eface) {
	if raceenabled {
		raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2E))
	}
	if msanenabled {
		msanread(elem, t.size)
	}
	x := mallocgc(t.size, t, true)
	// TODO: We allocate a zeroed object only to overwrite it with actual data.
	// Figure out how to avoid zeroing. Also below in convT2Eslice, convT2I, convT2Islice.
	typedmemmove(t, x, elem)
	e._type = t
	e.data = x
	return
}
```
内部调用了typedmemove做类型判断，所以一个类能否转化为某个接口是在runtime这一层做判断的。判断条件就是我之前所说的是否
实现了该接口所有的方法。
将上面的代码用eface图解就是
![2.jgp](2.jpg)
## 具体类型如何转换为iface
当接口中带有方法的时候，接口底层的实现就是通过iface结构实现的。我们下一个带方法的接口，然后反汇编一下。
``` golang
package main

import "fmt"

type EmpInter interface {
	GetNum() int
}

type EmpStruct struct {
	num int
}

func (es *EmpStruct) GetNum() int {
	return es.num
}

func main() {

	emps := EmpStruct{num: 1}
	var empi EmpInter
	empi = &emps
	fmt.Println(empi)
	fmt.Println(emps)

}
```
和之前操作一样，先编译
go build -gcflags "-l" -o ifacedemo  ifacedemo.go
然后反汇编找到指令
go tool objdump -s "main\.main" ifacedemo
生成的汇编指令如下
``` cmd
 ifacedemo.go:17       0x48ffb0                65488b0c2528000000      MOVQ GS:0x28, CX
  ifacedemo.go:17       0x48ffb9                488b8900000000          MOVQ 0(CX), CX
  ifacedemo.go:17       0x48ffc0                483b6110                CMPQ 0x10(CX), SP
  ifacedemo.go:17       0x48ffc4                0f86c1000000            JBE 0x49008b
  ifacedemo.go:17       0x48ffca                4883ec50                SUBQ $0x50, SP
  ifacedemo.go:17       0x48ffce                48896c2448              MOVQ BP, 0x48(SP)
  ifacedemo.go:17       0x48ffd3                488d6c2448              LEAQ 0x48(SP), BP
  ifacedemo.go:19       0x48ffd8                488d0521c30100          LEAQ runtime.types+111360(SB), AX
  ifacedemo.go:19       0x48ffdf                48890424                MOVQ AX, 0(SP)
  ifacedemo.go:19       0x48ffe3                e868b3f7ff              CALL runtime.newobject(SB)
  ifacedemo.go:19       0x48ffe8                488b442408              MOVQ 0x8(SP), AX
  ifacedemo.go:19       0x48ffed                4889442430              MOVQ AX, 0x30(SP)
  ifacedemo.go:19       0x48fff2                48c70001000000          MOVQ $0x1, 0(AX)
  ifacedemo.go:22       0x48fff9                488b0d48ce0400          MOVQ go.itab.*main.EmpStruct,main.EmpInter+8(SB), CX
  ifacedemo.go:22       0x490000                0f57c0                  XORPS X0, X0
  ifacedemo.go:22       0x490003                0f11442438              MOVUPS X0, 0x38(SP)
  ifacedemo.go:22       0x490008                48894c2438              MOVQ CX, 0x38(SP)
  ifacedemo.go:22       0x49000d                4889442440              MOVQ AX, 0x40(SP)
  ifacedemo.go:22       0x490012                488d4c2438              LEAQ 0x38(SP), CX
  ifacedemo.go:22       0x490017                48890c24                MOVQ CX, 0(SP)
  ifacedemo.go:22       0x49001b                48c744240801000000      MOVQ $0x1, 0x8(SP)
  ifacedemo.go:22       0x490024                48c744241001000000      MOVQ $0x1, 0x10(SP)
  ifacedemo.go:22       0x49002d                e8de98ffff              CALL fmt.Println(SB)
  ifacedemo.go:23       0x490032                488b442430              MOVQ 0x30(SP), AX
  ifacedemo.go:23       0x490037                488b00                  MOVQ 0(AX), AX
  ifacedemo.go:23       0x49003a                48890424                MOVQ AX, 0(SP)
  ifacedemo.go:23       0x49003e                e88d89f7ff              CALL runtime.convT64(SB)
  ifacedemo.go:23       0x490043                488b442408              MOVQ 0x8(SP), AX
  ifacedemo.go:23       0x490048                0f57c0                  XORPS X0, X0
  ifacedemo.go:23       0x49004b                0f11442438              MOVUPS X0, 0x38(SP)
  ifacedemo.go:23       0x490050                488d0da9c20100          LEAQ runtime.types+111360(SB), CX
  ifacedemo.go:23       0x490057                48894c2438              MOVQ CX, 0x38(SP)
  ifacedemo.go:23       0x49005c                4889442440              MOVQ AX, 0x40(SP)
  ifacedemo.go:23       0x490061                488d442438              LEAQ 0x38(SP), AX
  ifacedemo.go:23       0x490066                48890424                MOVQ AX, 0(SP)
  ifacedemo.go:23       0x49006a                48c744240801000000      MOVQ $0x1, 0x8(SP)
  ifacedemo.go:23       0x490073                48c744241001000000      MOVQ $0x1, 0x10(SP)
  ifacedemo.go:23       0x49007c                e88f98ffff              CALL fmt.Println(SB)
  ifacedemo.go:25       0x490081                488b6c2448              MOVQ 0x48(SP), BP
  ifacedemo.go:25       0x490086                4883c450                ADDQ $0x50, SP
  ifacedemo.go:25       0x49008a                c3                      RET
  ifacedemo.go:17       0x49008b                e860f2fbff              CALL runtime.morestack_noctxt(SB)
  ifacedemo.go:17       0x490090                e91bffffff              JMP main.main(SB)
  :-1                   0x490095                cc                      INT $0x3
  :-1                   0x490096                cc                      INT $0x3
  :-1                   0x490097                cc                      INT $0x3
  :-1                   0x490098                cc                      INT $0x3
  :-1                   0x490099                cc                      INT $0x3
  :-1                   0x49009a                cc                      INT $0x3
  :-1                   0x49009b                cc                      INT $0x3
  :-1                   0x49009c                cc                      INT $0x3
  :-1                   0x49009d                cc                      INT $0x3
  :-1                   0x49009e                cc                      INT $0x3
  :-1                   0x49009f                cc                      INT $0x3

```
抛开寄存器寻址和移动数据，我们查看call和重要的数据copy
19行LEAQ runtime.types+111360(SB), AX做了类型上的转换
19行CALL runtime.newobject(SB)调用了runtime包的newobject方法，开辟我们结构体的指针
22行MOVQ go.itab.*main.EmpStruct,main.EmpInter+8(SB), CX将我们结构体部分信息保存在itab中。
23行 CALL runtime.convT64(SB)， 实际上将64位的num转化为数据存在data域中。
这些函数都可以在runtime包中找到，读者可以自己阅读。另外，还有个重要函数convT2I
``` golang
func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
	t := tab._type
	if raceenabled {
		raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2I))
	}
	if msanenabled {
		msanread(elem, t.size)
	}
	x := mallocgc(t.size, t, true)
	typedmemmove(t, x, elem)
	i.tab = tab
	i.data = x
	return
}
```
这个函数将Type类型转化为Iface类型。类似的函数还有convT2Inoptr(Type转Iface指针)。读者可以自己阅读源码。
通过上面的调试和分析，我们知道iface中结构和eface略有不同，多出一个itab类型的结构，我们看看iface源码
``` golang
type iface struct {
	tab  *itab
	data unsafe.Pointer
}
```
和eface不同，iface的第一个字段是itab指针。我们继续查看itab的定义
``` golang
type itab struct {
	inter *interfacetype
	_type *_type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```
itab 各参数含义
inter和_type共同确认实际的类型信息，因为在接口中有方法的时候，itab要保存接口方法的一些额外信息，如名字，类型等。
hash 用于查询类型和判断类型，比如接口赋值，接口转换判断等。
fun 为具体的方法集，所有的方法保存在fun里。虽然fun大小为1，但是这其实是一个柔型数组，后面的地址空间连续且安全，
后面能存多少函数，取决于itab初始化多大的空间。

图解iface结构
![3.jpg](3.jpg)
结合代码，具象化的绘制一下
![4.jpg](4.jpg)
以上就是interface内部结构和动态调用的原理。根据fun方法集动态调用具体类的方法，从而实现了多态。
## gdb调试
除了可以通过反汇编的方式查看代码指令，其实通过汇编查看函数调用也是很不错的手段。
 go build -gcflags "-N -l" -o ifacedemo  ifacedemo.go
 先编译出可执行文件, -N 忽略优化
 然后gdb调试
 gdb ifacedemo
 进入gdb调试界面
 ``` cmd
 (gdb) list 20
15	}
16	
17	func main() {
18	
19		emps := EmpStruct{num: 1}
20		var empi EmpInter
21		empi = &emps
22		fmt.Println(empi)
23		fmt.Println(emps)
24	
 ```
 输入list 20 为了列举 20行左右代码，然后我们在22行处打个断点,执行run让程序跑在断点处停止
 ``` cmd
(gdb) break 22
Breakpoint 1 at 0x4872ea: file /home/secondtonone/workspace/goProject/src/golang-/ifacedemo/ifacedemo.go, line 22.
(gdb) run 
Starting program: /home/secondtonone/workspace/goProject/src/golang-/ifacedemo/ifacedemo 
[New LWP 9562]
[New LWP 9563]
[New LWP 9564]

Thread 1 "ifacedemo" hit Breakpoint 1, main.main () at /home/secondtonone/workspace/goProject/src/golang-/ifacedemo/ifacedemo.go:22
22		fmt.Println(empi)
 ```
 接下来我们查看下empi这个接口的数据信息
 ``` cmd
(gdb) p empi
$1 = {tab = 0x4d0ee0 <EmpStruct,main.EmpInter>, data = 0xc000078010}
 ```
 看得出empi是iface结构的，包含tab和data两个字段。
 我们查看下tab里的内容
 ``` cmd
(gdb) p empi.tab 
$2 = (runtime.itab *) 0x4d0ee0 <EmpStruct,main.EmpInter>
(gdb) p *empi.tab
$3 = {inter = 0x4a17a0, _type = 0x49f3c0, hash = 4144246241, _ = "\000\000\000", fun = {4747840}}
 ```
 可以看得出 tab是个地址，我们*解引用看到内部内容和上面所述一样，inter, _type, hash, 函数集合fun
 接下来我们将代码反汇编
 ``` cmd
 (gdb) disass
Dump of assembler code for function main.main:
   0x0000000000487260 <+0>:	mov    %fs:0xfffffffffffffff8,%rcx
   0x0000000000487269 <+9>:	lea    -0x58(%rsp),%rax
   0x000000000048726e <+14>:	cmp    0x10(%rcx),%rax
   0x0000000000487272 <+18>:	jbe    0x487443 <main.main+483>
   0x0000000000487278 <+24>:	sub    $0xd8,%rsp
   0x000000000048727f <+31>:	mov    %rbp,0xd0(%rsp)
   0x0000000000487287 <+39>:	lea    0xd0(%rsp),%rbp
   0x000000000048728f <+47>:	lea    0x1b22a(%rip),%rax        # 0x4a24c0
   0x0000000000487296 <+54>:	mov    %rax,(%rsp)
   0x000000000048729a <+58>:	callq  0x40b070 <runtime.newobject>
   0x000000000048729f <+63>:	mov    0x8(%rsp),%rax
   0x00000000004872a4 <+68>:	mov    %rax,0x68(%rsp)
   0x00000000004872a9 <+73>:	movq   $0x0,0x30(%rsp)
   0x00000000004872b2 <+82>:	movq   $0x1,0x30(%rsp)
   0x00000000004872bb <+91>:	mov    0x68(%rsp),%rax
   0x00000000004872c0 <+96>:	movq   $0x1,(%rax)
   0x00000000004872c7 <+103>:	xorps  %xmm0,%xmm0
   0x00000000004872ca <+106>:	movups %xmm0,0x70(%rsp)
   0x00000000004872cf <+111>:	mov    0x68(%rsp),%rax
   0x00000000004872d4 <+116>:	mov    %rax,0x50(%rsp)
   0x00000000004872d9 <+121>:	lea    0x49c00(%rip),%rcx        # 0x4d0ee0 <go.itab.*main.EmpStruct,main.EmpInter>
   0x00000000004872e0 <+128>:	mov    %rcx,0x70(%rsp)
   0x00000000004872e5 <+133>:	mov    %rax,0x78(%rsp)
   0x00000000004872ea <+138>:	mov    0x78(%rsp),%rax
   0x00000000004872ef <+143>:	mov    0x70(%rsp),%rcx
   0x00000000004872f4 <+148>:	mov    %rcx,0x80(%rsp)
   0x00000000004872fc <+156>:	mov    %rax,0x88(%rsp)
   0x0000000000487304 <+164>:	mov    %rcx,0x48(%rsp)
   0x0000000000487309 <+169>:	cmpq   $0x0,0x48(%rsp)
   0x000000000048730f <+175>:	jne    0x487316 <main.main+182>
   0x0000000000487311 <+177>:	jmpq   0x48743e <main.main+478>
   0x0000000000487316 <+182>:	test   %al,(%rcx)
   0x0000000000487318 <+184>:	mov    0x8(%rcx),%rax
   0x000000000048731c <+188>:	mov    %rax,0x48(%rsp)
   0x0000000000487321 <+193>:	jmp    0x487323 <main.main+195>
   0x0000000000487323 <+195>:	xorps  %xmm0,%xmm0
   0x0000000000487326 <+198>:	movups %xmm0,0x90(%rsp)
   0x000000000048732e <+206>:	lea    0x90(%rsp),%rax
   0x0000000000487336 <+214>:	mov    %rax,0x40(%rsp)
   0x000000000048733b <+219>:	test   %al,(%rax)
   0x000000000048733d <+221>:	mov    0x48(%rsp),%rcx
   0x0000000000487342 <+226>:	mov    0x88(%rsp),%rdx
   0x000000000048734a <+234>:	mov    %rcx,0x90(%rsp)
   0x0000000000487352 <+242>:	mov    %rdx,0x98(%rsp)
   0x000000000048735a <+250>:	test   %al,(%rax)
   0x000000000048735c <+252>:	jmp    0x48735e <main.main+254>
   0x000000000048735e <+254>:	mov    %rax,0xa0(%rsp)
   0x0000000000487366 <+262>:	movq   $0x1,0xa8(%rsp)
   0x0000000000487372 <+274>:	movq   $0x1,0xb0(%rsp)
   0x000000000048737e <+286>:	mov    %rax,(%rsp)
   0x0000000000487382 <+290>:	movq   $0x1,0x8(%rsp)
   0x000000000048738b <+299>:	movq   $0x1,0x10(%rsp)
   0x0000000000487394 <+308>:	callq  0x480c60 <fmt.Println>
=> 0x0000000000487399 <+313>:	mov    0x68(%rsp),%rax
   0x000000000048739e <+318>:	mov    (%rax),%rax
   0x00000000004873a1 <+321>:	mov    %rax,0x38(%rsp)
   0x00000000004873a6 <+326>:	mov    %rax,(%rsp)
   0x00000000004873aa <+330>:	callq  0x408950 <runtime.convT64>
   0x00000000004873af <+335>:	mov    0x8(%rsp),%rax
---Type <return> to continue, or q <return> to quit---
 ```
 显示的就是在main包main函数的反汇编。和我们之前用tool工具看到的一样。
 到此为止，interface内部结构和特性介绍完毕
 感谢关注我的公众号
 ![wxgzh.jpg](wxgzh.jpg)
 




