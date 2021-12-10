---
title: golang面试题汇总(二)
date: 2021-12-06 17:28:45
categories: [golang]
tags: [golang]
---
## 简介
总结一些笔试题
源码地址
[https://gitee.com/secondtonone1/go-interview-questions](https://gitee.com/secondtonone1/go-interview-questions)
## 面试题
#### 1 以下定义包内全局变量，正确的是
``` golang
A. var str string
B. str := ""
C. str = ""
D. var str = ""
```
<!--more-->
答案是AD，定义全局变量要有var 声明，可以写类型也可以不写，系统自动推导，:=实际上是两个表达式不能定义包内全局变量，会报错。
#### 2 指针访问
通过指针变量 p 访问其成员变量 name
``` golang
 p.name 或者(*p).name 我们能常用p.name即可
```
#### 3 接口
一个类只需要实现了接口要求的所有函数，我们就说这个类实现了该接口，实现类的时候，只需要关心自己应该提供哪些方法，不用再纠结接口需要拆得多细才合理，接口由使用方按自身需求来定义，使用方无需关心是否有其他模块定义过类似的接口，类实现接口时，不需要导入接口所在的包，因为go的interface体系降低了接口实现的耦合性。

#### 4 协程
协程和线程都可以实现程序的并发执行，协程比线程更轻量级，协程和线程一样会存在死锁问题，协程可以通过channel进行通信
#### 5 init函数
一个包中可以包含多个init函数，程序编译时，先导入包的init函数，再执行本包内的init函数，main包中也可以有init函数，init函数不可以被其他函数调用，init函数的执行顺序是按照导入的顺序执行的。
#### 6  for循环
go的for循环支持continue和break来控制循环，也提供了更高级的break，可以跳出到指定位置。for循环不支持以逗号为间隔的多个复制语句，必须使用平行赋值的方式来初始化多个变量。
#### 7  变参函数
``` golang
func add(args ...int) int {
	sum := 0
	for _, arg := range args {
		sum += arg
	}

	return sum
}
```
对于add调用可以采取如下方式
``` golang
	add(1, 2, 3)
	add([]int{1, 3, 5}...)
```
#### 8 强制类型转换
go的强制类型转换比较简单
``` golang
type MyStr string
var temp string = "Hello world!"
var newStr MyStr
newStr = MyStr(temp)
log.Println("newStr is ", newStr)
```
#### 9 const变量
const变量的定义，可以不写类型，交由系统自动推导。
``` golang
	const pai float64 = 3.1415926
	const (
		name = "zack"
		age  = 33
	)
```
但是不可以将变量赋值给常量
``` golang
    const newerr = errors.New("new error")
```
#### 10 bool变量
go的bool变量不支持和int做强制转换，也不支持将int类型赋值给bool变量，以下操作为错误的,编译会报错
``` golang
b := bool(1)
b = 1
```
#### 11 switch语句
switch语句可以包含多个case，每个表达式可以是常量，整数，字符串等等，也可以是channel读取等。不同于C++，case中要明确添加fallthrough才能继续执行紧跟的下一个case
#### 12 golang的this
golang中没有隐藏的this指针，方法施加的对象显示传递，没有被隐藏，可以叫this，that等等什么都可以。方法施加的对象不需要非得是指针，也不用非得叫this
#### 13 golang的引用类型
golang的引用类型包括map，slice，chan，interface，凡是底层有指针实现的都是引用类型。
#### 14 golang的指针
golang的指针，可以通过&取指针的地址，可以通过*取指针指向的数据
#### 15 main函数
main函数不能有参数，不能有返回值，必须在main包，可以通过flag包来获取和解析命令行参数
#### 16 关于nil
nil表示空，常用来给指针赋值，所以当用nil给某个指针赋值时一定要指明类型
``` golang
var x1 interface{} = nil
var x2 error = nil
```
#### 17 切片初始化
``` golang
	s := make([]int, 0)
	s = make([]int, 5, 10)
	s = []int{1, 2, 3, 4, 5}
```
#### 18 从切片删除元素
实现一个从slice中删除指定索引元素的函数
``` golang
func RemoveFromSlice(datasrc []int, index int) []int {
	if index > len(datasrc)-1 || index < 0 {
		log.Println("index invalid")
		return nil
	}

	return append(datasrc[:index], datasrc[index+1:]...)
}
```
这么做可以删除指定索引的元素，但是也存在隐患因为append会修改data的数据，举个例子
``` golang
    datasrc := []int{6, 1, 0, 5, 2, 9}
	data := RemoveFromSlice(datasrc, 0)
	log.Println("data is ", data)
	log.Println("datasrc is ", datasrc)
	data = RemoveFromSlice(datasrc, 3)
	log.Println("data is ", data)
```
程序输出
``` golang
 data is  [1 0 5 2 9]
 datasrc is  [1 0 5 2 9 9]
 len data is  6
 data is  [1 0 5 9 9]
```
可以看到删除功能都是正常的，但是经过第一次删除后datasrc被改变了，主要原因是RemoveFromSlice内部调用了append，append会修改参数datasrc的数据，append将datasrc数据由[6,1,0,5,2,9]经过第一次截取后做了拼接，此时append并未返回新的数据地址，因为不存在扩容，所以最后一个元素没变，还是9，所以datasrc数据为[1,0,5,2,9,9],这时再删除索引为3的元素2，data的值就是[1,0,5,9,9],这也是切片的双刃剑，我们用切片做参数一定要考虑修改后是否对外界有影响
如果想获取删除后的slice，并且不影响原slice，可以修改RemoveFromSlice函数如下
``` golang
func RemoveFromSliceCopy(datasrc []int, index int) []int {
	datatmp := make([]int, len(datasrc))
	copy(datatmp, datasrc)
	if index > len(datatmp)-1 || index < 0 {
		log.Println("index invalid")
		return nil
	}

	return append(datatmp[:index], datatmp[index+1:]...)
}
```
再来测试下
``` golang
    datasrc = []int{6, 1, 0, 5, 2, 9}
	data = RemoveFromSliceCopy(datasrc, 0)
	log.Println("data is ", data)
	log.Println("datasrc is ", datasrc)
	data = RemoveFromSlice(datasrc, 3)
	log.Println("data is ", data)
```
输出结果为
``` bash
data is  [1 0 5 2 9]
datasrc is  [6 1 0 5 2 9]
data is  [6 1 0 2 9]
```
通过copy操作实现slice深copy，这样修改copy副本就不会影响源slice了。
#### 19 方法和接口
如果Add函数的调用代码为
``` golang
func main() {
var a Integer = 1
var b Integer = 2
var i interface{} = &a
sum := i.(*Integer).Add(b)
fmt.Println(sum)
}
```
i可以转化为Integer指针并调用Integer方法，则Integer实现了Add方法，当然Integer*实现Add方法也可以。如下两种方式都可以
``` golang
type Integer int

func (a Integer) Add(b Integer) Integer {
	return a + b
}

func (a *Integer) Add(b Integer) Integer {
 	return *a + b
}
```
所以实现一个结构体或其指针的方法后，无论该类型的变量还是指针都可以调用其方法，但是实现结构体指针的方法有一个好处就是可以修改其内部变量。
接口约束，我们能用接口A定义了一个方法B后，凡是实现该方法B的结构体或指针，都可以给A赋值，A作为函数参数，接受实现其方法的结构体对象即可，如果结构体对象未实现该方法，则作为实参传递会报错。看个例子
``` golang
package main
 
import (
	"fmt"
)
 
type Bird interface {
	Fly() string
}
 
type Plane struct {
	Name string
}
 
func (plane *Plane) Fly() string {
	fmt.Println("plane fly")
	return "plane fly"
}
 
func main() {
	plane := Plane{Name: "plane"}
	plane.Fly()
 
	plane_p := &Plane{Name: "plane_p"}
	plane_p.Fly()
}
```
这个例子可以正常输出Fly的打印结果，因为*Plane实现了Fly方法。如果我们写一个函数，参数为Bird，内部做类型转换判断
``` golang
func CheckFly(bird Bird) {
	plane_p, b := bird.(*Plane)
	if b {
		fmt.Println("plane pointer convert success, is ", plane_p)
	}
}
```
如果我们调用
``` golang
CheckFly(&plane)
CheckFly(plane_p)
```
会正常输出
``` bash
plane pointer convert success, is  &{plane}
plane pointer convert success, is  &{plane_p}
```
如果调用
``` golang
CheckFly(plane)
```
会编译报错，因为Plane类没有实现Fly方法，不能传递给Bird接口，这就达到了接口约束的目的。同样修改CheckFly,新增Plane的转换也会报错
``` golang
func CheckFly(bird Bird) {
	plane_p, b := bird.(*Plane)
	if b {
		fmt.Println("plane pointer convert success, is ", plane_p)
	}

	plane, b := bird.(Plane)
	if b {
		fmt.Println("plane conver success, is ", plane)
	}
}
```
上述代码编译报错的原因是Plane没有实现Fly方法。
当然如果我们将Fly方法的调用者改为Plane，就不会报错了
``` golang
func (plane Plane) Fly() string {
	fmt.Println("plane fly")
	return "plane fly"
}
```
实现了Plane类的Fly方法，就隐式实现了指针的Fly方法。所以接口可以转换类类型和类指针类型，编译器不会报错。
关于接口使用和注意事项的文章
[https://llfc.club/category?catid=20RbopkFO8nsJafpgCwwxXoCWAs#!aid/21JhR9PJ3GNWCvTB9OjIHTzKb8C](https://llfc.club/category?catid=20RbopkFO8nsJafpgCwwxXoCWAs#!aid/21JhR9PJ3GNWCvTB9OjIHTzKb8C)

只要两个接口拥有相同的方法列表，即使次序不同，那么他们就是等价的，可以互相赋值。如果接口A的方法列表是接口B的方法列表的子集，那么接口B可以赋值给接口A，接口查询是否成功，需要在运行阶段才能确定，接口赋值可行在编译阶段就可确定

#### 20 vendor

vendor目录被添加到除了GOPATH和GOROOT之外的依赖目录查找的解决方案. go查找依赖包路径的解决方案如下
 1  在当前包下的vendor目录
 2  向上级目录查找，直到找到src下的vendor目录
 3  在GOPATH下面查找依赖包
 4  在GOROOT下查找依赖包
基本思路是将引用的外部包源代码放在当前工程的vendor目录下，go会优先从vendor目录查找依赖包，有了vendor目录后，打包当前的工程代码到其他机器的$GOPATH/src下都可以通过编译

#### 21 channel特性
1 给一个 nil channel发送数据，造成永远阻塞
2 从一个 nil channel接收数据，造成永久阻塞
3 向一个已经关闭的channel写数据，会造成panic
4 从一个已经关闭的channel读数据，如果缓冲区有数据，先读出数据，缓冲区为空，会读出空值
5 无缓冲的channel是同步的，有缓冲的channel是异步的
6 一个只用于读取int数据的单向channel var ch <- chan int
7 一个只用于写int数据的单项channel var ch chan <- int

#### 22 单元测试
go test 可以测试指定代码，该代码文件为*_test.go,文件内的函数如下
``` golang
func TestXXX( t *testing.T )
```
写一个测试代码helloworld_test.go
``` golang
package testinst
import "testing"

func TestHelloWorld(t *testing.T) {
	t.Log("hello world")
	t.Log("test hello world end")
}
```
然后可以通过go test helloworld_test.go 或者go test -v helloworld_test.go查看输出
``` cmd
=== RUN   TestHelloWorld
    helloworld_test.go:6: hello world
    helloworld_test.go:7: test hello world end
--- PASS: TestHelloWorld (0.00s)
PASS
ok      command-line-arguments  0.259s
```
我们新增一个TestAnother函数
``` golang
func TestAnother(t *testing.T) {
	t.Log("another test")
}
```
可以通过go test -v -run TestAnother helloworld_test.go 指定测试这个函数
``` cmd
=== RUN   TestAnother
    helloworld_test.go:11: another test
--- PASS: TestAnother (0.00s)
PASS
ok      command-line-arguments  0.277s
```
也可以Benchmark_开头写函数，用来做性能测试
``` golang
func Benchmark(b *testing.B) {
	b.Log("bench mark test")
	sum := 0
	for i := 0; i < 200; i++ {
		sum += i
	}
}
```
执行 go test -v -bench=. hellowrold_test.go 
-bench=.表示测试helloworld_test.go文件下所有基准测试。
此外还可以指定benchtime，benchmem等参数
``` cmd
=== RUN   TestHelloWorld
    helloworld_test.go:6: hello world
    helloworld_test.go:7: test hello world end
--- PASS: TestHelloWorld (0.00s)
=== RUN   TestAnother
    helloworld_test.go:11: another test
--- PASS: TestAnother (0.00s)
goos: windows
goarch: amd64
cpu: Intel(R) Core(TM) i7-7500U CPU @ 2.70GHz
Benchmark
    helloworld_test.go:15: bench mark test
    helloworld_test.go:15: bench mark test
    helloworld_test.go:15: bench mark test
    helloworld_test.go:15: bench mark test
    helloworld_test.go:15: bench mark test
    helloworld_test.go:15: bench mark test
Benchmark-4     1000000000
PASS
ok      command-line-arguments  0.268s
```
#### 23 闭包捕获range变量
如果用闭包捕获for循环的变量会有什么问题呢
``` golang
func range1() {
	strings := []string{"hello", "world", "zack"}
	for _, str := range strings {
		go func() {
			log.Println("in func str is ", str)
		}()
	}
}

func main() {
	range1()
	time.Sleep(time.Second)
}
```
这段代码会输出 zack zack zack
原因在于str为for循环的变量，通过闭包延长了str的生命周期，闭包捕获的是str的本身引用。当三次循环过后str的值变为zack，所以会输出三个zack，我们可以在关键部位打印str的地址和值，验证一下
``` golang
func range2() {
	strings := []string{"hello", "world", "zack"}
	for _, str := range strings {
		log.Println("str is ", str)
		log.Println("str addr is ", &str)
		go func() {
			log.Println("in func str is ", str)
			log.Println("in func str addr is ", &str)
		}()
	}
}

func main() {
	range2()
	time.Sleep(time.Second)
}
```
程序输出
``` cmd
str is  hello
str addr is  0xc000042240
str is  world
str addr is  0xc000042240
str is  zack
str addr is  0xc000042240
in func str is  zack
in func str addr is  0xc000042240
in func str is  zack
in func str addr is  0xc000042240
in func str is  world
in func str addr is  0xc000042240
```
可以看到每次for循环打印的str值是变化的，但是str地址没变，因为str就是一个变量，而go func()是在三次循环过后才调用，这就导致了输出str的值都为zack。所以我们能只要控制协程及时调用，就能保证str输出不同值。
``` golang
func range3() {
	strings := []string{"hello", "world", "zack"}
	for _, str := range strings {
		log.Println("str is ", str)
		log.Println("str addr is ", &str)
		go func() {
			log.Println("in func str is ", str)
			log.Println("in func str addr is ", &str)
		}()
		time.Sleep(time.Second)
	}
}

func main() {
	range3()
	time.Sleep(time.Second)
}
```
输出
``` cmd
str is  hello
str addr is  0xc000042240
in func str is  hello
in func str addr is  0xc000042240
str is  world
str addr is  0xc000042240
in func str is  world
in func str addr is  0xc000042240
str is  zack
str addr is  0xc000042240
in func str is  zack
in func str addr is  0xc000042240
```
通过sleep，保证了每次循环及时调用go func()这样，输出的str和每次遍历的记过一样。
所以从这个例子我们知道，不要用协程或者闭包捕获for循环的变量，因为for循环的变量值会不断变化，而协程很难在准确的时机获取其值，造成逻辑错误。
#### 24 defer链式调用
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

#### 25 字符串
字符串取索引获取的是字节的asc码值，通过切片截取获取的是字串
``` golang
    str := "Hello"
	log.Println(str[0])
    log.Println(str[0:1])
```
上述代码分别打印出72和H，如果对str[0]赋值，会编译报错
``` golang
str[0] = 'M'
```
编译报错
``` cmd
 cannot assign to str[0] (strings are immutable)
```
#### 26 recover和defer
defer 函数中可以使用recover捕获异常，recover要写在defer中，panic可以由本层或者上一层捕获。
具体原理和注意事项请参考
[defer注意事项](https://llfc.club/category?catid=20RbopkFO8nsJafpgCwwxXoCWAs#!aid/21zj2pliWS2qQQ6orF7agEO3oke)

#### 27 range注意事项
前面的例子说过defer 或者函数捕获 range 变量是捕获的引用，会导致问题，这个例子同样会出现问题，因为map存储的val为stu的指针
``` golang
func pase_student() {
	m := make(map[string]*student)
	stus := []student{
		{Name: "zhou", Age: 24},
		{Name: "li", Age: 23},
		{Name: "wang", Age: 22},
	}
	for _, stu := range stus {
		m[stu.Name] = &stu
	}

	for key, val := range m {
		log.Println("name is ", key)
		log.Println("age is ", val)
	}
}
```
程序输出
``` golang
name is  zhou
age is  &{wang 22}
name is  li
age is  &{wang 22}
name is  wang
&{wang 22}
```
因为遍历结束后stu变为{wang 22}，所以map的val为&{wang 22}，只需要用map存储stu的值就不会出现问题了
``` golang
func pase_student2() {
	m := make(map[string]student)
	stus := []student{
		{Name: "zhou", Age: 24},
		{Name: "li", Age: 23},
		{Name: "wang", Age: 22},
	}
	for _, stu := range stus {
		m[stu.Name] = stu
	}

	for key, val := range m {
		log.Println("name is ", key)
		log.Println("age is ", val)
	}
}
```
#### 28协程随机性和闭包
``` golang
func clouser() {
	runtime.GOMAXPROCS(1)
	wg := sync.WaitGroup{}
	wg.Add(20)
	for i := 0; i < 10; i++ {
		go func() {
			fmt.Println("A: ", i)
			wg.Done()
		}()
	}
	for i := 0; i < 10; i++ {
		go func(i int) {
			fmt.Println("B: ", i)
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```
上述程序输出
``` cmd
B:  9
A:  10
A:  10
A:  10
A:  10
A:  10
A:  10
A:  10
A:  10
A:  10
A:  10
B:  0
B:  1
B:  2
B:  3
B:  4
B:  5
B:  6
B:  7
B:  8
```
因为多个协程执行是随机的，所以输出A和B也是随机的，B的输出值是0~9, A的输出值为10，因为第一个循环，func捕获的是i的引用。
#### 29 组合和继承
``` golang
type People struct{}

func (p *People) ShowA() {
	fmt.Println("showA")
	p.ShowB()
}
func (p *People) ShowB() {
	fmt.Println("showB")
}

type Teacher struct {
	People
}

func (t *Teacher) ShowB() {
	fmt.Println("teachershowB")
}
```
main函数调用
``` golang
func main() {
	t := Teacher{}
	t.ShowA()
	t.ShowB()
	t.People.ShowB()
}
```
输出
``` cmd
showA
showB
teachershowB
showB
```
因为匿名组合实现了继承，所以Teacher类有了People的方法ShowA，所以t.ShowA调用的是People类的ShowA
#### 30 select随机性
``` golang
func maypanic() {
	runtime.GOMAXPROCS(1)
	int_chan := make(chan int, 1)
	string_chan := make(chan string, 1)
	int_chan <- 1
	string_chan <- "hello"
	select {
	case value := <-int_chan:
		fmt.Println(value)
	case value := <-string_chan:
		panic(value)
	}
}
```
由于select由随机性，所以上述代码有可能会panic
#### 31 defer 调用顺序和链式求值
``` golang
func calc(index string, a, b int) int {
	ret := a + b
	fmt.Println(index, a, b, ret)
	return ret
}
    a := 1
	b := 2
	defer calc("1", a, calc("10", a, b))
	defer calc("2", a, calc("20", a, b))
```
上述代码输出如下
``` bash
10 1 2 3
20 1 2 3
2 1 3 4
1 1 3 4
```
具体原理可参考[defer原理](https://llfc.club/category?catid=20RbopkFO8nsJafpgCwwxXoCWAs#!aid/21zj2pliWS2qQQ6orF7agEO3oke)

#### 32 make初始化长度
``` golang
func makeslice() {
	s := make([]int, 5)
	s = append(s, 1, 2, 3)
	fmt.Println(s)
}
```
程序输出
``` cmd
[0 0 0 0 0 1 2 3]
```
因为make初始化slice时指定长度为5，就会默认为slice初始化5个0
#### 33 lock锁
``` golang
func (ua *UserAges) Add(name string, age int) {
	ua.Lock()
	defer ua.Unlock()
	ua.ages[name] = age
}
func (ua *UserAges) Get(name string) int {
	if age, ok := ua.ages[name]; ok {
		return age
	}
	return -1
}
```
上述这段代码并不会引发死锁，但是会因为读操作未加锁导致读出来的数据不准确。
#### 34 chan阻塞
``` golang
func Iter(set *sync.Map) <-chan interface{} {
	ch := make(chan interface{})
	go func() {
		set.Range(func(k, v interface{}) bool {
			fmt.Println("key is ", k)
			fmt.Println("value is ", v)
			ch <- v
			return true
		})
		close(ch)
	}()
	return ch
}
```
上述代码会有什么问题呢？我们看看调用
``` golang
func main() {
	safe_map := sync.Map{}
	safe_map.Store("zack", 1024)
	safe_map.Store("bob", 2048)
	safe_map.Store("vivo", 9096)
	ch := Iter(&safe_map)
	<-ch
}
```
程序输出
``` bash
key is  zack
value is  1024
key is  bob
value is  2048
```
没有vivo的输出，主要原因是ch是无缓冲的，造成了协程写阻塞，而主协程退出后，子协程因为占有ch而无法被回收，造成资源泄露

#### 35 接口内部实现
``` golang
type People interface {
	Show()
}
type Student struct{}

func (stu *Student) Show() {
}

func live() People {
	var stu *Student
	return stu
}

func Self() *Student {
	var stu *Student
	return stu
}

type Empty interface {
}

func empty() Empty {
	var stu *Student
	return stu
}

func main(){
    if live() == nil {
		fmt.Println("AAAAAAA")
	} else {
		fmt.Println("BBBBBBB")
	}

	if Self() == nil {
		fmt.Println("AAAAAAA")
	} else {
		fmt.Println("BBBBBBB")
	}

	if empty() == nil {
		fmt.Println("AAAAAAA")
	} else {
		fmt.Println("BBBBBBB")
	}
}
```
上述代码输出
``` bash
BBBBBBB
AAAAAAA
BBBBBBB
```
*Student对象stu为空指针，赋值给接口，接口不是nil，因为接口要保存type，data，以及方法集等信息。接口具体结构请参考
[接口结构](https://llfc.club/category?catid=20RbopkFO8nsJafpgCwwxXoCWAs#!aid/21Lh5Wc9KHRclSzHXcMQ0ss5mQo)

#### 36 switch type
``` golang
package main
func GetValue() int {
	return 1
}
func main() {
	i := GetValue()
	switch i.(type) {
	case int:
		println("int")
	case string:
		println("string")
	case interface{}:
		println("interface")
	default:
		println("unknown")
	}
}
```
上述代码编译失败，会提示i不是interface类型
#### 37 函数返回值参数
``` golang
func funcMui(x,y int)(sum int,error){    
	return x+y,nil
}
```
上述代码会编译失败，因为返回值要么全带参数，要么全不带

#### 38 defer和返回值参数
``` golang
func DeferFunc1(i int) (t int) {
	t = i
	defer func() {
		t += 3
	}()
	return t
}
func DeferFunc2(i int) int {
	t := i
	defer func() {
		t += 3
	}()
	return t
}
func DeferFunc3(i int) (t int) {
	defer func() {
		t += i
	}()
	return 2
}

func main() {
	println(DeferFunc1(1))
	println(DeferFunc2(1))
	println(DeferFunc3(1))
}
```
程序输出 4, 1, 3。 defer的可以捕获返回值，并通过计算修改返回值。
#### 39 结构体比较
结构体比较时 
1  如果两个结构体内部成员类型不同，则不能比较
2  如果两个结构体内部成员类型相同，但是顺序不同，则不能比较
3  如果两个结构体内不含有无法比较类型，则无法比较
4  如果两个结构体类型，顺序相同，且不含有无法比较类型(slice, map),则可以进行比较
``` golang
sn1 := struct {
    age  int
    name string
}{age: 11, name: "qq"}
sn2 := struct {
    age  int
    name string
}{age: 11, name: "qq"}
if sn1 == sn2 {
    fmt.Println("sn1== sn2")
}
```
上面的代码会输出sn1 ==  sn2
如果新增sn3，与sn1比较
``` golang
sn1 := struct {
    age  int
    name string
}{age: 11, name: "qq"}
sn3 := struct {
    name string
    age  int
}{age: 11, name: "qq"}
 
//顺序不同无法比较
if sn1 == sn3 {
    fmt.Println("sn1== sn3")
}
```
则编译器汇编报错，提示miss matched struct，因为两个结构体类型相同但是成员的顺序不同，也无法比较
我们新增sn4，让两个结构体的内容不同
``` golang
sn3 := struct {
    name string
    age  int
}{age: 11, name: "qq"}
sn4 := struct {
    sex int
    age int
}{sex: 1, age: 23}
 
//类型不同无法比较
if sn4 == sn3 {
    fmt.Println("sn4== sn3")
}
```
编译器也会编译报错，提示两个结构体成员类型不同，无法比较
如果我们在结构体内部包含map类型
``` golang
sm1 := struct {
    age int
    m   map[string]string
}{age: 11, m: map[string]string{"a": "1"}}
sm2 := struct {
    age int
    m   map[string]string
}{age: 11, m: map[string]string{"a": "1"}}
//sm1 和 sm2 无法比较， 因为结构体内有无法比较的类型
if sm1 == sm2 {
    fmt.Println("sm1== sm2")
}
```
编译器仍然会报错，提示编译错误，无法比较两个结构体，因为有无法比较的类型map
``` golang
sps1 := struct {
    age  int
    jobs []string
}{age: 11, jobs: nil}
 
sps2 := struct {
     age  int
     jobs []string
}{age: 11, jobs: nil}
 
if sps1 == sps2 {
    fmt.Println("sps1 == sps2")
}
```
上述代码编译器仍然会报错，因为结构体内包含无法比较的类型slice
对于chan，指针，以及interface这些类型都可以比较， 如下代码都可以实现比较
``` golang
spch1 := struct {
	age int
	ch  chan int
}{age: 11, ch: nil}
 
spch2 := struct {
	age int
	ch  chan int
}{age: 11, ch: nil}
 
if spch1 == spch2 {
	fmt.Println("spch1 == spch2")
}
 
spi1 := struct {
	age int
	it  interface{}
}{age: 11, it: nil}
 
spi2 := struct {
	age int
	it  interface{}
}{age: 11, it: nil}
 
if spi1 == spi2 {
	fmt.Println("spi1 == spi2")
}

a := 1
b := 2
sp1 := struct {
    age  int
    data *int
}{age: 11, data: &a}
sp2 := struct {
     age  int
     data *int
}{age: 11, data: &b}
 
if sp1 == sp2 {
   fmt.Println("sp1== sp2")
}

```
#### 40 iota
iota是一个特殊常量，可以认为是一个可以被编译器修改的常量。
iota 在const关键字出现时将被重置为0，const中每新增一行常量声明将使 iota 计数加1，因此iota可作为const 语句块中的行索引。
``` golang
const (
	a = iota
	b
	c
	d
	e = "Hello World"
	f
	g = iota
)

func main() {
	fmt.Println(a)
	fmt.Println(b)
	fmt.Println(c)
	fmt.Println(d)
	fmt.Println(e)
	fmt.Println(f)
	fmt.Println(g)
}
```
程序输出
``` cmd
0
1
2
3
Hello World
Hello World
6
```
#### 41 alias
``` golang
func main() {
	type MyInt1 int
	type MyInt2 = int
	var i int = 9
	var i1 MyInt1 = i
	var i2 MyInt2 = i
	fmt.Println(i1, i2)
}
```
上述代码会报错，不能将i赋值给i1，因为i1为MyInt1类型。可以将i赋值给i2，因为alias声明MyInt2类型为int类型的别名。
#### 42 struct alias
``` golang
type User struct {
}
type MyUser1 User
type MyUser2 = User

func (i MyUser1) m1() {
	fmt.Println("MyUser1.m1")
}
func (i User) m2() {
	fmt.Println("User.m2")
}

func main(){
    var iu1 MyUser1
	var iu2 MyUser2
	iu1.m1()
	iu2.m2()
}
```
输出
``` cmd
MyUser1.m1
User.m2
```
#### 43 方法重名
``` golang
type T1 struct {
}

func (t T1) m1() {
	fmt.Println("T1.m1")
}

type T2 = T1
type MyStruct struct {
	T1
	T2
}

func main() {
    my:=MyStruct{}
    my.m1()
}
```
上述代码会编译报错 ambiguous selector my.m1，无论type alias还是type define 都有可能存在类组合后的方法重名情况，那么外部调用就要指明方法所属类名，改为如下调用就没有问题了。
``` golang
my.T1.m1()
my.T2.m1()
```
#### 44 变量作用域
``` golang
var ErrDidNotWork = errors.New("did not work")

func DoTheThing(reallyDoIt bool) (err error) {
	if reallyDoIt {
		result, err := tryTheThing()
		if err != nil || result != "it worked" {
			err = ErrDidNotWork
		}
	}
	return err
}
func tryTheThing() (string, error) {
	return "", ErrDidNotWork
}
func main() {
	fmt.Println(DoTheThing(true))
	fmt.Println(DoTheThing(false))
}
```
程序输出
``` cmd
nil
nil
```
因为err = ErrDidNotWork, 这个err为作用域内部的err，而返回的是外层err

#### 45 闭包延迟求值
``` golang
func test() []func() {
	var funs []func()
	for i := 0; i < 2; i++ {
		funs = append(funs, func() {
			println(&i, i)
		})
	}
	return funs
}

funs := test()
for _, f := range funs {
	f()
}
```
闭包捕获了i的最后值为2，所以结果为
``` cmd
0xc00000c0a8 2
0xc00000c0a8 2
```
如果想要打印不同的地址和变量值，可以用一个临时变量x存储i的值，闭包捕获x延长x的生命周期就可以了
``` golang
func test2() []func() {
	var funs []func()
	for i := 0; i < 2; i++ {
		x := i
		funs = append(funs, func() {
			println(&x, x)
		})
	}
	return funs
}

funs = test2()
	for _, f := range funs {
		f()
}
```
#### 46 闭包引用相同变量
``` golang
func test3(x int) (func(), func()) {
	return func() {
			println(x)
			x += 10
		}, func() {
			println(x)
		}
}

a, b := test3(100)
a()
b()
```
程序输出100， 110
#### 47 多个panic
``` golang
func panic1() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println(err)
		} else {
			fmt.Println("fatal")
		}
	}()
	defer func() {
		panic("deferpanic")
	}()
	panic("panic")
}
func main(){
    panic1()
}
```
当程序有多个panic，只有最后一个panic可以被捕获并恢复，程序第一次panic后会执行defer逻辑，defer又触发panic，所以程序输出deferpanic
#### 48 协程控制
目前有一个加载类接口
``` golang
type LoaderInter interface{
	Load()
}
```
要求实现一个类Loader，
1  包含Load方法，Load方法可以模拟实现一些超时或者阻塞操作。
2  另外实现一个Check函数，其参数为LoadInter类型,返回值为error类型，要求内部调用Load方法，
3  如果Load超时则Check函数返回超时错误，否则Check函数返回nil，超时时间设置为5s
4  用代码实现该需求，请勿修改接口和函数参数以及返回值
Check方法形式如下
``` golang
func Check(li LoaderInter) error {
    //...
	return nil
}
```
下面用代码实现上述需求
``` golang
type LoaderInter interface {
	Load()
}

func Check(li LoaderInter) error {
	ch := make(chan struct{})
	go func() {
		li.Load()
		ch <- struct{}{}
	}()
	select {
	case <-ch:
		return nil
	case <-time.After(time.Second * 5):
		return errors.New("load time out error")
	}
}

type Loader struct {
}

func (ld Loader) Load() {
	fmt.Println("Loader begin load data....")
	<-time.After(time.Second * 10)
	fmt.Println("Loader load data success...")
}

func main() {
	ld := &Loader{}
	err := Check(ld)
	if err != nil {
		fmt.Println("err is ", err)
	}

	time.Sleep(time.Second * 11)
}
```
程序输出
``` bash
Loader begin load data....
err is  load time out error
Loader load data success...
```
#### 49 协程同步
目前有一个加载类接口
``` golang
type LoaderInter interface{
	Load()
}
```
同时有一个ProducerInter接口, 返回LoaderInter，如果返回nil表明ProducerInter已经生产完毕
``` golang
type ProducerInter interface {
    Produce() LoaderInter
}
```
现要求实现一个Create函数,函数参数如下
``` golang
func Create(producer ProducerInter) {
    //...
}
```
要求
1 循环调用Produce函数产生多个LoaderInter，直到返回nil表明无需生产
2 每个LoaderInter调用自己的Load操作，Load操作为耗时或者阻塞io操作
3 保证最多同时并发运行五个LoadInter的Load操作

现在实现上述需求
``` golang
type Loader struct {
}

func (ld Loader) Load() {
	fmt.Println("Loader begin load data....")
	<-time.After(time.Second * 10)
	fmt.Println("Loader load data success...")
}

type ProducerInter interface {
	Produce() LoaderInter
}

type Producer struct {
	max int //最大产量
}

func (p *Producer) Produce() LoaderInter {
	if p.max <= 0 {
		return nil
	}

	ld := &Loader{}
	p.max--
	return ld
}

func Create(producer ProducerInter) {
	maxch := make(chan struct{}, 5)
	for {
		select {
		case maxch <- struct{}{}:
			ld := producer.Produce()
			if ld == nil {
				fmt.Println("producer end ,max ....")
				return
			}

			go func() {
				ld.Load()
				<-maxch
			}()
		}

	}
}
```
在main函数中调用
``` golang
func main() {
	p := Producer{max: 10}
	Create(&p)
}
```
我们初始化Producer最多产出10个loader，程序输出如下
``` bash
Loader begin load data....
Loader begin load data....
Loader begin load data....
Loader begin load data....
Loader begin load data....
Loader load data success...
Loader begin load data....
Loader load data success...
Loader begin load data....
Loader load data success...
Loader begin load data....
Loader load data success...
Loader begin load data....
Loader load data success...
Loader begin load data....
Loader load data success...
producer end ,max ....
```




