# 0.6.0

## 错误修正

- 修复TrackTouched逻辑不一致的问题。

- 确保所有的内部账户都存在于创世块，否则一些读取操作可能会崩溃。

- 修正了vm操作中对require()的不正确用法。在大多数情况下，不希望在没有他的情况下创建基础账户，特别是地址对应于一个合约的情况。当要创建用户账户时，要对地址空间进行检查。

- 修正处理快照分块边界导致崩溃的问题。该错误是由于对Trie证明密钥错误的唯一性假设所引起的。


## 改进

- 提高共识层在不稳定的TreeGraph场景下的性能。

- 完善节点通信的协议版本机制，并将协议版本升级为 V2。除了 msgid::THROTTLE (0xfe) 之外，这个改动是向后兼容的。

- 在同步协议和轻协议握手消息中加入chain_id字段，这样peers节点就可以从另一条Conflux链上断开对应的peers节点，例如测试网和另一个测试网。

- 保持network_id与chain_id相同。设置network_id只针对本地实验目的。

- 改进tx-pool内的的交易替换规则：现在一个交易可以用更高的燃料价格 或用相同燃料价格和更大的纪元高度替换相同的发送者和随机数值nonce。

- 将随机数nonce从64bits（位）改为256bits（位）

- 在PoW难度计算中引入基于随机数nonce的下限。这将有助于在未来抵御矿池之间的扣块攻击。由于该变化和对PoW进行细致的设计，矿池可以将随机数nonce的前128位作为服务器随机数nonce，矿池的参与者将无法判断他们是否已经开采到了一个区块。

- 改进stratum协议，使其更符合常规情况。现在，stratum协议可以正确的与外部矿工协同工作。

- 将 `deposit_list` 与 `vote_stake_list` 从 `Account` 中分离出来，并调整 `withdraw`, `deposit`, `vote_lock` 内部合约调用的燃料开销。现在这三个函数的燃料开销与 `deposit_list` 或 `vote_stake_list` 的长度有关。

- 在默认情况下，禁用交易索引持久性。这可以降低矿工的磁盘使用量。如果你想可靠地服务于与交易相关的RPC，你应手工在配置文件内设置 `persist_tx_index=true` 。

- 一个新的RPC ctx_getBlockRewardInfo用于在给定纪元内查询一个区块奖励信息。

- 通过使用键为前导含0、固定长度的大尾字节索引，值为对应数据的字节，应用简单的MPT树计算交易根和收据根，例如，交易根的摘要值，收据rlp为块收据根，而收据根对应于块收据根所在MPT树的Merkle根。

- 在blame vec摘要计算中使用原始字节而非rlp的原因是 每个vec元素的长度值固定位H256。

- 增加对CHAINID, SELFBALANCE, BEGINSUB, JUMPSUB, RETURNSUB 操作码的支持。

- call_virtual()内的NUMBER操作码目前能够返回正确的区块号。

- BLOCKHASH操作码现在返回最后一个块的摘要值(如， ``blockhash(block.number - 1)``) 
如果不是查询最后一个块的摘要值，则返回0。

- 禁止通过其他合约重复调用合约的情况。

- 将RPC中的 `cfx_getLogs` 中 `from_epoch` 的默认值从"earliest" 改为 "latest_checkpoint"。如果没有指定 `from_epoch` ，它将只发挥最新检查点后的日志。

- 改进存档及全节点日志文件过滤功能。将 `filter.to_epoch` 的缺省值设置为 `"latest_state"` 。 同时限制 `filter.block_hashes` 最多为128项。

# 0.5.0

## 提升

- 在收据中增加字段：gas_fee, gas_sponsored, storage_sponsored。在收据内累计gas_used而不是gas_charged。

- 当我们无法处理块请求时，会延迟对应请求，以避免浪费网络带宽。

- 将创世区块的燃料限制值设置为30_000_000。

- 定义gas_used为NotEnoughCash的交易燃料限制，与其他所有异常相同。

- 在RPC中增加对WebSockets的支持。

- cfx_gasPrice将返回至少1000000000，即1GDrip。

