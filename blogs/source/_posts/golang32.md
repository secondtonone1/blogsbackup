---
title: 谈谈golang的netpoll原理(二)
date: 2020-05-20 11:17:16
categories: [golang]
tags: [golang]
---
接上文我们查看了bind和listen流程，直到了listen操作会在内核初始化一个epoll表，并将listen的描述符加入到epoll表中
## 如何保证epoll表初始化一次
前文我们看到pollDesc的init函数中调用了runtime的pollOpen函数完成的epoll创建和描述符加入，这里再贴一次代码
``` golang
func (pd *pollDesc) init(fd *FD) error {
	serverInit.Do(runtime_pollServerInit)
	ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
	if errno != 0 {
		if ctx != 0 {
			runtime_pollUnblock(ctx)
			runtime_pollClose(ctx)
		}
		return errnoErr(syscall.Errno(errno))
	}
	pd.runtimeCtx = ctx
	return nil
}
```
runtime_pollServerInit link的是runtime/netpoll.go中的poll_runtime_pollServerInit函数
由于serverInit是sync.Once类型，所以runtime_pollServerInit只被初始化一次，而epoll模型的初始化就是在该函数完成
<!--more-->
``` golang
func poll_runtime_pollServerInit() {
	netpollGenericInit()
}

func netpollGenericInit() {
	if atomic.Load(&netpollInited) == 0 {
		lock(&netpollInitLock)
		if netpollInited == 0 {
			netpollinit()
			atomic.Store(&netpollInited, 1)
		}
		unlock(&netpollInitLock)
	}
}
```
netpollinit实现了不同模型的初始化，epoll的实现在runtime/netpoll_epoll.go中
``` golang
func netpollinit() {
	epfd = epollcreate1(_EPOLL_CLOEXEC)
	if epfd < 0 {
		epfd = epollcreate(1024)
		if epfd < 0 {
			println("runtime: epollcreate failed with", -epfd)
			throw("runtime: netpollinit failed")
		}
		closeonexec(epfd)
	}
	//...
}
```
可以看到上述代码里实现了epoll模型的初始化，所以对于一个M主线程只会初始化一张epoll表，所有要监听的文件描述符都会放入这个表中。
## 跟随accept看看goroutine挂起逻辑
当我们调用Listener的Accept时，Listener为接口类型，实际调用的为TCPListener的Accept函数
``` golang
func (l *TCPListener) Accept() (Conn, error) {
	if !l.ok() {
		return nil, syscall.EINVAL
	}
	c, err := l.accept()
	if err != nil {
		return nil, &OpError{Op: "accept", Net: l.fd.net, Source: nil, Addr: l.fd.laddr, Err: err}
	}
	return c, nil
}
```
Accept内部调用了accept函数，该函数内部实际调用netFD的accept
``` golang
func (ln *TCPListener) accept() (*TCPConn, error) {
	fd, err := ln.fd.accept()
	if err != nil {
		return nil, err
    }
    //...
}
```
在net/fd_unix.go中实现了linux环境下accept的操作
``` golang
func (fd *netFD) accept() (netfd *netFD, err error) {
	d, rsa, errcall, err := fd.pfd.Accept()
	if err != nil {
		if errcall != "" {
			err = wrapSyscallError(errcall, err)
		}
		return nil, err
	}

	if netfd, err = newFD(d, fd.family, fd.sotype, fd.net); err != nil {
		poll.CloseFunc(d)
		return nil, err
	}
	if err = netfd.init(); err != nil {
		netfd.Close()
		return nil, err
	}
	lsa, _ := syscall.Getsockname(netfd.pfd.Sysfd)
	netfd.setAddr(netfd.addrFunc()(lsa), netfd.addrFunc()(rsa))
	return netfd, nil
}
```
上述函数内部调用的是net/fd_unix.go内部实现的Accept函数
``` golang
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
	if err := fd.readLock(); err != nil {
		return -1, nil, "", err
	}
	defer fd.readUnlock()

	if err := fd.pd.prepareRead(fd.isFile); err != nil {
		return -1, nil, "", err
	}
	for {
		s, rsa, errcall, err := accept(fd.Sysfd)
		if err == nil {
			return s, rsa, "", err
		}
		switch err {
		case syscall.EAGAIN:
			if fd.pd.pollable() {
				if err = fd.pd.waitRead(fd.isFile); err == nil {
					continue
				}
			}
		case syscall.ECONNABORTED:
			// This means that a socket on the listen
			// queue was closed before we Accept()ed it;
			// it's a silly error, so try again.
			continue
		}
		return -1, nil, errcall, err
	}
}
```
上述函数就是tcp底层的函数了，accept(fd.Sysfd)监听fd.Sysfd描述符，等待可读事件到来，当可读事件到来后，就可以认为来了一个新的连接，从而创建一个新的描述符给新的连接。
当accept出现错误时，需要判断err类型，如果是EAGAIN说明当前没有连接到来，就调用waitRead等待连接，ECONNABORTED说明连接还未accept就断开了，可以忽略。
``` golang
func (pd *pollDesc) waitRead(isFile bool) error {
	return pd.wait('r', isFile)
}
```
进而调用pollDesc的wait操作
``` golang
func (pd *pollDesc) wait(mode int, isFile bool) error {
	if pd.runtimeCtx == 0 {
		return errors.New("waiting for unsupported file type")
	}
	res := runtime_pollWait(pd.runtimeCtx, mode)
	return convertErr(res, isFile)
}
```
wait函数中判断pd的runtime上下文是否正常，然后调用runtime包的poll_runtime_pollWait实现挂起等待
``` golang
func poll_runtime_pollWait(pd *pollDesc, mode int) int {
	err := netpollcheckerr(pd, int32(mode))
	if err != 0 {
		return err
	}
	if GOOS == "solaris" || GOOS == "illumos" || GOOS == "aix" {
		netpollarm(pd, mode)
	}
	for !netpollblock(pd, int32(mode), false) {
		err = netpollcheckerr(pd, int32(mode))
		if err != 0 {
			return err
		}
	}
	return 0
}
```
poll_runtime_pollWait运行在内核M线程中，轮询调用netpollblock，所以内核M线程一直在轮询检测netpollblock返回值，当其返回true时循环就可以退出，从而用户态协程就可以继续运行了。
``` golang
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
	gpp := &pd.rg
	if mode == 'w' {
		gpp = &pd.wg
	}

	// set the gpp semaphore to WAIT
	for {
		old := *gpp
		if old == pdReady {
			*gpp = 0
			return true
		}
		if old != 0 {
			throw("runtime: double wait")
		}
		if atomic.Casuintptr(gpp, 0, pdWait) {
			break
		}
	}
	if waitio || netpollcheckerr(pd, mode) == 0 {
		gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
	}
	old := atomic.Xchguintptr(gpp, 0)
	if old > pdWait {
		throw("runtime: corrupted polldesc")
	}
	return old == pdReady
}
```
netpollblock内部根据读模式还是写模式，获取pollDesc成员变量的读协程或者写协程地址，然后判断其状态是否为pdReady，这里要详细说一下，golang阻塞一个用户态协程是要将其状态设置为0(正在运行)或者pdWait(阻塞)，这里为0，所以逻辑继续往下走，之后做了一个原子操作将gpp设置为pdWait状态，接着根据这个状态，执行gopark函数，阻塞住用户态协程。当内核想激活用户协程时gopark会返回，然后该函数判断gpp是否为pdReady，从而激活用户态协程。
``` golang
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
	if reason != waitReasonSleep {
		checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
	}
	mp := acquirem()
	gp := mp.curg
	status := readgstatus(gp)
	if status != _Grunning && status != _Gscanrunning {
		throw("gopark: bad g status")
	}
	mp.waitlock = lock
	mp.waitunlockf = unlockf
	gp.waitreason = reason
	mp.waittraceev = traceEv
	mp.waittraceskip = traceskip
	releasem(mp)
	// can't do anything that might move the G between Ms here.
	mcall(park_m)
}
```
gopark将用户态协程放在等待队列中，然后调用mcall触发汇编代码。之后会检测调用unlockf函数，如果unlockf返回false则说明可以解锁用户态协程了。另外官网的注释说unlockf不要访问用户态协程的stack，因为G's stack可能会在gopark和unlockf之间被移除。到目前为止，我们理解了用户态协程挂起原理。
## epoll就绪后如何激活用户态协程
想知道如果激活挂起的用户态协程，就要先看看epoll_wait判断就绪事件后怎么处理的。runtime/netpoll_epoll.go中实现了epollwait逻辑
``` golang
func netpoll(delay int64) gList {
	if epfd == -1 {
		return gList{}
	}
	//...
	var events [128]epollevent
retry:
	n := epollwait(epfd, &events[0], int32(len(events)), waitms)
	if n < 0 {
		if n != -_EINTR {
			println("runtime: epollwait on fd", epfd, "failed with", -n)
			throw("runtime: netpoll failed")
		}
		// If a timed sleep was interrupted, just return to
		// recalculate how long we should sleep now.
		if waitms > 0 {
			return gList{}
		}
		goto retry
	}
	var toRun gList
	for i := int32(0); i < n; i++ {
		ev := &events[i]
		if ev.events == 0 {
			continue
		}
        //...
		var mode int32
		if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
			mode += 'r'
		}
		if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
			mode += 'w'
		}
		if mode != 0 {
			pd := *(**pollDesc)(unsafe.Pointer(&ev.data))
			pd.everr = false
			if ev.events == _EPOLLERR {
				pd.everr = true
			}
			netpollready(&toRun, pd, mode)
		}
	}
	return toRun
}
```
可以看出netpoll函数调用epollwait返回就绪事件列表，然后遍历就绪的事件列表，从事件类型中取出pollDesc数据，调用netpollready将曾经挂起的协程放入gList中，然后返回该列表
``` golang
func netpollready(toRun *gList, pd *pollDesc, mode int32) {
	var rg, wg *g
	if mode == 'r' || mode == 'r'+'w' {
		rg = netpollunblock(pd, 'r', true)
	}
	if mode == 'w' || mode == 'r'+'w' {
		wg = netpollunblock(pd, 'w', true)
	}
	if rg != nil {
		toRun.push(rg)
	}
	if wg != nil {
		toRun.push(wg)
	}
}
```
netpollready调用了unblock函数，并且将协程写入glist中
``` golang
func netpollunblock(pd *pollDesc, mode int32, ioready bool) *g {
	gpp := &pd.rg
	if mode == 'w' {
		gpp = &pd.wg
	}

	for {
		old := *gpp
		if old == pdReady {
			return nil
		}
		if old == 0 && !ioready {
			// Only set READY for ioready. runtime_pollWait
			// will check for timeout/cancel before waiting.
			return nil
		}
		var new uintptr
		if ioready {
			new = pdReady
		}
		if atomic.Casuintptr(gpp, old, new) {
			if old == pdReady || old == pdWait {
				old = 0
			}
			return (*g)(unsafe.Pointer(old))
		}
	}
}
```
netpollunblock函数修改pd所在协程的状态为0，表示可运行状态，所以netpoll函数内部做了这样几件事，根据就绪事件列表找到对应的协程，将挂起的协程状态设置为0表示可运行，然后将该协程放入glist中。在runtime/proc.go中findrunnable会判断是否初始化epoll，如果初始化了则调用netpoll，从而获取glist，然后traceGoUnpark激活挂起的协程
``` golang
func findrunnable() (gp *g, inheritTime bool) {
	_g_ := getg()
    //...
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
		if list := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}
    }
    //...
}
```
以上就是golang网络调度和协程控制的原理，golang通过epoll和用户态协程调度结合的方式，实现了高并发的网络处理，这种思路是值得日后我们设计产品借鉴的。
感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)

