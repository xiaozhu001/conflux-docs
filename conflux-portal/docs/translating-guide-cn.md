# ConfluxPortal的翻译指导

ConfluxPortal浏览器扩展支持通过将语言环境文件放入 `app/_locales` 的形式添加新的翻译。

- [MDN的扩展国际化指南](https://developer.mozilla.org/en-US/Add-ons/WebExtensions/Internationalization)

## 添加一种新的语言

- 每一种支持的语言都会在 `app/_locales` 内以一种语言子标签名文件夹的形式进行表示(样例： `app/_locales/es/`)。 (查询语言的子标签名可以使用[r12a"查找"工具](https://r12a.github.io/app-subtags/)或[维基百科列表](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes))。
- 在该文件夹内需要有一个文件 `messages.json`。
- 开始您的翻译的一个简单的方法就是首先 **拷贝一份** `app/_locales/en/messages.json` 文件（英语翻译版本），随后，为每一个应用内消息, **翻译 `message` 键值** 。
- **`description` 键值** 仅仅是为了增加翻译内容的上下文，它本身**并不需要被翻译**。
- 在[语言索引](https://github.com/Conflux-Chain/conflux-portal/blob/master/app/_locales/index.json)内添加 `app/_locales/index.json`。


这就好了！当ConfluxPortal被加载到电脑上时，其对应语言将被设置为系统语言，他们将看到您的翻译内容，而不是默认的翻译内容。

## 测试

为了自动化检查是否在翻译时遗漏了任何短语，我们提供了一个只要您知道如何使用命令就可以运行的脚本。脚本为：

```
node development/verify-locale-strings.js $YOUR_LOCALE
```

其中 `$YOUR_LOCALE` 使您的本地语言字符串 (样例： `es`)，即您使用的语言文件夹名。 

为了验证您的翻译是否能在应用内工作，需要为ConfluxPortal[构建一个语言拷贝](https://github.com/Conflux-Chain/conflux-portal#building-locally)。您需要更改您的浏览器语言、操作系统语言，并重新启动浏览器（抱歉，这些步骤确实有些繁琐！）。