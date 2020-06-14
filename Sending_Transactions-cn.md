## 转账交易

交易是区块链上的一个标准行为。它们常常在ConfluxPortal中通过调用

```plain
cfx_sendTransaction
```
来启动。它们可以涉及简单的发送ether，也可以发送token，创建一个新的智能合约，或以任何方式改变区块链上的状态。它们通常由外部账户的签名或简单的密钥对发起。
在ConfluxPortal中，直接使用

```plain
conflux.sendAsync
```
发送交易将涉及以下的选项：
```plain
const transactionParameters = {
nonce: '0x00', // ignored by ConfluxPortal
gasPrice: '0x09184e72a000', // customizable by user during ConfluxPortal confirmation.
gas: '0x2710', // customizable by user during ConfluxPortal confirmation.
to: '0x0000000000000000000000000000000000000000', // Required except during contract publications.
from: accounts[0], // must match user's active address.
value: '0x00', // Only required to  by ConfluxPortal.
}
 
conflux.sendAsync({
method: 'cfx_sendTransaction',
params: [transactionParameters],
from: conflux.selectedAddress,
}, callback)
```
### 交易参数

很多交易参数都是由ConfluxPortal处理的，但了解所有交易参数的作用很有用。

#### Nounce [可忽略]

这个字段被ConfluxPortal忽略了。

在Conflux中，每笔交易都有一个nonce。这是为了让每笔交易只能被区块链处理一次。此外，为了成为有效的交易，nonce必须是0，或者是前一个数字的交易必须已经被处理。

这意味着对于一个给定的账户来说，交易总是按顺序处理的，因此递增nonce是一个非常敏感的问题，很容易出错，特别是当用户使用同一账户与多个应用程序进行交互，并且有待处理的交易时，可能会经过多个设备。

基于这些原因，ConfluxPortal目前并没有为应用开发者提供任何方法来定制其建议的交易nonce，而是协助用户自行管理其交易队列。

#### Gas price燃料价格[可选]

可选参数--最好用在私有区块链上。

在Conflux中，待处理的交易池将其燃料价格作为一种拍卖竞价提供给验证者，以将此交易纳入一个区块中，以换取交易费。这意味着高手续费可能意味着更快的处理速度，也意味着更昂贵的交易。

ConfluxPortal帮助用户在Conflux主网和流行的测试网络上选择有竞争力的天然气价格。我们向MyCrypto的朋友维护的API提出请求，并允许用户在 "慢"、"中 "和 "快 "之间选择天然气价格。

我们无法了解所有区块链上的天然气市场，因为这需要一些深度分析。出于这个原因，虽然您可以在我们的主托管网络上安全地忽略这个参数，但在您的应用程序比我们更了解目标网络的情况下，您可能希望建议一个气体价格。

#### Gas limit 燃料极限[可选]

可选参数。对Dapp开发者来说很少有用。

Gas limit是一个高度可选的参数，我们会自动为它计算一个合理的价格。你可能发现你的智能合约因为某些原因而受益于自定义gas limit。

#### To [半可选]

一个十六进制的Conflux地址。与接收人进行交易时需要（除合约创建外的所有交易）。

当没有to值但有data值时，合约就会创建。

#### Value [可选]

以十六进制编码的网络本位货币值发送。在Conflux主网络中是cfx，以Drip为单位，也就是1e-18ether。

请注意，Conflux中经常使用的这些数字比原生JavaScript数字的精度高得多，如果没有预设，可能会导致不可预测的结果。出于这个原因，我们强烈建议在修改用于公链的value时参考[BigInt](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt)或[BN.js](https://github.com/indutny/bn.js/)。

#### Data [半可选]

智能合约创建时需要。

这个字段也用于指定合约方法及其参数。你可以在[solidity ABI规范](https://solidity.readthedocs.io/en/develop/abi-spec.html)上了解更多关于这些数据是如何编码的。

#### ChainID [当前忽略]

ChainID目前是由用户当前选择的网络在

>conflux.networkVersion

处得出。未来我们可能会允许同时连接多个网络的方式，到时候这个参数就会变得很重要，所以现在习惯性的加入可能会很有用。

