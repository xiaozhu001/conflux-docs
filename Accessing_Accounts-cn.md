

​       用户的账户在Conflux中的各种情境中使用，它们作为整洁的标志，但没有任何用途比*调用钱包方法*更重要。这些方法涉及到签名或交易审批。所有这些方法都需要转出账户作为函数参数。

* `cfx_sendTransaction`

     一旦您连接到一个用户，您可以随时通过检查 conflux.selectedAddress 来重新检查当前账户。

    如果你想在地址改变时收到通知，我们有一个event可以供你们使用：

```plain
conflux.on('accountsChanged', *function* (accounts) {
 // Time to reload your interface with accounts[0]!
})
```
如果返回的数组中的第一个账户不是你所期望的账户，你应该通知用户！在未来，账户组可能包含多个账户。但是，数组中的第一个账户将继续被认为是用户的 "选定 "账户。
