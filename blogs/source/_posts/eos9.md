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
controller_impl结构体的实例的唯一指针my。controller核心功能都是通过impl实现调用的。
``` cpp
struct controller_impl {
   controller&                    self;
   chainbase::database            db;       // 用于存储已执行的区块，这些区块可以回滚，一旦提交就成了不可逆的区块
   chainbase::database            reversible_blocks; ///< a special database to persist blocks that have successfully been applied but are still reversible
   block_log                      blog;
   optional<pending_state>        pending;  // 尚未出的块
   block_state_ptr                head;     // 当前区块的状态
   fork_database                  fork_db;  // 用于存储可分叉区块的数据库，可分叉的区块都放到这里
   wasm_interface                 wasmif;   // wasm虚拟机的runtime
   resource_limits_manager        resource_limits;  // 资源管理
   authorization_manager          authorization;    // 权限管理
   controller::config             conf;
   chain_id_type                  chain_id;
   bool                           replaying = false;
   bool                           in_trx_requiring_checks = false; ///< if true, checks that are normally skipped on replay (e.g. auth checks) cannot be skipped
}
```
controller_impl 几个重要的成员
db用于存储已执行的区块，这些区块可以回滚，一旦提交就不可逆
reversible_blocks用于存储已执行，但是这些区块是可逆的
pending用于存放当前正在生产或者验证（收到）的区块
fork_db用于存放所有区块（包括自己生产和收到的），这些区块链是可分叉的
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

插件捕捉处理： chain_plugin连接该信号，
``` cpp
void chain_plugin::plugin_initialize(const variables_map& options) {
my->accepted_block_header_connection = my->chain->accepted_block_header.connect(
            [this]( const block_state_ptr& blk ) {
               my->accepted_block_header_channel.publish( priority::medium, blk );
            } );
}
```
由信号槽转播到channel，accepted_block_header_channel发布该区块,bnet_plugin订阅该channel
``` cpp
void bnet_plugin::plugin_startup() {
   my->_on_accepted_block_header_handle = app().get_channel<channels::accepted_block_header>()
                                         .subscribe( [this]( block_state_ptr s ){
                                                my->on_accepted_block_header(s);
                                         });
}
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
3. accepted_block
发射时机: commit_block函数，fork_db以及重播的处理结束后，发射承认区块的信号，携带pending状态区块数据。
插件捕捉处理
1 net_plugin连接该信号，绑定处理函数，打印日志的同时调用dispatch_manager::bcast_block，传入区块数据。send_all向所有连接发送广播
2 chain_plugin连接该信号，由信号槽转播到channel，accepted_block_channel发布该区块
``` cpp
void chain_plugin::plugin_initialize(const variables_map& options) {
    my->accepted_block_connection = my->chain->accepted_block.connect( [this]( const block_state_ptr& blk ) {
         my->accepted_block_channel.publish( priority::high, blk );
      } );
}
```
bnet_plugin订阅该channel
``` cpp
void bnet_plugin::plugin_startup() {
      my->_on_accepted_block_handle = app().get_channel<channels::accepted_block>()
                                         .subscribe( [this]( block_state_ptr s ){
                                                my->on_accepted_block(s);
                                         });
}
```
``` cpp
void on_accepted_block( const block_state_ptr& s ) {
           verify_strand_in_this_thread(_strand, __func__, __LINE__);
           const auto& id = s->id;

           _local_head_block_id = id;
           _local_head_block_num = block_header::num_from_id(id);

           if( _local_head_block_num < _last_sent_block_num ) {
              _last_sent_block_num = _local_lib;
              _last_sent_block_id  = _local_lib_id;
           }

           purge_transaction_cache();

           for( const auto& receipt : s->block->transactions ) {
              if( receipt.trx.which() == 1 ) {
                 const auto& pt = receipt.trx.get<packed_transaction>();
                 const auto& tid = pt.id();
                 auto itr = _transaction_status.find( tid );
                 if( itr != _transaction_status.end() )
                    _transaction_status.erase(itr);
              }
           }

           maybe_send_next_message(); /// attempt to send if we are idle
        }
```

on_accepted_block函数，删除缓存中的所有事务，遍历接收到的区块的事务receipt，获得事务的打包对象，事务id，在多索引表_transaction_status中查找该id，如果找到了则删除。接下来如果在空闲状态下，尝试发送下一条pingpong心跳连接信息。
3 mongo_db_plugin连接该信号，绑定其mongo_db_plugin_impl::accepted_block函数，传入区块内容。
``` cpp
my->accepted_block_connection.emplace( chain.accepted_block.connect( [&]( const chain::block_state_ptr& bs ) {
            my->accepted_block( bs );
         } ));
