---
id: cli_sub_commands
title: CLI Sub-commands
custom_edit_url: https://github.com/Conflux-Chain/conflux-doc/edit/master/docs/cli.md
keywords:
  - conflux
  - cli
  - sdk
---

Conflux CLI子命令是一个命令行界面的集合，它允许您与本地或远程Conflux节点进行交互。

## 账户管理
`account` 允许你在本地管理您的账户。

### new
在本地创建一个新账户。
#### Usage
```text
$ ./conflux.exe account new --help
conflux.exe-account-new
Create a new account (and its associated key) for the given --chain (default conflux).

USAGE:
    conflux.exe account new [OPTIONS]

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
        --keys-iterations <NUM>    Specify the number of iterations to use when deriving key from the password (bigger is more secure). [default: 10240]
        --password <FILE>          Provide a file containing a password for unlocking an account. Leading and trailing whitespace is trimmed.
```
#### 样例

`./conflux.exe account new`

### list
列出本机的所有账户。
#### 使用方法
```text
$ ./conflux.exe account list --help
conflux.exe-account-list
List existing accounts of the given --chain (default conflux).

USAGE:
    conflux.exe account list

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information
```
#### 样例
`./conflux.exe account list`

### import
通过JSON UTC keystore文件导入账户。
#### 使用方法
```text
$ ./conflux.exe account import --help
conflux.exe-account-import
Import accounts from JSON UTC keystore files to the specified --chain (default conflux)

USAGE:
    conflux.exe account import --import-path <PATH>...

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
        --import-path <PATH>...    A list of file paths to import.
```
#### 样例
`./conflux.exe account import --import-path ./keystores`

## 公共的应用程序接口API
公共应用程序接口API允许通过JSON-RPC协议的HTTP连接与本地或远程的Conflux节点进行交互。所有的公共应用程序接口都在 `rpc` 子命令下，通过使用默认的 `url` 选项访问本地机器上的JSON-RPC应用程序接口。

```text
OPTIONS:
        --url <url>    URL of RPC server [default: http://localhost:12539]
```
要访问远程Conflux节点的JSON-RPC应用程序接口API，请指定正确的 `--url` 选项（如http://10.1.5.6:12537) 。出于对安全方面的考虑，JSON-RPC仅允许本地访问。可以通过在***default.toml***文件中配置 `jsonrpc_http_port` 手动启用远程访问。

```toml
# jsonrpc_tcp_port=12536
jsonrpc_http_port=12537
# jsonrpc_local_tcp_port=12538
jsonrpc_local_http_port=12539
```
所有可用命令如下：
```text
$ ./conflux.exe rpc --help
conflux.exe-rpc
RPC based subcommands to query blockchain information and send transactions

USAGE:
    conflux.exe rpc [OPTIONS] <SUBCOMMAND>

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
        --url <url>    URL of RPC server [default: http://localhost:12539]

SUBCOMMANDS:
    balance                  Get balance of specified account
    best-block-hash          Get the best block hash
    block-by-epoch           Get block by epoch
    block-by-hash            Get block by hash
    block-with-assumption    Get block by hash with pivot chain assumption
    blocks                   Get blocks of specified epoch
    call                     Executes a new message call immediately without creating a transaction
    code                     Get bytecode of specified contract
    local                    Local subcommands (requires jsonrpc_local_http_port configured)
    epoch                    Get epoch number
    estimate-gas             Executes a call request immediately without creating a transaction and returns the gas used
    help                     Prints this message or the help of the given subcommand(s)
    nonce                    Get nonce of specified account
    price                    Get recent mean gas price
    receipt                  Get receipt by transaction hash
    send                     Send a signed transaction and return its hash
    tx                       Get transaction by hash
```

### 获取余额
`./conflux.exe rpc balance --address 0xa70ddf9b9750c575db453eea6a041f4c8536785a`

### 获取一次性随机数
`./conflux.exe rpc nonce --address 0xa70ddf9b9750c575db453eea6a041f4c8536785a`

### 获取当前的纪元数
`./conflux.exe rpc epoch`

### 获取区块信息
- 获取最佳区块摘要值： `./conflux.exe rpc best-block-hash`
- 通过纪元获取区块： `./conflux.exe rpc block-by-epoch --epoch latest_state`
- 通过摘要值获取区块： `./conflux.exe rpc block-by-epoch --hash 0xf756b4...c0a6d1`
- 以纪元为单位获取区块： `./conflux.exe rpc blocks --epoch latest_state`

