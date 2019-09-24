---
title: Go(05)slice知识介绍和使用
date: 2019-05-08 16:22:50
categories: [golang]
tags: [golang]
---
## golang 的引用类型和内置类型变量
golang 中变量类型分为引用类型和值类型(也叫作内置类型)
### 1.值类型：变量直接存储值，内存通常在栈中分配。

值类型：基本数据类型int、float、bool、string以及数组和struct

### 2.引用类型：变量存储的是一个地址，这个地址存储最终的值。内存通常在 堆上分配。通过GC回收。

引用类型：指针、slice、map、chan等都是引用类型。这类型变量需要通过make构造

## golang中函数传参只有一种方式
golang中函数传递参数，只有值传递一种，也就是实参内容按照值copy方式传递给形参。
当函数的形参变量类型为指针,slice,map,chan等类型时，虽然实参和形参地址不同，但是内部指向了同一个地址，所以可以达到修改指定空间数据的目的。
不要着急，接下来我会写一写小demo帮助大家理解。
<!--more-->
## 数组
先把这段代码写一遍看看结果
``` golang
//数组声明方法
	var bytearray [8]byte //长度为8的数组
	fmt.Println(bytearray)
	var pointarray [4]*float64 //指针数组
	fmt.Println(pointarray)
	var mularray [3][5]int
	fmt.Println(mularray)
	fmt.Printf(" pointarray len is %v\n", len(pointarray))
	//数组遍历
	for i := 0; i < len(pointarray); i++ {
		fmt.Println("Element", i, "of array is", pointarray[i])
	}

	//采用range遍历
	for i, v := range pointarray {
		fmt.Println("Array element [", i, "]=", v)
	}
```
上边提供了数组的声明方式 var 数组名 [数组长度] 元素类型，
同时给出了两种数组遍历方式：
1 len(数组名) 可以获取数组大小，然后遍历
2 采用range遍历，第一个返回值是索引，第二个返回值是对应的内容
int 类型数组初始值为0，指针类型数组初始值为nil
结果如下:
``` bash
[0 0 0 0 0 0 0 0]
[<nil> <nil> <nil> <nil>]
[[0 0 0 0 0] [0 0 0 0 0] [0 0 0 0 0]]
 pointarray len is 4
Element 0 of array is <nil>
Element 1 of array is <nil>
Element 2 of array is <nil>
Element 3 of array is <nil>
Array element [ 0 ]= <nil>
Array element [ 1 ]= <nil>
Array element [ 2 ]= <nil>
Array element [ 3 ]= <nil>
```
前文说过数组是值类型变量，我们写个函数，在函数内部修改形参数组的变量内容，看是否会对实参影响
``` golang
func modify(array [5]int) {
	array[0] = 200
    fmt.Println("In modify(), array values:", array)
}
func main(){
    array := [5]int{1, 2, 3, 4, 5}
	modify(array)
	fmt.Println("In main(), array values:", array)
}

```
结果如下
``` bash
In modify(), array values: [200 2 3 4 5]
In main(), array values: [1 2 3 4 5]
```
说明实参没有被函数修改。那么既然golang传递变量的方式都是值传递，是不是就没办法通过函数修改外部变量了呢？
肯定不是的，可以通过引用类型变量修改，比如指针，slice，map，chan等都可以在函数体内修改，从而影响外部实参的内容。
下面通过slice说明这一点
## slice切片
先看代码
``` golang
array := [5]int{1, 2, 3, 4, 5}
//根据数组生成切片
//切片
	var mySlice []int = array[:3]
	fmt.Println("Elements of array")
	for _, v := range array {
		fmt.Print(v, " ")
	}
	fmt.Println("\nElements of mySlice: ")
	for _, v := range mySlice {
		fmt.Print(v, " ")
    }
    
    //直接创建元素个数为5的数组切片
	mkslice := make([]int, 5)
	fmt.Println("\n", mkslice)
	//创建初始元素个数为5的切片，元素都为0，且预留10个元素存储空间
	mkslice2 := make([]int, 5, 10)
	fmt.Println("\n", mkslice2)
	mkslice3 := []int{1, 2, 3, 4, 5}
	fmt.Println("\n", mkslice3)

	//元素遍历
	for i := 0; i < len(mkslice3); i++ {
		fmt.Println("mkslice3[", i, "] =", mkslice3[i])
	}

	//range 遍历
	for i, v := range mkslice3 {
		fmt.Println("mkslice3[", i, "] =", v)
	}

```
### 生成切片有三种方式
1 通过数组或者切片截取生成新的切片
2 通过make生成 如mkslice := make([]int, 5)
3 直接初始化 如mkslice3 := []int{1, 2, 3, 4, 5}
切片遍历和数组遍历类似，上面结果如下
``` bash
Elements of array
1 2 3 4 5
Elements of mySlice:
1 2 3

 [0 0 0 0 0]

 [0 0 0 0 0]

 [1 2 3 4 5]
mkslice3[ 0 ] = 1
mkslice3[ 1 ] = 2
mkslice3[ 2 ] = 3
mkslice3[ 3 ] = 4
mkslice3[ 4 ] = 5
mkslice3[ 0 ] = 1
mkslice3[ 1 ] = 2
mkslice3[ 2 ] = 3
mkslice3[ 3 ] = 4
mkslice3[ 4 ] = 5
```
### 获取切片大小和容量
``` golang
//获取size和capacity
	mkslice4 := make([]int, 5, 10)
	fmt.Println("len(mkslice4):", len(mkslice4))
	fmt.Println("cap(mkslice4):", cap(mkslice4))
```
获取大小采用len,获取实际开辟的容量用cap
### 切片添加和删除
``` golang
//末尾添加三个元素
	mkslice4 = append(mkslice4, 1, 2, 3)
	fmt.Println("mkslice4 is : ", mkslice4)

	mkslice4 = append(mkslice4, mkslice3...)
	fmt.Println("mkslice4 is : ", mkslice4)
```
采用append 方式可以添加切片数据，但是要注意将append赋值给要存储结果的slice
append有两种用法，第一种是多个参数，第一个参数是slice，后边是要加的多个元素。
第二种是第一个参数为slice，第二个参数为slice展开，slice...表示把slice中元素一个个展开加入。
切片的删除较为麻烦，比如说删除第n个元素，就是截取n-1之前的序列和n之后的序列进行拼接。
``` golang
    mkslice4 := make([]int, 0)
	
	//末尾添加三个元素
	mkslice4 = append(mkslice4, 1, 2, 3)
	fmt.Println("mkslice4 is : ", mkslice4)
	mkslice3 := []int{1, 2, 3, 4, 5}
	mkslice4 = append(mkslice4, mkslice3...)

	fmt.Println("mkslice4 is : ", mkslice4)
	mkslice4 = append(mkslice4[:4-1], mkslice4[4:]...)
	fmt.Println("mkslice4 is : ", mkslice4)
```

