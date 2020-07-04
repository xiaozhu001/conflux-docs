# 移植ConfluxPortal到新环境的指引

ConfluxPortal已经持续开发了近一年时间。我们也在逐渐发现一些有用的抽象允许我们更容易的成长。这几层结合起来，可以让ConfluxPortal越来越容易移植到新的环境和语境中（尽管它可以更容易，如果您卡住了，请联系我们！）。

在我们开始之前，我们的基本架构是值得熟悉的。

![ConfluxPortal-architecture-diagram](./architecture.png)

`ConfluxPortal-background` 描述了位于 `app/scripts/background.js` 的Web扩展单例文件，该上下文实例化了一个 `ConfluxPortal Controller` 实例，代表用户的账户，与区块链的连接以及与新Dapp的交互。

当访问新站点时，WebExtension会在页面上下文中创建一个可以在`app/scripts/contentscript.js` 中看到的新 `ContentScript` 。该脚本表示每页的设置过程：为每个页面创建 `web3` 应用程序接口，并通过端口API(封装在[stream abstraction](https://github.com/substack/stream-handbook)中)
与后端脚本建立连接，并在任意加载前注入到DOM中。

你可以选择使用该流接口连接ConfluxPortal控制器，通过连接[confluxportal-inpage-provider](https://github.com/yqrashawn/conflux-portal-inpage-provider#readme)可以在流中包裹任何传输内容。但你也可以像这样为每个域构建一个提供商：

```javascript
const providerFromEngine = require('@yqrashawn/cfx-json-rpc-middleware/providerFromEngine')

/**
* returns a provider restricted to the requesting domain
**/
function incomingConnection (domain, getSiteMetadata) {
  const engine = confluxPortalController.setupProviderEngine(domain, getSiteMetadata)
  const provider = providerFromEngine(engine)
  return provider
}
```

请注意如果采取这种方式，你需要负责在页面关闭时清空过滤器：

```
const filterMiddleware = engine._middleware.filter(mid => mid.name === 'filterMiddleware')[0]
filterMiddleware.destroy()
```

### getSiteMetadata()

该方法用于使用代表请求域的图像和文本信息增强我们的确认屏幕。

这应该返回一个用对象解析的承诺，且该对象具有如下属性：

- `name`：请求的站点名；
- `icon`：代表站点标志的URI。

### 使用流接口

只有当你打算构建[confluxportal-inpage-provider](https://github.com/yqrashawn/conflux-portal-inpage-provider#readme)时才使用它。

The most confusing part about porting ConfluxPortal to a new platform is the way
we provide the
[js-conflux-sdk](https://github.com/Conflux-Chain/js-conflux-sdk#readme) API
over a series of streams between contexts. Once you understand how we create the
[ConfluxportalInpageProvider](https://github.com/yqrashawn/conflux-portal-inpage-provider#readme)
in the [inpage.js script](../app/scripts/inpage.js), you will be able to
understand how the
[extension-port-stream](https://github.com/MetaMask/extension-port-stream) is
just a thin wrapper around the [Port postMessage
API](https://developer.chrome.com/extensions/runtime#property-Port-postMessage)
(see the [Chrome documentation for extension message
passing](https://developer.chrome.com/extensions/messaging#connect) for more
information). A similar stream API can be wrapped around any communication
channel to communicate with the `ConfluxPortalController` via its
`setupUntrustedCommunication(stream, domain)` method.

### The ConfluxPortal Controller

The core functionality of ConfluxPortal all lives in what we call [The
ConfluxPortal
Controller](https://github.com/Conflux-Chain/conflux-portal/blob/master/app/scripts/metamask-controller.js).
Our goal for this file is for it to eventually be its own javascript module that
can be imported into any JS-compatible context, allowing it to fully manage an
app's relationship to Conflux.

#### Constructor

When calling `new ConfluxPortal(opts)`, many platform-specific options are
configured. The keys on `opts` are as follows:

- initState: The last emitted state, used for restoring persistent state between
  sessions.
- platform: The `platform` object defines a variety of platform-specific
  functions, including opening the confirmation view, and opening websites.
- encryptor - An object that provides access to the desired encryption methods.

##### Encryptor

An object that provides two simple methods, which can encrypt in any format you
prefer. This parameter is optional, and will default to the browser-native
WebCrypto API.

- encrypt(password, object) - returns a Promise of a string that is ready for
  storage.
- decrypt(password, encryptedString) - Accepts the encrypted output of `encrypt`
  and returns a Promise of a restored `object` as it was encrypted.


##### Platform Options

The `platform` object has a variety of options:

- reload (function) - Will be called when ConfluxPortal would like to reload its
  own context.
- openWindow ({ url }) - Will be called when ConfluxPortal would like to open a
  web page. It will be passed a single `options` object with a `url` key, with a
  string value.
- getVersion() - Should return the current ConfluxPortal version, as described
  in the current `CHANGELOG.md` or `app/manifest.json`.

#### [confluxportal.getState()](https://github.com/Conflux-Chain/conflux-portal/blob/master/app/scripts/metamask-controller.js#L450)

This method returns a javascript object representing the current ConfluxPortal
state. This includes things like known accounts, sent transactions, current
exchange rates, and more! The controller is also an event emitter, so you can
subscribe to state updates via `confluxportal.on('update', handleStateUpdate)`.
State examples available
[here](https://github.com/Conflux-Chain/conflux-portal/tree/develop/development/states)
under the `confluxportal` key. (Warning: some are outdated)

#### [confluxportal.getApi()](https://github.com/Conflux-Chain/conflux-portal/blob/develop/app/scripts/metamask-controller.js#L467-L718)

Returns a JavaScript object filled with callback functions representing every
operation our user interface ever performs. Everything from creating new
accounts, changing the current network, to sending a transaction, is provided
via these API methods. We export this external API on an object because it
allows us to easily expose this API over a port using
[dnode](https://www.npmjs.com/package/dnode), which is how our WebExtension's UI
works!

### The UI

The ConfluxPortal UI is essentially just a website that can be configured by
passing it the API and state subscriptions from above. Anyone could make a UI
that consumes these, effectively re-skinning ConfluxPortal.

You can see this in action in our file
[ui/index.js](https://github.com/Conflux-Chain/conflux-portal/blob/master/ui/index.js).
There you can see the background connection being passed in, which is
essentially the ConfluxPortal controller. With access to that object, the UI is
able to initialize a whole React/Redux app that relies on this API for its
account/blockchain-related/persistent states.

## Putting it Together

As an example, a WebExtension is always defined by a `manifest.json` file. [In
ours](https://github.com/Conflux-Chain/conflux-portal/blob/master/app/manifest.json#L31),
you can see that
[background.js](https://github.com/Conflux-Chain/conflux-portal/blob/master/app/scripts/background.js)
is defined as a script to run in the background, and this is the file that we
use to initialize the ConfluxPortal controller.

In that file, there's a lot going on, so it's maybe worth focusing on our
ConfluxPortal controller constructor to start. It looks something like this:

```javascript
const controller = new ConfluxPortalController({
    // User confirmation callbacks:
    showUnconfirmedMessage: triggerUi,
    showUnapprovedTx: triggerUi,
    // initial state
    initState,
    // platform specific api
    platform,
})
```
Since `background.js` is essentially the Extension setup file, we can see it
doing all the things specific to the extension platform:

- Defining how to open the UI for new messages, transactions, and even requests
  to unlock (reveal to the site) their account.
- Provide the instance's initial state, leaving ConfluxPortal persistence to the
  platform.
- Providing a `platform` object. This is becoming our catch-all adapter for
  platforms to define a few other platform-variant features we require, like
  opening a web link. (Soon we will be moving encryption out here too, since our
  browser-encryption isn't portable enough!)

## Ports, streams, and js-conflux-sdk!

Everything so far has been enough to create a ConfluxPortal wallet on virtually
any platform that runs JS, but ConfluxPortal's most unique feature isn't being a
wallet, it's providing an Conflux-enabled JavaScript context to websites.

ConfluxPortal has two kinds of [duplex stream
APIs](https://github.com/substack/stream-handbook#duplex) that it exposes:
- [metamask.setupTrustedCommunication(connectionStream,
  originDomain)](https://github.com/Conflux-Chain/conflux-portal/blob/master/app/scripts/metamask-controller.js#L1725) -
  This stream is used to connect the user interface over a remote port, and may
  not be necessary for contexts where the interface and the metamask-controller
  share a process.
- [metamask.setupUntrustedCommunication(connectionStream,
  originDomain)](https://github.com/Conflux-Chain/conflux-portal/blob/master/app/scripts/metamask-controller.js#L1696) -
  This method is used to connect a new web site's js-conflux-sdk API to
  ConfluxPortal's blockchain connection. Additionally, the `originDomain` is
  used to block detected phishing sites.

### js-conflux-sdk as a Stream

If you are making a ConfluxPortal-powered browser for a new platform, one of the
trickiest tasks will be injecting the js-conflux-sdk API into websites that are
visited. On WebExtensions, we actually have to pipe data through a total of
three JS contexts just to let sites talk to our background process (site ->
contentscript -> background).

To see how we do that, you can refer to the [inpage
script](https://github.com/Conflux-Chain/conflux-portal/blob/master/app/scripts/inpage.js)
that we inject into every website. There you can see it creates a multiplex
stream to the background, and uses it to initialize what we call the
[ConfluxPortalInpageProvider](https://github.com/yqrashawn/conflux-portal-inpage-provider#readme),
which you can see stubs a few methods out, but mostly just passes calls to
`sendAsync` through the stream it's passed! That's really all the magic that's
needed to create a web3-like API in a remote context, once you have a stream to
ConfluxPortal available.

In `inpage.js` you can see we create a [`postMessage
Stream`](https://github.com/Conflux-Chain/conflux-portal/blob/develop/app/scripts/inpage.js#L50),
that's just a class we use to wrap WebExtension postMessage as streams, so we
can reuse our favorite stream abstraction over the more irregular API surface of
the WebExtension. In a new platform, you will probably need to construct this
stream differently. The key is that you need to construct a stream that talks
from the site context to the background. Once you have that set up, it works
like magic!

If streams seem new and confusing to you, that's ok, they can seem strange at
first. To help learn them, we highly recommend reading Substack's [Stream
Handbook](https://github.com/substack/stream-handbook), or going through
NodeSchool's interactive command-line class [Stream
Adventure](https://github.com/workshopper/stream-adventure), also maintained by
Substack.

## Conclusion

I hope this has been helpful to you! If you have any other questions, or points
you think need clarification in this guide, please [open an issue on our
GitHub](https://github.com/Conflux-Chain/conflux-portal/issues/new/)!
