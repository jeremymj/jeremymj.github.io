---
title: Substrate 问题记录
tags: 区块链
abbrlink: dd1b667b
date: 2020-01-11 21:09:14
---
## 节点运行
要是以前有通过命令行运行substrate,建议先执行一次 `substrate pruge-chain`
以开发模式运行 `substrate --dev` <!-- more -->
若节点在指定的时间内不能生成块 可以使用参数 `--execution-block-construction Native` <br>
若节点需要对外提供访问服务：可以在启动参数添加 `--ws-external --rpc-external` <br> 
因此一个完整的启动命令可以是：
`substrate --dev --execution-block-construction Native --ws-external --rpc-external`
## 日志过滤
当节点运行起来后，若需要查看日志 可以使用参数 `--log debug`会输入大量的debug 日志,只是大量这种日志 会影响观察 可以使用设置命令行参数
`RUST_LOG="runtime=debug,substrate=error"`，这种方式启动起来后日志就会大量的减少,同时会把所有的错误日志显示出来

#[cfg(feature="std")]
println!("fee is:{:?}",fee);

启动命令：`RUST_LOG="runtime=debug,substrate=error" ./substrate --dev --execution-block-construction Native --ws-external --rpc-external`

## 模块编写教程

自定义Module，编写自己基于链上的应用，（如加密猫），使用脚本命令 curl https://getsubstrate.io -sSf 安装环境；

数据还原 ./target/release/substratekitties purge-chain --dev
运行   ./target/release/substratekitties --dev

在使用polkadot-js/api

调用keyring相关函数，遇到ExtError: @polkadot/wasm-crypto has not been initialized 这种问题

是因为keyring 依赖于sr25519,它只能通过WASM调用，并且是异步方式初始化；所有解决方式：

- 要是单机使用，可以在调用new 之前，明确的初始化加密模块；
- 在api中并行调用，在构造keyring之前，先调用Api.call，因为API初始化是异步的，因此也要初始化crypto init

## 源码编译

可以编译里面的node 节点或者node-template, 编译的时候 需要使用nightly版本的toolchain (20190903),可以在项目的根目录下通过配置文件指定 rust-toolchain；
echo "nignely" > rust-toolchain

## 智能合约的编写

### 编写工具

- 基础工具安装
sudo apt install -y curl jq tar
WABT: The WebAssembly Binary Toolkit:提供WebAssembly 二进制工具集，包含的工具如下：
- **wat2wasm**: translate from WebAssembly text format to the WebAssembly binary format
- **wasm2wat**: the inverse of wat2wasm, translate from the binary format back to the text format (also known as a .wat)
- **wasm-objdump**: print information about a wasm binary. Similiar to objdump.
- **wasm-interp**: decode and run a WebAssembly binary file using a stack-based interpreter
- **wat-desugar**: parse .wat text form as supported by the spec interpreter (s-expressions, flat syntax, or mixed) and print "canonical" flat format
- **wasm2c**: convert a WebAssembly binary file to a C source and header
- **wasm-strip**: remove sections of a WebAssembly binary file
- **wasm-validate**: validate a file in the WebAssembly binary format
- **wast2json**: convert a file in the wasm spec test format to a JSON file and associated wasm binary files
- **wasm-opcodecnt**: count opcode usage for instructions
- **spectest-interp**: read a Spectest JSON file, and run its tests in the interpreter

使用脚本安装：`curl https://raw.githubusercontent.com/paritytech/scripts/master/install-wasm-binaries.sh -sSf |bash -s`,

`cargo install pwasm-utils-cli --bin wasm-prune --force`

- 在安装基础的编译环境后，可以编译智能合约编写工具 使用命令

  cargo install --force --git https://github.com/paritytech/ink cargo-contract，在安装成功后，可以使用命令：cargo contract new 合约名
  创建合约的提示性代码

