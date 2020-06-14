---
id: getting_started
title: Getting Started
custom_edit_url: https://github.com/Conflux-Chain/conflux-portal-docs/edit/master/docs/en/portal/Main_Concepts/Getting_Started.md
---
要使用ConfluxPortal进行开发，首先你要将ConfluxPortal安装到你的开发设备上。[点此下载。](https://github.com/Conflux-Chain/conflux-portal/releases)

运行之后，你可以看到控制台新的浏览器标签中有一个可用的`window.conflux`对象，这是ConfluxPortal与你交互的方式。 

你可以[点击此处](local://base_request.html/API_Reference/Conflux_Provider.md)查看该对象完整的API。

## 基本项目

### ConfluxJS浏览器检测

你的APP要做的第一件事就是确认你是否在使用ConfluxPortal，非常简单，使用检查命令即可：`if (typeof
window.conflux !== 'undefined') { /* deal with it */ }`. 

### 运行测试网络

在ConfluxPortal右上角的菜单中，你可以选择当前接入的网络，在几个常用的默认选项中，你可以找到`Custom RPC`和`Localhost 12539`，这些都有助于连接一个测试区块链，比如[Conflux本地网络](https://github.com/yqrashawn/conflux-local-network-lite#readme)。如果你用`npm i -g
ganache-cli && ganache-cli`安装了`npm`，就可以快速安装启动测试网络。

<!-- Ganache has some great features for starting it up with different states. If you -->
<!-- start it with the `-m` flag, you can feed it the same seed phrase you have in -->
<!-- your ConfluxPortal, and the test network will give your first 10 accounts 100 -->
<!-- test ether each, which makes it easier to start work.  -->

你的安全词能够控制你所有的账户，因此请保存至少一个安全词用于开发，和其他任何你存储实际资产的安全词要分开。用ConfluxPortal保存多个安全词的一个简单的方法就是使用多个浏览器配置文件，每个文件都可以安装各自的干净的拓展项。

#### 重置本地Nonce计算

如果你在运行一个测试区块链，重启它之后，ConfluxPortal有可能会被混淆，因为ConfluxPortal是基于网络状态*和*已知的已发送交易计算[Nonce](local://base_request.html/Sending_Transactions.md#nonce-[ignored])的。

清除ConfluxPortal的交易队列，并有效地重置它的nonce计算，可以使用`Settings`中的`Reset Account`按钮实现（在右上角的菜单中）。

### 检测ConfluxPortal

如果你想区分ConfluxPortal和其他与Conflux兼容的浏览器，可以用`conflux.isConfluxPortal`进行检测。

### 用户状态

当前使用这个API时，有一些状态性的问题需要考虑：

- 当前的网络是什么？
- 当前的账户是什么？

这两个问题在`conflux.networkVersion`和`conflux.selectedAddress`中同时需要考虑。 你也可以用event监听变化，参见（[API参考](local://base_request.html/API_Reference/Conflux_Provider.md)）

### 连接到ConfluxPortal

“连接”或“登录”到ConfluxPortal意味着“访问用户的Conflux账户”。

你**只能**在响应直接用户操作时发起连接请求，比如点击按钮，而**不能**在页面加载时发起请求。

我们建议你在你的DApp中设置一个连接ConfluxPortal的按钮，点击按钮即可调用以下方法：  

```javascript
conflux.enable()
```

这种承诺返回函数使用十六进制前缀的conflux地址数组进行解析，可在发起交易时用作一般帐户参考。

随着时间的推移，这个方法将发展为包含其他各种参数的方法，帮助你的站点在安装过程中向用户请求所需的所有安装选项。

由于返回值是一个Promise，如果你使用的是`async`函数，你的日志会像下面这样：

```javascript
const accounts = await conflux.enable()
const account = accounts[0] // We currently only ever provide a single account,
                            // but the array gives us some room to grow.
```

## 选择Convience Library

Convience Library存在的原因有很多。

有些是简化了用户界面元素的建立，有些完全管理了用户账户引导，其他的会给你提供各种和只能条约交互的方法、各种API参考，从promises到callbacks，再到strong types等等。

提供者API本身非常简单，封装[Conflux
JSON-RPC](https://conflux-chain.github.io/conflux-doc/json-rpc/)的格式化消息，因此开发者们通常会使用convenience library与提供者交互，像[js-conflux-sdk](https://www.npmjs.com/package/js-conflux-sdk)、
[conffle](https://github.com/liuis/conffle#readme)等等。通过这些工具，一般能找到足够多的可以与提供者交互的文件，而不需要读这些低水平的API。