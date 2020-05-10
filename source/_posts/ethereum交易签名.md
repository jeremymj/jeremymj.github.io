---
title: ethereum交易签名
tags:
  - Ethereum
  - 交易签名
  - 数据编码
abbrlink: ed176158
date: 2020-05-10 12:21:57
---

　　今天把前段在实现ETH交易过程中遇到的问题做一下记录，算是对这部分知识的复习。要想在以太坊中完成一笔交易，可以借助节点对外开放的`eth_sendTransaction`或`eth_sendRawTransaction`这两个JsonRpc接口。<!--more-->我们只需要拼接好这个接口需要的参数再通过网络提交到节点即可。本文的重点是叙述怎么来拼接这个目标参数，由于在钱包开发中主要使用`eth_sendRawTransaction`这个接口，下面就对这接口所需要的参数进行阐述。
## 交易构造
　　通过查阅Ethereum的文档，知道构造原始交易所需要的结构如下所示：
```rust
/// Description of a Transaction, pending or in the chain.
#[derive(Debug, Default, Clone, PartialEq, Deserialize, Serialize)]
pub struct RawTransaction {
    /// Nonce
    pub nonce: U256,
    /// Recipient (None when contract creation)
    pub to: Option<H160>,
    /// Transfered value
    pub value: U256,
    /// Gas Price
    #[serde(rename = "gasPrice")]
    pub gas_price: U256,
    /// Gas amount
    pub gas: U256,
    /// Input data
    pub data: Vec<u8>,
}
```
注意：自己在实现过程中定义原始交易参数结构体时，必须要确保**参数顺序相同**。在构造交易时会将该结构体使用RLP方式编码，当节点收到原始交易时会按照这个结构对收到的数据做编码还原；
从`RawTransaction`我们知道在一笔交易中涉及到`nonce`,`to`，`value`,`gas_price`,`gas,data`这五个属性。这下面是对这些属性的说明：
### nonce
　　在区块链交易中都需要有nonce这个参数，用于表示当前发送地址的第几笔交易，能够确保交易的顺序，防止节点被交易重放攻击。注意：
- 初始交易是从编号0开始的，可以通过调用`eth_getTransactionCount`获取当前交易的值；
- 节点在验证交易时会校验当前的nonce是否为正确的值，若交易中的nonce值比真实值小，则这笔交易会直接返回nonce错误；
- 若交易中的nonce值比真实值大，则会将该交易放入一个`future`队列里面，等待中间缺失的交易进来后再继续验证这笔交易；
### 地址(to)
　　`to`表示交易的接受地址，这个地址通常可以是一个外部地址（该地址对应的私钥有实际的控制者），表示一笔转账操作；也可以是一个内部地址(合约地址)，表示合约调用。
### value
　　这个值表示该交易需要转给目标地址的`ETH`数量，需要注意填写的数值是最小单位（wei）,通常在钱包界面上填写的数值单位是`ether`，在处理的时候需要做单位转换；这里有几种情况需要注意：
- 填写正常的数值且to 地址是一个外部地址，表示将指定数量的token转到指定地址；
- 填写的数值为0，且to地址是一个外部地址，这种操作除了浪费gas之外，没有任何作用；
- 当to地址是一个合约地址时且value不为空，这样的交易也会将发送者地址上的eth代币转到合约地址上，由于合约地址是内部地址，没人知道对应的私钥，所以这些token会成为死币；
### gas_price
　　表示每个单位的gas值多少wei，我们知道Ethereum EVM在执行过程中每个操作都需要消耗gas,gas是一个`不随ETH波动而波动的单位定义`，用于表示调用者愿意为这笔交易每个步骤花费多少token，通常情况下调用者给出的`gas_price`价格越高，矿工会优先执行执行；为了防止恶意抬高gas_price，Ethereum针对`gas_price`有专门的算法来进行控制，开发者在构造交易的时候，直接调用节点提供的rpc接口`eth_gasPrice`即可；还需要注意的是，这里的数值也需要填写**最小的单位wei**。
### gas
　　表示愿意交易在执行过程中，愿意支付的最大gas数量，也就是我们常见的gas_limit数值；若是指定的gas数量太少，交易在执行的过程中gas消耗完后，就会返回错误；若是指定的gas数量比较大，调用者账户上也有足够的gas可扣，则交易会正常的执行，在执行完后会将多余的gas返回到调用者的账户上。
### data
　　在以太坊中针对每笔交易都可以填写附加信息，针对这个字段的内容我打算放到下一篇关于合约调用的文章中。
以上部分就是构造一个以太坊交易所需要的基本数据，要是构造的交易能够在链上被虚拟机执行，还需要对交易进行签名操作；
## 编码
在以太坊中，数据的编码使用`RLP`的编码方式来进行的，关于编码实现的具体方式，可以查看[wiki](https://github.com/ethereum/wiki/wiki/%5B%E4%B8%AD%E6%96%87%5D-RLP),只是需要注意在编码的过程中，需要注意字段的顺序

```rust
 //对RawTransaction 数据使用RLP编码
    fn encode(&self, rlp: &mut RlpStream) {
        rlp.append(&self.nonce);
        rlp.append(&self.gas_price);
        rlp.append(&self.gas);
        if let Some(ref t) = self.to {
            rlp.append(t);
        } else {
            rlp.append(&vec![]);
        }
        rlp.append(&self.value);
        rlp.append(&self.data);
    }
```
之所以按照`fn encode`方式将数据添加到`RlpStream`实例中，是因为节点验证交易时会进行如下方式的数据解码
```rust
//还原出原RLP数据格式
impl rlp::Decodable for UnverifiedTransaction {
	fn decode(d: &Rlp) -> Result<Self, DecoderError> {
		//rlp编码 针对数据长度 是有特征？？
		if d.item_count()? != 9 {
			return Err(DecoderError::RlpIncorrectListLen);
		}
		let hash = keccak(d.as_raw());
		Ok(UnverifiedTransaction {
			unsigned: Transaction {
				nonce: d.val_at(0)?,
				gas_price: d.val_at(1)?,
				gas: d.val_at(2)?,
				action: d.val_at(3)?,
				value: d.val_at(4)?,
				data: d.val_at(5)?,
			},
			v: d.val_at(6)?,
			r: d.val_at(7)?,
			s: d.val_at(8)?,
			hash,
		})
	}
}
```
## 签名生成
　　以太坊使用Secp256k1算法来签名，我们知道签名都有计算源数据的hash值过程。针对交易hash值的计算过程如下：
```rust
  fn hash(&self, chain_id: Option<u64>) -> [u8; 32] {
        let mut stream = RlpStream::new();
        stream.begin_unbounded_list();
        self.encode(&mut stream);
        if let Some(n) = chain_id {
            stream.append(&n);
            stream.append(&U256::zero());
            stream.append(&U256::zero());
        }
        stream.finalize_unbounded_list();
        keccak(stream.out().as_slice())
    }
```
其中`chain_id`表示当前交易用于的目标链，以太坊使用这个参数的目的是防止在不同的链上提交相同的数据，让节点不能提前检查出风险交易，增加了链的稳定性。可以通过构造`JsonRpc`请求数据`{"method":"eth_chainId","params":[],"id":1,"jsonrpc":"2.0"}`获取到；
hash数据签名的过程如下：
```rust
fn ecdsa_sign(hash: &[u8], private_key: &[u8], chain_id: u64) -> EcdsaSig {
    let s = Secp256k1::signing_only();
    let msg = Message::from_slice(hash).unwrap();
    let key = SecretKey::from_slice(private_key).unwrap();
    let (v, sig_bytes) = s.sign_recoverable(&msg, &key).serialize_compact();

    EcdsaSig {
        v: v.to_i32() as u64 + chain_id * 2 + 35,
        r: sig_bytes[0..32].to_vec(),
        s: sig_bytes[32..64].to_vec(),
    }
}
```
签名最后得到的`v`、`r`、`s`值和交易结构体编码的数据一起构成了`eth_sendRawTransaction`这个接口所需要的数据；下面是完整的交易签名实现：
```rust
 pub fn sign(&self, private_key: &[u8], chain_id: Option<u64>) -> Vec<u8> {
        let hash_data = self.hash(chain_id);
        let sig = ecdsa_sign(&hash_data, private_key, chain_id.unwrap());
        let mut r_n = sig.r;
        let mut s_n = sig.s;
        while r_n[0] == 0 {
            r_n.remove(0);
        }
        while s_n[0] == 0 {
            s_n.remove(0);
        }
        let mut tx = RlpStream::new();
        tx.begin_unbounded_list();
        self.encode(&mut tx);
        tx.append(&sig.v);
        tx.append(&r_n);
        tx.append(&s_n);
        tx.finalize_unbounded_list();
        tx.out()
    }
```
　　从上述代码可以看出，我们通常听到的交易签名，其实是包含了通过`keccak256`计算交易hash值、使用`Sec256K1`对Hash签名并将签名结果和交易详情编码在一起，最后输出`Hex`格式数据的过程。
## 总结
　　通过上面的描述详细说明了以太坊交易构成的过程，这也是当前钱包在构造用户交易的核心过程，该过程涉及到的内容很多，后续可以继续对里面的知识点进行阐述。