## 编译ABI文件

  创建合约ABI json文件 `cargo contract generate-metadata`
  
  当使用substrate 版本v1.0.0时 ink 使用版本efe69028cc5bd9ec86bfc15f52162b39c560c194 可以正常的使用
  
## 创建智能合约

### 支持的数据类型

在合约中使用可以存储一些简单的值，使用storage::Value\<T\>,其中T包含简单的基本类型，比如bool,, u{8,16,32,64,128}、i{8,16,32,64,128}, AccountId, Balance 以及他们的 tuples and arrays类型。

```rust
 struct MyContract {
        // Store some AccountId
        my_address: storage::Value<AccountId>,
        // Store some Balance
        my_balance: storage::Value<Balance>,
    }
```

也可以使用复杂点值类型，比如String或者map形式，在使用的时候 需要指定引入的特定路径；

```rust

use ink_core::env::{AccountId, Balance};
// Note that you will need to import the `String` type to use it
use ink_core::memory::string::String;

contract! {
    struct MyContract {
        // Store a string
        my_string: storage::Value<String>,
        // Store a key/value map; AccountId -> Balance
        my_map: storage::HashMap<AccountId, Balance>,
    }
}
```

### 合约的发布

编写的合约必须要 impl Deploy  trait,实现包含在里面的方法，deploy 这个方法在合约发布的时候只执行一次，在执行的时候，可以传递参数，但是传递的参数类型有严格的限制，当前只支持用户传递基本的数据类型比如：bool, u{8,16,32,64,128}, i{8,16,32,64,128} 和  SRML primitives  中定义的，比如AccountId、 Balance这种类型

```rust
contract! {
    struct MyContract {
        ...
    }

    impl Deploy for MyContract {
        fn deploy(&mut self, a: bool, b: i32) {
            // Deployment logic that runs once upon contract creation using `a` and `b`
        }
    }
}
```

### 注意事项

在合约存储里面使用到的 Value，为了能够正确的访问，需要在deploy中进行初始化。假如没有初始化在和约中定义的 storage::Value 属性，在使用他们的时候，会造成合约panic！

```rust
contract! {
    struct MyContract {
        // Store a bool
        my_bool: storage::Value<bool>,
        // Store a key/value map
        my_map: storage::HashMap<AccountId, Balance>,
    }

    impl Deploy for MyContract {
        /// Initializes our state to `false` upon deploying our smart contract.
        fn deploy(&mut self) {
            self.my_bool.set(false);
        }
    }
    ...
}
```

### 功能实现

将Public 和Private 方法功能单独写，Public 方法写pub(external)

```rust
impl MyContract {
    // Public functions go here
    pub(external) fn my_public_function(&self) {
        ...
    } 
}

impl MyContract {
    // Private functions go here
    fn my_private_function(&self) {
        ...
    }
}

```

### 合约测试

cargo +nightly test

### 合约发布

## 遇到的问题

- substarate master分支的代码 需要使用nightly版本编译链版本来进行编译；
- 在使用substrate --dev方式启动的时候，出现 DB corrupted: Invalid argument: You have to open all column的问题，需要先删除指定内容：rm -rf ~/.local/share/substrate/

## 节点启动

### 启动私链

- 创建启动参数模板 `./target/release/node-template build-spec --chain=local > customSpec.json`
- 修改`customSpec.json`中 aura、grandpa中包含的地址，其中aura 中的地址用于区块的生成，grandpa中的地址用于最终块的确认；
- 使用命令 `./target/release/node-template build-spec --chain customSpec.json --raw > customSpecRaw.json`生成启动需要的创世文件
- 在所有的节点中，要作为验证者，都需要使用相同的customSpecRaw.json文件来启动
- 最先启动的节点使用命令：`./node-template --chain ./customSpecRaw.json --base-path /tmp/node1 --port 30333 --ws-port 9944 --rpc-port 9933 --validator --name node1`
- 后续启动的节点，需要增加根节点的地址`./node-template --chain ./customSpecRaw.json --base-path /tmp/node2 --port 30334 --ws-port 9945 --rpc-port 9934 --validator --name node2 --bootnodes /ip4/192.168.1.6/tcp/30333/p2p/QmVPKXseHEdaDBBLonLyJfWwUEQwV9XNAuEZNFxtJu4dnS`
- 在启动成功后，需要通过rpc方式将在 `customSpec.json`指定地址对应的助记词，aura、grandpa算法需要的公钥传入到对应的节点中；
- 将key传入节点后，需要重新启动节点，这样节点就可以正常的生成块、确认块了；
- 若在启动的时候，没有添加 --validator 这个参数，则该节点就只能同步数据，不能参与区块的生成与最终块确认；


