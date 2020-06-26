# 私密偏好

有时候我们希望能够在实际环境内测试一个还没有准备好供公众使用的新特性。

其中一个例子是我们的"与手机同步"功能，在支持手机的版本上线之前，退出该功能是没有任何意义的。

要启用这类功能，首先需要打开后端控制台，随后可以使用全局方法 `global.setPreference(key, value)` 。

例如，如果特征标志是一个名为 `useNativeCurrencyAsPrimaryCurrency` 的布尔值，你可以输入 `setPreference('useNativeCurrencyAsPrimaryCurrency', true)` 。

