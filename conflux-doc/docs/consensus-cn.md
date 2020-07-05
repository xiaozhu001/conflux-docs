# Conflux共识层设计与实现

Conflux共识层处理所有从同步层接收到的区块，基于Conflux的GHAST共识算法产生区块的总顺序，并调用其底层的**交易执行引擎**，按照确定的顺序运行交易。它提供了协助**区块生成器**准备新区块骨架的必要信息。它同时还会将已处理过的交易情况通知**交易池**，以便交易池能够做出更好的交易选择决策。

本文档是为想要了解Conflux共识层（在目录：core/src/consensus中）的rust实现细节的读者提供了一个顶层描述。如果需要查看更多的实现细节，可以关注代码内的注释。如果需要了解更多Conflux共识算法的信息，可以关注Conflux协议规范和论文(https://arxiv.org/abs/1805.03870) 。 

## 设计目标

共识层有以下的设计目标。 

1. 在后台按照共识算法持续处理新的区块。

2. 我们希望将共识图内的每个区块的内存使用量最小化。即使采用了检查点机制，在正常情况下，图中也会包含300K-500K个区块，而在面临liveness攻击时，甚至会包含1M个以上的区块。这可能会为内存带来压力。

3. 我们需要快速处理每个区块。因为全/归档节点从*创世*开始追赶网络时，快速的块处理进程对保持追赶周期较短是非常重要的。

4. 对潜在的攻击有很强的抵御能力。恶意攻击者可能会在树图中的任意位置生成不良的区块。

## 架构和组件

### 共识图

`ConsensusGraph` (core/src/consensus/mod.rs)是共识层的主要结构。同步层使用 `BlockDataManager` 构建 `ConsensusGraph` ，其中 `BlockDataManager` 负责将所有区块的元数据信息存储在磁盘上。
`ConsensusGraph::on_new_block()` 是将区块发送至 `ConsensusGraph` 结构并处理的关键函数。它也提供了一组公开函数以便查询区块/交易的状态。这应该是与其他组件交互的主要接口。

### ConsensusGraphInner

`ConsensusGraphInner` (core/src/consensus/consensus_inner/mod.rs) 是 `ConsensusGraph` 的内部结构。 `ConsensusGraph::on_new_block()` 在函数开始时获取内部结构的写入锁。其余的是只需要获取读取锁的查询函数。

`ConsensusGraphInner` 的内部结构是相当复杂的。通俗来讲，其维护了两类信息。一是整个树图的状态信息，如当前的*主轴链*、*时钟链*、*难度*等。二是每个区块的状态信息（如每个区块的 `ConsensusGraphNode` 结构）。每一个区块都对应与一个用于获取其信息的 `ConsensusGraphNode` 结构。
当区块第一次进入 `ConsensusGraphInner` 时，它将被插入 ``ConsensusGraphInner::arena : Slab<ConsensusGraphNode>`` 。在slab中的索引将会成为 `ConsensusGraphInner` 中的区块arena索引。 我们使用arena索引替代 `H256` 的原因是开销更低。在谈到算法机制及其实现时我们会重新提到 `ConsensusGraphInner` 和 `ConsensusGraphNode` 字段。

### ConsensusNewBlockHandler

`ConsensusNewBlockHandler`
(core/src/consensus/consensus_inner/consensus_new_block_handler.rs) 包含一组用于处理新区块的例程。从理论上说，由于主要对内部结构进行操作，该代码实质上是 `ConsensusGraphInner` 的一部分。然而，这些例程都是 `on_new_block()` 的子例程且consensus_inner/mod.rs已经非常复杂了。因此我们决定将他们放在单独的文件中。

### ConsensusExecutor

`ConsensusExecutor` (core/src/consensus/consensus_inner/consensus_executor.rs)
是独立交易执行线程的接口结构。 `ConsensusExecutor::enqueue_epoch()` 允许其他线程发送一个以异步方式执行给定主轴链区块的纪元的执行任务。一旦计算完成，结果状态根会被存储到 `BlockDataManager` 中。如果需要，其他线程可以通过调用 `ConsensusExecutor::wait_for_result()` 以等待纪元的执行。在当前的实现中， `ConsensusExecutor` 也包含了计算区块奖励的例程，包括 `get_reward_execution_info()` 及其子例程。

### ConfirmationMeter

`ConfirmationMeter` (core/src/consensus/consensus_inner/confirmation_meter.rs)
conservatively calculates the confirmation risk of each pivot chain block. Its
result will be useful for the storage layer to determine when it is *safe* to
discard old snapshots. It can also be used to serve RPC queries about block
confirmation if we decide to provide such RPC.

### AnticoneCache and PastsetCache

`AnticoneCache` (core/src/consensus/anticone_cache.rs) and `PastsetCache`
(core/src/consensus/pastset_cache.rs) are two structs that implement customized
caches for data structures in `ConsensusGraphInner`. In the implementation of
the inner struct, we need to calculate and store the anticone set and the past
set of each block. However, it is not possible to store all of these sets in
memory. We therefore implement cache style data structures to store sets for
recently inserted/accessed blocks. If an anticone/past set is not found in the
cache, we will recalculate the set in the current inner struct implementation. 

## Important Algorithmic Mechanisms

There are several important algorithmic mechanisms in the Conflux Consensus
Layer. Here we will talk about them from the implementation aspect. See XXX for
the algorithmic reasoning behind them.

### Pivot Chain and Total Order

The basic idea of the Conflux consensus algorithm is to first make everyone
agree on a pivot chain. It then expands the total order from the pivot chain to
cover all blocks with a topological sort. As long as the pivot chain does not
change/reorg, the total order of blocks will stay the same, so does the derived
order of transactions. 

Comparing with Bitcoin/Ethereum, the consensus in Conflux has two key
differences: 

1. *almost every block* will go into the total order, not just the agreed pivot
chain.

2. The transaction validity and the block validity are *independent*. For example, a
transaction is invalid if it was included before or it cannot carry out due to
insufficient balance. Such invalid transactions will become noop during the
execution. However, *unlike Bitcoin and Ethereum blocks containing such
transactions will not become invalid*.

In `ConsensusGraphInner`, the arena index of the current pivot chain blocks are
stored in order in the `pivot_chain[]` vector. To maintain it, we calculate the
lowest common ancestor (LCA) between the newly inserted block and the current best
block following the GHAST rule. If the fork corresponding to the newly inserted
block for the LCA ended up to be heavier, we will update the `pivot_chain[]` from
the forked point.

### Timer Chain

Blocks whose PoW quality is `timer_chain_difficulty_ratio` times higher than the target
difficulty are *timer blocks*. The `is_timer` field of the block will be set to
True. The consensus algorithm then finds the longest timer block chain (more
accurately, with greatest accumulated difficulty) similar to the Bitcoin
consensus algorithm of finding the longest chain. The arena index of this
longest timer chain will be stored into `timer_chain[]`. 

The rationale of the timer chain is to provide a coarse-grained measurement of
time that cannot be influenced by a malicious attacker. Because timer blocks
are rare and generated slowly (if `timer_chain_difficulty_ratio` is properly
high), a malicious attacker cannot prevent the growth of the timer chain unless
it has the majority of the computation power. Therefore how many timer chain
blocks appear in the past set of a block is a good indication about the latest
possible generation time of the block. We compute this value for each block and
store it in `timer_chain_height` field of the block.

### Weight Maintenance with Link-Cut Tree

To effectively maintain the pivot chain, we need to query the total weight of a
subtree. Conflux uses a Link-Cut Tree data structure to maintain the subtree
weights in O(log n). The Link-Cut Tree can also calculate the LCA of any two nodes
in the TreeGraph in O(log n). The `weight_tree` field in `ConsensusGraphInner`
is the link-cut tree that stores the subtree weight of every node. Note that
the implementation of the Link-Cut Tree is in the utils/link-cut-tree
directory.

### Adaptive Weight

If the TreeGraph is under a liveness attack, it may fail to converge under one
block for a while. To handle this situation, the GHAST algorithm idea is to
start to generate adaptive blocks, i.e., blocks whose weights are redistributed
significantly so that there will be many zero weight blocks with a rare set of
very heavy blocks. Specifically, if the PoW quality of an adaptive block is
`adaptive_heavy_block_ratio` times of the target difficulty, the block
will have a weight of `adaptive_heavy_block_ratio`; otherwise, the block will
have a weight of zero. This effectively slows down the confirmation
temporarily but will ensure the consensus progress.

Because adaptive weight is a mechanism to defend against rare liveness attacks,
it should not be turned on during the normal scenario. A new block is adaptive
only if: 1) one of its ancestor blocks is still not the dominant subtree
comparing to its siblings, and 2) a significantly long period of time has passed
between the generation of that ancestor block and the new block (i.e., the
difference of `timer_chain_height` is sufficiently large). `ConsensusGraphInner::adaptive_weight()`
and its subroutines implement the algorithm to determine whether a block is
adaptive or not. Note that the implementation uses another link-cut-tree
`adaptive_tree` as a helper. Please see the inlined comments for the
implementation details. 

