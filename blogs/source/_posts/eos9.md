---
title: eos源码剖析之controller
date: 2019-04-09 11:01:39
categories: 技术开发
tags: [区块链eos]
---
## controller::block_status，区块状态枚举类，包括：
irreversible = 0，该区块已经被当前节点应用，并且被认为是不可逆的。
validated = 1，这是由一个有效生产者签名的完整区块，并且之前已经被当前节点应用，因此该区块已被验证但未成为不可逆。
complete = 2，这是一个由有效生产者签名的完整区块，但是还没有成为不可逆，也没有被当前节点应用。
incomplete = 3，这是一个未完成的区块，未被生产者签名也没有被某个节点生产。
实际上块的状态就是，1未签名未应用 2已签名未应用 3 已签名和应用，未变成不可逆 4 变成不可逆
## controller的私有成员：
apply_context,应用上下文，处理节点应用区块的上下文环境，其中包含了迭代器缓存iterator_cache<key_value_object>
transaction_context，事务上下文环境。包括controller，db的session，signed_transaction等。
mutable_db()，返回一个chainbase::database的引用
controller_impl结构体的实例的唯一指针my。controller核心功能都是通过impl实现调用的。从fork_db比如获取区块信息
<!--more-->
## controller的信号
controller 包括了以下几个信号：
signal<void(const signed_block_ptr&)> pre_accepted_block; // 预承认区块(承认其他节点广播过来的区块是正确的)
signal<void(const block_state_ptr&)> accepted_block_header; // 承认区块头(对区块头做过校验)
signal<void(const block_state_ptr&)> accepted_block; // 承认区块
signal<void(const block_state_ptr&)> irreversible_block; // 不可逆区块
signal<void(const transaction_metadata_ptr&)> accepted_transaction; // 承认事务
signal<void(const transaction_trace_ptr&)> applied_transaction; // 应用事务(承认其他节点数据要先校验，通过以后可以应用在本地节点)
signal<void(const header_confirmation&)> accepted_confirmation; // 承认确认
signal<void(const int&)> bad_alloc; // 内存分配错误信号
所有信号的发射时机都是在controller中。
分别介绍每个信号发射和接收时机

1. pre_accepted_block
发射时机：push_block函数，在producer_plugin中 on_incoming_block中会对已签区块校验，包括不能有pending块，不能push空块，区块状态不能是incomplete。通过校验后，调用push_block会发射该信号，携带该区块。
on_incoming_block函数
``` cpp
void producer_plugin::on_incoming_block(const signed_block_ptr& block) {
         //判断是否在fork_db中
         auto existing = chain.fetch_block_by_id( id );
         if( existing ) { return; }
         // 开始创建一个blockstate
         auto bsf = chain.create_block_state_future( block );
         // 清除 pending状态，将交易放到unappliedtrx map中
         chain.abort_block();
         // 当抛出异常时，重启loop
         auto ensure = fc::make_scoped_exit([this](){
            schedule_production_loop();
         });
         // push the new block
         bool except = false;
         try {
            chain.push_block( bsf );
         } catch ( const guard_exception& e ) {
            chain_plug->handle_guard_exception(e);
            return;
         } catch( const fc::exception& e ) {
            elog((e.to_detail_string()));
            except = true;
         } catch ( boost::interprocess::bad_alloc& ) {
            chain_plugin::handle_db_exhaustion();
            return;
         }
        
```
on_incoming_block做了这样几个事，
1 判断forkdb是否有该新来的块，
2 其次根据该块投递到线程池生成blockstate，
3 清除之前的pending状态，pending中的交易取出放到unapplied map中.(以后会通过schedule_production_loop调用producer_plugin::start_block处理unapplied map,内部调用controller::start_block重新组织pending)
4 为防止producer异常退出，设置schedule_production_loop重启。
5 调用controller的push_block
``` cpp
void controller::push_block( std::future<block_state_ptr>& block_state_future ) {
      controller::block_status s = controller::block_status::complete;
      EOS_ASSERT(!pending, block_validate_exception, "it is not valid to push a block when there is a pending block");

      auto reset_prod_light_validation = fc::make_scoped_exit([old_value=trusted_producer_light_validation, this]() {
         trusted_producer_light_validation = old_value;
      });
      try {
         block_state_ptr new_header_state = block_state_future.get();
         auto& b = new_header_state->block;
         emit( self.pre_accepted_block, b );

         fork_db.add( new_header_state, false );

         if (conf.trusted_producers.count(b->producer)) {
            trusted_producer_light_validation = true;
         };
         emit( self.accepted_block_header, new_header_state );

         if ( read_mode != db_read_mode::IRREVERSIBLE ) {
            maybe_switch_forks( s );
         }

      } FC_LOG_AND_RETHROW( )
   }
```
在push_block中发送了pre_accepted_block信号
插件捕捉处理： chain_plugin连接该信号，在plugin_initialize中绑定了信号的处理
但是该channel没有订阅者。
``` cpp
// relay signals to channels
      my->pre_accepted_block_connection = my->chain->pre_accepted_block.connect([this](const signed_block_ptr& blk) {
         auto itr = my->loaded_checkpoints.find( blk->block_num() );
         if( itr != my->loaded_checkpoints.end() ) {
            auto id = blk->id();
            EOS_ASSERT( itr->second == id, checkpoint_exception,
                        "Checkpoint does not match for block number ${num}: expected: ${expected} actual: ${actual}",
                        ("num", blk->block_num())("expected", itr->second)("actual", id)
            );
         }

         my->pre_accepted_block_channel.publish(priority::medium, blk);
      });
```
2. accepted_block_header

