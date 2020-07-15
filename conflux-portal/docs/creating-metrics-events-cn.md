## 创建度量事件

 方法 `metricsEvent` 会通过上下文向所有组件开放。该方法在 `conflux-portal/ui/app/helpers/higher-order-components/metametrics/metametrics.provider.js` 内完成。所有组件可以通过在代码内添加设置上下文属性类型调用它：

```
static contextTypes = {
  t: PropTypes.func,
  metricsEvent: PropTypes.func,
}
```

随后可通过 `this.context` 访问。

如下为一个度量事件调用样例：

```
this.context.metricsEvent({
  eventOpts: {
    category: 'Navigation',
    action: 'Main Menu',
    name: 'Switched Account',
  },
})
```

### 基本模式

任意 `metricsEvent` 调用会传递一个必须包含 `eventOpts` 属性的对象。而 `eventOpts` 本身是一个包含三个属性的对象，包括：
- 类别：根据在matomo.org实例中设置的模式，对事件的分类
- 动作：通常描述事件发生的页面，有时描述的是页面中的关键部分
- 名称：事件的特定描述


### 隐含的属性Implicit properties

所有的度量事件在被调用时会发送如下数据：
- network
- environmentType
- activeCurrency
- accountType
- numberOfTokens
- numberOfAccounts

这些数据通过metametrics供应商添加到度量事件内。

### 自定义变量Custom Variables

度量事件可以包含自定义变量。自定义变量被包含在  `customVariables` 属性内，且该属性是传递给 `metricsEvent` 的第一个参数的一级属性。

例如:
```
this.context.metricsEvent({
  eventOpts: {
    category: 'Settings',
    action: 'Custom RPC',
    name: 'Error',
  },
  customVariables: {
    networkId: newRpc,
    chainId,
  },
})
```

自定义变量可以有自定义的属性名和值，且值可以为字符串或数字。

**为引入自定义变量，需要完成一系列必要的操作步骤**

1. 首先必须在 `conflux-portal/ui/app/helpers/utils/metametrics.util.js` 文件内的 `//Custom Variable Declarations` 下声明一个与自定义变量属性名相同的常量。
1. 随后必须将该属性名添加到 `customVariableNameIdMap` 声明内
    1. id值要在1-5之间。
    1. 在一个给定的url上，不能超过5个自定义变量的id。
