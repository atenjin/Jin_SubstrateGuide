# Substrate 入门 - 区块头 -（五）

关于“区块链”的块链结构这里就不再赘述，听过这个名词的应该都多少在心里能想到大概的样子。因此本文直接开始从Substrate的区块头结构开始讲解。

总体来说，substrate的卖点就是一个区块链框架，因此实际上对于区块头，交易体等一系列区块链中的基本元素而言，都是可以**自行定制**的，并非是固定的结构体。然而由于区块头中含有的一些证明及一些属性与区块链的运行过程（基于状态），共识（共识证明）等方面是“强耦合”的，因此一般情况下在Substrate中使用的区块头都直接使用了Substrate默认提供的，**极少有需求需要对其进行更改**。

## substrate核心与用户可在框架下编写的模块的分界线

参考本系列第四篇[《Substrate 入门 - 项目结构 -（四）》](https://zhuanlan.zhihu.com/p/98684002)中的描述，应该清楚的知道，位于`/bin`目录下的代码属于框架外的，可以参考的，位于`primitives`及`client`里面的代码属于Substrate框架内的，一般情况下不改动的。

因此位于不同的文件夹下即是Substrate框架与用户编写代码的分界线。

## 区块头

总所周知，在区块链的块结构中，一个块由区块头（block_header）与该区块下的交易体共同构成。

而**共识**实际上共识的就是区块头，区块头就是对这个区块信息的**摘要**与**证明**。

## Substrate的区块头

### 1. 框架内

Substrate提供了一个Header的模板，位于`/primitives/runtime/src/generic/header.rs`文件中：

```rust
pub struct Header<Number: Copy + Into<U256> + TryFrom<U256>, Hash: HashT> {
	/// The parent hash.
	pub parent_hash: Hash::Output,
	/// The block number.
	#[cfg_attr(feature = "std", serde(
		serialize_with = "serialize_number",
		deserialize_with = "deserialize_number"))]
	pub number: Number,
	/// The state trie merkle root
	pub state_root: Hash::Output,
	/// The merkle root of the extrinsics.
	pub extrinsics_root: Hash::Output,
	/// A chain-specific digest of data useful for light clients or referencing auxiliary data.
	pub digest: Digest<Hash::Output>,
}
```

这里简单对这个模板做介绍：

* parrent_hash：父区块hash，这里就不多做解释了，这是“区块链”概念的根本
* number：区块高度，对于状态链必须，对于utxo链非必须（指代类似Bitcoin这类模型，而不是使用状态来模拟utxo），总体来说如果记录在header里面在其他部分的设计下可以简化一些，否则其他组件就会设计的比较复杂
* `state_root`：状态根，参考本系列[《Substrate 入门 - 具备状态的链 -（三）》](https://zhuanlan.zhihu.com/p/96866051)，是对区块执行后状态变更的**证明**
* `extrinsics_root`：交易根，代表了该header下的区块体中交易的内容与顺序，是对区块信息的**摘要**
* `digest`，区块附加信息。digest翻译名“摘要”，但是这里笔者觉得这个类型更像是对区块信息的一些附加信息的集合，当前Substrate的文章及官方文档几乎都没对这个属性的剖析，但是其实它是相当重要的一个组成部分，后文会做分析。

### 2. 框架外

我们浏览一个模板范例`bin/node`中的例子：

文件`/bin/node/runtime/src/lib.rs` L528 左右

```rust
pub type Header = generic::Header<BlockNumber, BlakeTwo256>;
pub type Block = generic::Block<Header, UncheckedExtrinsic>; // Block 引用了上面Header的结构，UncheckedExtrinsic是交易，后续文章会介绍。因此Block中是可以看得到Header的结构，而这个结构是由用户定义了传入
```

L112行左右

```rust
impl frame_system::Trait for Runtime {
// ...
	type Header = generic::Header<BlockNumber, BlakeTwo256>; // 可以看到这里定义runtime内部Header的结构与上面对于Header的定义是一致的。
// ...
}
```

我们已知 node 中是用户编写的代码，因此我们可以注意到，用户代码中的`Header`是可以由**用户自由定义**的，包括并不限于如下改动：

* 保留Substrate的框架内默认Header模板，即`generic::Header`，也就是位于`/primitives/runtime/src/generic/header.rs`中的定义：
  * BlockNumber 是用 u32, u64，抑或时u128 等
  * 对于BlockHash的计算使用的hash函数（这里是`BlakeTwo256`）

* 重新使用自己定义的Header
  * 用户可自己定义自己的Header结构，一般情况下不适用，因为Header实际上和共识等组件是强耦合的，开发者**唯有在知道自己该改的情况**下才需要走这条路。

### 简单分析

以上总结了在Substrate框架中Header的定义，可以看出，由于Substrate的框架属于状态类型的链，因此其提供的Header的模板是偏向于状态类型的定义，而且其Header类型提供的信息实际上和以太坊十分接近，不过一个很大的不同是将以太坊的收据根`receipt_root`移除了。

不过相对应的，Header中提供了一个新的类型digest，实际上这个类型是一个能附加很多额外信息的类型，例如对于收据根就可以附加在这里的，比如是pow的链需要提供nonce和bits这类的难度信息也可以附加到digest，除此之外其还提供了pos中出块者的签名，共识证明信息等等相当重要的信息。接下来做简单介绍。

### digest 的介绍

前文已经说过，digest更像是对于该区块其他所有附加信息的一个集合，现在简要分析如下：

见文件`/primitives/runtime/src/generic/digest.rs` L31 行左右：

```rust
pub struct Digest<Hash: Encode + Decode> {
	/// A list of logs in the digest.
	pub logs: Vec<DigestItem<Hash>>,
}
```

我们可以看到，digest实际上是一个`DigestItem`的列表集合，因此可能每一个块都不同，其长度也不是固定的。因此**如何解析Digest实际上是这条链的协议之一**，是这条链的开发者应该重点进行设计的地方。

`DigestItem`的定义见 L77 行

```rust
pub enum DigestItem<Hash> {
	ChangesTrieRoot(Hash),
	PreRuntime(ConsensusEngineId, Vec<u8>),
	Consensus(ConsensusEngineId, Vec<u8>),
	Seal(ConsensusEngineId, Vec<u8>),
	/// Some other thing. Unsupported and experimental.
	Other(Vec<u8>),
}
```

由于本系列文章针对的是Substrate入门，因此本文仅对这些类型做介绍，至于如何使用这些及扩展不做详细说明。

* `ChangesTrieRoot`：事实上是类似以太坊收据根`receipt_root`的一种加强版本（暂定），是对该块中**每一条交易执行变更的证明**，仅仅在该链genesis的时候决定是否启用。默认**不启用**。
* `PreRuntime`：对于部分PoS算法中必须，在Aura和Babe中用于记录轮到某个出块者出块的证明，如`Aura`中的`slot`，Babe中还会记录VRF证明等（见`/primitives/consensus/babe/src/digest.rs:L44`）。
* `Consensus`：对于部分PoS算法必须，用于记录共识算法中需要记录的特别的信息。如Aura/Babe中使用这个记录每个epoch验证者切换的列表。
* `Seal`：所有PoS算法必须，很多情况下用于记录这个产出这个区块的**验证者的签名**，因为在PoS中需要出块者使用的自己的私钥对区块进行签名，表面这个区块是由自己产出而不是别人产出。（PoW是靠难度，因此不需要签名）
* `Other`：扩展字段，因此**一般对区块头的扩展**都可以放在这里面，这里相当于编写链的人需要小心制定协议，如通过顺序，或者关键字等提取信息。这里可以扩展如：
  * PoW中的难度nbits与难度证明nonce

以上内容在编写轻节点以及区块链前端的时候相当重要，因为很多关键信息需要从Digest中解析得到（如这个区块的出块者，轻节点获取验证者变更的信息等）

这里特别强调以下对于DigestItem的编码的`Encode/Decode`需要遵从如下定义：

```rust
pub enum DigestItemType {
	ChangesTrieRoot = 2,
	PreRuntime = 6,
	Consensus = 4,
	Seal = 5,
	Other = 0,
}
```

对于`Encode/Decode`的特点后续会写一篇文章专门描述。这里是需要提醒前端的开发者需要特别注意这个编码顺序。

## 总结

以上即是对Substrate的区块头定义的介绍，这里总体注意如下几点：

1. Substrate的区块头是可以自由定义的，但是一般情况下应该沿用模板，除非有充足的理由去改变它
2. Substrate的区块头是偏向“状态链”的定义
3. Substrate的`Digest`属性需要特别注意，很多与共识相关的信息会记录在这里。