### Partial Invalid

Note that the past set of a new block denotes all the blocks that the generator
of the new block observes at the generation time. Therefore, from the past set
of a new block, other full nodes could determine whether it chooses the correct
parent block and whether it should be adaptive or not. 

The Conflux consensus algorithm defines those blocks who choose incorrect
parents or fill in incorrect adaptive status as *partial invalid blocks*. For a
partial invalid block, the `partial_invalid` field will be set to True. The
algorithm requires the partial invalid blocks being treated differently from
the normal blocks in three ways:

1. All honest nodes will not reference directly or indirectly partial invalid
blocks until a significant period of time. This time period is measured with
the `timer_chain_height` and the difference has to be more than
`timer_chain_beta`. Yes, it means that if another otherwise perfectly fine
block referencing the partial invalid block, both of these two blocks will not
be referenced for a while.

2. Partial invalid blocks will have no block reward. They are extremely
unlikely to get any reward anyway because of their large anticone set due to
the first rule.

3. Partial invalid blocks are excluded from the timer chain consideration.

To implement the first rule, the `on_new_block()` routine in
`ConsensusNewBlockHandler` is separated into two subroutine
`preactivate_block()` and `activate_block()`. `preactivate_block()` compute and
determine whether a block is partial invalid or not, while `activate_block()`
fully integrate a block into the consensus graph inner data structures. For
every new block, the field `active_cnt` tracks how many inactive blocks it
references. A block is inactive if it references directly or indirectly a
partial invalid block. `activate_block()` will be called on a block only when
`active_cnt` of the block becomes zero. The field `activated` denotes whether a
block is active or not. For partially invalid blocks, their activation will be
delayed till the current timer chain height of the ledger is `timer_chain_beta`
higher than the invalid block. Newly generated blocks will not reference any
inactive blocks, i.e., these inactive blocks are treated as if they were not in
the TreeGraph.