启动之前，需要先生成密钥对SR25519 用于块生成，ED25519 用于最终确认区块：
密钥对1：

Sr25519: aura

- 助记词：`clip organ olive upper oak void inject side suit toilet stick narrow`
- Secret seed: 0x4bd2b2c1dad3dbe3fa37dc6ad5a4e32ddd8ad84b938179ac905b0622880e86e7
- Public key (hex): 0x9effc1668ca381c242885516ec9fa2b19c67b6684c02a8a3237b6862e5c8cd7e
- Account ID: 0x9effc1668ca381c242885516ec9fa2b19c67b6684c02a8a3237b6862e5c8cd7e
- SS58 Address: 5FfBQ3kwXrbdyoqLPvcXRp7ikWydXawpNs2Ceu3WwFdhZ8W4

Ed25519: gran

- Secret phrase `clip organ olive upper oak void inject side suit toilet stick narrow` is account:
- Secret seed: 0x4bd2b2c1dad3dbe3fa37dc6ad5a4e32ddd8ad84b938179ac905b0622880e86e7
- Public key (hex): 0xb48004c6e1625282313b07d1c9950935e86894a2e4f21fb1ffee9854d180c781
- Account ID: 0xb48004c6e1625282313b07d1c9950935e86894a2e4f21fb1ffee9854d180c781 
- SS58 Address: 5G9NWJ5P9uk7am24yCKeLZJqXWW6hjuMyRJDmw4ofqxG8Js2

密钥对2：

SR25519

- Secret phrase `paper next author index wedding frost voice mention fetch waste march tilt` is account:
- Secret seed: 0x4846fedafeed0cf307da3e2b5dfa61415009b239119242006fc8c0972dde64b0
- Public key (hex): 0x74cca68a32156615a5923c67024db70da5e7ed36e70c8cd5bcf3556df152bb6d
- Account ID: 0x74cca68a32156615a5923c67024db70da5e7ed36e70c8cd5bcf3556df152bb6d
- SS58 Address: 5EhrCtDaQRYjVbLi7BafbGpFqcMhjZJdu8eW8gy6VRXh6HDp

ED25519

- Secret phrase `paper next author index wedding frost voice mention fetch waste march tilt` is account:
- Secret seed: 0x4846fedafeed0cf307da3e2b5dfa61415009b239119242006fc8c0972dde64b0
- Public key (hex): 0x0fe9065f6450c5501df3efa6b13958949cb4b81a2147d68c14ad25366be1ccb4
- Account ID: 0x0fe9065f6450c5501df3efa6b13958949cb4b81a2147d68c14ad25366be1ccb4
- SS58 Address: 5CRZoFgJs4zLzCCAGoCUUs2MRmuD5BKAh17pWtb62LMoCi9h

密钥对3:
sr25519::

- Secret phrase `crack amused shed reduce consider phone stage sniff drama clinic unveil retire`
- Public key:0x0668354f654d4d7954a90b3819f32ed05eaed8394cc3ae4c75c803e92f527168
- Account ID:0x0668354f654d4d7954a90b3819f32ed05eaed8394cc3ae4c75c803e92f527168
- SS58 Address: 5CD76oHb5PQRxvga6kWVvZxRD35gkUq1k2q87DPVc73L9abK

