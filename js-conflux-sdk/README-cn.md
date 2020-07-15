# js-conflux-sdk

基于JavaScript的Conflux软件开发套件

## 安装

`npm install js-conflux-sdk`

## 使用方法

[API文档](https://zh-hans.developer.conflux-chain.org/docs/js-conflux-sdk/javascript_sdk)

### Nodejs
```javascript
const { Conflux } = require('js-conflux-sdk');

async function main() {
  const cfx = new Conflux({
    url: 'http://testnet-jsonrpc.conflux-chain.org:12537',
    defaultGasPrice: 100,
    defaultGas: 1000000,
    logger: console,
  });

  const balance = await cfx.getBalance('0xbbd9e9be525ab967e633bcdaeac8bd5723ed4d6b');
  console.log(balance); // 937499420597305000n
}

main();
```
require deep nested file/dir  

``` javascript
const util = require('js-conflux-sdk/src/util');
```

### Frontend前端

#### 通用模块定义规范
``` javascript
import { Conflux } from 'js-conflux-sdk/dist/js-conflux-sdk.umd.min.js';
```

or  

``` html
<script type="text/javascript" src="node_modules/js-conflux-sdk/dist/js-conflux-sdk.umd.min.js"></script>
<script type="text/javascript">
  const cfx = new window.Conflux.Conflux({
    url: 'http://testnet-jsonrpc.conflux-chain.org:12537',
    defaultGasPrice: 100,
    defaultGas: 1000000,
    logger: console,
  });
</script>
```

## 变更记录

[查看](./CHANGE_LOG-cn.md)


## 样例

[样例](https://github.com/Conflux-Chain/js-conflux-sdk/tree/master/example)
