

用户的账户在Conflux中的各种应用场境中都有使用，它们的标志整洁，但没有任何用途比调用_wallet methods_方法更重要。由于这类方法涉及到签名或交易审批。所有这些方法都需要转出账户作为函数参数。

- `cfx_sendTransaction`

     一旦您连接到一个用户，您可以随时通过检查 conflux.selectedAddress 来重新检查当前账户。

    如果你想在地址改变时收到通知，我们有一个event可以供你们使用：

```javascript
conflux.on('accountsChanged', function (accounts) {
  // Time to reload your interface with accounts[0]!
})
```

如果返回的数组中的第一个账户不是你所期望的账户，你应该通知用户！在未来的工作中，账户数组内可能包含多个账户。但是，数组中的第一个账户将继续被认为是用户"选定"的账户。