ed25519

- Secret phrase `crack amused shed reduce consider phone stage sniff drama clinic unveil retire`
- Public key:0x0ec5a5a5affcadc155ce1c092e112041a64916293397a9c502a8a50744fa1ea7
- Account ID:0x0ec5a5a5affcadc155ce1c092e112041a64916293397a9c502a8a50744fa1ea7
- SS58 Address: 5CQ5Evw5ZcYKLpcf4DWwEazR9qR6Hrk3MNCUc6Lyi6imyzJy

密钥对4:
sr25519::

- Secret phrase `hollow pigeon project utility island laundry spell six recipe soccer swing gold`
- Public key:0x9831764843144e14ee80acfee2ef8ab3ad28904af20078139bd5ac1b5f17bf0c
- Account ID:0x9831764843144e14ee80acfee2ef8ab3ad28904af20078139bd5ac1b5f17bf0c
- SS58 Address: 5FWFq2mwnYN11dYQ5uRF6oGm8i4WDY8x65w4DZTTuPwJWLTb

ed25519

- Secret phrase `hollow pigeon project utility island laundry spell six recipe soccer swing gold`
- Public key:0x74335f953d21b6de5c5b4602d03b6905c55394cd77a3dac43dc2e394283ca396
- Account ID:0x74335f953d21b6de5c5b4602d03b6905c55394cd77a3dac43dc2e394283ca396
- SS58 Address: 5Eh4fymh2VWTkhevSy28giV2Vnv3Lbz2Lnq8tJciud7usKVx

#### 启动默认私链

- ./substrate --base-path /tmp/alice --chain=local --alice --port 30333 --ws-port 9944 --rpc-port 9933 --telemetry-url ws://telemetry.polkadot.io:1024

- ./substrate --base-path /tmp/bob --chain=local --bob --port 30332 --execution-block-construction Native --ws-external --rpc-external --validator --pool-limit 10000 --bootnodes /ip4/127.0.0.1/tcp/30333/p2p/QmNNfEjLjyZZ8kT8CEDFqCUs9XKWBCysVHRPqjxM16Pz6A

#### 启动自定义私链

种子节点:

- ./substrate --base-path /home/jeremy/work/chain-data/node1 --chain ./customSpecRaw_node.json --port 30333 --ws-port 9944 --rpc-port 9933 --validator --name node1  --pool-limit 10000

验证节点:

- ./substrate --base-path /home/jeremy/work/chain-data/node2 --chain ./customSpecRaw_node.json --port 30334 --ws-port 9945 --rpc-port 9934  --validator  --pool-limit 10000  --bootnodes /ip4/127.0.0.1/tcp/30333/p2p/QmQMqUY9T1Szfj1zk4VLEE7zAR72tpt2EvdZSFPXEwprBU

需要注意保存数据的位置：默认情况下`/tmp`路径下的数据在重启之后，数据就消失了

**这两种启动方式的区别：最主要区别为使用的密钥不一样**

## polkadot-js/api

调用keyring相关函数，遇到ExtError: @polkadot/wasm-crypto has not been initialized 这种问题

是因为keyring 依赖于sr25519,它只能通过WASM调用，并且是异步方式初始化；所有解决方式：

- 要是单机使用，可以在调用new 之前，明确的初始化加密模块；
- 在api中并行调用，在构造keyring之前，先调用Api.call，因为API初始化是异步的，因此也要初始化crypto init 

### 使用js 查看链相关的参数

当api链接到一个节点，第一件事情是获取节点的`metadata`,元数据提供了当前节点哪些方法可以使用。api 的调用格式为: `api.<type>.<module>.<section>`

api提供了几个快捷使用的默认参数:

