---
title: 对io.readfull的理解
date: 2021-11-30 11:33:00
categories: [golang]
tags: [golang]
---
## 问题描述
有些人提出了io.readfull可能导致读取数据溢出的问题，有人说如果ReadAtLeast函数中，n>min时，就会直接返回n，而readfull传进去的min是len(buf),所以最终可能返回的n>len(buf)？

## 问题解答
ReadFull调用的时ReadAtLeast
<!--more-->
``` golang
func ReadFull(r Reader, buf []byte) (n int, err error) {
	return ReadAtLeast(r, buf, len(buf))
}
```
那ReadFull返回的结果就是len(buf) 不会出现n>len(buf)情况
因为ReadAtLeast源码返回三种情况
``` golang
func ReadAtLeast(r Reader, buf []byte, min int) (n int, err error) {
	if len(buf) < min {
		return 0, ErrShortBuffer
	}
	for n < min && err == nil {
		var nn int
		nn, err = r.Read(buf[n:])
		n += nn
	}
	if n >= min {
		err = nil
	} else if n > 0 && err == EOF {
		err = ErrUnexpectedEOF
	}
	return
}
```
1 数据读取的缓冲区buf小于min，这样直接返回buffer太短的错误，这是合理的。你要读取min长度，但是你传递的缓存buf又不够大，api就直接报错。
2 数据读取的缓冲区buff大于0小于min，这种情况是由于意外中断了读取，所以返回ErrUnexpectedEOF错误。
3 正常情况下返回的n就是大于等于min，并且error为nil。
比如我们这样调用
``` golang
//开辟100个字节的buf
buf := make([]byte,100)
//con为网络连接描述符， 从con中最少读取10个字节
io.ReadAtLeast(con, buf, 10)
```
上述代码读取到buf的数据是可能大于10个字节的，比如在con的tcp读缓冲区数据大于10个字节的时候
如果改为
``` golang
//开辟10个字节的buf
buf := make([]byte,10)
//con为网络连接描述符， 从con中最少读取10个字节
io.ReadAtLeast(con, buf, 10)
```
那么ReadAtLeast读取的数据就是10个字节，不会大于10个字节。
因为ReadAtLeast内部循环调用Read函数达到n>=min，而Read传递的参数为buf[n:]，所以即使因为tcp缓冲区数据不足多次Read也不会导致多读，因为Read(buf)只会读取>=0并且<=len(buf)长度的数据，在循环中不断地的控制buf[n:]保证了最多只会读取len(buf)长度。
但是ReadFull没问题，因为ReadFull本意就是调用ReadAtLeast，读取buf长度，也只会读取buf长度的数据。所以核心问题不是ReadFull的问题，是使用者对该API的不理解，只要保证ReadAtLeast(con, buf, min) 中buf的大小等于min就不会有问题，ReadAtLeast在不出错的情况下只会返回min个字节
不会出现溢出情况，对于网络粘包也不会出现bug
对于这个问题，我之前也思考良久，并加以实践和测试，在公司编写了基于tcp的网络服务器，采用的就是ReadFull函数，用于对讲业务，单节点7000并发连接测试粘包未出现bug。
我的代码段是这样的
``` golang
//处理TCP粘包
func (pi *ProtocolImpl) ReadPacket(conn net.Conn) (interface{}, error) {
	buff := make([]byte, 4)
	_, err := io.ReadAtLeast(conn, buff[:4], 4)
	if err != nil {
		//fmt.Println("read at least error ", err.Error())
		return nil, common.ErrReadAtLeast
	}

	var msgpacket *MsgPacket = new(MsgPacket)
	value, err := pi.ParaseHead(msgpacket, buff[:4])

	msgpacket, ok := value.(*MsgPacket)
	if !ok {
		fmt.Println("it's not msgpacket type")
		return nil, common.ErrTypeAssertain
	}

	if components.MaxMsgId < msgpacket.Head.Id {
		return nil, errors.New("msg id is too big")
	}

	if components.MaxMsgLen < msgpacket.Head.Len {
		return nil, common.ErrMsgLenLarge
	}

	if uint16(len(msgpacket.Body.Data)) < msgpacket.Head.Len {
		msgpacket.Body.Data = make([]byte, msgpacket.Head.Len)
	}

	if _, err = io.ReadFull(conn, msgpacket.Body.Data[:msgpacket.Head.Len]); err != nil {
		//fmt.Println("err is ", err.Error())
		return nil, common.ErrReadAtLeast
	}

	return msgpacket, nil
}
```
先用ReadAtLeast读取头部四字节，然后解包分析，进而得到包体的长度，然后用ReadFull从con中读取包体长度的数据。
感兴趣可以看一下我的源码
[https://github.com/secondtonone1/wentmin](https://github.com/secondtonone1/wentmin)