### 切片的copy
copy函数提供了切片的深层复制，而赋值操作(=)紧紧是浅拷贝。
看看赋值操作，我们修改slice内部元素数据，其他slice是否会受到影响
``` golang
    oldslice := []int{1, 2, 3, 4, 5}
	newslice := oldslice[:3]
	newslice2 := oldslice
	fmt.Println("newslice is :", newslice)
	fmt.Println("newslice2 is :", newslice2)
	fmt.Printf("newslice addr is : %p \n", &newslice)
	fmt.Printf("newslice2 addr is:  %p \n", &newslice2)
	oldslice[0] = 1024
	fmt.Println("newslice is :", newslice)
	fmt.Println("newslice2 is :", newslice2)
```
输出一下
``` bash
newslice is : [1 2 3]
newslice2 is : [1 2 3 4 5]
newslice addr is : 0xc00005a400
newslice2 addr is:  0xc00005a420
newslice is : [1024 2 3]
newslice2 is : [1024 2 3 4 5]
```
可以看到oldslice修改后，newslice和newslice2都受到影响了，即便他们地址不同。
为什么呢?这要追溯到slice内部实现
``` golang
type Slice struct {
    ptr   unsafe.Pointer        // Array pointer
    len   int               // slice length
    cap     int               // slice capacity
}
```
Slice 内部其实存放了一个指针ptr，这个ptr指向的地址就是存放数据连续空间的首地址，len表示空间当前长度，cap表示空间实际开辟了多大。
如下图
![1.jpg](1.jpg)
那如何深copy元素到另一个slice呢？就是copy函数了
``` golang
    slice1 := []int{1, 2, 3, 4, 5}
	slice2 := []int{5, 4, 3}
	copy(slice2, slice1)
	fmt.Println("after copy.....")
	fmt.Println("slice1: ", slice1)
	fmt.Println("slice2: ", slice2)
	slice2[0] = 1024
	slice2[1] = 999
	slice2[2] = 1099
	fmt.Println("after change element slice2...")
	fmt.Println("slice1: ", slice1)
	fmt.Println("slice2: ", slice2)
	copy(slice1, slice2)
	fmt.Println("after copy.....")
	fmt.Println("slice1: ", slice1)
	fmt.Println("slice2: ", slice2)
```
结果如下
``` bash
after copy.....
slice1:  [1 2 3 4 5]
slice2:  [1 2 3]
after change element slice2..
slice1:  [1 2 3 4 5]
slice2:  [1024 999 1099]
after copy.....
slice1:  [1024 999 1099 4 5]
slice2:  [1024 999 1099]
```
可以看到copy(destslice,srcslice)，当destslice 大小< srcslice时，只拷贝destslice大小的数据。
也就是说copy的大小取决于destslice和srcslice最小值
另外copy后，修改slice2元素，slice1也不会受到影响，是深copy。
感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)