- 将getstatus RPC从test移至common，并改名为cfx_getStatus。

## 漏洞修复

- 修复从诚信伙伴处收到UnexpectedResponse，导致对方降级的问题。

- 移除部分非真的调试断言。

- 禁止将CALLCODE和DELEGATECALL用于内部合约。

- RPC现在返回正确块对应的RLP大小。

- 修复了一个可能引发乐观性执行陷入到恐慌状态的竞争条件。

- 填入正确的块燃料限制值以便进行挖矿。

- 修正交易早期执行错误检查中的定义和逻辑。

- 使用block_count - 目标难度计算中的1，是因为它是指数分布参数（过去算力）的无偏估计。

## 提升

- 添加cfx_getConfirmationRiskByHash RPC，通过区块摘要值获取确认风险。

- 添加getTransactionsFromPool调试RPC来收集池中的事务。

# 0.4.0

## 漏洞修复

- 修正交易池组件中潜在的崩溃bug。

- 在读取操作时，不再标记OverlayAccount为脏。该操作会影响状态根。

- 对于不存在的账户，不要在sub_balance为0时标记OverlayAccount为脏。

- 为无效区块添加缺失的交易验证。

## 提升

- 改进RPC的交易地址检查

- 将测试网PoW改为使用双keccak算法

## 针对EVM的更新

- 在交易开始时，决定质押存储者（负责贮存抵押物）。

- 只在EVM执行结束时检查存储质押的存储限额和余额。

# 0.3.2

## 漏洞修复

- 修正了GetBlockHashesByEpoch包含检查点前的区块可能导致的节点崩溃问题。

- 快速修复可能插入到共识矿工线程中的重复块。

# 0.3.0

## 区块链核心更新（不向下兼容）

- 更改Conflux的地址方案。所有正规地址现在都以0x1开始。所有合约地址现在都以0x8开始。需要注意的是您的只需要把十六进制地址中的第一个字符替换为 ``0x1`` 即可。 例如你的地址是0x7b5c......，在本次更新后将变为0x1b5c......。

- 改变状态Merkle根的计算方法。Merkle的计算方法是基于
对在构造的原始keccak输入字节字符串上计算，而不是在经过序列化处理的rlp上计算；检查compressed_path是否从一个字节的第二个半字节（4bits）开始；确保即使拥有构造好的keccak输入字符串，敌手任然难以构造一个压缩路径以创建一个与节点Merkle值相同相同的路径Merkle值。

- 现在每个纪元有执行200个区块的限制。如果在纪元内超过200个区块时，只有最后200个区块会被执行。这一改动的目的是为了对抗隐藏和短时间内生成大量区块这类的拒绝服务攻击（DoS）。

- 在ConfluxMPT树内添加对storage_root的支持。将存储根定义为忽略压缩路径情况下在账户存储子树根节点上计算出的Merkle根。通过设置StorageLayout值强制存储根节点的存在。

需要使用最新的SDK工具以连接主链，否则您的交易会由于无效而被拒绝。

## 针对EVM的更新

- 将SSTORE操作中使用的燃料改为5000，而不再管zero-ness
是否改变。在存储条目释放时，也不在对燃料进行退回。 A

## 针对RPC/CLI的更新

- 将CLI接口子命令从 `debug` 转为 `local`。其功能性上保持不变。

- 增加了cfx_getSkippedBlocksByEpoch这一RPC操作以查询每一纪元内跳过的区块。

- 添加了一个可以使用本地RPC查询跳过区块的CLI接口。

- 重构RPC接口，目前大多数RPC接受HEX参数，并返回HEX参数。

## 漏洞修复

- 修复了一个可能导致P2P层不能正确传递过期区块的问题。

- 修复了Conflux RPC可能返回错误估计值的问题。

- 修复了如果调用者无足够余额，虚拟调用RPC可能失败的问题。

- 修复了发送待处理请求使区块无法收到的问题。

- 修复了not-graph-ready致密区块不能完全接收的问题。


## 提升

- 使共识层优先考虑有意义的区块。这将提高其面对DoS攻击的整体性能。在理想情况下，还将优先考虑自采区块。