发射时机1： commit_block函数，
``` cpp
void commit_block( bool add_to_fork_db ) {
         if (add_to_fork_db) {
            pending->_pending_block_state->validated = true;
            auto new_bsp = fork_db.add(pending->_pending_block_state, true);
            emit(self.accepted_block_header, pending->_pending_block_state);
            head = fork_db.head();
            EOS_ASSERT(new_bsp == head, fork_database_exception, "committed block did not become the new head in fork database");
         }     
      }
```
add_to_fork_db为true,将_pending_block_state->validated设置为true,_pending_block_state放入fork_db,然后发送accepted_block_header

发射时机2： push_block函数，获取区块的可信状态,发射完pre_accepted_block以后，添加可信状态至fork_db，然后发射accepted_block_header信号，携带fork_db添加成功后返回的状态区块。

插件捕捉处理： chain_plugin连接该信号，由信号槽转播到channel，accepted_block_header_channel发布该区块。bnet_plugin订阅该channel，绑定bnet_plugin_impl的on_accepted_block_header函数，该函数涉及到线程池等概念，将会在bnet_plugin插件的部分详细分析。遍历线程池，转到session会话下的on_accepted_block_header函数执行。如果传入区块与本地时间相差6秒以内则接收，之外不处理。接收处理时先从本地多索引库表block_status中查找是否已存在，不存在则插入block_status结构对象，如果不是远程不可逆请求以及不存在该区块，或者该区块不是来自其他节点的情况，要在区块头通知集合中插入该区块id。

bnet_plugin订阅该信号，绑定处理函数，函数体实现了日志打印。
``` cpp
my->_on_accepted_block_header_handle = app().get_channel<channels::accepted_block_header>()
                                         .subscribe( [this]( block_state_ptr s ){
                                                my->on_accepted_block_header(s);
                                         });
```
绑定了回调函数，回调函数内部调用bnet_plugin_impl的on_accepted_block_header(s);
``` cpp
void on_accepted_block_header( const block_state_ptr& s ) {
           ...
           const auto& id = s->id;
           if( fc::time_point::now() - s->block->timestamp  < fc::seconds(6) ) {
              auto itr = _block_status.find( id );
              //_remote_request_irreversible_only(对端请求可逆)且(之前状态集合中没有该块或没收到过该块)
              if( !_remote_request_irreversible_only && ( itr == _block_status.end() || !itr->received_from_peer ) ) {
                 //加入通知集合，通知对方防止对方重复发送区块头
                 _block_header_notices.insert( id );
              }
              //加入状态集合
              if( itr == _block_status.end() ) {
                 _block_status.insert( block_status(id, false, false) );
              }
           }
        }
```

插件捕捉处理2： 