### Anticone, Past View, and Ledger View

In order to check the partial invalid status of each block, we need to operate
under the *past view* of the block to determine its correct parent and its
adaptivity. This is different from the current state of the TreeGraph or we
call it the *ledger view*, i.e., all blocks in the anticone and the future set
of the block are excluded. Because we process blocks in topological order, the
future set of a new block is empty. We therefore need to eliminate all anticone
blocks only.

`compute_and_update_anticone()` in `ConsensusNewBlockHandler` computes the
anticone set of a new block. Note that because the anticone set may be very
large, we have two implementation level optimizations. First, we represent the
anticone set as a set of barrier nodes in the TreeGraph, i.e., a set of
subtrees where each block in the subtrees is in the anticone set. Second, we
will maintain the anticone set of the recently accessed/inserted blocks
only. When checking whether a block is valid in its past view or not (e.g., in
`adaptive_weight()` and in `check_correct_parent()`), we first cut all barrier
subtrees from the link-cut weight trees accordingly to get the state of the
past view. After the computation, we restore these anticone subtrees.

### Check Correct Parent

To check whether a new block chooses a correct parent block or not, we first
compute the set of blocks inside the epoch of the new block assuming that 
the new block is on the pivot chain. We store this set to the field
`blockset_in_own_view_of_epoch`. We then iterate over every candidate block in
this set to make sure that the chosen parent block is better than it.
Specifically, we find out the two fork blocks of the candidate block and the
parent block from their LCA and make sure that the fork of the parent is
heavier. This logic is implemented in `check_correct_parent()` in
`ConsensusNewBlockHandler`.

Note that `blockset_in_own_view_of_epoch` may become too large to hold
consistently in memory as well. Especially if a malicious attacker tries to
generate invalid blocks to blow up this set. The current implementation will
only periodically clear the set and only keep the sets for pivot chain blocks.
Note that for pivot chain blocks, this set will also be used during the
transaction execution.

### Fallback Brute Force Methods

There are situations where the anticone barrier set is too large if a malicious
attacker tries to launch a performance attack on Conflux. This will make the
default strategy worse than O(n) because there is a factor of O(log n) for each
block in the barrier set when we do the link-cut tree chopping. To this end, we
implemented a brute force routine `compute_subtree_weights()` to compute the
subtree weights of each block in a past view for O(n). We also implement
`check_correct_parent_brutal()` and `adaptive_weight_impl_brutal()` to use the
brute-force computed subtree weight to do the checking instead. 

### Force Confirmation

