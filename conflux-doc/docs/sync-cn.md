# 区块同步过程

## 同步图
通过设计同步图是为了组织新到区块（从对等节点收到、从本地存储加载、或自挖取），即使过去的区块尚未被完全收集。一旦一个区块的所有前序区块都被收集到同步图内，它将被调度到共识图中以进行进一步的处理。

区块头和区块体分别通过不同的进程进入同步图，其原因是通常情况下，区块头和区块体是通过点对点层分别传输的。同步图的图结构是根据区块头的到达情况构建的。在图结构内每一个区块都表示为图中的一个节点，节点间通过区块间的父/子和引用/被引关系进行连接。

同步图对新到区块的有效性进行检验。未通过有效性检查的区块是无效的，不会被派发给共识图。有效性检查过程按如下方式进行：

1.	检查区块的父区块或被引区块是否无效，如果其中一个无效，则该块同样无效。
2.	根据区块头中设置的难度，检查区块头中的随机数是否正确设置，即检查该区块对应矿工是否正确解决了工作量证明POW难题。
3.	检查区块头中的引用数是否大于阈值（200），如果是，则该区块是无效的。
4.	检查区块父区块或被引区块中是否存在重复的哈希值，如果存在，则该区块无效。
5.	检查区块头中自定义区段的字节长度是否超过阈值（64），如果超过，则该区块无效。
6.	检查区块高度是否比其父区块高度大1，如果不是，则区块无效。
7.	检查区块的时间戳是否大于或等于其父区块的试驾戳，如果不是，则该区块无效。
8.	检查区块燃料限制数是否设置正确。
9.	检查区块难度是否设置正确。
10.	根据区块体中的交易，检查区块头是否包含了正确的交易根信息。
11.	检查区块体中的交易是否具有有效的签名结构。
12.	检查区块主体中的交易总大小是否超出区块大小限制（800KB），如果是，则该区块无效。
13.	检查区块体中交易的燃料限制综合是否超出整个区块的燃料限制上界，如果是，则该区块无效。

有效性检查中的1-9部分只使用区块头中的信息，10-13部分会使用区块体中的信息。有效性检查的6-9部分需要诸如父节点信息一样的图结构信息，当一个区块过去所有区块的头部信息都进入同步图时，会对该区块进行检查。为了加快区块中继过程，当一个区块的区块头和区块体信息都进入同步图，且其过去所有区块的头部信息都进入时，就可以将区块中继传递给对等节点。而不需要等待一个区块过去所有区块的区块体信息被接收后才能中继该区块。这可能会导致中继无效的区块，但由于所有被中继的区块都已经有了有效的难度和工作量证明POW设置，所以攻击者在实现这类攻击的同时也需要付出相应的算力成本。

### 图结构维护
同步图的节点结构定义如下：
```c
pub struct SynchronizationGraphNode {
    pub block_header: Arc<BlockHeader>,
    // The status of graph connectivity in the current block view.
    pub graph_status: u8,
    // Whether the block body is ready.
    pub block_ready: bool,
    // Whether parent is in old era and already reclaimed
    pub parent_reclaimed: bool,
    // The index of the parent of the block.
    pub parent: usize,
    // The indices of the children of the block.
    pub children: Vec<usize>,
    // The indices of the blocks referenced by the block.
    pub referees: Vec<usize>,
    // The number of blocks referenced by the block but
    // haven't been inserted in synchronization graph.
    pub pending_referee_count: usize,
    // The indices of the blocks referencing the block.
    pub referrers: Vec<usize>,
    // the timestamp in seconds when graph_status updated
    pub last_update_timestamp: u64,
}

```

The graph structure is maintained by the fields `parent`, `children`, `referees`, and `referrers`. Each node has a `graph_status` field representing its status changing during the period from the time when the header arrives to the time when the block is ready to be dispatched to consensus graph. 

When a block header enters synchronization graph, it is first checked whether this block is already processed by synchronization graph. 
If not, it will be added into the graph and the graph structure will be updated accordingly. 
First, the *parent/children* edge will be established. 
If the parent of this newly arrived block hasn’t come into synchronization graph, the graph uses a collection `children_by_hash` to handle this. 
This is a map from block hash to a set of graph nodes. 
In this case, the graph node representing this newly arrived block header will be added into the graph node set mapped from the parent block hash of this block. 
This functions as a bookkeeping to remember the relation between the parent block hash and this newly arrived block. 
Once this parent block comes into the synchronization graph in the future, the corresponding edge between the parent and child nodes can be established and the hash of the parent block will be removed from the `children_by_hash` map. 
Secondly, the *referees/referrers* edges will also be established. 
Similarly, the synchronization graph uses a map `referrers_by_hash` to remember the relation between the unarrived referee block and the newly arrived referrer block. 
The graph node also maintains a `pending_referee_count` field to remember how many referees of the block haven’t come into the synchronization graph.

