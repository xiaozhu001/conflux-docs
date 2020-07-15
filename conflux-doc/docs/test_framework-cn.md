# 测试框架

该框架使用 `python3` 实现。它可以设置多个Conflux节点，并在本地测试分布式系统的行为。可以通过设置节点配置信息、调用节点RPC或直接发送P2P消息实现对节点的控制。

所有相关的文件都包含在 `tests` 目录中。

在项目目录下使用 `cargo build --release` 编译源代码后，可以运行 `tests/test_all.py` 来运行所有包含python的测试。

## 一个测试样例

下面是一个测试示例：设置了2个结点，并让每个节点分别生成了一些区块，最后将它们连接起来以检查它们是否能接收对方生成的区块。
```python
from test_framework.test_framework import ConfluxTestFramework
from test_framework.util import *

class ExampleTest(ConfluxTestFramework):
    def set_test_params(self):
        self.setup_clean_chain = True
        self.num_nodes = 2

    def setup_network(self):
        self.setup_nodes()
        # connect_sample_nodes(self.nodes, self.log)

    def run_test(self):
        self.nodes[0].generate(1, 0)
        assert (self.nodes[0].getblockcount() == 2)

        self.nodes[1].generate(2, 0)
        assert (self.nodes[1].getblockcount() == 3)

        connect_nodes(self.nodes, 0, 1)
        sync_blocks(self.nodes)
        assert (self.nodes[0].getblockcount() == 4)
        self.log.info("PASS")

if __name__ == '__main__':
    ExampleTest().main()
```

该框架会

1. 调用 `set_test_params` 来设置基本的测试初始化参数。
2. 根据 `set_test_params` 中设置的参数设置测试目录和节点配置。默认情况下，会创建一个临时目录，所有文件都会保存在该目录中。例如设置 `self.num_nodes = 2` 会初始化两个节点的目录。
3. 调用 `setup_network` 来添加节点并连接它们。此处的 `self.setup_nodes()` 会在步骤2中设置的目录下运行预编译的Conflux可执行二进制文件以添加2个Conflux节点。在该步骤我们不会连接它们，因为我们希望节点在开始时是分开的。
4. 调用 `run_test` 运行实际的测试代码。

运行 `self.setup_nodes()` 后， `self.nodes` 是一个 `TestNode` 的列表，每个节点都可以用来和对应的Conflux节点进行交互。例如，如果要通过调用名为 `getblockcount` 的RPC获取节点0中的块数，只需要通过运行 `self.nodes[0].getblockcount` 即可返回一个整数。

`connect_nodes(self.nodes, 0, 1)` 连接节点0和节点1。 `sync_blocks(self.nodes)` 等待所有节点具有相同的枢轴链。它们都是通过调用RPC实现的，更多有用的功能将在[实用功能列表](#实用功能列表)中介绍。

## 发送P2P消息

调用 `start_p2p_connection(self.nodes)` 后，每个 `TestNode` 的 `p2p` 字段将被初始化为一个用Python语言编写的模拟Conflux节点，该模拟节点将被连接到对应 `TestNode` 控制的Conflux进程。随后，你就可以使用python代码发送和接收P2P消息了。下面是一个关于如何使用 `p2p` 与Conflux节点交互的例子。

```python
    def run_test(self):
        def assert_length(_node, msg):
            assert_equal(len(msg.headers), 1)
        h = WaitHandler(self.nodes[0].p2p, GET_BLOCK_HEADERS_RESPONSE, assert_length)
        self.nodes[0].p2p.send_protocol_msg(GetBlockHeaders(hashes=[self.nodes[0].p2p.genesis.hash]))
        h.wait()
```

该样例尝试用P2P请求（而不是使用RPC）从节点0处获取创世区块头，且保证只返回头信息。 

`WaitHandler` 将等待指定消息类型的第一条消息，并对收到的消息运行相应的函数。 `p2p.send_protocol_msg` 用于发送一个用RLP编码的消息。 `h.wait()` 等待并处理第一次收到的 `GET_BLOCK_HEADERS_RESPONSE` 消息。注意， `WaitHandler` 在初始化后就已经开始进行监听。

## 配置

默认情况下，测试将使用由 `cargo` 构建的发行版可执行二进制文件。如果你想使用另一个路径的文件（例如调试版二进制文件），你可以再运行测试前将环境变量 `CONFLUX` 设置为使用的二进制文件的完整路径。

TODO

## 实用功能列表

TODO

## 现有的Python测试介绍

TODO