The Conflux consensus algorithm will *force confirm* a block if 1) there are
`timer_chain_beta` consecutive timer chain blocks under the subtree of the
block and 2) afterward there are at least `timer_chain_beta` timer chain blocks
following (not required in the subtree though). Force confirmation means that
new blocks should follow this block as their ancestor no matter what, ignoring
subtree weights. Though extremely unlikely a force confirmed block will have
lesser weights than its siblings.

The force confirmation mechanism is to enable checkpoint, which we will
describe later. It is based on the rationale that:

1. Reverting a `timer_chain_beta` length timer chain is impossible.

2. Therefore force confirmed block will always move along the pivot chain, not
drifting between its siblings.

We compute the accumulative LCA of the last `timer_chain_beta` timer chain
blocks and store it at the `timer_chain_accumulative_lca[]` field. This vector
is `timer_chain_beta` shorter than `timer_chain[]` because the force confirm
needs at least `timer_chain_beta` timer chain block trailing, so their LCAs do
not matter. `check_correct_parent()` and `adaptive_weight()` and their
subroutines also respect this force confirm point during their checking.
Specifically, any fork before the force confirm height is ignored.

Note that this force confirm rule is also defined based on *past view* of each
block. With the computed anticone information, `compute_timer_chain_tuple()` in
`ConsensusGraphInner` computes the timer chain related information of each
block under its past view. The results of this function include the difference of
the `timer_chain[]`, `timer_chain_accumulative_lca[]`, and `timer_chain_height`
between the ledger view and the past view. We can use the diff and the current
ledger view values to get the past view values.

### Era

In order to implement the checkpoint mechanism, the Conflux consensus algorithm split the
graph into eras. Every era contains `era_epoch_count` epochs. For example, if the
`era_epoch_count` is 50000, then there is a new era every 50000 epochs. The
pivot chain block at the height 50000 will be the genesis of a new era.
At the era boundary, there are several differences from the normal case.

1. A block will enter the total order for execution only if 1) it is under the
subtree of the previous era genesis and 2) it is inside the past set of the next era genesis in
the pivot chain.

2. Anticone penalty calculation for the block reward does not go across the era
boundary.

### Checkpoint

Inside `ConsensusGraphInner`, there are two key height pointers, the current
checkpoint era genesis block height (`cur_era_genesis_height`) and the current
stable era genesis block height (`cur_era_stable_height`). These two height pointers
will always point to some era genesis (being a multiple of `era_epoch_count`).
Initially, both of these two pointers will point to the true genesis (height
0).

A new era genesis block becomes stable (i.e., `cur_era_stable_height` moves) if
the block is *force confirmed* in the current TreeGraph. A stable era genesis
block becomes a new checkpoint (i.e., `cur_era_genesis_height` moves) if:

1. The block is *force confirmed in the past view of the stable era genesis block*. 

2. In the anticone of this block, there is no timer chain block.

`should_move_stable_height()` and `should_form_checkpoint_at()` in
`ConsensusNewBlockHandler` are invoked after every newly inserted block to test
the above two conditions. Generally speaking, the stable era genesis block will never be
reverted off the pivot chain. Any block in the past set of the checkpoint block
is no longer required for the future computation of the consensus layer.
Therefore, after a new checkpoint is formed, `make_checkpoint_at()` in
`ConsensusNewBlockHandler` is called to clean up those blocks that are not in
the future set of the new checkpoint.

Note that the checkpoint mechanism also changes how we handle a new block. For
a new block:

1. If the new block is outside the subtree of the current checkpoint, we only
need to insert a stub into our data structure (because a block under the
subtree may be indirectly referenced via this stub block). We do not need to
care about such a block because it is not going to change the timer chain and it
is not going to be executed.

2. If the past set of the new block does not contain the stable era genesis block, we
do not need to check the partial invalid status of this block. This is because
this block will not change the timer chain (recall our assumption that the timer
chain will not reorg for more than `timer_chain_beta` blocks) and future blocks can reference
this block directly (since the timer chain difference is already more than `timer_chain_beta`).

### Deferred Execution

Because the TreeGraph pivot chain may oscillate temporarily, we defer the
transaction execution for `DEFERRED_STATE_EPOCH_COUNT` epochs (default 5).
After a pivot chain update, `activate_block()` routine will enqueue the
execution task of the new pivot chain except for the last five epochs. It calls
`enqueue_epoch()` in `ConsensusExecutor` to enqueue each task.

### Block Reward Calculation