### 获取交易
`./conflux.exe rpc tx --hash 0x718532fe76dbd8c4208c6c5a79588db35c0bf97e7d8a0faa5988ba66ad88b74c`

### 获取收据
`./conflux.exe rpc receipt --hash 0x718532fe76dbd8c4208c6c5a79588db35c0bf97e7d8a0faa5988ba66ad88b74c`

### 发送一笔经过签名的交易
使用HEX格式编码发送一笔经过签名处理的交易。通常来说，该应用程序接口使用Java-Script应用程序接口来发送经过编码的交易。如果需要使用CLI发送交易，建议使用私有应用程序接口[发送交易](#发送交易)。

`./conflux.exe rpc send --raw-bytes 0x...`

### 杂项
- 获取合约代码： `./conflux.exe rpc code --address 0xa70ddf9b9750c575db453eea6a041f4c8536785a`
- 获取近期平均的燃料价格： `./conflux.exe rpc price`

## 私有应用程序接口
私有应用程序接口**仅**允许通过HTTP连接使用JSON-RPC协议与本地Conflux节点进行交互。私有应用程序接口是提供给用户协助其管理本地Conflux节点，需要在***default.toml***配置文件中配置 `jsonrpc_local_http_port` 。

此外，私有应用程序接口也能够帮助开发者进行调试、测试以及在监控Conflux节点的运行情况。

所有的私有应用程序接口都在 `local` 子命令下，通过默认的 `url` 选项在本机上访问JSON-RPC应用程序接口。

```text
$ ./conflux.exe rpc local --help
conflux.exe-rpc-local
Debug subcommands (requires jsonrpc_local_http_port configured)

USAGE:
    conflux.exe rpc local [OPTIONS] <SUBCOMMAND>

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
        --url <url>    URL of RPC server [default: http://localhost:12539]

SUBCOMMANDS:
    consensus-graph-state    Get the consensus graph state
    help                     Prints this message or the help of the given subcommand(s)
    net                      Network subcommands
    send                     Send a transaction and return its hash
    sync-phase               Get the current synchronization phase
    test                     Test subcommands (used for test purpose only)
    txpool                   Transaction pool subcommands
```
### net
`net` 子命令可以帮助检查P2P点对点网络状态。

#### 样例
- 列出全部已连接的P2P节点： `./conflux.exe rpc local net session`
- 列出单个P2P节点： `./conflux.exe rpc local net session --id <node_id>`
- 检查网络出口： `./conflux.exe rpc local net throttling`

### txpool
`txpool` 子命令能够帮助检查交易池。

#### 样例
- 列出交易池状态： `./conflux.exe rpc local txpool status`
- 详细列出交易信息： `./conflux.exe rpc local txpool content`
- 列出交易摘要: `./conflux.exe rpc local txpool inspect`
- 详细检查一项交易： `./conflux.exe rpc local txpool inspect-one --hash <tx_hash>`

### sync-phase
获取本地Conflux节点的同步阶段信息。

`./conflux.exe rpc local sync-phase`

### 发送交易
向本地Conflux节点发送交易。

#### 使用
```text
$ ./conflux.exe rpc local send --help
conflux.exe-rpc-local-send
Send a transaction and return its hash

USAGE:
    conflux.exe rpc local send [OPTIONS] --from <ADDRESS> --password <STRING> --value <HEX>

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
        --data <HEX>           Hash of the method signature and encoded parameters
        --from <ADDRESS>       Transaction from address
        --gas <HEX>            Gas provided for transaction execution [default: 0x5208]
        --gas-price <HEX>      Transaction gas price [default: 0x2540BE400]
        --nonce <HEX>          Transaction nonce
        --password <STRING>    Used to decrypt private key of sender to sign transaction
        --to <ADDRESS>         Transaction to address (empty to create contract)
        --url <url>            URL of RPC server [default: http://localhost:12537]
        --value <HEX>          value sent with this transaction
```

#### 样例
Alice向Bob转账5个Drip（其中1 CFX = 10^18 Drip）。注意，Alice的地址必须在本地机器留存，否则请先为Alice[创建](#new)账户。 

`./conflux.exe rpc local send --from <alice_address> --to <bob_address> --value 0x5 --password 123456`

Alice创建了一个燃料为3000000的合约。可以使用***solc*** 编译合约以获取字节码数据。

`./conflux.exe rpc local send --from <alice_address> --value 0x0 --gas 0x‭2DC6C0‬ --data <HEX_contract_bytecodes> --password 123456`
