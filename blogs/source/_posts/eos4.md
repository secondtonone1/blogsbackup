---
title: eos交易同步过程和区块生产过程源码分析
date: 2019-01-30 10:29:25
categories: 技术开发
tags: [区块链eos]
---
## 交易同步过程
1 通过命令cleos调用 cleos transfer ${from_account} ${to_account} ${quantity} 发起交易
2 eos调用chain_plugin 的push_transaction,内部调用注册好的方法。
``` cpp
app().get_method<incoming::methods::transaction_async>();
```
<!--more-->
代码截图如下
![1.png](1.png)
在producer_plugin插件的plugin_initialize函数中提前注册了incoming::methods::transaction_async。
![2.png](2.png)
所以实际调用了producer_plugin_impl的on_incoming_transaction_async(trx, persist_until_expired, next )函数,
传递的参数分别是pretty_input,true,和一个lambda匿名函数
``` cpp
app().get_method<incoming::methods::transaction_async>()(pretty_input, true, [this, next](const fc::static_variant<fc::exception_ptr, transaction_trace_ptr>& result) -> void{
         if (result.contains<fc::exception_ptr>()) {
            next(result.get<fc::exception_ptr>());
         } else {
            auto trx_trace_ptr = result.get<transaction_trace_ptr>();

            try {
               chain::transaction_id_type id = trx_trace_ptr->id;
               fc::variant output;
               try {
                  output = db.to_variant_with_abi( *trx_trace_ptr, abi_serializer_max_time );
               } catch( chain::abi_exception& ) {
                  output = *trx_trace_ptr;
               }

               next(read_write::push_transaction_results{id, output});
            } CATCH_AND_CALL(next);
         }
      });
```
3 在producer_plugin_impl的on_incoming_transaction_async中调用controller的 push_transaction，并执行trx。
将交易插入_pending_incoming_transactions中
![3.png](3.png)
接下来调用了controller的push_transaction函数
![4.png](4.png)
4 controller的push_transaction函数中发送消息accepted_transaction，而目前常用的网络插件为bnet_plugin,net_plugin目前已作为备用,
由于在bnet_plugin的startup函数中绑定了消息回调函数
![5.png](5.png)
``` cpp
my->_on_appled_trx_handle = app().get_channel<channels::accepted_transaction>()
                                .subscribe( [this]( transaction_metadata_ptr t ){
                                       my->on_accepted_transaction(t);
                                });
```
通过app().get_channel将channels::accepted_transaction信号和lambda表达式绑定起来，所以当controller发送信号accepted_transaction就会调用这个lambda表达式传递参数，从而调用bnet_plugin_impl中on_accepted_transaction函数。
5  bnet_plugin_impl通过on_accepted_transaction将消息广播到其他节点
``` cpp
void on_accepted_transaction( transaction_metadata_ptr trx ) {
            if( trx->implicit || trx->scheduled ) return;
            for_each_session( [trx]( auto ses ){ ses->on_accepted_transaction( trx ); } );
         }
```
6 其他节点收到消息后，进入on处理流程，发送transction消息
bnet_plugin处理消息函数
![6.png](6.png)
transaction消息处理
![7.png](7.png)
通过
``` cpp
app().get_channel<incoming::channels::transaction>().publish(p);
```
发送transaction消息。
7 producer_plugin绑定了消息处理的回调函数
![8.png](8.png)
收到消息后，调用on_incoming_transaction_async，调用controller的 push_transaction，并执行trx。
以上就是eos交易同步过程。
## 区块生产过程
整体的区块生产流程
![9.png](9.png)
1 检查自己是否是生产者，一个生产者500ms出一次块，共出12次之后切换生产者。
2 对上次确认的区块到本次的区块做BFT签名，涉及函数set_confirmed和maybe_promote_pending,具体可以参看controller中start_block函数
3 等待一个出块周期500ms
4 计算action的merkle root
5 计算transaction的merkle root
6 对区块签名
7 提交区块到DB
8 递归调用schedule_production_loop
## 区块同步过程
1 参考区块生产过程，producer_plugin循环生产区块，先start_block处理BFT签名并确定不可逆的区块数，之后produce_block调用controller
2 controller使用finalize_block计算merkle root，使用commit_block提交到fork database中，fork db会依据1中计算的不可逆区块数，将不可逆的区块删除，并发送irreversible消息,controller绑定了该消息处理的回调函数
``` cpp
fork_db.irreversible.connect( [&]( auto b ) {
                                 on_irreversible(b);
                                 });
```
3 controller收到消息后，调用on_irreversible处理发送irreversible_block消息。bnet注册了该消息处理的回调函数
``` cpp
my->_on_irb_handle = app().get_channel<channels::irreversible_block>()
                                .subscribe( [this]( block_state_ptr s ){
                                       my->on_irreversible_block(s);
                                });
```
4 bnet_plugin 收到消息后，调用on_irreversible_block处理。并广播给其他节点。
5 controller发送accepted_block_header和accepted_block消息。
6 producer_plugin收到消息后， 调用on_block，calc_dpos_last_irreversible计算不可逆块。
7 bnet_plugin/net_plugin 收到之后广播到其他节点。
8 其他节点的bnet_plugin/net_plugin收到P2P消息后，发送block消息
``` cpp
void on( const signed_block_ptr& b ) {
           peer_ilog(this, "received signed_block_ptr");
           if (!b) {
              peer_elog(this, "bad signed_block_ptr : null pointer");
              EOS_THROW(block_validate_exception, "bad block" );
           }
           status( "received block " + std::to_string(b->block_num()) );
           //ilog( "recv block ${n}", ("n", b->block_num()) );
           auto id = b->id();
           mark_block_status( id, true, true );

           app().get_channel<incoming::channels::block>().publish(b);

           mark_block_transactions_known_by_peer( b );
        }
```
9 Producer_plugin收到block消息后调用controller的push_block函数
10 Controller调用apply_block判断如果新收到的block比原有的链长，则切换到新链上
11 Controller调用finalize_block计算merkle root，使用commit_block提交到DB
以上就是区块同步过程。
感谢关注我的公众号
![1.jpg](1.jpg)