Because there is no explicit coinbase transaction in Conflux, all block rewards
are computed implicitly during the transaction execution. In Conflux, the block
reward is determined by the base reward and the penalty ratio based on the total weight of
its anticone blocks divided by its epoch pivot block's target difficulty. This anticone set only
considers blocks appearing no later than the next `REWARD_EPOCH_COUNT` epochs.
Specifically, if there is a new era then the anticone set will not count across
the era boundary as well. `get_pivot_reward_index()` in `ConsensusExecutor`
counts this reward anticone threshold.
`get_reward_execution_info_from_index()` in `ConsensusExecutor` and its
subroutines compute this anticone set given the threshold point in the pivot
chain.

### Blaming Mechanism

It is infeasible to validate the filled state root of a block because we
would need to execute all transactions in a different order in the past view of
that block. Instead, we will only ask full nodes to validate the state root
results on the current pivot chain. It then fills a blame number to indicate
how many levels ancestors from the parent who do not have correct state root.
When this number is greater than zero, the filled deferred state root becomes a
Merkel H256 vector that contains the corrected state roots of the ancestors
along with the correct one. `get_blame_and_deferred_state_for_generation()` in
`ConsensusGraph` computes the blame information for the block generation.
`first_trusted_header_starting_from()` in ``ConsensusGraph`` is a useful helper
function to compute the first trustworthy header based on the subtree blame
information.

## Multi-Thread Design

The consensus layer has one thread dedicated to processing new blocks from the
synchronization layer and one thread dedicated to executing transactions. It of
course also has a set of interface APIs that RPC threads and synchronization
threads may call.

### Consensus Worker

``Consensus Worker`` is a thread created by the synchronization layer. During
the normal running phase, every new block will be sent to a channel connecting
the synchronization thread and the consensus worker thread. The consensus work
thread consumes each block one by one and invokes `consensus::on_new_block()`
to process it. Note that the synchronization layer ensures the new block to be
*header-ready* when it is delivered to `Consensus Worker`, i.e., all of its
ancestor/past blocks are already delivered to the consensus layer before itself. 
This enables the consensus layer to always deal with a well-defined
direct acyclic graph without holes.

One advantage of having a single thread to be dedicated to the consensus
protocol is that it simplifies the protocol implementation a lot. Because the
details of the consensus protocol are complicated and the implementation involves
many sophisticated data structure manipulations, the single thread design makes
sure that we do not need to worry about deadlocks or races. Upon the entrance
of `consensus::on_new_block()`, the thread acquires the write lock of the inner
of the consensus struct (i.e., ConsensusGraphInner). During the normal phase,
this thread should be the only one modifying the inner struct of the consensus
layer.

### Consensus Execution Worker

`Consensus Execution Worker` is a thread created at the start of the consensus
layer. It is dedicated to transaction execution. There is a channel connecting
`Consensus Worker` with `Consensus Execution Worker`. Once the consensus
protocol determines the order of the pivot chain, it will send an `ExecutionTask`
for each epoch in the pivot chain to the channel. These tasks will be picked up
by the `Consensus Execution Worker` thread one by one. The thread loads the
previous state before the executed epoch from the storage layer as the input,
runs all transactions in the executed epoch (see
``ConsensusExecutor::process_epoch_transactions()``), and produces the result
state as the output.

The rationale of separating the transaction execution from the consensus
protocol implementation is for performance. With our *blaming mechanism*, the
execution result state is completely separated from the consensus protocol
implementation. The *deferred execution mechanism* gives us extra room to
pipeline the consensus protocol and the transaction execution. It is therefore
not wise to block the `Consensus Worker` thread to wait for the execution
results from coming back.

## Key Assumptions, Invariants, and Rules

If you want to write code to interact with the Conflux consensus layer, it is
very important to understand the following assumptions and rules.

1. The consensus layer assumes that the passed `BlockDataManager` is in a
consistent state. It means that the `BlockDataManager` contains the correct current
checkpoint/stable height. Blocks before the checkpoint and the stable height
are properly checked during previous execution and they are persisted into the
`BlockDataManager` properly. The consensus layer **does not check** the results
it fetches from the block data manager. If it is inconsistent, the consensus
layer will execute incorrectly or crash!

2. Besides the subroutines of `on_new_block()`, **no one should hold the write
lock of the inner struct**! Right now the only exception for this rule is
`assemble_new_block_impl()` because of computing the adaptive field and this is
not good we plan to change it. Acquiring the write lock of the inner struct
is very likely to cause deadlock given the complexity of the Consensus layer
and its dependency with many other components. Always try to avoid this!