```
accepted_block_connection.emplace 插入信号连接器，当accepted_block从controller发出后，mongodb同样捕获到该信号，调用my->accepted_block( bs );

插件捕捉处理④： producer_plugin连接该信号，执行其on_block函数，传入区块数据。
``` cpp
my->_accepted_block_connection.emplace(chain.accepted_block.connect( [this]( const auto& bsp ){ my->on_block( bsp ); } ));
```
on_block该函数主要是
函数首先做了校验，包括时间是否大于最后签名区块的时间以及大于当前时间，还有区块号是否大于最后签名区块号。校验通过以后，活跃生产者账户集合active_producers开辟新空间，插入计划出块生产者。接下来利用set_intersection取本地生产者与集合active_producers的交集(如果结果为空，说明本地生产者没有出块权利不属于活跃生产者的一份子)。
将结果存入一个迭代器，迭代执行内部函数，如果交集生产者不等于接收区块的生产者，说明是校验别人生产的区块，如果是相等的不必做特殊处理。校验别人生产的区块，首先要在活跃生产者的key中找到匹配的key(本地生产者账户公钥)，否则说明该区块不是合法生产者签名抛弃不处理。接下来，获取本地生产者私钥，组装生产确认数据字段，包括区块id，区块摘要，生产者，签名。更新producer插件本地标志位_last_signed_block_time和_last_signed_block_num。最后发射信号confirmed_block，携带以上组装好的数据。但经过搜索，项目中目前没有对该信号设置槽connection。在区块创建之前要为该区块的生产者设置水印用来标示该区块的生产者是谁。

4. irreversible_block
发射时机
on_irreversible函数，更改区块状态为irreversible的函数，操作成功最后发射该信号。
插件捕捉处理
``` cpp
 my->irreversible_block_connection = my->chain->irreversible_block.connect( [this]( const block_state_ptr& blk ) {
         my->irreversible_block_channel.publish( priority::low, blk );
      } );
```
chain_plugin连接该信号，由信号槽转播到channel,irreversible_block_channel发布该区块。bnet_plugin订阅该channel，
``` cpp
my->_on_irb_handle = app().get_channel<channels::irreversible_block>()
                                .subscribe( [this]( block_state_ptr s ){
                                       my->on_irreversible_block(s);
                                });
```
依然线程池遍历会话，执行on_new_lib函数，当本地库领先时可以清除历史直到满足当前库，或者直到最后一个被远端节点所知道的区块。最后如果空闲，尝试发送下一条pingpong心跳连接信息。
3 mongo_db_plugin连接该信号，执行applied_irreversible_block函数，仍旧参照mongo配置项的值决定是否储存区块、状态区块以及事务数据，然后将区块数据塞入队列等待消费。
4 producer_plugin连接该信号，绑定执行函数on_irreversible_block，设置producer成员_irreversible_block_time的值为区块的时间。

5. accepted_transaction

发射时机： 
1 push_scheduled_transaction函数，推送计划事务时，将事务体经过一系列转型以及校验，接着发射该信号，承认事务。
2 push_transaction函数，新事务到大状态区块，要经过身份认证以及决定是否现在执行还是延期执行，最后要插入到pending区块的receipt接收事务中去。当检查事务未被承认时，发射一次该信号。最后全部函数处理完毕，再次发射该信号。
插件捕捉处理
1 chain_plugin连接该信号，由信号槽转播到channel，accepted_transaction_channel发布该事务。
``` cpp
my->accepted_transaction_connection = my->chain->accepted_transaction.connect(
            [this]( const transaction_metadata_ptr& meta ) {
               my->accepted_transaction_channel.publish( priority::low, meta );
            } );
```
bnet_plugin订阅该channel，线程池遍历会话，执行函数on_accepted_transaction。
``` cpp
 my->_on_appled_trx_handle = app().get_channel<channels::accepted_transaction>()
                                .subscribe( [this]( transaction_metadata_ptr t ){
                                       my->on_accepted_transaction(t);
                                });
```
``` cpp
void on_accepted_transaction( transaction_metadata_ptr t ) {
           auto itr = _transaction_status.find( t->id );
           if( itr != _transaction_status.end() ) {
              if( !itr->known_by_peer() ) {
                 //对端未接受过该交易，修改过期时间
                 _transaction_status.modify( itr, [&]( auto& stat ) {
                    stat.expired = std::min<fc::time_point>( fc::time_point::now() + fc::seconds(5), t->packed_trx->expiration() );
                 });
              }
              return;
           }

           transaction_status stat;
           stat.received = fc::time_point::now();
           //每一次事务被“accepted”，都会延时5秒钟。
           //每次一个区块被应用，所有超过5秒未被应用的但被承认的事务都将被清除。
           stat.expired  = stat.received + fc::seconds(5);
           stat.id       = t->id;
           stat.trx      = t;
           //添加到multiindex中，用于判断是否发送过该交易，防止重复发送
           _transaction_status.insert( stat );

           maybe_send_next_message();
        }
```

2 mongo_db_plugin连接该信号，执行函数accepted_transaction，校验加入队列待消费。

6. applied_transaction

发射时机
1 push_scheduled_transaction函数，事务过期时间小于pending区块时间处理后发射该信号。反之大于等于处理后发射该信号。当事务的sender发送者不为空且没有主观失败的处理后发射该信号。基于生产和校验的主观修改，主观处理后发射该信号，非主观处理发射该信号。
2 push_transaction函数，发射两次该信号，逻辑较多，这段包括以上那个函数的可读性很差，注释几乎没有。
插件捕捉处理
1 net_plugin连接该信号，绑定函数applied_transaction，打印日志。
2 chain_plugin连接该信号，由信号槽转播到channel，原理基本同上，不再重复。
3 mongo_db_plugin同上。

7. accepted_confirmation

发射时机： 
1 push_confirmation函数，推送确认信息，在此阶段不允许有pending区块存在，接着fork_db添加确认信息，发射该信号。
插件捕捉处理
2 net_plugin连接该信号，绑定函数accepted_confirmation，打印日志。
插件捕捉处理
1： chain_plugin连接该信号，由信号槽转播到channel，基本同上。