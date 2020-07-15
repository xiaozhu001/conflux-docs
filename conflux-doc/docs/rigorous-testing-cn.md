# 针对Conflux的严格测试工具

确保像Conflux这样的区块链系统的正确性是一项艰巨的任务。Conflux Rust实现的仓库中带有一些严格的测试工具和脚本。

请注意，在某些终端中，打开文件描述符的默认最大数量可能不够。如果您使用带有默认zsh终端的Mac则尤其如此。您将需要将限制更改为更大的数字，如下所示：

```bash
$ ulimit -n 22288
```

## 单元测试和集成测试

单元测试与rust代码一起提供。可以从源代码级别编译Conflux之后，可以通过 `cargo test
--release --all` 执行。有关更多的信息，请参考[启程](https://conflux-chain.github.io/conflux-doc/get_started/)。集成测试是存储在目录 `tests/scripts` 中以 `_test.py` 为结尾的python测试脚本。从代码级别编译Conflux的*发行*版本后。可以通过运行 `tests/test_all.py` 一起运行所有集成测试。这些测试会在Conflux Rust实现的每次提交中例行执行。

## 共识的模糊测试工具

在`core/benchmark/consensus/test`目录下，有一款随机模糊测试工具，用于对共识组件进行测试。它按照如下步骤进行工作。 `core/benchmark/consensus/test/gen-random-graph.cpp`是Conflux树图共识算法的C++实现（速度较慢），以及一个随机图生成器，它能以特殊格式生成随机的树图块。`consensus_bench` 能够处理该输入格式，运行Conflux共识并将其结果与速度较慢的C++实现进行比较。 `iter-gen-random.py` 是一个反复调用生成-处理-比较过程的Python脚本。下面给出了运行该工具的方法：

```bash
$ cd core/benchmark/consensus/test
$ g++ -O2 -o gen-random-graph gen-random-graph.cpp
$ ./iter-gen-random.py 10000 3 30 10 10 100
```

Python脚本在找到错误或用户手工终止之前不会停止。如果Python脚本发现错误，则 `rand.in` 文件将会对 `consensus_bench` 程序的错误触发输入进行对应。传递给Python脚本的六个参数分别对应于每个测试用例随机生成的块数，分别为
`TIMER_CHAIN_BLOCK_DIFFICULTY_RATIO` 参数、 `TIMER_CHAIN_BETA` 参数、 `ADAPTIVE_WEIGHT_BETA` 参数、
`HEAVY_BLOCK_DIFFICULTY_RATIO` 参数以及 `ERA_EPOCH_COUNT` 参数。您可以将任何合法的共识参数传递给Python。这些数字是默认选择的，我们通过经验发现它们对检测错误很有帮助。

Python脚本还将打印出测试中共识图的处理速度。预期速度为每秒约1000个区块（在MacBook Pro 2019笔记本电脑上），而在m5a.xlarge上为每秒约350个区块。如果报告的速度大大低于预期的速度，则通常意味着潜在的性能问题。对于每个发行版，我们使用的默认参数至少需要执行一个小时的模糊测试。

请注意，如果您暴力的终止了该脚本（你很可能喜欢这样做）。它会留下两到三个前缀带有 `__` 和 `sqlite_db` 的临时目录。您应该手工删除这些目录。

## 随机追踪测试

`tests/conflux_tracing.py` 是一个包含错误注入能力的随机测试脚本。它将以固定的节点数启用Conflux网络，并注入节点崩溃、db崩溃和节点重启。在运行期间，它会不断从不同的节点上获取状态，并验证这些节点是否有树图和快状态的共识。要运行Conflux追踪测试您需要首先从源码级别编译Conflux Rust实现的发布版本。随后你可以调用如下脚本：

```bash
$ tests/conflux_tracing.py run
```

随后Python脚本将启动10个不同的实例及一个模拟实例。它将不停地运行，直至发现一个错误（不一致的状态或意外的崩溃）。碎玉每一个发型版本，我们都会执行该跟踪脚本至少一小时。

如果出现错误，它将生成跟踪文件`snapshot*.json` 和
`txs*.json`来帮助诊断问题。请注意，如果你暴力终止该脚本（你可能会这样做）。它也会生成上述文件，所以你可能要手动清理它们。

## 交易传递及性能测试

`tests/scripts/one_click.sh` 以及其余的bash脚本都位于同一个目录中，可以在AWS上自动部署Conflux网络，以测试简单的交易TPS和交易池性能。您可以按照以下方式运行该测试：

1. 首先，您需要下载并安装AWS CLI工具。 正确配置CLI工具的AWS凭证。

2. 使您的默认公钥在*the us-west-2 region*中登记为命名密钥对。

3. 确定您要测试的Conflux仓库的分支。请注意，此脚本从包含Conflux rust实现的GitHub存储库中提取源代码，并即时对其进行编译。您不能使用此脚本运行本地Conflux副本。如果您未指定存储库/分支名称，它将从GitHub的官方Conflux-rust存储库中提取源代码。

4. 运行以下命令：

```bash
$ cd tests/scripts
$ ./one_click.sh key-pair-name 20 branch-name [repo-name]
```

这将在us-west-2区域与随机交易生成器一起启动20个实例。设置实验环境大约需要15分钟，随后需要20分钟才能完成运行。最后，它将报告TPS性能。预期的良好TPS数量约为4000TPS。如果您获得的TPS数量远低于预期，则在事务池或存储层会出现性能下降。对于每个发行版，我们都会运行此脚本来测试其性能。

## 存储基准测试

Conflux中的存储层通常面临着性能瓶颈。 `core/benchmark/storage` 因此包含了一个基准工具以度量存储层的性能，并在执行中消除其它层。我们还转换了以太坊网络的历史支付交易信息（大约钱400万个区块）作为基准进行跟踪。一下是运行存储基准测试的步骤：

1. 从AWS S3的 `conflux-storage-bench` 存储桶中下载 `foundation.json` 及 `eth_from_0_to_4141811_txs.rlp.tar.gz` 。

2. 解压缩rlp历史文件以获得 `eth_from_0_to_4141811_txs.rlp` 。

3. 进入 `core/benchmark/storage` 目录并运行 `cargo build --release` 编译二进制文件 `storage_bench` 。

4. 创建一个临时目录 `tmp_storage_db` 用于保存实验中生成的区块链数据库。

5. 执行下列命令：

```bash
$ cd core/benchmark/storage
$ RUST_BACKTRACE=full target/release/storage_bench run -g /path/to/foundation.json -t /path/to/eth_from_0_to_4141811_txs.rlp -d /path/to/tmp_storage_db --txs_to_process 30000000 --skip 1156773812
```

该命令会处理已经过解析处理过的历史记录文件中的前3000万个交易，随后退出。计算存储层实现的处理吞吐量时，对该命令的运行时间进行计时是一个不错的想法。其具体性能将很大程度上取决于磁盘I/O的质量。在MacBook Pro2019中，吞吐量为25000-30000TPS。在m5a.xlarge中，吞吐量为15000-20000TPS，如果性能低于预期，则表明存储层实际表现存在潜在的退步。对于每一个发行版本，我们将运行该测试以检查存储层性能。

## 共识性能基准工具

共识的实现通常来说速度很快，在正常情况下每秒可以处理近一千个区块。但是，如果树图不稳定且包含很多分支时，共识组件可能会失败并回退到慢速例程中。在不稳定的情况下，性能是至关重要的，因为它与DoS攻击期间的追赶速度相对应。
`tests/attack_bench` 包含一系列python脚本，用于对攻击场景下的共识性能进行基准测试：

1. `fork_same_height_merge.py` 创建了一个包含月95000个区块的不稳定树图。在书图中具有三个分支，且每个分支中都有以固定高度连接的星形分叉。它对应于共识处理引擎所面对的最坏情况。MacBook Pro 2019的预期速度约为70个区块/秒，而m5a.xlarge的预期速度越位45个区块/秒。

2. `fork_same_height_hiding.py` 测试攻击者尝试在固定高度主动挖矿，隐藏所挖区块随后一并释放的情况。它测量的是受害人在该情况下的区块生成能力。预期的生成速度总是在1分钟内多余1000个区块。

3. `fork_same_height_attack.py` 测试与2类似但攻击者不隐藏区块的攻击。预期的生成速度通常为10秒内生成超过100个区块。

4. `fork_chain_hiding.py` 测试了攻击者试图主动挖掘一条单独的链，将所挖区块隐藏起来随后一并释放的场景。受害者的预期生成速度通常为10秒内生成超过100个块。

5. `fork_chain_attack.py`测试了与4类似但攻击者不隐藏区块的攻击。受害者的预期生成速度通常为10秒内生成超过100个块。

请注意，2、3和5是长期运行的测试脚本，你也可以在速度稳定后终止执行。对于每一个版本，我们都会运行这些脚本，以确保不会出现性能退步。