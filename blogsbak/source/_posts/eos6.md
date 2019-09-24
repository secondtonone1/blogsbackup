---
title: EOCS跨链核心技术内幕
date: 2019-02-11 22:16:36
categories: 技术开发
tags: [区块链eos]
---
EOCS跨链技术的核心就是ICP模块，ICP即Inter Chain Protocol(跨链交互协议)，下面着重介绍ICP工作原理和实现细节。
## Inter Chain Protocol(ICP)
### ICP Overview
建立于EOSIO软件堆栈之上的ICP跨链基础设施，可应用于EOSIO兼容的同构区块链的跨链通信。ICP设计之初，就考虑怎样以一种无侵入、可安全验证、去中心化的方式来实现EOS多链并行的同构跨链网络。经过对业界最前沿的跨链技术的研究（包括BTC Relay、Cosmos、Polkadot等），并结合对EOSIO软件实现细节的差异化剖析，ICP采取了比较现实的跨链方案：
#### 实现类似于轻节点的跨链基础合约，对对端区块链进行区块头跟随和验证，使用Merkle树根和路径验证跨链交易的真实性，依赖全局一致的顺序来保证跨链交易同时遵循两条链的共识。
#### 实现无需信任、去中心化的跨链中继，尽最大可能地向对端传递区块头和跨链交易，并对丢失、重复、超时、伪造的跨链消息进行合适的处理。
<!--more-->
整体框架图
![3.jpg](3.jpg)
## ICP Relay
ICP中继作为nodeos的插件，可随nodeos节点部署。部署模式上有几点需要说明：
不需要每个nodeos都开启ICP中继插件。
尽量多的nodeos开启ICP中继插件，将有助于保证跨链中继工作的不中断。
如果所有中继均瘫痪，将中断后续跨链交易进行，但不会影响已经发生的跨链交易；中继恢复后，将造成中断期某些跨链交易超时，但不会影响后续跨链交易的安全验证（这类似于所有nodeos节点瘫痪也会造成EOS区块链暂停）。
本端ICP中继可以连接对端多个ICP中继。
本端开启了ICP中继的nodeos之间可链内P2P互连(net_plugin/bnet_plugin)，但不可ICP P2P互连(icp_relay_plugin)。
本端ICP中继插件负责向本端跨链合约查询或发送交易，但不能直接向对端跨链合约查询或发送交易，而只能借助于与对端ICP中继的P2P通信。
ICP 通信和工作流程
![4.jpg](4.jpg)
## ICP Network
基于EOSIO的两条同构区块链，需对称部署一对跨链中继和跨链合约。那么要达成多条区块链之间的ICP跨链通信，可在每两条链之间都这样部署。其实，从ICP基础设施的角度来说，ICP只负责两条区块链之间的跨链通信。如果要建立无感知的平滑跨越数条区块链的跨链通信网络，可在ICP基础合约之上编写合约构建跨链网络协议(Inter Chain Network Protocol)。
ICP 跨链网络通信原理
![5.jpg](5.jpg)
## ICP 插件
在eos基础上新增了icp_plugin和eoc_plugin, 该功能为可扩充性增加，支持迭代更新，和eos核心功能零耦合。eos网络启动方式分为bnet_plugin和net_plugin模式，所以eocs实现了icp_plugin(对应icp_plugin模式)和eoc_plugin(对应net_plugin模式)
这两种插件之后会详细解析源码。
## ICP 合约底层实现
![1.jpg](1.jpg)
ICP合约底层实现分为types,merkle,icp,fork四个模块，分别为数据消息类型管理，merkle控制类，icp核心关系，fork类。具体实现细节之后会详细讲解
目前为止EOCS核心跨链内幕已经初步介绍完毕，源码解析和思想设计后续不断更新
谢谢关注我的公众号
![2.jpg](2.jpg)

