---
title: EOCS中继网络源码剖析
date: 2019-02-17 20:47:03
categories: [区块链]
tags: [区块链eos]
---
EOCS中继网络实现了两种模式，分别是eoc模式和icp模式，今天先介绍eoc模式。
## eoc_relay_plugin 源码剖析
<!--more-->
在eoc_relay_plugin.cpp中
``` cpp
static appbase::abstract_plugin& _eoc_relay_plugin = app().register_plugin<eoc_relay_plugin>();
```
向appbase中注册了eoc_relay_plugin插件。
``` cpp
void eoc_relay_plugin::set_program_options(options_description&, options_description& cfg);
```
该函数将config中配置的参数设置导入eoc_relay_plugin中，内部调用了plugin_initialize和init_eoc_relay_plugin函数，设置eoc_relay_plugin必备的参数。
接着调用
``` cpp
 void eoc_relay_plugin::plugin_startup()
      {
         ilog("starting eoc_relay_plugin");
         //sendstartaction();
         auto &chain = app().get_plugin<chain_plugin>().chain();
         FC_ASSERT(chain.get_read_mode() != chain::db_read_mode::IRREVERSIBLE, "icp is not compatible with \"irreversible\" read_mode");

         relay_->start();
         chain::controller &cc = relay_->chain_plug->chain();
         {
            cc.applied_transaction.connect(boost::bind(&eoc_icp::relay::on_applied_transaction, relay_.get(), _1));
            cc.accepted_block_with_action_digests.connect(boost::bind(&eoc_icp::relay::on_accepted_block, relay_.get(), _1));
            cc.irreversible_block.connect(boost::bind(&eoc_icp::relay::on_irreversible_block, relay_.get(), _1));
         }

         relay_->start_monitors();

         for (auto seed_node : relay_->connect_to_peers_)
         {
            connect(seed_node);
         }

         if (fc::get_logger_map().find(icp_logger_name) != fc::get_logger_map().end())
            icp_logger = fc::get_logger_map()[icp_logger_name];

         // Make the magic happen
      }
```
该函数主要是实现插件的启动，内部绑定了controller的applied_transaction,accepted_block_with_action_digests,irreversible_block信号，当收到这些信号，就会自动调用relay的三个函数。
上面就是eoc_relay_plugin的核心通信代码，内部的网络消息流程和net_plugin类似，通过icp_relay类完成网络消息的控制，以及回调函数的逻辑处理。
``` cpp
void relay::start_monitors( )
{
    connector_check.reset(new boost::asio::steady_timer( app().get_io_service()));
      start_conn_timer(connector_period, std::weak_ptr<icp_connection>());
}
```
该函数设置了定时器，检测连接是否成功，不成功则定时重连。
``` cpp
void relay::start() {
   ioc_ = std::make_unique<boost::asio::io_context>(num_threads_);

   timer_ = std::make_shared<boost::asio::deadline_timer>(app().get_io_service());

   auto address = boost::asio::ip::make_address(endpoint_address_);
   update_local_head();
    if( acceptor ) {
         acceptor->open(listen_endpoint.protocol());
         acceptor->set_option(tcp::acceptor::reuse_address(true));
         try {
           acceptor->bind(listen_endpoint);
         } catch (const std::exception& e) {
           ilog("eoc_relay_plugin::plugin_startup failed to bind to port ${port}",
             ("port", listen_endpoint.port()));
           throw e;
         }
         acceptor->listen();
         ilog("starting listener, max clients is ${mc}",("mc",max_client_count));
         start_listen_loop();
      }

}
```
该函数启动了eoc插件的网络监听事件，start_listen_loop就是控制的网络事件循环监听。
``` cpp
void relay::send_icp_net_msg(const icp_net_message& msg)
 {
    send_icp_notice_msg(icp_notice_message{local_head_});
    for( const auto& c : connections )
    {
       c->send_icp_net_msg(msg);
    }
       
 }
```
中继间通信是通过网络tcp点对点发送的，该函数就是封装了icp_net_message发送的功能。

``` cpp
void relay::on_applied_transaction(const transaction_trace_ptr& t) {
   //ilog("on applied transaction");
  //......
   for (auto& action: t->action_traces) {
      if (action.receipt.receiver != action.act.account) 
      {
         ilog("action.receipt.receiver != action.act.account");
         continue;
      }
      if (action.act.account != local_contract_ or action.act.name == ACTION_DUMMY) { // thirdparty contract call or icp contract dummy call
         //ilog("on applied transaction action dumy!!!");
         for (auto& in: action.inline_traces) {
            if (in.receipt.receiver != in.act.account) continue;
            if (in.act.account == local_contract_ and in.act.name == ACTION_SENDACTION) {
               for (auto &inin: in.inline_traces) {
                  if (inin.receipt.receiver != inin.act.account) continue;
                  if (inin.act.name == ACTION_ISPACKET) {
                     auto seq = icp_packet::get_seq(inin.act.data, st.start_packet_seq);
                     st.packet_actions[seq] = send_transaction_internal{ACTION_ONPACKET, inin.act, inin.receipt};
                  }
               }
            }
         }
      } else if (action.act.account == local_contract_) {
         if (action.act.name == ACTION_ADDBLOCKS or action.act.name == ACTION_OPENCHANNEL) {
            ilog( "on_applied_transaction ,action name is ${p}",("p", action.act.name) );
            app().get_io_service().post([this] {
               update_local_head();
            });
         } else 
         {
             //....
         }
   }

   if (st.empty()) return;

   //send_transactions_.insert(st);
    update_send_transaction_index(st);
}

```
该函数是跨链交易处理的核心逻辑，分别处理了不同类型的action，如ACTION_ADDBLOCKS，ACTION_OPENCHANNEL等。
同样，由于eoc_relay既要发送消息，也要接收消息，所以封装了几个消息处理函数如
``` cpp
void relay::handle_message( icp_connection_ptr c, const icp_notice_message & msg)
 {
     ilog("received icp_notice_message");
      if (not msg.local_head_.valid()) return;

   app().get_io_service().post([=, self=shared_from_this()] {
      if (not peer_head_.valid()) {
         clear_cache_block_state();
      }
      peer_head_ = msg.local_head_;
      });

    // TODO: check validity
    icp_peer_ilog(c, "received icp_notice_message");
    ilog("received icp_notice_message");
   
 }
```
该函数处理icp_notice_message。详细细节，读者可以下载源码仔细读一读，eoc_relay_plugin核心代码剖析就介绍这些了。