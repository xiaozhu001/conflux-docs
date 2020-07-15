在本文档中，我们将介绍使用命令行接口工具包创建和管理您的密钥对、创建交易、对交易进行签名处理以及提交交易到Conflux的过程。

## 获取密钥管理工具
您可以从[这里]()下载Conflux的密钥管理工具包(**keymgr**)，或者你可以直接从Conflux的源代码中构建 **keymgr** ，步骤如下。
```markdown
$ git clone ssh://git@base.conflux-chain.org:2222/source/conflux-rust.git
$ cd conflux-rust/key_manager/cli
$ cargo build
```
**keymgr** 的二进制文件存储在 `conflux-rust/key_manager/cli/target/` 中。

## 使用KeyMgr创建密钥对
可以使用 **keymgr** 来创建你的私/公钥对。
```markdown
$ keymgr generate random
```
样例输出为：
```markdown
secret:  074842cdfa28a02fd23f244126618bcb49588a5530e7135dcd8c86aa3fbf0103
public:  7827b388197a9b4c4c97aafff400b1d168439b0b6b2428dad9a8f8ec461789155a9318c7d0d38a2e696e41c99faa0e7f7ab55bc21814b6e7809936f1d51ee5b0
address: 71e177b579a4b1ad24382f4b559f479ca0099572
``` 
***address*** 是由公钥导出的160位账户ID。你可以在任何只有你知道的地方保存你的私钥。

## 生成、签署和提交代币转移交易。Conflux提供的JavaScript库 **CfxWeb** 可以帮助用户以编程的方式生成交易。以下是一个示例的代码片段。

```Javascript
#!/usr/bin/env node

var Tx = require('ethereumjs-tx');
var secretKey = Buffer.from('e331b6d69882b4cb4ea581d88e0b604039a3de5967688d3dcffdd2270c0fd109', 'hex')
var rawTx = {
 nonce: '0x00',
 gasPrice: '0x09184e72a000',
 gasLimit: '0x2710',
 to: '0x0000000000000000000000000000000000000000',
 value: '0x00',
 data: '0x7f7465737432000000000000000000000000000000000000000000000000000000600057'
}
var tx = new Tx(rawTx);
tx.sign(secretKey);
console.log('0x' + serializedTx.toString('hex'));

var Web3 = require('web3');
var web3 = new Web3();
web3.setProvider(new web3.providers.HttpProvider('http://localhost:12345'));
let answer = web3.cfx.sendRawTransaction('0x' + serializedTx.toString('hex'));
console.log(answer);

```  
要运行上述代码，需要先安装Node.js和CfxWeb.js。