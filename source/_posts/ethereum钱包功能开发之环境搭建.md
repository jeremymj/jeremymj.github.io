---
title: ethereum钱包功能开发之环境搭建
tags:
  - Ethereum
  - JsonRpc
abbrlink: 426a759f
date: 2020-03-01 19:15:59
---
　　最近在进行ethereum钱包功能的开发，在这里将实现钱包功能过程中遇到的问题进行记录。<!--more-->从当前的进度来看，打算分为几篇文章来记录开发过程中的经历，算是对自己工作过程中进行一个记录与总结。
## 常见工具介绍
　　在进行Ethereum平台相关内容开发时可能会看到`Geth`，`Mist`,`Truffle`,`Ganache`等，这里对他们的区别做一个简单区分：
- Geth:是Ethereum的一个客户端实现，用于节点搭建等，是使用golang来实现的；
- Parity Ethereum,号称最快、最安全、最稳定的一个Ethereum客户端实现，是使用Rust来开发的；
- Mist:是etherum官方出的一个区块链浏览器，能够让用于进行交易，合约调用等；
- Truffle：简单的理解它是一个**以太坊合约开发工具**，它可以在智能合约开发、测试、发布等过程更方便，关于Truffle更多详细资料，[查看这里]()
- Ganache：是一个为开发者提供的**私有Ethereum 区块链客户端**, 可以用于本地部署, 开发, 测试应用程序, 测试代码,[更多介绍查看这里](https://truffleframework.org/docs/getting_started/client)
## 技术架构
　　为便于提供不同平台的支持，我们使用Rust来实现钱包底层的功能，借助Rust交叉编译的便捷性，能够很方便的编译出目标平台所需要的动态库。借助Flutter在界面上实现的跨平台技术，能够解决Andorid,IOS等平台的一致性问题。关于[Rust](https://www.rust-lang.org/zh-CN/)、[Flutter](https://flutter.dev/)的介绍可以点击链接进行了解，关于搭建Rust、Flutter交叉编译环境过程,可以[参考这篇文章](https://dev.to/robertohuertasm/rust-once-and-share-it-with-android-ios-and-flutter-286o)。本文重点对支持ethereum开发所需的环境搭建进行叙述。
## 环境搭建
　　由于前期有开发[Substrate](https://substrate.dev/)的经验，知道ethereum与substrate来源的关系，开发过程中需要搭建一个节点来接收JsonRpc的请求。所以在ethereum钱包开发中也需要节点来参与调试。按照这种思路，我首先想到的是在本地直接运行一个ethereum节点，接入测试链或者搭建一个私链。由于使用Rust的原因，节点的选择上就自然的想到了Parity Ethereum（由于某些原因现在已经转移到[OpenEthereum DAO](https://discordapp.com/invite/JCuNu3m)）。
## 节点程序编译
　　从github上取回[最新的代码](https://github.com/OpenEthereum/open-ethereum)，进入项目的根目录，直接使用rust的编译命令`Cargo build --release`进行编译，等待几分钟之后就可以编译完成，编译生成的可执行程序在`./target/release/`目录下。直接运行`./parity`默认会启动一个全节点，会进行区块数据同步等操作，针对我们仅仅是想测试来说，不需要接入主网参与数据同步。
### 可执行程序介绍
在运行`./parity --help`后，可以看到程序支持的一些可选参数：
- --jsonrpc-apis，可选的值有：all, safe, debug, web3, net, eth, pubsub, personal, signer, parity, parity_pubsub,parity_accounts, parity_set, traces, rpc, secretstore, shh, shh_pubsub.这些rpc接口不是在每个节点都对外开放的，节点的类型也很多，所以在开发的时候确保所选的节点有开放你所需要的rpc接口这点很重要，涉及到技术方案的选择以及实现难度、工作量等问题。
- --chain=[CHAIN]，可选的值有poacore, tobalaba, expanse,musicoin, ellaism, mix, callisto, morden, ropsten, kovan, rinkeby,goerli, kotti, poasokol, testnet, or dev，根据需要启动的节点类型，指定对应的chain。
- --bootnodes,这个值是需要接入的节点地址，每个节点启动的时候都会输出当前自身节点的地址，用于其他节点的直接连接，地址格式如下：`enode://xxxx@ip:port`
　　除了跟启动参数外，`./parity`还提供很多工具命令，比如节点自身也具有钱包管理的功能，能够生成账户地址:运行`parity account new`，输入密码之后就会生成一个账户地址，同时该账户的私钥信息保存在`~/.local/share/io.parity.ethereum/keys/ethereum/`路径下。若是在启动节点的时候需要启用挖矿功能，则直接添加 `--author [地址]`，比如：`./parity --author 0037a6b811ffeb6e072da21179d11b1406371c63`,地址是去掉`0x`前缀的
## 节点启动
　　由于我仅仅是想测试依赖节点的Rpc服务来验证钱包拼接的数据结构是否满足接口要求，所以我使用的启动方式是使用`parity --config dev`启动ETH开发节点，同时该节点默认存在一个地址`0x00a329c0648769a73afac7f9381e08fb43dbea72`，对应的私钥为：`0x4d5db4107d237df6a3d58ee5f70ae63d73d7658d4026f2eefd2f204c81682cb7`（这个私钥仅用于开发测试，正式使用时请保存好自己的私钥），该账户中存在一定数目的测试ETH　token，可以用于后续的开发测试。
## 常用Rpc
　　虽然有不同的类型的节点，但是他们都有eth开头的最基础Rpc接口，我们可以借助这些接口实现我们的交易数据提交等功能，以下列举出的常用接口使用方式：
- `eth_getTransactionCount`，熟悉区块链开发的都清楚每笔交易都需要一个nonce,用于避免交易的重放攻击，确保交易的执行顺序，在拼接交易之前需要获取交易发起方当前总共发起的交易数，比如`
curl --data '{"method":"eth_getTransactionCount","params":["0x00a329c0648769a73afac7f9381e08fb43dbea72"],"id":1,"jsonrpc":"2.0"}'  -H "Content-Type: application/json" -X POST localhost:8545`
- `eth_sendRawTransaction`，用于构造离线交易签名，这种方式在钱包开发里面使用的场景比较多，需要依赖特定的节点就可以将交易提交到链上进行验证，这里稍微麻烦的是Rpc请求参数的构造，这在后面的文章做详细的介绍。该API请求数据格式:`curl --data '{"method":"eth_sendRawTransaction","params":["0xf86d808253e8837a1200941c9baedc94600b2d1c8a6d2bad1744e6182f300e8609184e72a0008568656c6c6f29a0a65b500258e5cf458db262758786e5c327285c924df687b8e9ce28e2fccb9451a07359f8dba84300d950bb534ac71a8d0eccb2a7e01a8cb70041c886588424be1c"],"id":1,"jsonrpc":"2.0"}' -H "Content-Type: application/json" -X POST localhost:8545`。 
- `eth_getTransactionByHash`，在`eth_sendRawTransaction`的Rpc请求中会返回当前交易的结果。假如交易参数验证通过，会返回当前交易的hash，通过该Rpc可以查看该交易的具体详情，该API数据请求格式如下：`curl --data '{"method":"eth_getTransactionByHash","params":["0x1486d4ff14e7fc6f991c17633e2b7a65ae52fc7a1be33845773a561a6955929e"],"id":1,"jsonrpc":"2.0"}' -H "Content-Type: application/json" -X POST localhost:8545`
- `eth_getBalance`，针对交易来说，比较重要的一个功能是查询指定地址上的ETH token数量,该API数据请求格式如下:`curl --data '{"method":"eth_getBalance","params":["0x00a329c0648769a73afac7f9381e08fb43dbea72"],"id":1,"jsonrpc":"2.0"}' -H "Content-Type: application/json" -X POST localhost:8545`
　　以上是在钱包开发中使用频率较高的一些RPC接口，关于parity Ethereum的介绍以及使用文档，可以查看[这里](https://wiki.parity.io/Parity-Ethereum)了解更多。
## 总结
　　以上就是使用Ethereum来实现钱包功能所需要了解的一些内容，关于用户账户、交易构成、合约调用等内容后续慢慢的更新。虽然前期有substrate开发的经验，但是在实现的过程中也遇到了一些问题，我想把遇到的问题记录下来，再把解决问题过程中查阅的内容消化后记录下来，算是自己的学习笔记，可以用于自己以后复习。
