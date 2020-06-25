---
id: internal_contract
title: Internal Contract
custom_edit_url: https://github.com/Conflux-Chain/conflux-rust/edit/master/internal_contract/README.md
keywords:
  - conflux
  - contract
---

Conflux引入了一些内置的内部合约，以便更好地进行系统维护和链上治理。而本文档将介绍如何使用这些内部合约。

后续文档使用 [js-conflux-sdk](https://github.com/Conflux-Chain/js-conflux-sdk) 作为案例

## 通过赞助使用合约的方法

Conflux实现了一种赞助机制以补贴用户使用智能合约。因此，一个账户余额为0的新用户，只要执行赞助（通常由Dapps的运营商赞助），就能够直接调用智能合约。 通过引入内置的 SponsorControl 合约已记录对智能合约的赞助信息。

**SponsorControl** 智能合约为每个用户建立的合约维护以下信息:
+ `sponsor_for_gas`: 为提供燃料费用补贴的账户；
+ `sponsor_for_collateral`: 为提供存放抵押补贴的账户；
+ `sponsor_balance_for_gas`: 可用于燃料费用补贴的余额；
+ `sponsor_balance_for_collateral`: 可用于存放抵押补贴的余额；
+ `sponsor_limit_for_gas_fee`: 赞助者愿意为每笔交易提供的资助上限；
+ `whitelist`: 为合约愿意资助的正常账户清单，特殊的全0地址特指所有的正常账户。只有合约本身才有权更改该清单。

有两类资源可以被赞助：燃料费用及存储抵押物。

+ *对于燃料费用*: 如果一笔交易使用非空的 `sponsor_for_gas` 调用智能合约且交易发送者处于合约的 `whitelist` 列表内，且交易指定的燃料费用在 `sponsor_limit_for_gas_fee` 范围内，交易的燃料消耗将从合约的 `sponsor_balance_for_gas` 中支付（如果足够的话），而不是由交易发送者的账户余额支付，如果 `sponsor_balance_for_gas` 无法承担燃料消耗，则交易失败。否则，交易发送者应支付燃料费用。 
+ *对于存储抵押物*: 如果一笔交易使用非空的 `sponsor_balance_for_collateral` 调用智能合约且交易发送者处于合约的 `whitelist` 列表内，在执行交易的过程中抵押担保物将从智能合约的 `sponsor_balance_for_collateral` 中扣除，并将这些修改后的存储条目所有者相应设置为合约地址。 否则，交易发送方应在执行过程中支付抵押担保物。

### 赞助更新

可以通过调用SponsorControl合同来更新燃料赞助费用和抵押担保物。要替换合约中的 `sponsor_for_gas` ，新的赞助人应当向合约转移比当前合约 `sponsor_balance_for_gas` 更多的资金，并为 `sponsor_limit_for_gas_fee` 设定一个新值。除非原有的 `sponsor_limit_for_gas_fee` 无法负担赞助，否则， `sponsor_limit_for_gas_fee` 的新值应不低于原有赞助者设置的限额。而且，转入的资金应当是新限额的1000倍及以上，这样至少可以补贴 `1000` 个调用C的交易。 如满足上述条件，则剩余的 `sponsor_balance_for_gas` 将退还给之前的赞助账户 `sponsor_for_gas` ，随后根据新赞助商的规范更新 `sponsor_balance_for_gas` ， `sponsor_balance_for_gas` ，及 `sponsor_limit_for_gas_fee` 。

对 `sponsor_for_collateral` 的替换操作与上文描述非常类似，只是没有类似于燃料费用的限制。新的赞助人应当向C转入比现有赞助人提供的资金更多的资金作为合约的抵押担保物。那么当前赞助人 `sponsor_for_collateral` 的担保物将被全额退还，如 `sponsor_balance_for_collateral` 与合约使用的抵押担保总额之和，此外两个抵押担保字段均按照新赞助人的要求进行相应的变更。一个合约账户允许成为赞助人。

### 接口

内建的合约地址为 `0x8ad036480160591706c831f0da19d1a424e39469`. 内部合约的abi可以在 [这里](https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/metadata/SponsorWhitelistControl.json) 以及 [这里](https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/contracts/SponsorWhitelistControl.sol)找到。

+ `set_sponsor_for_gas(address contract, uint upper_bound)` ：如果有人希望向合约地址 `contract` 赞助燃料费用， 他/她（也可以是合约账户）可以在调用该函数的同时向地址 `0x8ad036480160591706c831f0da19d1a424e39469` 传输代币。 参数 `upper_bound` 是指赞助者为单笔交易支付的燃料费用上限。 传输的代币量至少为参数 `upper_bound` 的1000倍。赞助者可能会被赞助更多代币同时设置更高的上界参数的赞助者所替换。当前赞助者也可以调用该函数并向该合约传输更多代币。在当前赞助者账户余额小于参数 `upper_bound` 时 ， `upper_bound` 值将被动态调低。
+ `set_sponsor_for_collateral(address contract_addr)`： 如果有人希望向地址为 `contract` 的合约赞助担保金，他/她（也可以是合约账户）可以在调用该函数的同时向地址 `0x8ad036480160591706c831f0da19d1a424e39469`传输代币。赞助者可能会被传输更多代币的新赞助者替换而当前赞助者也可通过调用该函数向合约传输更多代币。
+ `add_privilege(address[] memory)`:  合约可通过调用该函数向白名单中加入部分正常账户地址。这意味着，若 `sponsor_for_gas` 被设置，合约会向白名单内的账户支付燃料费用，若 `sponsor_for_collateral` 被设置，则合约会向白名单内账户支付担保金。合约可通过使用特殊地址 `0x0000000000000000000000000000000000000000` ，能够将所有账户加入到白名单中。
+ `remove_privilege(address[] memory)`：合约可通过调用该函数从白名单中移除正常账户。

当调用函数时的交易值 `set_sponsor_for_gas` 及 `set_sponsor_for_collateral` 代表发送者（新赞助者）愿意支付的代币数量。每个合约通过调用 `add_privilege` 及 `remove_privilege` 来维护它的白名单列表 `whitelist` 。

### 例子

假设有一个如下所示的简单合约。
```solidity
pragma solidity >=0.4.15;

import "https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/contracts/SponsorWhitelistControl.sol";

contract CommissionPrivilegeTest {
    mapping(uint => uint) public ss;

    function add(address account) public payable {
        SponsorWhitelistControl cpc = SponsorWhitelistControl(0x8ad036480160591706c831f0DA19D1a424e39469);
        address[] memory a = new address[](1);
        a[0] = account;
        cpc.add_privilege(a);
    }

    function remove(address account) public payable {
        SponsorWhitelistControl cpc = SponsorWhitelistControl(0x8ad036480160591706c831f0DA19D1a424e39469);
        address[] memory a = new address[](1);
        a[0] = account;
        cpc.remove_privilege(a);
    }

    function foo() public payable {
    }

    function par_add(uint start, uint end) public payable {
        for (uint i = start; i < end; i++) {
            ss[i] = 1;
        }
    }
}
```

部署合约后，地址为 `contract_addr` ，如果有人希望赞助燃料费用，他/她可以发送如下交易：
```javascript
const PRIVATE_KEY = '0xxxxxxx';
const cfx = new Conflux({
  url: 'http://testnet-jsonrpc.conflux-chain.org:12537',
  defaultGasPrice: 100,
  defaultGas: 1000000,
  logger: console,
});
const account = cfx.Account(PRIVATE_KEY); // create account instance

const sponsor_contract_addr = '0x8ad036480160591706c831f0da19d1a424e39469';
const sponsor_contract = cfx.Contract({
  abi: require('./contracts/sponsor.abi.json'),
  address: sponsor_contract_addr,
});
sponsor_contract.set_sponsor_for_gas(contract_addr, your_upper_bound).sendTransaction({
  from: account,
  value: your_sponsor_value
}).confirmed();
```

至于赞助者的存储抵押物，只需要更换 `set_sponsor_for_gas(contract_addr, your_upper_bound)` 函数为 `set_sponsor_for_collateral(contract_addr)` 函数并使用。

通过使用 `add_privilege` 和 `remove_privilege` 为合约维护白名单 `whitelist` 。含有全0的特殊地址 `0x0000000000000000000000000000000000000000` 意味着任何人都在 `whitelist` 内。 需要谨慎的使用它。

```javascript
you_contract.add(white_list_addr).sendTransaction({
  from: account,
})

you_contract.remove(white_list_addr).sendTransaction({
  from: account,
})
```

在这一操作后，在调用 `you_contract.foo()` 或 `you_contract.par_add(1, 10)` 时存在于 `whiltelist` 内的账户将不再支付任何费用。

## 管理

引入 **AdminControl** 合约的目的是为了更好地维护其他智能合约，特别是那些没有适当销毁程序而临时生成的智能合约：它记录了每个用户建立的智能合约的管理员，并根据相应管理员的要求进行销毁处理。

The default administrator of a smart contract is the creator of the contract, i.e. the sender α of the transaction that causes the creation of the contract. The current administrator of a smart contract can transfer its authority to another normal account by sending a request to the AdminControl contract. Contract accounts are not allowed to be the administrator of other contracts, since this mechanism is mainly for tentative maintenance. Any long term administration with customized authorization rules should be implemented inside the contract, i.e. as a specific function that handles destruction requests.

At any time, the administrator `addr` of an existing contract has the right to request destruction of the contract by calling AdminControl. However, the request would be rejected if the collateral for storage of contract is not zero, or `addr` is not the current administrator of the contract. If `addr` is the current administrator of the contract and the collateral for storage of contract is zero, then the destruction request is accepted and
processed as follows:
1. the balance of the contract will be refunded to `addr`;
2. the `sponsor_balance_for_gas` of the contract will be refunded to `sponsor_for_gas`;
3. the `sponsor_balance_for_collateral` of the contract will be refunded to `sponsor_for_collateral`;
4. the internal state in the contract will be released and the corresponding collateral for storage refunded to owners;
5. the contract is deleted from world-state.

### The Interfaces

The contract address is `0x8060de9e1568e69811c4a398f92c3d10949dc891`. The abi for the internal contract could be found [here](https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/metadata/AdminControl.json) and [here](https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/contracts/AdminControl.sol).

+ `set_admin(address contract, address admin)`: Set the administrator of contract `contract` to `admin`. The caller should be the administrator of `contract` and it should be a normal account. Caller should make sure that `contract` should be an address of a contract and `admin` should be a normal account address. Otherwise, the call will fail.

+ `destroy(address contract)`: Perform a suicide of the contract `contract`. The caller should be the administrator of `contract` and it should be a normal account. If the collateral for storage of the contract is not zero, the suicide will fail. Otherwise, the `balance` of `contract` will be refunded to the current administrator, the `sponsor_balance_for_gas` will be refunded to `sponsor_for_gas`, the `sponsor_balance_for_collateral` will be refunded to `sponsor_for_collateral`.

### Examples

Consider you have deployed a contract whose address is `contract_addr`. The administrator can call `AdminControl.set_admin(contract_addr, new_admin)` to change the administrator and call `AdminControl.destroy(contract_addr)` to kill the contract. 

```javascript
const PRIVATE_KEY = '0xxxxxxx';
const cfx = new Conflux({
  url: 'http://testnet-jsonrpc.conflux-chain.org:12537',
  defaultGasPrice: 100,
  defaultGas: 1000000,
  logger: console,
});
const account = cfx.Account(PRIVATE_KEY); // create account instance

const admin_contract_addr = '0x8060de9e1568e69811c4a398f92c3d10949dc891';
const admin_contract = cfx.Contract({
  abi: require('./contracts/admin.abi.json'),
  address: admin_contract_addr,
});
// to change administrator
admin_contract.set_admin(contract_addr, new_admin).sendTransaction({
  from: account,
}).confirmed();

// to kill the contract
admin_contract.destroy(contract_addr).sendTransaction({
  from: account,
}).confirmed();
```

## Staking Mechanism

Conflux introduces the staking mechanism for two reasons: first, staking mechanism provides a better way to charge the occupation of storage space (comparing to “pay once, occupy forever”); and second, this mechanism also helps in defining the voting power in decentralized governance.

At a high level, Conflux implements a built-in **Staking** contract to record the staking information of all accounts. By sending a transaction to this contract, users (both external users and smart contracts) can deposit/withdraw funds, which is also called stakes in the contract. The interest of staked funds is issued at withdrawal, and depends on both the amount and staking period of the fund being withdrawn

### Interest Rate

The staking interest rate is currently set to 4% per year. Compound interest is implemented in the granularity of blocks.

When executing a transaction sent by account `addr` at block `B` to withdraw a fund of value `v` deposited at block `B'`, the interest is calculated as follows:

```
interest issued = v * (4% / 63072000)^T
```

where `T = BlockNo(B)−BlockNo(B')` is the staking period measured by number of blocks, and `63072000` is the expected number of blocks generated in `365` days with the target block time `0.5` seconds. 

### Staking for Voting Power

See the details in [Conflux Protocol Specification](https://confluxnetwork.org/developer/).

### The Interfaces

The contract address is `0x843c409373ffd5c0bec1dddb7bec830856757b65`. The abi for the internal contract could be found [here](https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/metadata/Staking.json) and [here](https://github.com/Conflux-Chain/conflux-rust/blob/master/internal_contract/contracts/Staking.sol).

+ `deposit(uint amount)`: The caller can call this function to deposit some tokens to Conflux Internal Staking Contract. The current annual interest rate is 4%.
+ `withdraw(uint amount)`: The caller can call this function to withdraw some tokens to Conflux Internal Staking Contract. It will trigger a interest settlement. The staking capital and staking interest will be transferred to the user's balance in time. All the withdrawal applications will be processed on a first-come-first-served basis according to the sequence of staking orders.
+ `vote_lock(uint amount, uint unlock_block_number)`: This function is related with Voting Rights in Conflux. Staking users can choose the voting amount and locking maturity by locking a certain amount of CFX in a certain maturity from staking. The `unlock_block_number` is measured in the number of blocks since genesis block.

### Examples

```javascript
const PRIVATE_KEY = '0xxxxxxx';
const cfx = new Conflux({
  url: 'http://testnet-jsonrpc.conflux-chain.org:12537',
  defaultGasPrice: 100,
  defaultGas: 1000000,
  logger: console,
});
const account = cfx.Account(PRIVATE_KEY); // create account instance

const staking_contract_addr = '0x843c409373ffd5c0bec1dddb7bec830856757b65';
const staking_contract = cfx.Contract({
  abi: require('./contracts/staking.abi.json'),
  address: staking_contract_addr,
});
// deposit some amount of tokens
staking_contract.deposit(your_number_of_tokens).sendTransaction({
  from: account,
}).confirmed();

// withdraw some amount of tokens
staking_contract.withdraw(your_number_of_tokens).sendTransaction({
  from: account,
}).confirmed();

// lock some tokens until some block number
staking_contract.vote_lock(your_number_of_tokens, your_unlock_block_number).sendTransaction({
  from: account,
}).confirmed();
```
