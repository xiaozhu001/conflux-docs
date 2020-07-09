# Conflux共识层设计与实现

Conflux共识层处理所有从同步层接收到的区块，基于Conflux的GHAST共识算法产生区块的总顺序，并调用其底层的**交易执行引擎**，按照确定的顺序运行交易。它提供了协助**区块生成器**准备新区块骨架的必要信息。它同时还会将已处理过的交易情况通知**交易池**，以便交易池能够做出更好的交易选择决策。

本文档是为想要了解Conflux共识层（在目录：core/src/consensus中）的rust实现细节的读者提供了一个顶层描述。如果需要查看更多的实现细节，可以关注代码内的注释。如果需要了解更多Conflux共识算法的信息，可以关注Conflux协议规范和论文(https://arxiv.org/abs/1805.03870) 。 

## 设计目标

共识层有以下的设计目标。 

1. 在后台按照共识算法持续处理新的区块。

2. 我们希望将共识图内的每个区块的内存使用量最小化。即使采用了检查点机制，在正常情况下，图中也会包含300K-500K个区块，而在面临存活性攻击时，甚至会包含1M个以上的区块。这可能会为内存带来压力。

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

`ConfirmationMeter` (core/src/consensus/consensus_inner/confirmation_meter.rs)保守的计算每个主轴链区块的确认风险。其结果能够帮助存储层确定何时丢弃旧快照是*安全*的。如果我们决定提供这样的RPC，它也可以用来服务于区块确认的RPC查询。

### AnticoneCache and PastsetCache

`AnticoneCache` (core/src/consensus/anticone_cache.rs) 及 `PastsetCache`
(core/src/consensus/pastset_cache.rs)是两个实现了 `ConsensusGraphInner` 中数据结构的自定义缓存结构。 在内部结构的实现中，我们需要计算和存储anticone集和过去每个区块的集合。我们因此实现了缓存式的数据结构来存储最近插入/访问的区块集合。如果anticone/过去区块集合没有在缓存内找到，我们将在当前内部结构的实现中重新计算集合。

## 重要的算法机制

在Conflux的共识层中含有一系列重要的算法机制。在这里我们将从其实现的方面进行探讨。参见XXX背后的算法推理。

### 主轴链和总顺序

Conflux共识算法的基本思想是先让大家任何主轴链。随后它通过拓扑排序将主轴链的总顺序扩展到覆盖所有区块。由于主轴链不会改动/重组，区块的总顺序和派生的交易顺序将保持不变。

与比特币/以太坊相比，Conflux中的共识有两个关键的区别：

1. *几乎所有区块* 都会进入到总顺序中，而不是仅仅接受主轴链。

2. 交易的有效性和区块的有效性是 *相互独立的* 。例如如果某项交易在之前已被纳入记录或因为余额不足而无法进行，则该交易无效。这类无效交易在执行期间会成为空操作。然而*比特币和以太坊中包含这类交易的区块不会变为无效状态*。

在 `ConsensusGraphInner` 中，当前主轴链的arena索引会按顺序存储在 `pivot_chain[]` 向量中。为了维护它，我们依照GHAST规则计算新插入的区块和当前最优区块之间的最近公共祖先（LCA）。如果对应于最近公共祖先的新插入区块的分叉最终变得更重，我们将从分叉点处更新 `pivot_chain[]` 。

### 时钟链

PoW质量比目标难度高 `timer_chain_difficulty_ratio` 倍的区块是*时钟区块*。区块中的 `is_timer` 字段会被设置为真。共识算法随后按照类似于比特币共识算法查找最长链的方法查找最长的时钟链（更精确地说，是具有最高的累计难度）。最长时钟链的arena索引会被存储至 `timer_chain[]` 中。

时钟链的基本原理是为了提供一种不受恶意攻击者影响的粗粒度时间度量方案。由于时钟区块稀少且生成速度较慢（如果`timer_chain_difficulty_ratio` 适当高），一个恶意的攻击者除非具备大部分的算力，否则无法限制时钟链的增长。因此，在过去的区块集中出现多少个时钟链能够很好地表示区块最可能的新生成时间。我们为每一个区块计算该值并将其存入到区块中的 `timer_chain_height` 字段。

### 使用动态树维护权重信息

为了更高效的的维护主轴链，我们需要查询子树的总权重。Conflux使用动态树数据结构以O(log n)的效率维护子树权重信息。动态树还可以以O(log n)的效率树图中任意两个节点的最近公共祖先（LCA）。 `ConsensusGraphInner` 中的 `weight_tree` 字段是存储每个节点子树权重的动态树信息。请注意，动态树的实现位于utils/link-cut-tree目录下。

### 自适应权重Adaptive Weight

如果树图正在面临存活性攻击时，可能会在一段时间里无法收敛到一个块中。为了处理这种情况，GHAST算法思路是开始生成自适应区块，即那些权重明显被重新分配的块，因此将会有许多零权重的块，其中也会包含非常少的非常重的块。特别的，如果一个自适应块的工作量证明质量是目标难度的 `adaptive_heavy_block_ratio` 倍，该区块的权重会是 `adaptive_heavy_block_ratio`； 否则该区块的权重为0。这样的思路会暂时的减缓确认速度，但能够确保共识进度。

由于自适应权重是一种能够防御稀有存活性攻击的机制，因此在正常情况下不应将其打开。只有在以下的情况下，新区块才是一个自适应的区块：1）它的一个祖先块与它的兄弟节点相比，仍旧不是主要的子树，和2）祖先区块的生成时间和新区块的生成时间之间已经有了很长一段时间（即，`timer_chain_height` 的差距足够大）。 `ConsensusGraphInner::adaptive_weight()` 及其子例程实现了决策区块是否是自适应块的算法。需要注意的是其实现使用到了另一个动态树 `adaptive_tree` 作为辅助。请查看内嵌注释以获取实现细节。

### 部分无效

需要注意的是，新生成区块的过去集合表示块生成器在生成区块时观察到的所有区块。因此，通过新生成区块的过去集合，其他全节点可以确定其是否选择了正确的父区块以及其是否为自适应块。

Conflux共识算法将这些选取不正确父区块和填充不正确自适应状态的区块定义为*部分无效区块*。对于一个部分无效区块，其 `partial_invalid` 字段将被设置为真。算法需要按照三种方式将部分无效区块与正常区块区分对待：

1. 所有的诚实节点在相当长的一段时间内不会直接或间接引用这些部分无效区块。这段时间使用 `timer_chain_height` 度量，且差值必须大于 `timer_chain_beta` 。是的，这意味着如果另一个完美无缺的区块引用了部分无效区块，则这两个区块在一段时间内都不会被引用。

2. 部分无效区块会没有区块奖励。由于规则一，它们的光锥外区块集合较大，因此它们极有可能不获得任何奖励。

3. 在时钟链的考虑中会排除部分无效区块。

为了实现规则一，在 `ConsensusNewBlockHandler` 内的 `on_new_block()` 例程会被分割为 `preactivate_block()` 和 `activate_block()` 两个子例程。 `preactivate_block()` 计算并确定区块是否为部分无效区块，而 `activate_block()` 会将区块集成到共识图的内部数据结构中。对于每个新区块， `active_cnt` 追踪其引用了多少个不活跃区块。如果一个区块直接或间接的引用一个部分无效区块，该区块将会是不活跃区块。只有在区块的
`active_cnt` 变为0时 `activate_block()` 才会被调用。 `activated` 表示区块是否活跃。对于部分无效区块来说，其激活操作将被延迟知道当前账本的时钟链比无效区块高 `timer_chain_beta` 时。新生成的区块不会引用任何不活跃区块，如，这些不活跃块将被视为不在树图中。

### 光锥外、过往视图和账本视图

为了检查每个区块的部分无效状态，我们需要对区块的*过往视图*进行操作以决定其正确的父区块及适应性。这与树图的当前状态不同，或者说我们称之为或称之为*账本视图*，即所有在光锥外块集合和未来集合中的区块都会被排除在外。由于我们会按照拓扑顺序处理区块，新区块的未来集合为空。因此，我们只需要消除所有的光锥外区块即可。

`ConsensusNewBlockHandler` 中的 `compute_and_update_anticone()` 负责计算新区块的光锥外块集合。要注意的是由于光锥外块集合可能会非常大，我们在实现时进行了两个层面的优化。一，在树图中将光锥外块集合标识为障碍节点集合，即树内每一个区块都在光锥外块集合中的一组子树。二，我们只会维护最近访问/插入区块的光锥外块集合。当检查一个区块在其过往视图中是否是有效时（如，`adaptive_weight()` 和 `check_correct_parent()`），我们首先对动态权重树中的障碍子树进行剪枝，得到过往视图的装填。在计算后，我们会恢复这些光锥外子树。

### 检查正确的父区块

为了检查一个新的区块是否选取了正确的父区块，通过假设新区块在主轴链上，我们首先计算了新区块所在纪元内的区块集。将集合存储到字段 `blockset_in_own_view_of_epoch` 处。随后，我们对这个集合中的每一个候选区块进行迭代，以确保所选的父区块比它更好。具体来说，我们从其最近公共祖先中找出候选块和父块的两个分叉块并确保父区块所在的分叉权重较高。这一逻辑在
`ConsensusNewBlockHandler`的 `check_correct_parent()` 中实现。

需要注意的是 `blockset_in_own_view_of_epoch` 可能会变得太大而无法在内存中保持一致。尤其是如果恶意攻击者尝试生成无效区块以至于溢出该集合。当前的实现中会定期清理集合，并且仅让集合只保留主轴链区块。请注意对于主轴链区块，该集合还会在交易执行期间使用。

### 蛮力法的的应变计划

如果恶意攻击者尝试向Conflux发起性能攻击，就会出现光锥外障碍区块集合过大的情况。这将使默认策略比O(n)更糟糕，因为在进行动态树剪枝时，障碍集中的每个区块都会有一个O(log n)因素。为此，我们实现了一种蛮力例程 `compute_subtree_weights()` ，以便以O(n)的效率计算过去视图中每个区块的子树权重。我们也实现了
`check_correct_parent_brutal()` 和 `adaptive_weight_impl_brutal()` 检查使用蛮力计算得到的子树权重。

### 强制确认

Conflux共识算法会*强制确认*一个区块如果：1）区块的子树下有
`timer_chain_beta` 个连续的时钟链区块及2）之后至少有 `timer_chain_beta` 个时钟链区块跟随（尽管在子树中不是必需的）。强制确认意味着无论如何，新区块都应将该区块作为其祖先区块，而忽略子树权重。尽管被强制确认区块的权重几乎不可能比其兄弟节点轻。

强制确认机制用于启用检查点，我们将在随后进行介绍。基于以下理由：

1. 几乎不可能恢复 `timer_chain_beta` 长度的时钟链。

2. 因此强制确认的块将始终沿主轴链移动，而不是在其同级区块之间四处移动。

我们计算最后一个 `timer_chain_beta` 时钟链区块的累计最近公共祖先，并将其存储在 `timer_chain_accumulative_lca[]` 字段中。该项量是比 `timer_chain[]` 短 `timer_chain_beta` ，由于强制确认至少需要有 `timer_chain_beta` 个时钟链区块尾随，因此它们的最近公共祖先就显得无关紧要了。 `check_correct_parent()` 和 `adaptive_weight()` 及它们的子例程在进行检查期间也同样遵守这一强制确认思路。具体的，强制确认高度前的任何分叉都会被忽略。

注意，该强制确认规则也是基于对每个区块“过往视图”定义的。通过使用计算得到的光锥外信息， `ConsensusGraphInner` 中的 `compute_timer_chain_tuple()` 结合其过往视图计算每个区块与时钟链相关的信息。该函数的结果包括分账视图和过往视图之间 `timer_chain[]` 、 `timer_chain_accumulative_lca[]` 和 `timer_chain_height`的差异。我们可以使用差异和当前的分账视图值获取过往视图值。

### 时代

为了实现检查点机制，Conflux共识算法将图划分为时代（era）。每一个时代都包含 `era_epoch_count` 个纪元。例如，如果
`era_epoch_count` 为50000，那么每50000个纪元就是一个新时代。高度为50000的主轴链区块将会成为新时代的创世点。在时代的边界上，相比一般情况有几个不同。

1. 一个区块只有在以下情况时才会进入全序中执行： 1）它在上一个时代创世点的子树下，及2）它在下一个时代创世过往集的主轴链中。

2. 区块奖励的光锥外惩罚计算不会横跨时代边界。

### 检查点

在 `ConsensusGraphInner` 内有两个关键的高度指针，当前检查点时代创世区块高度（`cur_era_genesis_height`）及当前稳定时代创世区块高度（`cur_era_stable_height`）。这两个高度指针总是会指向一些时代的创世区块（是 `era_epoch_count` 的倍数）。初始时，这两个指针都将指向真正的创世块（高度为0）。

如果在区块在当前树图中被*强制确认*，则新的时代的创世块将变得稳定（即 `cur_era_stable_height` 指针会移动）。如果出现以下情况，稳定时代的创世区块将成为新的检查点（即 `cur_era_genesis_height` 指针会移动）：

1. 该区块*在稳定时代创世区块的过往视图中已经被强制确认*。

2. 在该区块的光锥外中，没有任何时钟链区块。

`ConsensusNewBlockHandler` 中的 `should_move_stable_height()` 及 `should_form_checkpoint_at()` 会在每一个区块新插入后被调用以测试上述两个条件。一般来说，稳定时代的创世区块永远不会从主轴链中消失（reverted off）。共识层的未来计算时，已不再需要检查点区块过往集合中的任何区块。因此，在新的检查点形成时， `ConsensusNewBlockHandler` 中的 `make_checkpoint_at()` 会被调用以清除这些不再新检查点未来集中的区块。

值得注意的是检查点机制也改变了我们对新区块的处理方式。对于一个新区块来说：

1. 如果新区块在当前检查点的子树外，则只需要在我们的数据结构中插入一个存根（因为子树下的一个区块可能会被此存根区块间接引用）。我们不需要关心这类区块，因为它既不会改动时钟链也不会被执行。

2. 如果新区块的过往集合不包含稳定时代的创世区块，则不需要检查该区块的部分无效状态。这是由于该区块不会修改时钟链（回想一下我们的假设，即重组后时钟链不会超过 `timer_chain_beta` 个区块）且随后的区块可以直接参考该区块（因为时钟链的差距已经超过 `timer_chain_beta` ）。

### 延期执行

由于树图的主轴链可能会出现临时性的震荡，所以我们会推迟 `DEFERRED_STATE_EPOCH_COUNT` 个纪元（默认为5）的交易执行。随着主轴链的更新， `activate_block()` 例程将会在排除最后5个纪元的情况使新主轴链的执行任务进入队列。通过在 `ConsensusExecutor` 中调用
`enqueue_epoch()` 来使每个任务进入队列。

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
