# 变更记录

## v0.10.1

* 添加format.bytes特性

```
format.bytes('abcd'); // <Buffer 61 62 63 64>
format.bytes([0, 1, 2, 3]); // <Buffer 00 01 02 03>
```

* 添加智能合约方法及&事件类型或签名索引

```
// solidity
function override(bytes memory str) public
function override(string memory str) public
```

```
contract.override('str'); // Error: can not determine override

contract['override(string)']('str'); // specify override method by type
contract['0x227ffd52']('str'); // specify override method by signature
```


## v0.10.0-alpha

* 添加 `getStatus` 函数

```
cfx.getStatus()
```

* 移除 `getRiskCoefficient` 并使用 `getConfirmationRiskByHash` 作为替代

```
// old
cfx.getRiskCoefficient(epochNumber)

// new
cfx.getConfirmationRiskByHash(blockHash)
```

* 移除 `getAccount` ，因为它是内部RPC。

* 使用 `require` 替换 `import` 以生成代码。

## v0.9.2

* 为Conflux添加defaultStorageLimit和defaultChainId

```
// old
const cfx = new Conflux({
  url: 'http://localhost:8000',
  defaultGasPrice: 100,
  defaultGas: 100000,
})

// new
const cfx = new Conflux({
  url: 'http://localhost:8000',
  defaultGasPrice: 100,
  defaultGas: 100000,
  defaultStorageLimit: 4096,
  defaultChainId: 0,
})
```

## v0.9.1

* abi隐式转换字符串为数字

solidity方法： `function add(uint,uint) public returns (uint);`

```
// old
await contract.add(1, '2'); // error! can not accept string 

// new version
await contract.add(1, '2'); // good, converting string to number
```

## v0.9.0-beta

* 格式为JSBI.BigInt的nonce一次性随机数

```
nonce = await cfx.getNextNonce(...)

// old
100000

// new
JSBI.BigInt(100000)
```

* 格式化交易区域

```
tx = await cfx.getTransactionByHash(txHash)
// old
{
  storageLimit: "0x64",
  chainId: "0x0",
  epochHeight: "0x400",
  ...
}

// new
{
  storageLimit: JSBI.BigInt(100), // JSBI
  chainId: 0,
  epochHeight: 1024,
  ...
}
```

* unit返回字符串

```
// old
unit.fromCFXToGDrip(123) => JSBI.BigInt(123000000000)
unit.fromCFXToGDrip('0.1234567891') => Error('Cannot convert JSBI.BigInt')

// new
unit.fromCFXToGDrip(123) => "123000000000"
unit.fromCFXToGDrip('0.1234567891') => "123456789.1"
```

* 将合约字段 "code" 重命名为 "bytecode"

```
// old
cfx.Contract({code, abi, address})

// new
cfx.Contract({bytecode, abi, address})
```

* abi decodeData方法和decodeLog方法将返回对象

```
result = contract.abi.decodeData('0x....')

// old
["Tom", JSBI.BigInt(18)]

// new
{
  name: 'func'
  fullName: 'func(string name, uint age)',
  type: 'func(string,uint)',
  signature: '0x812600df',
  array: ["Tom", JSBI.BigInt(18)],
  object: {
    name: "Tom",
    age: JSBI.BigInt(18),
  }
}
```
