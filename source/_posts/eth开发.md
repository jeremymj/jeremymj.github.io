---
title: eth开发
abbrlink: 8a63a1af
date: 2020-01-12 23:13:29
tags: 
- 区块链
- 签名
---
使用parity 版本来搭建开发测试节点；
 <!-- more -->
## 启动参数
使用parity客户端，启动命令:
--jsonrpc-apis=
可选项: all, safe, debug,
        web3, net, eth, pubsub, personal, signer, parity, parity_pubsub,
        parity_accounts, parity_set, traces, rpc, secretstore, shh, shh_pubsub.

--jsonrpc-experimental


--chain=[CHAIN]

classic, poacore, tobalaba, expanse,
        musicoin, ellaism, mix, callisto, morden, ropsten, kovan, rinkeby,
        goerli, kotti, poasokol, testnet, or dev

--bootnodes 
enode://a5fe4a83c7ef89a68918ae2fe1ecec0f4fc2d3582c0fafe2c6d5695f2d2511f66651bf756fbae90f6ca82dd7efeb525d80f48b5be305d1cd4149100300f51623@192.168.1.6:30303

 --db-path

 --base-path=[PATH]

/home/jeremy/work/eth-db

parity --geth --chain ropsten --author 0xda9b1a939350dc7198165ff84c43ce77a723ef73

若需要挖矿，则需要先生成账户，运行`parity account new`，得到地址 0x3cdc082d3b6b5e0588501552d35ed3bb5feccc70；
允许节点挖矿：`parity --author 0037a6b811ffeb6e072da21179d11b1406371c63`

生成地址:0x7b02dca46711be2664310f4fe322c8bd35a9bd2a

测试链:
parity --config dev

address:0x00a329c0648769a73afac7f9381e08fb43dbea72

pri_key:0x4d5db4107d237df6a3d58ee5f70ae63d73d7658d4026f2eefd2f204c81682cb7

{"method":"eth_getTransactionCount","params":["0x00a329c0648769a73afac7f9381e08fb43dbea72"],"id":1,"jsonrpc":"2.0"}

{"method":"eth_getBalance","params":["0x00a329c0648769a73afac7f9381e08fb43dbea72"],"id":1,"jsonrpc":"2.0"}

{"method":"eth_call","params":[{"from":"0x00a329c0648769a73afac7f9381e08fb43dbea72","to":"0xa94f5374fce5edbc8e2a8697c15331677e6ebf0b","value":"0x186a0"}],"id":1,"jsonrpc":"2.0"}

{"method":"eth_chainId","params":[],"id":1,"jsonrpc":"2.0"}

{"method":"eth_blockNumber","params":[],"id":1,"jsonrpc":"2.0"}

{"method":"eth_accounts","params":[],"id":1,"jsonrpc":"2.0"}

## 交易生成

1、地址生成；
2、密钥管理；
3、交易生成
     生成交易
     transfer（from,to）

## 交易

{"method":"eth_sendTransaction","params":[{"from":"0x00a329c0648769a73afac7f9381e08fb43dbea72","to":"0x7b02dca46711be2664310f4fe322c8bd35a9bd2a","gas":"0x76c0","gasPrice":"0x9184e72a000","value":"0x9184e72a","data":"0xd46e8dd67c5d32be8d46e8dd67c5d32be8058bb8eb970870f072445675058bb8eb970870f072445675"}],"id":1,"jsonrpc":"2.0"}

## 交易验证

https://download.mycrypto.com/

82c445eb270de865e83576a9f80964a60593295845a08e94d88eb61a994eb155

## RPC 调用
{"method":"eth_call","params":[{"from":"0x407d73d8a49eeb85d32cf465507dd71d507100c1","to":"0xa94f5374fce5edbc8e2a8697c15331677e6ebf0b","value":"0x186a0"}],"id":1,"jsonrpc":"2.0"}
