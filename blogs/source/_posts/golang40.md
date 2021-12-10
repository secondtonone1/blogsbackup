---
title: golang面试题汇总(一)
date: 2021-12-03 16:26:44
categories: [golang]
tags: [golang]
---
## 简介
陆续总结一些面试常常会问到的问题，对知识体系做一个梳理
源码地址[https://gitee.com/secondtonone1/go-interview-questions](https://gitee.com/secondtonone1/go-interview-questions)
<!--more-->
## 面试题
### 1 简述go协程调度原理
go协程调度模型图
[https://llfc.club/articlepage?id=21XXEPY1IoqquP7phaftDjf9Rt4](https://llfc.club/articlepage?id=21XXEPY1IoqquP7phaftDjf9Rt4)
简述原理:
go的协程是通过MPG模型调用的，M为内核态线程,G为用户态协程,P为处理器,系统会通过调度器从全局队列找到G分配给空闲的M，P会选择一个M来运行，M和G的数量不等，P会有一个本地队列表示M未处理的G,M本身有一个正在处理的G，M每次处理完一个G就从本地队列里取一个G，并且更新P的schedtick字段，如果本地队列没有G，则从全局队列一次性取走G/P个数的G，如果全局队列里也没有，就从其他的P的本地队列取走一半。go1.2之前goroutine是轮询式调度，之后改为抢占式，弱抢占式。
弱抢占式:
如果某个P的schedtick一直没有递增，说明这个P一直在执行一个G任务，如果超过一定时间就会为G增加标记，并且该G执行非内联函数时中断自己并把自己加到队尾。
### 2 struct能否比较
结构体比较时 
1  如果两个结构体内部成员类型不同，则不能比较
2  如果两个结构体内部成员类型相同，但是顺序不同，则不能比较
3  如果两个结构体内不含有无法比较类型，则无法比较
4  如果两个结构体类型，顺序相同，且不含有无法比较类型(slice, map, )
### 3 defer关键字
一个函数定义了多个defer函数，defer的调用顺序和栈一样，先进后出，最先调用的是最后写的defer。函数将返回值入栈，然后执行析构，在析构之前要执行defer的操作。defer使用的注意事项
#### 3.1 defer常用来释放变量
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
#### 3.2 defer被声明时，其参数是实时解析和捕获
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
#### 3.3 defer调用顺序为栈式调用
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
#### 3.4 defer可以捕获函数返回值
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
### 4 select有什么作用
select是用来控制并发访问的技术，当有多个逻辑要监控时，可以写入select的不同的case，同时也能捕获外界通知，也可以控制协程退出和并发等。select的case表达式必须为一个channel类型，所有case都会被求值，自上而下，从左而又。
如果有多个case满足条件，则随机选择一个执行，如果所有case都不满足条件，且实现了defautl逻辑，则走入default逻辑。如果没有实现default逻辑，则阻塞等待，知道某个case条件满足为止。
break可以跳出select执行，所以要注意配合for循环的select，如果想要跳出循环，请使用return或者break+标签的方式跳转到某个位置。
我们实现一个逻辑，不断的从channel中读取数据
1  要求监听外界退出的通知，收到通知后退出并返回
2  要求从channel中读取数据，超过5s未读出则打印超时,然后继续等待读取,这期间有退出信号要及时退出
3  保证channel中有数据则优先读取
``` golang
func NoneBlockRead(ctx context.Context, chread chan interface{}, wt *sync.WaitGroup) {
	for {
		select {
		case <-ctx.Done():
			wt.Done()
			log.Println("receive main exit signal")
			return
		case data := <-chread:
			log.Println("receive ", data)
		default:
			select {
			case <-time.After(time.Second * 5):
				log.Println("time out")
			case data := <-chread:
				log.Println("receive ", data)
			case <-ctx.Done():
				log.Println("receive main exit signal")
				wt.Done()
				return
			}
		}
	}
}
```
NoneBlockRead优先捕获主线程的ctx的退出信号和chread读取数据，如果上述条件都不满足，则进入default逻辑启动定时器检测超时，并监控退出信号和chread的数据读取。
我们实现主函数
``` golang
func main(){
    var wg = &sync.WaitGroup{}
	chread := make(chan interface{})
	wg.Add(1)
	ctx, _ := context.WithTimeout(context.TODO(), time.Second*10)
	go NoneBlockRead(ctx, chread, wg)

	select {
	case <-time.After(time.Second * 7):
		chread <- 1
	}
	wg.Wait()
}
```
主函数中设置定时器7秒后才像chread写入数据，所以NoneBlockRead会先打印超时，然后读出数据，接着等待3秒后自动退出，因为ctx十秒会触发cancel逻辑。
程序输出
``` cmd
2021/12/06 11:28:03 time out
2021/12/06 11:28:05 receive  1
2021/12/06 11:28:08 receive main exit signal
```
### 5 context的作用
context在多个goroutine中运行是安全的，用于信息传递，协程管理等。我们再实现一个从channel读取数据的函数
``` golang
func NoneBlockRead2(ctx context.Context, chread chan interface{}, wt *sync.WaitGroup) {
	defer func() {
		wt.Done()
	}()
	select {
	case <-ctx.Done():
		log.Println("receive main exit signal")
		return
	case data := <-chread:
		log.Println("receive data is ", data)
		return
	}
}
```
这个函数就是通过传递一个ctx达到控制协程的目的
``` golang
func main(){
    var wg = &sync.WaitGroup{}
	chread := make(chan interface{})
	wg.Add(1)
	ctx2, _ := context.WithTimeout(context.TODO(), time.Second*3)
	go NoneBlockRead2(ctx2, chread, wg)
	wg.Wait()
}
```
### 6 主协程如何等待其余协程退出
有很多种方式，为每个协程传递一个channel，当每个协程退出时向该channel写入数据或者关闭该channel，主协程读取channel捕获退出信号。或者采用waitgroup也可以。
``` golang
func main(){
    var wg = &sync.WaitGroup{}
    for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			wg.Done()
		}()
	}

	wg.Wait()
}
```
或者
``` golang
func main(){
    waitch := make(chan int, 5)
	for i := 0; i < 5; i++ {
		go func() {
			waitch <- 1
		}()
	}

	for i := 0; i < 5; i++ {
		<-waitch
	}
}
```
### 7 slice的扩容原理
slice底层的结构为一个结构体，包含data数据域和实际的长度len，以及容量cap,每次append都要判断当前长度是否达到len，如果达到就将cap扩充为len的2倍，然后移动数据到新的内存空间，所以append会返回新的slice，内部包含了数组的地址，使用slice要用make初始化
``` golang
a := make([]int, 1,2)
```
开辟一个slice，长度len为1，cap为2
### 8 go的map是有序的吗？
go的map底层是hash table，也就是hmap类型，对key值进行hash运算，将低八位用来确定value存在那个bucket中，高八位与bucket的tophash进行比较，确定key是否存在。出现hash碰撞后，会将bucket的overflow指向一个新的bucket，形成一个单向链表
map排序可以通过取出key值构造slice再排序
``` golang
	data_m := make(map[int]int)
	data_m[1] = 111
	data_m[2] = 222
	data_m[3] = 333
	keys := make([]int, 0, len(data_m))
	for k := range data_m {
		keys = append(keys, k)
	}

	sort.Slice(keys, func(i, j int) bool {
		return keys[i] < keys[j]
	})

	for _, val := range keys {
		log.Println(data_m[val])
	}
```
### 9 实现set
利用map，建立value为ture，key为对应键值即可。要实现安全的set，可以用sync.map
``` golang
    data_set := make(map[int]bool)
	data_set[1] = true
	data_set[2] = true
	if val, ok := data_set[3]; !ok {
		log.Println("key not exist")
		return
	} else {
		log.Println("val is ", val)
	}
```
### 如何实现生产者消费者
用sync.map存储topic，主线程用waitGroup等待退出，生产者和消费者之间分别用两个chan通信，详见
[https://llfc.club/category?catid=20RbopkFO8nsJafpgCwwxXoCWAs#!aid/21Xe63IHvSfz8AEOulnUP4wrHZs](https://llfc.club/category?catid=20RbopkFO8nsJafpgCwwxXoCWAs#!aid/21Xe63IHvSfz8AEOulnUP4wrHZs)
### 10 GC原理
go采用三色标记法回收内存，程序开始创建的对象全部为白色，gc扫描后将可到达的对象标记为灰色，再从灰色对象中找到其引用的其他对象，将其标记为灰色，将自身标记为黑色，重复上述步骤，直到找不到灰色对象为止。最后对所有白色对象清除。gc采用标记清除(mark and sweep)算法的STW(STOP THE WORLD)的操作，标记阶段runtime把所有线程全部冻结掉，所有线程全部冻结意味着用户逻辑也是暂停的，这样所有对象都不会被修改，在清除阶段用户逻辑和清除操作时可并行的，因为白色对象意味着用户不再使用。go引入写屏障机制，在写操作之间和之后内存的修改被系统感知，然后重新标记，这时也会有短暂的stw，所以新生成的对象一律都是灰色的。如果一个黑色对象引用了曾经标记的白色对象，则写屏蔽机制被触发，向gc发送信号，gc重新扫描，将其着色为灰色。
### 11 go的堆栈原理
栈: 由操作系统自动分配和释放，存放在函数的参数值，局部变量的值等。
堆: 程序员自己分配并释放，如果不释放，可能由os回收，也可能不会(C++需要手动释放)，go对于堆空间的回收有垃圾回收算法，不一定是成为孤儿对象就立即回收，所以堆的回收较栈缓慢，栈则是从存储空间开辟，调用完立即释放。
go 的内存分配遵循内存分配逃逸原则，所谓逃逸分析(escape analysis)是指由编译器决定内存分配的位置，不需要程序员指定。在函数中申请一个对象，如果分配在栈中，则函数结束自动回收，如果分配在堆中，则gc回收，以下场景会造成内存逃逸
1  申请并开辟指针对象，指针对象在堆空间
2  闭包，内部函数引用了外部函数的变量，则变量变为堆开辟
3  栈空间不足，在堆上开辟
逃逸分析减低gc压力，go开发不见得使用指针就是更高效，指针为四字节作为数据传递可以减少开销，但是回收要由gc管理，所以在传递频繁的场景尽量使用指针。
