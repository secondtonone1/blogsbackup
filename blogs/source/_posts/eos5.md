---
title: EOCS框架概述和剖析
date: 2019-02-10 12:28:52
categories: [区块链]
tags: [区块链eos]
---
## 什么是EOCS？
EOCS(Enterprise Operation Cross System),是一个基于eosio底层框架实现的企业级跨链操作系统，旨在实现和EOS主链通信的并行链，是真正意义的跨链，支持高效稳定的跨链交易，为跨链生态建设提供更稳定和安全的平台。作为可与EOS主链交互操作的第一条并行链，EOCS Chain力图实现安全可靠、快捷便利的跨链资产转移、跨链智能合约调用。任何人都可以在EOCS Chain并行链上开发或使用跨链DAPP。

## 为什么选择EOCS？
### 互操作性
EOCS Chain并行链允许EOS主链与主流公有链、联盟链、私有链互相通信和价值交换。
### 可扩展性
通过多链互联互操作，EOCS Chain可帮助EOS实现史无前例的无限扩展。
### 开发友好性
EOCS Chain延续了EOS软件堆栈的WebAssembly机制，可以非常轻松地开发DAPP。
<!--more-->
## EOCS发展路径
![1.jpg](1.jpg)

## EOCS核心竞争力
### EOCS Chain并行链与EOS主链之间的同构跨链，涉及以下组件：
同构跨链协议（Isomorphic Inter-Chain Protocol, ICP） 同构跨链合约，在并行链和主链上同时部署，支持跨链协议包的解析，证明的验证和存储，以及EOS原生币（EOS）、EOCS Chain原生币（EOCS）、EOS代币的跨链资产转移 同构跨链通道，通过逻辑证明确保通道建立的稳定性和安全性。 中继者，将跨链协议包在并行链和主链之间安全快速地传输
### EOCS Chain异构跨链尝试与探索
我们相信未来的区块链不仅在去中心化社区中得到商业落地前景，千万中小企业同样需要区块链作为价值传递的基础服务，未来不仅是公有链、联盟链还是企业内部的私有链，都需要在一个公用网络中进行价值传递和证明。 作为第一条EOS同构并行链，我们将在开发EOCS Chain的基础上，继续探索和研究异构链的跨链协议，不仅要为EOS生态做出支持百万TPS的并行链体系，更要为整个EOS体系连接异构链做出创造性的贡献，作为连接EOS主链及整个EOS跨链群体系与其他区块链链的纽带，为所有异构区块链公链、联盟链、私有链实现安全、快捷、无限扩展的区块链生态体系!
## EOCS整体框架简图
![2.jpg](2.jpg)
## 如何使用EOCS
### 编译和部署
EOCS 支持多种Linux操作系统，mac，centos，ubuntu等等，可以去github下载源码并编译，源码下载地址，[https://github.com/eocschain/eocs](https://github.com/eocschain/eocs)。
在自己的工作目录(可自己设定)执行命令 git clone https://github.com/eocschain/eocs 更新下载源码。下载后文件组织结构如下
![3.png](3.png)
在该目录下执行eosio_build.sh,会生成build目录，执行成功会提示build success!!!
### 填写配置
在~/.local/share/eosio目录下有config和data文件夹，修改config.ini即可。
``` python
# Override default WASM runtime (eosio::chain_plugin)
wasm-runtime = wabt

# print contract's output to console (eosio::chain_plugin)
# 方便观察跨链合约打印信息
contracts-console = true

# The local IP and port to listen for incoming http connections; set blank to disable. (eosio::http_plugin)
# 链1为127.0.0.1:8888，链2为127.0.0.1:8889
http-server-address = 127.0.0.1:8888 # 或 127.0.0.1:8889

# The endpoint upon which to listen for incoming connections (eosio::icp_relay_plugin)
# 链1为0.0.0.0:8765，链2为0.0.0.0:8766
icp-relay-endpoint = 0.0.0.0:8765 # 或 0.0.0.0:8766

# The number of threads to use to process network messages (eosio::icp_relay_plugin)
# icp-relay-threads = 

# Remote endpoint of other node to connect to (may specify multiple times) (eosio::icp_relay_plugin)
# 链1为127.0.0.1:8766，链2为127.0.0.1:8765；其实只要填一个，使得两条链的ICP插件能够连接上
icp-relay-connect = 127.0.0.1:8766 # 或 127.0.0.1:8765

# The chain id of icp peer (eosio::icp_relay_plugin)
# 链1填写链2的chain id，链2填写链1的chain id，可参考后文获取方式后再填写
icp-relay-peer-chain-id = 630f427c3007b42929032bc02e5d6fded325b3e2caf592f963070381b2787a9d

# The peer icp contract account name (eosio::icp_relay_plugin)
# 对端ICP合约账户名；链1填写链2上跨链合约账户名，链2填写链1上跨链合约账户名
icp-relay-peer-contract = eocseosioicp

# The local icp contract account name (eosio::icp_relay_plugin)
# 本端ICP合约账户名；链1填写链1上跨链合约账户名，链2填写链2上跨链合约账户名
icp-relay-local-contract = eocseosioicp

# The account and permission level to authorize icp transactions on local icp contract, as in 'account@permission' (eosio::icp_relay_plugin)
# ICP插件向本端ICP合约发送交易时使用的账户名
icp-relay-signer = eocseosrelay@active

# The actual host:port used to listen for incoming p2p connections. (eosio::net_plugin)
# 链1为0.0.0.0:9876，链2为0.0.0.0:9877
p2p-listen-endpoint = 0.0.0.0:9876 # 或 0.0.0.0:9877


# Limits the maximum time (in milliseconds) that is allowed a pushed transaction's code to execute before being considered invalid (eosio::producer_plugin)
# 设置足够大的最大交易执行时间，可参看ICP Challenges中关于计算量的说明
max-transaction-time = 300

# ID of producer controlled by this node (e.g. inita; may specify multiple times) (eosio::producer_plugin)
# 这里测试链仅使用生产者eosio
producer-name = eosio

# 填写账户eosio的公私钥，这里使用了默认值
signature-provider = EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV=KEY:5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3

# 插件
plugin = eosio::chain_api_plugin
```
### 启动节点
进入build/programs/nodes,执行nodes,启动节点
到此为止，EOCS概述和节点启动简介完毕，下一篇为大家带来EOCS的跨链设计和源码剖析