- `api.genesisHash` - The genesisHash of the connected chain
- `api.runtimeMetadata` - The metadata as retrieved from the chain
- `api.runtimeVersion` - The chain runtime version (including spec/impl. versions and types)
- `api.libraryInfo` - The version of the API, i.e. @polkadot/api v0.90.1

在使用js keyring 来导入备份的密码时，若要签名，需要先将pair解密；

```js
  const account_obj = JSON.parse(account_str);//将备份的json字符串转换成json对象
  const keyring = new Keyring({ type: 'sr25519' });//初始化keyring
  const from_account = keyring.addFromJson(account_obj);//得到account实例
    from_account.decodePkcs8("123456");//使用密码进行解密，用于签名 可以通过 islock()方法来判断是否需要解密，这样就可以在后续的签名中直接使用该账户了，为确保安全在签名完成后需要将该账户继续锁住，使用 lock()方法
```

在substrate 中，关于交易的检查相关操作 是通过trait的方式来定义的，具体内容可以查看位置
`primitives/sr-primitives/src/traits.rs`

## 功能测试

通过终端方式 使用websocket 提交jsonrpc，使用方式为：`wscat -c ws://127.0.0.1:9944`，若是没有安装`wscat`,可以使用命令：`sudo apt install node-ws`，若在启动的时候，提示`Error: Cannot find module 'ws'`模块没有找到的错误，可以试试使用管理员的方式来运行`sudo`

{"id":44,"jsonrpc":"2.0","method":"state_subscribeStorage","params":[["0xe70a9997c8a0d1ae4b47094ca78234e0da641adc9838487b4fe5b60ab577daf3"]]}

## 模块调试

在模块开发中，若需要新增加自定义的结构体，同时该结构体具有debug功能，定义如下:

```rust
use sp_runtime::RuntimeDebug;

#[cfg_attr(feature = "std", derive(Serialize, Deserialize, Debug))]
#[derive(Encode, Decode, Clone, PartialEq, Eq, RuntimeDebug)]
pub struct Offer<Balance,AccountId> {
    pub order: Order<Balance,AccountId>,
    pub sender: AccountId,
}
```

需要在substrate中使用一些标准库的代码，可以使用如下方式:`sp_std::prelude::*`

同时在Cargo.toml文件 `[feature]` 部分添加一行 `sp-std/std`，

```toml
[dependencies]
rstd = { package = "sp-std", git = "https://github.com/blockxlabs/substrate", branch = "master", default-features = false }

[features]
default = ["std"]
std = [
    ...
    "rstd/std",
    ...
]
```

The portion of the file we're interested in is the Aura authorities (used for creating blocks) and the GRANDPA authorities (used for finalizing blocks). That section looks like this

怎么来运行一个自己的公链，运行这样的一个链 需要的配置文件参数怎么来设置？
是否需要一个界面来查看当前节点的出块情况？
跟节点的nodeid 是怎么算出来的？

RPC调用 存在最大连接数限制
发布的时候 需要生成节点使用的密钥，但是针对现存在的代码，node-template的代码版本GenesisConfig 没有更新；

需要知道自己怎么来修改node中的验证节点

节点的token机制怎么来设计？（这一部分暂时不对外公开）

针对节点，对外暴露的方式为 用户通过证书号来进行查询；

node_template node中GenesisConfig 结构体相应的元素不一样？

以私链方式运行，出块时间变为60s 但是以开发模式运行节点，出块时间为6s

前4个块速度很慢，以私链方式启动，产生块的时间是设置的2倍时间；

设置期望出块时间为3s,实际在运行的过程中，当数据量大的情况下，实际出块时间最差发现有8s的；为确保交易最终被确认，需要的时间为实际出块的3倍；

查看一个区块里面的交易个数？是否能够扩大交易个数；

node 验证节点能否动态的增加？

怎么还原？？

运行时 能否使用 同步节点 不参与验证 只是数据同步？


区块数据的导出与导入；

部署节点，需要部署的节点有同步数据节点，验证节点；在验证节点，尽量不要使用`--rpc-external` and `--ws-external`方式

