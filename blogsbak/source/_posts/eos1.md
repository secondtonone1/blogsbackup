---
title: eos源码分析和应用(一) 调试环境搭建
date: 2018-09-02 08:28:43
categories: 技术开发
tags: [区块链eos]
---
eos基于区块链技术实现的开源引擎，开发人员可以基于该引擎开发DAPP(分布式应用)。下面搭建在windows环境下的虚拟机，并且安装eos引擎，以及配合vscode实现断点调试。
<!--more-->
## 创建vmware虚拟机安装ubuntu系统
去下载vmware虚拟机，然后安装。
[vmware虚拟机链接地址](https://www.vmware.com/products/workstation-pro.html)
ubuntu系统下载16.04版本以上的，下载地址
[ubuntu下载地址](https://www.ubuntu.com/download/desktop)
下面创建虚拟机，选择创建一个新的虚拟机
![创建虚拟机](1.png)
选择自定义
![自定义](2.png)
![按步骤](3.png)
选择稍后安装系统镜像
![4.png](4.png)
操作系统选择linux
![5.png](5.png)
安装位置自己设定
![6.png](6.png)
设置处理器数量，根据自己机器酌情设置
![7.png](7.png)
内存设置，eos编译要求至少7G内存，我设置8G，如果机器内存不够，可以设置小一点，之后改eos_build.sh中的设置就可以。
![8.png](8.png)
网络设置走默认就行
![9.png](9.png)
![10.png](10.png)
![11.png](11.png)
![12.png](12.png)
存储空间我设置了80G，根据自己机器设置，至少40G空间
![13.png](13.png)
虚拟机数据存放位置
![14.png](14.png)
![15.png](15.png)
虚拟机安装好了
![16.png](16.png)
点击编辑虚拟机设置，点击cd/dvd ，选择使用ISO映像文件
![17.png](17.png)
确定后，点击运行虚拟机，自动安装ubuntu，ubuntu具体安装选择不做赘述。
## 编译eos，运行eos
1 进入自己用户目录，创建文件夹，然后clone 代码
git clone https://github.com/EOSIO/eos--recursive
![18.png](18.png)
2 下载后进入eos目录，执行eosio_build.sh脚本，出现如下显示，则编译成功。
我输入sudo ./eosio_build.sh, 等待编译完成
![19.png](19.png)
如果出现boost ,mongodb等下载失败，无法执行成功，那么修改eos/scripts/eosio_build_ubuntu.sh，注释掉connect下载等操作，然后手动下载放入eos查找的目录即可。同样的道理，内存不足7G，空间不足40G，eosio_build_ubuntu.sh脚本会exit，注释掉exit代码即可继续编译。
编译成功后，可以执行以下命令运行节点,当前目录为eos
``` bash
cd ./build/programs/nodeos
./nodeos -e -p eosio --plugin eosio::chain_api_plugin --plugin eosio::history_api_plugi
```
那另起一个shell终端，执行cleos查看当前网络信息
``` bash
cd build/programs/cleos
./cleos get info
```
eos 根目录在~/.local/share/eosio/,
~/.local/share/eosio/nodeos/config/目录下有config.ini 和genesis.json两个文件，通过配置config.ini，可以直接运行节点，不需要带参数。
config.ini 配置如下
``` bash
genesis-json = ./genesis.json
block-log-dir = blocks
readonly = 0
send-whole-blocks = true
enable-stale-production = true
http-server-address = 127.0.0.1:8888
p2p-listen-endpoint = 0.0.0.0:9876
p2p-server-address = localhost:9876
allowed-connection = any
#p2p-peer-address = 47.105.111.1:7771
p2p-peer-address = 192.168.1.59:7771
#p2p-peer-address = localhost:9877
required-participation = 33
 
 
#Private key: 5JZ5Wwb8uQbi3A7DmMsD2zevcKCYw1pxmitij1x4xCjU8gv7ucj
#Public key: EOS6a5pr4DS4CksCQSHqTdKMPbAdCyrE4b7QExDwTuCxH1vbkYMqG
 
# key for eosio 
#producer-name = eosio
private-key = ["EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV","5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3"]
 
# actinve key for bp.a
producer-name = p1
private-key = ["EOS6a5pr4DS4CksCQSHqTdKMPbAdCyrE4b7QExDwTuCxH1vbkYMqG","5JZ5Wwb8uQbi3A7DmMsD2zevcKCYw1pxmitij1x4xCjU8gv7ucj"]
 
# actinve key for bp.b
producer-name = p2
private-key = ["EOS5NiFNF4bG7T49S6f7qVXMAt4RN2WM211s77UZrwD4cz2Xu6gw9","5JKkei9CFtawsvnHt728DUQaahcjHm5nqJsNgZzna9XZKq8eA5c"]
 
# actinve key for bp.c
producer-name = p3
private-key = ["EOS59rjXxZLjRnUEdErjtCEN8fihQnMmdsWYSz7jaeruPEoSeyCHz","5JBDtjPbUeV2Hte6ZuFE5ny9RtuUujWEKG1u2yYPw2jmkCR7A4Y"]
 
# actinve key for bp.d
producer-name = p4
private-key = ["EOS5psRxWMGyQS4HPNY8fa4PDhgP53vD4AZ6w24Z9HUCTxXKEH7Ey","5JQPYAtWxdzGsJkBpHyWBV18N2rzFtMjcBwxvfndS3KXe4oQu3L"]
 
 
 
#plugin = eosio::producer_plugin
plugin = eosio::chain_api_plugin
#plugin = eosio::account_history_api_plugin
#plugin = eosio::wallet_plugin
#plugin = eosio::wallet_api_plugin
plugin = eosio::http_plugin
plugin = eosio::net_plugin
plugin = eosio::net_api_plugin
```
genesis.json
``` bash
{
  "initial_timestamp": "2018-06-08T08:08:08.888",
  "initial_key": "EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV",
  "initial_configuration": {
    "max_block_net_usage": 1048576,
    "target_block_net_usage_pct": 1000,
    "max_transaction_net_usage": 524288,
    "base_per_transaction_net_usage": 12,
    "net_usage_leeway": 500,
    "context_free_discount_net_usage_num": 20,
    "context_free_discount_net_usage_den": 100,
    "max_block_cpu_usage": 200000,
    "target_block_cpu_usage_pct": 1000,
    "max_transaction_cpu_usage": 150000,
    "min_transaction_cpu_usage": 100,
    "max_transaction_lifetime": 3600,
    "deferred_trx_expiration_window": 600,
    "max_transaction_delay": 3888000,
    "max_inline_action_size": 4096,
    "max_inline_action_depth": 4,
    "max_authority_depth": 6
  }
}

```
这样直接执行就可以了
``` bash
cd ./build/programs/nodeos
./nodeos
```
## 配置vscode，设置断点调试
eosio_build.sh脚本，把第51行CMAKE_BUILD_TYPE=Release修改成CMAKE_BUILD_TYPE=Debug，执行./eosio_build.sh,这样生成debug版本才可以断点调试。
ubuntu 软件中心下载visualstudio code， 进入软件界面，导入eos项目。
![20.png](20.png)
1 配置任务，如图所示菜单路径：任务->配置任务,选择使用模板创建tasks.json文件，MSBuild执行生成目标。
在eos工程目录下创建一个tasks.json文件，并打开，如下所示
![21.png](21.png)
按照如下修改配置
``` bash
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "label": "eosio_build",
            "type": "shell",
            "command": "cd build && make nodeos -j4",
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "problemMatcher": []
        }
    ]
}
```
2 菜单：调试->添加配置..vscode会在eos工程目录下创建launch.json文件，如下图
![22.png](22.png)
修改launch.json文件
``` bash
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
 
        {
            "name": "(gdb) Launch",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/programs/nodeos/nodeos",
            //"args": ["--genesis-json","/home/secondtonone1/.local/share/eosio/nodeos/config/genesis.json"],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}/build",
            "environment": [],
            "externalConsole": true,
            "MIMode": "gdb"
        }
    ]
}
```
mac 系统请设置MIMode成lldb形式
3 任务->运行任务，选择eosio_build，vscode会执行一次代码编译，以后修改代码后，可以直接在vs中修改代码编译
![23.png](23.png)
在main函数处设置断点
4 菜单：调试->启动调试或F5
![24.png](24.png)
程序运行到断点处暂停，可以F10单步调试，也可以F5跳过继续运行下一个节点，左侧Debug目录点击，可以看到调用的堆栈信息和变量信息。
到此为止，eos编译运行，以及调试环境搭建完了。下一篇源码分析，eos整个流程运行机制。
谢谢关注我的公众号
 ![1.jpg](1.jpg)








 













