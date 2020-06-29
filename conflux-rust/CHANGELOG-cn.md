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

- Stop marking OverlayAccount dirty on read access. This will influnce the state root. 

- Do not mark OverlayAccount dirty in sub_balance 0 for non-existence account.

- Add missing transaction verifications for invalid block.

## Improvements

- Improve the transaction address check at RPC

- Change the test net PoW to use double keccak

## EVM Updates

- Decide the storage owner (who is responsible for storage collateral) at the beginning of the transaction. 

- Only check the storage limit and balance for storage collateral at the end of EVM execution. 

# 0.3.2

## Bug Fixes

- Fix an issue that GetBlockHashesByEpoch containing blocks before checkpoint may cause the node to crash.

- Quick fix for possible duplicate block inserted into the consensus worker thread.

# 0.3.0

## Blockchain Core Updates (Not Backward Compatible)

- Changes the address scheme of Conflux. All normal address now start with 0x1.
All smart contracts address now start with 0x8. Note that your private key will
still work as long as you replace the first character in your hex address with
``0x1``. For example, if your address is 0x7b5c..., after this update your
address will change to 0x1b5c...

- Changes the state Merkle root calculation method. Merkle is calculated based
on constructed raw keccak input byte string instead of serialized rlp; checks if
compressed_path starts on the second nibble of a byte; makes sure that with the
constructed keccak input string adversary cannot construct a compressed path to
create a path Merkle of the same value as a node Merkle.

- Each epoch now has a limit of executing 200 blocks. If there are more than
200 blocks in an epoch. Only the last 200 blocks will be executed. This change
is designed to battle DoS attacks about hiding and generating a lot of blocks
suddenly.

- Add storage_root support in Conflux MPT. Define storage root as the Merkle
root computed on the account's storage subtree root node, ignoring the
compressed path; 2) force the storage root node to existing by setting a
StorageLayout value at the storage node. 

You need to use new SDK tools to connect with the main chain, otherwise your
transaction will be rejected as invalid. 

## EVM Updates

- Change the gas used in SSTORE operation to 5000 gas, no matter the zero-ness
is changed or not. And we no longer refund gas in releasing storage entry. 

## RPC/CLI Updates

- Change the CLI interface subcommand from `debug` to `local`. Its
functionality remains the same.

- Add a RPC cfx_getSkippedBlocksByEpoch to query skipped blocks of an epoch

- Add a corresponding CLI interface to query skipped blocks via local RPC

- Refactor RPC interface now most RPC takes HEX parameters and returns HEX

## Bug Fixes

- Fix an issue that may cause the P2P layer to not propagate out-of-era blocks properly

- Fix an issue that Conflux RPC may return incorrect estimate.

- Fix an issue that virtual call RPC may fail if the caller does not have enough balance

- Fix an issue that failing to send a pending request can make a block not received.

- Fix an issue that not-graph-ready compact blocks are not fully received.


## Improvements

- Make the consensus layer to prioritize meaningful blocks first. It will
improve the overall performance in facing of DoS attacks. It will also
prioritize self-mined blocks as a desirable effect.
