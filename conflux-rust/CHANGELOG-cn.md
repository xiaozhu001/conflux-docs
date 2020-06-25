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

- Compute transaction root and receipts root by a simple MPT where the key is
the index in big endian bytes representation of fixed length with leading zero
and the value is the bytes representation of the corresponding data, e.g.
transaction hash for transaction root, Receipt rlp for block receipts root.
The receipts root is the Merkle root of the simple MPT of block receipts roots.

- Use raw bytes in blame vec hash calculation instead of rlp because each 
element of the vec is fixed length H256.

- Add support for CHAINID, SELFBALANCE, BEGINSUB, JUMPSUB, RETURNSUB opcodes.

- NUMBER opcode in call_virtual() now returns the correct block number.

- BLOCKHASH opcode now returns the last block hash (i.e., ``blockhash(block.number - 1)``) 
or zero if not querying the last block hash.

- Disable reentrancy of contract calling through other contracts. 

- Change the default value of `from_epoch` in RPC `cfx_getLogs` from "earliest" to "latest_checkpoint".
Now if no `from_epoch` is specified, it will only return logs after the latest checkpoint.

- Improve archive and full node log filtering. Change `filter.to_epoch` default to `"latest_state"`. Limit `filter.block_hashes` to up to 128 items.

# 0.5.0

## Improvements

- Add fields in Receipt: gas_fee, gas_sponsored, storage_sponsored. Accumulate gas_used in Receipt, not gas_charged.

- Delay block requests when we cannot process them to avoid wasting network bandwidth.

- Set block gas limit for Genesis block to 30_000_000.

- Define gas_used to be transaction gas limit for NotEnoughCash, the same as all other exceptions.

- Add support for WebSockets in RPC.

- cfx_gasPrice will return a price with at least 1000000000, i.e. 1GDrip.

- Move getstatus RPC from test to common, and renamed with cfx_getStatus.

## Bug Fixes

- Fix UnexpectedResponse from honest peers that causes peer demotion.

- Remove some untrue debug assert.

- Forbidden CALLCODE and DELEGATECALL to internal contracts.

- RPC now returns the correct rlp size of the block

- Fix a race condition that may cause optimistic execution to panic.

- Fill in correct block gas limit value for mining.

- Fix definitions and logics in transaction early execution error checking.

- Use block_count - 1 in target difficulty calculation because it's the unbiased estimation of exponential distribution parameter (past mining power).

## Improvements

- Add cfx_getConfirmationRiskByHash RPC to get confirmation risk by block hash.

- Add getTransactionsFromPool debug RPC to collect transactions in pool.

# 0.4.0

## Bug Fixes

- Fix a potential crash bug in the transaction pool compoenent

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
