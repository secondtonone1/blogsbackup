---
title: eos 源码net_plugin分析
date: 2019-01-25 17:08:26
categories: 技术开发
tags: [区块链eos]
---
1  net_plugin_impl::connect(connection_ptr c) 函数用于解析地址，内部异步回调async_resolve
<!--more-->
![1.png](1.png)
async_resolve 传递了lambda表达式，如果err为零，则调用connect连接指定的地址
![2.png](2.png)
2   void net_plugin_impl::connect( connection_ptr c, tcp::resolver::iterator endpoint_itr ) 该函数将connection连接指定地址
内部调用async_connect进行异步连接，如果连接成功，回调lambda表达式。lambda表达式中取出connection的weak_ptr，这么做为防止shared_ptr互相引用导致内存无法释放。如果连接失败关闭socket连接，如果地址错误关闭socket重新连接。
![3.png](3.png)
连接成功后，调用start_session，开始监听该描述符的读事件，并且开始发送握手消息send_handshake()

3   bool net_plugin_impl::start_session( connection_ptr con )
开始连接，内部调用start_read_message()，并且增加连接会话数量。
![4.png](4.png)
4  void net_plugin_impl::start_read_message( connection_ptr conn ) 读取消息函数，内部较为复杂。
![5.png](5.png)
内部将shared_ptr connection_ptr 赋值给connection_wptr(weak_ptr)类型变量weak_conn,也是为了防止互相引用shared_ptr造成内存无法回收，minimum_read为要读取的字节数，如果conn->outstanding_read_bytes 非0,则等于conn->outstanding_read_bytes，否则等于消息头大小。同时判断是否设置低水位标记，如果设置了低水位标记，则将低水位大小设置为取minimum_read和固定的max_socket_read_watermark的最小值。同时实现lambda表达式赋值给completion_handler，completion_handler主要功能判断传输字节是否全部传输完成。未完成，则返回差值。

5  接着   void net_plugin_impl::start_read_message( connection_ptr conn )内部调用了async_read，设置异步读取数据，有消息到来会触发回调lambada表达式。当completion_handler返回0停止读取。lambda函数内部，conn->outstanding_read_bytes.reset()重置请求等状态，
![6.png](6.png)
conn->pending_message_buffer.bytes_to_write()表示pending_message_buffer剩余空间，还可读取多少数据。
conn->pending_message_buffer.
conn->pending_message_buffer.bytes_to_read() 表示pending_message_buffer已经读取多少 字节，用户需要从
pending_message_buffer中读取多少。
conn->outstanding_read_bytes表示buffer还有多少字节没有读取。

判断bytes_transferred > conn->pending_message_buffer.bytes_to_write(),视为异常，因为可用空间不足了 ，没办法接收。
conn->pending_message_buffer.advance_write_ptr(bytes_transferred); 将写指针向后推进bytes_transferred字节。
write_ptr指向了pending_message_buffer的可用空间。bytes_to_write()返回可用字节数。
如果conn->pending_message_buffer.bytes_to_read() > 0，则会一直循环，直到读完buffer中所有数据。
如果bytes_in_buffer < message_header_size，则将头部未读取的数据放入outstanding_read_bytes中。否则说明读完包头数据，pending_message_buffer.peek，读取4字节数据写入message_lenghth,确定数据大小。
![7.png](7.png)
auto total_message_bytes = message_length + message_header_size;为整个数据包大小，bytes_in_buffer >= total_message_bytes 说明此时buffer已经接受完数据，调用 conn->process_next_message()处理。否则，就将剩余未读完的数据放入outstanding_read_bytes中，
available_buffer_bytes表示可用字节数，如果可用字节数不足，则扩充pending_message_buffer大小。
  
6 bool connection::process_next_message(net_plugin_impl& impl, uint32_t message_length)处理消息。
![8.png](8.png)
位移代码没看明白，注释的意思是保存序列化前的原始信息。之后，blk_buffer.resize(message_length);重新设置大小为message_length长度，
auto index = pending_message_buffer.read_index();找到当前的读索引，peek函数将message_length长度的数据读入blk_buffer中。
之后创建datastream ds用来反序列化。将消息写入net_message中。接着定义了msgHandler 对象m，构造函数中传入net_plugin_impl和connection共享指针，调用msg.visit(m)。msgHandler的定义和实现
![9.png](9.png)
msgHandler重载了()运算符，当类msgHandler对象传参(msg)时会调用impl.handle_message(c,msg)。如msgHandler h(msg)实际调用的
impl.handle_message(c,msg)。
![10.png](10.png)
net_message为static_variant类型，static_variant类中实现了visit函数
![11.png](11.png)
visit内部调用apply函数，apply函数内部调用了visitor对象传参data，visitor就是上一层的msgHandler。
![12.png](12.png)
net_plugin_impl::handle_message 根据参数传入不同，实现了不同的函数调用。
![13.png](13.png)
这就是所说的visitor模式。通过net_message(static_variant类型)调用封装的visit函数，visit内部调用了visitor对象的传参，只要重载()(param)就可以实现调用。msgHandler重载operator()(const T& msg)，所以调用impl.handle_message(c,msg)，而msg为不同类型，所以可实现上面的不同调用。