在节点在运行的期间，需要对节点状况进行监控？这个监控使用哪种方式呢 监控对象为: 磁盘空间

查看功能模块介绍：palletStaking、palletBabe、palletIndices、palletBalances、palletSession、palletDemocracy、palletTreasury、palletGrandpa

### 密钥类型

在初始化的时候，initial_authorities包括的地址如下: Vec<(AccountId, AccountId, GrandpaId, BabeId, ImOnlineId, AuthorityDiscoveryId)>,是用过**get_authority_keys_from_seed**这个函数得到的，通过seed，会生成几种类型的id：

AccountId：  包含`get_account_id_from_seed`是使用sr25519算法得到的，

GrandpaId 使用算法ed25519

BabeId 是AuthorityId，通过`pub type AuthorityId = app::Public;` app_crypto!(sr25519, BABE);
ImOnlineId： `pub type AuthorityId = app_sr25519::Public;`
AuthorityDiscoveryId：pub type AuthorityId = app::Public; app_crypto!(sr25519, AUTHORITY_DISCOVERY);

```
   [
    "5GNJqTPyNqANBkUVMN1LPPrxXnFouWXoe2wNSmmEoLctxiZY",//用于控制节点手续费账户 sr25519
    {
        "grandpa": "5FA9nQDVg267DEd8m1ZypXLBnvN7SFxYwV7ndqSYGiN9TTpu", //ed25519
        "babe": "5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY", //sr25519
        "im_online": "5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY",//sr25519
        "authority_discovery": "5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY"//sr25519
    }
],
```

在json文件中配置的key,在运行的时候，需要指定key的类型，类型定义如下:

```rust
/// Key type for Babe module, build-in.
	pub const BABE: KeyTypeId = KeyTypeId(*b"babe");
	/// Key type for Grandpa module, build-in.
	pub const GRANDPA: KeyTypeId = KeyTypeId(*b"gran");
	/// Key type for controlling an account in a Substrate runtime, built-in.
	pub const ACCOUNT: KeyTypeId = KeyTypeId(*b"acco");
	/// Key type for Aura module, built-in.
	pub const AURA: KeyTypeId = KeyTypeId(*b"aura");
	/// Key type for ImOnline module, built-in.
	pub const IM_ONLINE: KeyTypeId = KeyTypeId(*b"imon");
	/// Key type for AuthorityDiscovery module, built-in.
	pub const AUTHORITY_DISCOVERY: KeyTypeId = KeyTypeId(*b"audi");
	/// A key type ID useful for tests.
	pub const DUMMY: KeyTypeId = KeyTypeId(*b"dumy");

```

./substrate export-blocks --base-path /home/jeremy/work/chain-data/node1 --chain ./customSpecRaw_node.json  /home/jeremy/work/chain-data/backup

./substrate export-blocks --base-path /home/jeremy/work/chain-data/node1 --chain /home/jeremy/work/temp/substrate/target/release/customSpecRaw_node.json --to 1950

5G6PuXb39m5jZjnKae5a1BnYpKSmvmQfkRFHADGuapR6hLwe

## 存储使用遇到的问题

在针对map进行操作时，若需要更改一个map的值，可以通过先获取最新的值，修改后再insert 进map; 还有一种方法是直接使用mutate方法，这种方式,但是在实际测试中发现不能将最新的值进行更新；若是不使用`checked_sub`这种方式，测试发现存在溢出现象;

```rust
 <Balances<T>>::mutate(id,&from,|from_balance| {
let new_balance = *from_balance.checked_sub(&value);
if new_balance.is_none(){
  DispatchError::Other("underflow in calculating allowance");
 }else{
 new_balance.unwrap();
}
});
```

## 模块编写遇到的问题

当前模块编写中，针对moudle中,验证失败原因 抛出更详细的错误提示；需要进一步查看相关解决方法；