A graph node may be in the following 5 status:
```c
// This block is an invalid block.
const BLOCK_INVALID: u8 = 0;
// Just get the header of the block.
const BLOCK_HEADER_ONLY: u8 = 1;
// The headers of all the blocks in the past set of this block have already entered
// synchronization graph. 
const BLOCK_HEADER_GRAPH_READY: u8 = 2;
// Both the headers and bodies of all the blocks in the past set of this block have
// entered synchronization graph.
const BLOCK_GRAPH_READY: u8 = 3;
```

当一个区块头刚刚进入同步图并触发图中创建和添加一个新的图节点时，该节点的初始状态为 `BLOCK_HEADER_ONLY` 。
由于图结构的更新，图中其他节点的状态也可能发生变化。
这类变化的效果是通过节点向其所有子孙进行广度优先搜索遍历实现的。
在这个遍历过程中，对于每一个节点，
1) 如果它是无效的，那么它的所有子孙节点都是无效的；
2) 如果它是新设置的 `BLOCK_HEADER_GRAPH_READY` ，则需要对它进行一系列与图相关的有效性检查（6-9）。
如果它通过了这些检查，随后需检查其区块体是否已进入同步图内（通过检查图节点中的 `block_ready` 字段）。
如果是，则该区块就可以被转发了。而该区块可能会影响使得其后代成为 `BLOCK_HEADER_GRAPH_READY` 。
需要注意的是，这不能使其后代成为 `BLOCK_GRAPH_READY` ，因为新到达区块头的原始节点（广度优先搜索过程的起点）只能是 `BLOCK_HEADER_GRAPH_READY` 。

当区块体进入同步图内，相应的图节点应当已存在于同步图中，否则该区块将被忽略（当进行垃圾收集时可能会出现这种情况）。
该节点的 `block_ready` 字段现在将被设置为true。
随后，该区块会进过相应的有效性检查（10-13）。
而同样的，这个新到达的区块体也会改变其一些后代的状态。
这也是通过从该节点进行广度优先搜索遍历实现的。在该遍历过程中，对于每个节点，
1) 如果它是无效的，则它的所有子孙节点都是无效的；
2) 如果它是新设置的 `BLOCK_HEADER_GRAPH_READY` ，其将被派发至共识图。
这可能会使其部分后代成为 `BLOCK_GRAPH_READY` 。 
如果新到达块的主体至少为 `BLOCK_HEADER_GRAPH_READY` ，则可以成为被中继转发的对象。

### Garbage Collect Dangling Blocks
一些（敌手）节点可能会向全节点发送一些永远不能处于 `BLOCK_GRAPH_READY` 状态的区块，例如在进行DDOS攻击或在消息严重延迟的情况下，会使该区块不再属于当前的检查点。
这些区块将被保存在同步图中，为避免内存资源的浪费，最终应当被作为垃圾回收。 
为做到这一点，同步图会维护一组代表图尚未准备好的区块边界区块信息。
一个区块会被判定为边界区块如果
1) 区块状态非 `BLOCK_GRAPH_READY` 但其父节点是；或 
2) 其父区块没有进入到同步图内。

为了对这些图未准备好的区块进行垃圾回收，将从这些区块的边界开始，并删除它及其所有的子代区块，这些子代区块可以通过对子边和引用边使用广度优先搜索遍历实现。
我们必须从同步图中删除所有边界区块的字区块，其原因与区块同步过程的设计有关。
在区块同步的过程中，它将按照新到达的区块父边和引用边常识获取缺失的祖先块。当遇到一个已经存在于同步图中的祖先区块时，进程会停止跟踪其父边和参考边。
因此，如果祖先区块是某个已经被垃圾收集的图未准备好区块的子孙区块，那么这个被移除的未准备好的块将永远不会再有机会从对等节点处获取。
但是，这个被移除的边缘区块可能仅仅是因为临时的网络状况变差，以后可能会恢复。

垃圾收集过程是周期性触发的，在每一次收集过程中，它仅仅尝试移除长时间处于未准备状态的边缘区块及其子区块。为了获取时间信息，每个图节点都包含字段 `last_update_timestamp` 以记录上一次更新节点状态的时间戳。

一个可以对同步过程进行优化的方法是：不总是从对等节点处获取内存中缺失的新到达的区块的父区块和引用区块。
它检查该区块的高度是否远远小于本地最佳纪元区块的高度。
如果是这样，那么这个区块的祖先极有可能存储于本地数据库中，值得我们努力从本地数据库中获取这些区块。
这也是仅仅根据对等节点信息避免进行不必要的后向树状图遍历的有效方法，这一点我们在[恢复过程](recovery-cn.md)一节也有比较详细的介绍。