# Conflux-Rust中的Stratum协议

## 设计目标

Rust是一块能够开发诸如Conflux等分布式系统的优秀语言，但对于具备开发能力的矿工来说并不友好，大部分矿工通常都会使用C/C++这类性能高且能够与GPU进行交互的语言进行开发。因此，在Conflux-Rust中设计了一个类Stratum的协议，以帮助外部矿工连接到Conflux。

需要注意的是，为使协议简洁，类Stratum协议的仅设计适用于自挖矿，如，Conflux-Rust只与属于相同实体的矿工本地或远程机器上的进程建立连接。没有被设计成矿池服务器的形式。 对于那些希望运行矿池的用户，推荐构建能够连接到Conflux-Rust且属于矿工自己的定制代理服务器。

## 通用工作流程

一般情况下，一个外部矿工会通过Stratum端口（默认32525）连接到Conflux-Rust，下面列出了通用的工作流程。

1. 矿工通过TCP连接到Stratum端口。Conflux-Rust必须在启用Stratum的配置下运行。

2. 矿工听过使用TCP流发送 `mining.subscribe` RPC调用。它会告知Conflux-Rust矿工名字。 `mining.subscribe` 起到了基本的口令认证作用，口令可以通过设置Conflux-Rust的配置文件实现。

3. 成功订阅后，Conflux-Rust会继续通过TCP流发送 `mining.notify` RPC调用。每一个 `mining.notify` 都对应于一个需要矿工求解的新工作量证明（PoW）问题。按照预期，矿工将求解其最后收到的一个工作任务。

4. 无论矿工合适求解出该工作量证明问题，他会通过调用 `mining.submit` RPC 调用将求解结果（如，随机数nonce）返回至Conflux-Rust。

5. 矿工在该过程中的任何时间都可以直接断开连接并退出。

## RPC接口

### 服务端RPCs

#### mining.subscribe
开始订阅来自Stratum服务器的工作量证明通知。

##### 参数
 1. WORKER_ID，字符串 - 矿工名
 2. Secret，空或32字节，与配置中口令的keccak结果相对应的秘密值。如果没有启用口令，则为空。

##### 返回值
`Bool` - `true` 如果成功， `false` 如果失败。

##### 样例
```
// 请求
'{"jsonrpc":"2.0","method":"mining.subscribe","params":["cfxmine", ""],"id":1}'

// 响应
{
  "jsonrpc": "2.0",
  "result": "true",
  "id": 1
}
```
---

#### mining.submit
向Stratum服务器提交工作量证明的解

##### 参数
 1. WORKER_ID，字符串 - 矿工名
 2. JOB_ID， 16进制字符串 - 工作的标识符，通常与工作量证明问题的摘要值相同。
 3. NONCE， 32字节16进制字符串 - 工作量证明问题的随机数解
 4. HASH， 32字节16进制字符串 - 已求解的工作量证明问题的摘要值

##### 返回值
`Array` - 如果成功，则为 `true` ，如果不成功，则第一个元素为 `false` ，第二个元素会用字符串说明原因。

##### 样例
```
// 请求
'{"jsonrpc":"2.0","method":"mining.submit","params":["cfxmine", "0x2106e1162d1199483fa010bcaa7d4f05b23b85d456b4a7089d787ae2e880deaf","0x21b49d385865819a171ed8cd9d9f80acc468e501f3486d3600000000000c786c","0x2106e1162d1199483fa010bcaa7d4f05b23b85d456b4a7089d787ae2e880deaf"],"id":1}'

// 响应
{
  "jsonrpc": "2.0",
  "result": ["true"],
  "id": 1
}

{
  "jsonrpc": "2.0",
  "result": ["false", "invlaid nonce"],
  "id": 1
}
```
---

### 客户端通知

需要注意的是，尽管服务器使用一个类RPC请求通知挖矿客户端，但其实这并不是一个真正的RPC -- 它不期望客户端返回任何响应。客户端仅仅会更新其求解的工作量证明问题并在找到对应解时进行提交。

#### mining.notify
通知客户端一个新的工作量证明问题。

##### 参数
 1. JOB_ID，16进制字符串 - 工作标识符。
 2. HASH， 32字节 - 工作量证明问题的摘要值。
 3. BOUNDARY， U256 - 问题的难度边界。为了使随机数nonce有效，计算所产生的摘要值必须小于BOUNDARY。

##### 样例
```
// 请求
'{"jsonrpc":"2.0","method":"mining.notify","params":["0x4e08db21d43a7696afa3d00ed948568210f3ab3f34673f1d17198625ec175a9c","0x4e08db21d43a7696afa3d00ed948568210f3ab3f34673f1d17198625ec175a9c","0x1a4e3422948568210f3ab3f34673f1d17198625ec175a9c"],"id":3}'

---

