---
title: golang channel详解和协程优雅退出
date: 2019-11-08 15:57:04
categories: [golang]
tags: [golang]
---
## 非缓冲chan，读写对称
非缓冲channel,要求一端读取，一端写入。channel大小为零，所以读写操作一定要匹配。
``` golang
func main() {
	nochan := make(chan int)
	go func(ch chan int) {
		data := <-ch
		fmt.Println("receive data ", data)
	}(nochan)
	nochan <- 5
	fmt.Println("send data ", 5)
}
```
我们启动了一个协程从channel中读取数据，在主协程中写入，程序的运行流程是主协程优先启动，运行到nochan<-5写入是阻塞，然后启动协程读取，从而完成协程间通信。
程序输出
``` cmd
receive data  5
send data  5
```
<!--more-->
如果将启动协程的代码放在nochan<-5下边，这样会造成主协程阻塞，无法启动协程，一直挂起。
``` golang
func main() {
	nochan := make(chan int)
	nochan <- 5
	fmt.Println("send data ", 5)
	go func(ch chan int) {
			data := <-ch
			fmt.Println("receive data ", data)
		}(nochan)	
}
```
上述代码在运行时golang会直接panic，日志输出dead lock警告。
我们可以通过go run -race 选项检测并运行，是可以看到主协程一直阻塞，子协程无法启动的。
## WaitGroup 待时而动
``` golang
func main() {
	nochan := make(chan int)
	waiter := &sync.WaitGroup{}
	waiter.Add(2)
	go func(ch chan int, wt *sync.WaitGroup) {
		data := <-ch
		fmt.Println("receive data ", data)
		wt.Done()
	}(nochan, waiter)

	go func(ch chan int, wt *sync.WaitGroup) {
		ch <- 5
		fmt.Println("send data ", 5)
		wt.Done()
	}(nochan, waiter)
	waiter.Wait()
}
```
通过waitgroup管理两个协程，主协程等待两个子协程退出。
``` cmd
receive data  5
send data  5
```
## range 自动读取
使用range可以自动的从channel中读取，当channel被关闭时，for循环退出,否则一直挂起
``` golang
func main() {
	catchan := make(chan int, 2)
	go func(ch chan int) {
		for i := 0; i < 2; i++ {
			ch <- i
			fmt.Println("send data is ", i)
		}
		//不关闭close，主协程将无法range退出
		close(ch)
		fmt.Println("goroutine1 exited")
	}(catchan)
    
    for data := range catchan {
		fmt.Println("receive data is ", data)
	}

	fmt.Println("main exited")
}
```
输出如下
``` cmd
receive data is  0
send data is  0
send data is  1
goroutine1 exited
receive data is  1
main exited
```
如果不写close(ch)，主协程将一直挂起，编译会出现死锁panic。
可以通过go run -race 选项检查看到主协程一直挂起。
## 缓冲channel, 先进先出
非缓冲channel内部其实是一个加锁的队列，先进先出。先写入的数据优先读出来。
``` golang
func main() {
	catchan := make(chan int, 2)
	go func(ch chan int) {
		for i := 0; i < 2; i++ {
			ch <- i
			fmt.Println("send data is ", i)
		}
	}(catchan)
	for i := 0; i < 2; i++ {
		data := <-catchan
		fmt.Println("receive data is ", data)
	}
}
```
输出如下
``` cmd
send data is  0
send data is  1
receive data is  0
receive data is  1
```
主协程从catchan中读取数据，子协程先catchan中写数据。主协程运行到读取位置先阻塞，子协程启动后向catchan中写数据后，主协程继续读取。
如果将主协程的for循环卸载go启动子协程之前，会造成编译警告死锁，当然可以通过go run -race 查看到主协程一直挂起。
## 读取关闭的channel
从关闭的channel中读取数据，优先读出其中没有取出的数据，然后读出存储类型的空置。循环读取关闭的channel不会阻塞，会一直读取空值。可以通过读取结果的bool值判断该channel是否关闭。
``` golang
func main() {
	nochan := make(chan int)
	go func(ch chan int) {
		ch <- 100
		fmt.Println("send data", 100)
		close(ch)
		fmt.Println("goroutine exit")
	}(nochan)
	data := <-nochan
	fmt.Println("receive data is ", data)
	//从关闭的
	data, ok := <-nochan
	if !ok {
		fmt.Println("receive close chan")
		fmt.Println("receive data is ", data)
	}
	fmt.Println("main exited")
}
```
输出如下
``` cmd
receive data is  100
send data 100
goroutine exit
receive close chan
receive data is  0
main exited
```
主协程运行到data := <- nochan阻塞，子协程启动后向ch中写入数据，并关闭ch,此时主协程继续执行，取出一个数据后，再次取出为空值，并且ok为false表示ch已经被关闭。
## 切忌重复关闭channel
重复关闭channel会导致panic
``` golang
func main() {
	nochan := make(chan int)
	go func(ch chan int) {
		close(ch)
		fmt.Println("goroutine exit")
	}(nochan)

	data, ok := <-nochan
	if !ok {
		fmt.Println("receive close chan")
		fmt.Println("receive data is ", data)
	}
	//二次关闭
	close(nochan)
	fmt.Println("main exited")
}
```
输出如下
``` cmd
goroutine exit
receive close chan
receive data is  0
panic: close of closed channel
```
子协程退出后，主协程读取到退出信息，主协程再次关闭chan导致主协程崩溃。
## 切忌向关闭的channel写数据
向关闭的channel写数据会导致panic
``` golang
func main() {
	nochan := make(chan int)
	go func(ch chan int) {
		close(ch)
		fmt.Println("goroutine1 exit")
	}(nochan)

	data, ok := <-nochan
	if !ok {
		fmt.Println("receive close chan")
		fmt.Println("receive data is ", data)
	}

	go func(ch chan int) {
		<-ch
		fmt.Println("goroutine2 exit")
	}(nochan)

	//向关闭的channel中写数据
	nochan <- 200
	fmt.Println("main exited")
}
```
主线程运行到nochan读取数据阻塞，此时子协程1关闭，主协程继续执行获知nochan被关闭，然后启动子协程2，继续运行nochan<-200，此时nochan已被关闭，导致panic，效果如下
``` cmd
receive close chan
receive data is  0
goroutine1 exit
goroutine2 exit
panic: send on closed channel
```
## 切忌关闭nil的channel
关闭nil值的channel会导致panic
``` golang
func main() {
	var nochan chan int = nil
	go func(ch chan int) {
		//关闭nil channel会panic
		close(ch)
		fmt.Println("goroutine exit")
	}(nochan)

	//从nil channel中读取会阻塞
	data, ok := <-nochan
	if !ok {
		fmt.Println("receive close chan")
		fmt.Println("receive data is ", data)
	}
	fmt.Println("main exited")
}
```
主协程定义了一个nil值的nochan，并未开辟空间。运行至data, ok := <-nochan 阻塞，此时启动子协程，关闭nochan，导致panic
效果如下
``` cmd
panic: close of nil channel
```
## 读或写nil的channel都会阻塞
向nil的channel写数据，或者读取nil的channel也会导致阻塞。
``` golang
func main() {
	var nochan chan int = nil
	go func(ch chan int) {
		fmt.Println("goroutine begin receive data")
		data, ok := <-nochan
		if !ok {
			fmt.Println("receive close chan")
		}
		fmt.Println("receive data is ", data)
		fmt.Println("goroutine exit")
	}(nochan)
	fmt.Println("main begin send data")
	//向nil channel中写数据会阻塞
	nochan <- 100
	fmt.Println("main exited")
}
```
如果直接编译系统会判断死锁panic，我们用go run -race main.go死锁检测，并运行，看到主协程一直挂起，子协程也一直挂起。
结果如下
``` cmd
goroutine begin receive data
main begin send data
```
主协程和子协程都阻塞了，一直挂起。
## select 多路复用，大巧不工
select 内部可以写多个协程读写，通过case完成多路复用，其结构如下
``` golang
select {
	case ch <- 100:
		...
	case <- ch2:
		...
	dafault:
		...
}
```
如果有多个case满足条件，则select随机选择一个执行。否则进入dafault执行。
我们可以利用上面的九种原理配合select创造出各种并发场景。
## 总结
1 当我们不使用一个channel时将其置为nil，这样select就不会检测它了。
2 当多个子协程想获取主协程退出通知时，可以从同一个chan中读取，如果主协程退出则关闭这个chan，那么所有从chan读取的子协程就会获得退出消息。从而实现广播。
3 为保证协程优雅退出，关闭channel的操作尽量放在对channel执行写操作的协程中。
## 并发实战
假设有这样的需求：
1 主协程启动两个协程，协程1负责发送数据给协程2，协程2负责接收并累加获得的数据。
2 主协程等待两个子协程退出，当主协程意外退出时通知两个子协程退出。
3 当发送协程崩溃和主动退出时通知接收协程也要退出，然后主协程退出
4 当接收协程崩溃或主动退出时通知发送协程退出，然后主协程退出。
5 无论三个协程主动退出还是panic，都要保证所有资源手动回收。
下面我们用上面总结的十招完成这个需求
``` golang
	datachan := make(chan int)
	groutineclose := make(chan struct{})
	mainclose := make(chan struct{})
	var onceclose sync.Once
	var readclose sync.Once
	var sendclose sync.Once
	var waitgroup sync.WaitGroup
	waitgroup.Add(2)
```
datachan: 用来装载发送协程给接收协程的数据
groutineclose: 用于发送协程和接收协程之间关闭通知
onceclose: 保证datachan一次关闭。
readclose: 保证接收协程资源一次回收。
sendclose: 保证发送协程资源一次回收。
waitgroup: 主协程管理两个子协程。
接下来我们实现发送协程
``` golang
go func(datachan chan int, gclose chan struct{}, mclose chan struct{}, group *sync.WaitGroup) {
		defer func() {
			onceclose.Do(func() {
				close(gclose)
			})
			sendclose.Do(func() {
				close(datachan)
				fmt.Println("send goroutine closed !")
				group.Done()
			})
		}()

		for i := 0; i < 100; i++ {
			select {
			case <-gclose:
				fmt.Println("other goroutine exited")
				return
			case <-mclose:
				fmt.Println("main goroutine exited")
				return
				/*
					default:
						datachan <- i
				*/
			case datachan <- i:
			}
		}
	}(datachan, groutineclose, mainclose, &waitgroup)
```
发送协程在defer函数中回收了和接收协程公用的chan，也主动关闭了数据chan，这么做保证关闭不会panic。此外还对group做了释放。
其实将datachan <- i 放在default分支也是可以的。但是为了保证接收协程退出后该发送协程也要及时退出，就放在case逻辑中，这样不会死锁。
发送协程累计发送100次数据给接收协程，然后退出。
接下来我们实现接收协程
``` golang
go func(datachan chan int, gclose chan struct{}, mclose chan struct{}, group *sync.WaitGroup) {
		sum := 0
		defer func() {
			onceclose.Do(func() {
				close(gclose)
			})
			readclose.Do(func() {
				fmt.Println("sum is ", sum)
				fmt.Println("receive goroutine closed !")
				group.Done()
			})
		}()

		for i := 0; ; i++ {
			select {
			case <-gclose:
				fmt.Println("other goroutine exited")
				return
			case <-mclose:
				fmt.Println("main goroutine exited")
				return
			case data, ok := <-datachan:
				if !ok {
					fmt.Println("receive close chan data")
					return
				}
				sum += data
			}
		}
	}(datachan, groutineclose, mainclose, &waitgroup)
```
和发送协程一样，接收协程也通过once操作保证公用的通知chan只回收一次。然后回收了自己的资源。接收协程一直循环获取数据，如果收到主协程退出或者发送协程退出的通知，就退出。
接下来我们继续编写主协程的等待和回收操作
``` golang
defer func() {
		close(mainclose)
		time.Sleep(time.Second * 5)
	}()

	waitgroup.Wait()
	fmt.Println("main exited")
```
这些逻辑我们都写在main函数里即可。主协程通过waitgroup等待两个协程，并通过defer通知两个协程退出。
运行代码效果如下
``` cmd
send goroutine closed !
receive close chan data
sum is  4950
receive goroutine closed !
main exited
```
可以看出发送协程退出接收协程也退出了，接收协程正好计算100次累加，数值为4950。主协程也退出了。
## 测试接收协程异常退出
接下来我们测试接收协程异常退出后，发送协程和主协程退出是否回收资源。
我们将接收协程的case逻辑改为i>=20时该接收协程主动panic
``` golang
	case data, ok := <-datachan:
		if !ok {
					fmt.Println("receive close chan data")
					return
				}
			sum += data
		if i >= 20 {
					panic("receive goroutine test panic !!")
				}
```
运行代码看下效果
``` cmd
recover !
close gclose channel
sum is  210
receive goroutine closed !
other goroutine exited
send goroutine closed !
main exited
defer main close
```
我们在接收协程的defer里增加了recover逻辑，可以看到三个协程都正常退出并回收了各自的资源。
## 测试主协程主动退出
我们将主协程的等待代码去掉，并且在defer中增加延时退出，方便看到两个协程退出情况
``` golang
	defer func() {
		fmt.Println("defer main close")
		close(mainclose)
		time.Sleep(time.Second * 10)
	}()

	time.Sleep(time.Second * 10)
	fmt.Println("main exited")
```
运行看效果
``` cmd
main exited
defer main close
main goroutine exited
sum is  88074498378441
receive goroutine closed !
main goroutine exited
send goroutine closed !
```
看到三个协程正常退出，并回收了资源。
## 源码下载
[https://github.com/secondtonone1/golang-/tree/master/channelpractice](https://github.com/secondtonone1/golang-/tree/master/channelpractice)
