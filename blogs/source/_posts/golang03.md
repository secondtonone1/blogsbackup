---
title: Go(03)golang 基本变量和demo
date: 2019-04-27 20:08:37
categories: [golang]
tags: [golang]
---
本文介绍基本的golang变量和简单的demo，该系列博文由浅入深，带领大家初入golang。
## golang 基本类型
``` golang
var v1 int  //整形
var v2 string //字符串
var v3 [10]int  //数组
var v4 []int   //切片
var v5 struct{   //结构体
    f int
}
var v6 * int  //指针
var v7 map[string]int  //map，key为string,value为int
var v8 func(a int) int  //函数对象
var(
    v9 int
    v10 string
)
```
如注释所说，上面就是以后会用到的基本类型，切片可以理解为C++的vector，python的list。
可以用括号把两个变量括起来一起定义。
<!--more-->
## 一个简单的golang程序
``` golang
package main
import "fmt"

func main()
{
    //整形
    var v1 int
    //字符串  
    var v2 string 
    //变量赋值
    v1 = 10
    v2 := "hello"
    fmt.Println("v2:",v2)
    //变量初始化
    v11  := "day02"
    fmt.Println("v11:",v11)
    //第二种初始化方式
    var v12 int = 13

    //变量交换
    v1, v12 = v12, v1
    fmt.Printf("v11:%v, v12:%v\n",v1,v12)
    _, _, nickName := GetName()
    fmt.Printf("nickName : %v\n", nickName)
    //常量
    const Pi float64 = 3.141592653
    const zero = 0.0
    const (
        size int64 = 1024
        eof = -1
    )
    const u, v float32 = 0,3
    const a, b, c = 3, 4, "foo"
}
```
下面说下各行含义
package main 表示该代码属于main包，其他文件引用该文件函数或变量需要引入包名main
import "fmt" 引入fmt包，这样就可以用fmt包里的打印函数fmt.Println了。
var v1 int 定义了一个int类型的v1变量
v1 = 10 将v1赋值为10，同样v2:="hello"也是将v2赋值为"hello"
fmt.Println是fmt包定义的打印函数，这里用来打印变量
v11 := "day02"，由于没有定义v11，所以:=直接定义并初始化v11
var v12 int = 13 也是初始化的方式，定义并初始化了v12
golang允许交换两个变量，不需要写额外的变量缓存转换，v1,v12 = v12,v1
## 常量
定义变量用var,定义常量用const，同样可以将常量放在()里一起定义或初始化。
golang里有iota变量，第一次为0，以后每次出现加1，如下所示
``` golang
//iota 表示初始化常量为0，之后每次出现iota，iota自增1
    const ( // iota被重设为0
        c0 = iota // c0 == 0
        c1 = iota // c1 == 1
        c2 = iota // c2 == 2
        )
    
    const (
        a1 = 1 << iota  //1左移0位
        b2 = 1 << iota  //1左移1位
        c3 = 1 << iota  //1左移2位
    )
    fmt.Printf("a1 : %v, b2 : %v, c3 : %v\n", a1, b2, c3)

    const (
        Sunday = iota
        Monday
        Tuesday
        Wednesday
        Thursday
        Friday
        Saturday
        numberOfDays // 这个常量没有导出
        )
        //同Go语言的其他符号（symbol）一样，以大写字母开头的常量在包外可见

```
## bool类型
``` golang
    //bool 类型
    var bvar bool 
    bvar = true
    bvar2 := (1==2) //bvar 被推导为bool类型
    fmt.Printf("bvar: %v, bvar2: %v\n",bvar, bvar2)
```
## 字符串
``` golang
 //字符串
    var str string
    str = "Hello world"
    ch := str[0]
    fmt.Printf("The length of \"%s\" is %d \n ", str, len(str))
    fmt.Printf("The first character of \"%s\" is %c.\n ",str, ch)
    str2 := ",I'm mooncoder"
    fmt.Println(str+str2)

    hellos := "Hello,我是中国人"
    //字符串遍历,utf-8编码遍历,每个字符byte类型，8字节
    for i:=0; i < len(hellos); i++{
        fmt.Printf("%c", hellos[i])
   }
   //unicode 遍历,每个字符rune类型，变长
   for i, ch := range hellos{
       fmt.Printf("%c\t",ch)
       fmt.Printf("%d\n",i)
   }
```
上面的例子可以自己写一遍，提供了两种遍历方式，第一种是根据hellos字符串长度遍历，每次取字节中的内容打印，由于有汉字所以会乱码。
采用range遍历，实际上是按照unicode变长遍历，打印出每个变长字节内容，不会乱码。
今天介绍到此为止，下一期介绍slice,map等复杂类型。
[源码下载地址](https://github.com/secondtonone1/golang-)
感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)



