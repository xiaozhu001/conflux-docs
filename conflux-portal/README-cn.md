# ConfluxPortal浏览器扩展
[![生成状态](https://circleci.com/gh/Conflux-Chain/conflux-portal.svg?style=svg)](https://circleci.com/gh/Conflux-Chain/conflux-portal?branch=cfx-develop)

可以在[我们的官方网站](https://github.com/Conflux-Chain/conflux-portal/releases)获取最新版本的ConfluxPortal。获取使用ConfluxPortal的帮助，可以访问我们的[用户支持网站](https://github.com/Conflux-Chain/conflux-portal/issues/new/choose)。

ConfluxPortal支持火狐，谷歌Chrome浏览器以及基于Chromium开发的浏览器。我们推荐使用最新的浏览器可用版本。


希望学习如何开发ConfluxPortal兼容的应用，可以访问我们的[开发者文档](https://conflux-chain.github.io/conflux-portal-docs/).

要了解如何为ConfluxPortal项目本身做出贡献，请访问我们的[内部文档](https://github.com/Conflux-Chain/conflux-portal/tree/develop/docs).

## 在本地生成

- 安装[Node.js](https://nodejs.org)版本10
    - 如果您正在使用[nvm](https://github.com/creationix/nvm#installation)
      (推荐) 运行 `nvm use` 能够自动为您选择正确的node版本。
- 安装[Yarn](https://yarnpkg.com/en/docs/install)
- 安装依赖： `yarn`
- 使用 `yarn dist` 生成项目到 `./dist/` 文件夹。
- 可选地，启动开发者生成模式（如带有日志和文件监控功能）可运行 `yarn start` 代替。
    - 为与应用程序一道启动[React DevTools](https://github.com/facebook/react-devtools)
      以及[Redux DevTools Extension](http://extension.remotedev.io)
      要使用 `yarn start:dev`.
      - React DevTools will open in a separate window; no browser extension is
        required
      - Redux DevTools需要作为浏览器扩展安装。启动
        Redux Remote Devtools以访问Redux状态日志。既可以通过在网络浏览器内点击右键弹出上下文菜单展开Redux DevTools面板，点击打开远程的DevTools也可以通过点击Redux DevTools扩展图标随后点击打开远程开发工具的方法方法获取状态日志。
        - 您还需要勾选"Use custom (local) server"复选框
          在远程DevTools的设置中，使用默认服务器配置（主机 `localhost` ，端口 `8000` ，安全连接复选框取消勾选）

Uncompressed builds can be found in `/dist`, compressed builds can be found in
`/builds` once they're built.

## 贡献

你可以[在这里获得我们的内部文档](https://github.com/Conflux-Chain/conflux-portal/tree/develop/docs).

### 执行测试

使用命令 `yarn test` 执行测试。

通过使用命令 `yarn watch` 执行连续的观察过程以进行测试。

可以使用 `yarn lint` 自行运行linter。

## 体系结构

[![体系结构图](./docs/architecture.png)][1]

## 开发

```bash
yarn
yarn start
```

## 生成以便发布

```bash
yarn dist
```

## 其他文档

- [如何将自定义版本添加到Chrome浏览器](./docs/add-to-chrome-cn.md)
- [如何将自定义版本添加到火狐浏览器](./docs/add-to-firefox-cn.md)
- [如何ConfluxPortal添加一种新的语言翻译版本](./docs/translating-guide-cn.md)
- [发布指引](./docs/publishing-cn.md)
- [如何将ConfluxPortal移植到一个新的平台](./docs/porting_to_new_environment-cn.md)

[1]: http://www.nomnoml.com/#view/%5B%3Cactor%3Euser%5D%0A%0A%5Bconfluxportal-ui%7C%0A%20%20%20%5Btools%7C%0A%20%20%20%20%20react%0A%20%20%20%20%20redux%0A%20%20%20%20%20thunk%0A%20%20%20%20%20ethUtils%0A%20%20%20%20%20jazzicon%0A%20%20%20%5D%0A%20%20%20%5Bcomponents%7C%0A%20%20%20%20%20app%0A%20%20%20%20%20account-detail%0A%20%20%20%20%20accounts%0A%20%20%20%20%20locked-screen%0A%20%20%20%20%20restore-vault%0A%20%20%20%20%20identicon%0A%20%20%20%20%20config%0A%20%20%20%20%20info%0A%20%20%20%5D%0A%20%20%20%5Breducers%7C%0A%20%20%20%20%20app%0A%20%20%20%20%20confluxportal%0A%20%20%20%20%20identities%0A%20%20%20%5D%0A%20%20%20%5Bactions%7C%0A%20%20%20%20%20%5BbackgroundConnection%5D%0A%20%20%20%5D%0A%20%20%20%5Bcomponents%5D%3A-%3E%5Bactions%5D%0A%20%20%20%5Bactions%5D%3A-%3E%5Breducers%5D%0A%20%20%20%5Breducers%5D%3A-%3E%5Bcomponents%5D%0A%5D%0A%0A%5Bweb%20dapp%7C%0A%20%20%5Bui%20code%5D%0A%20%20%5Bjs-conflux-sdk%5D%0A%20%20%5Bconfluxportal-inpage%5D%0A%20%20%0A%20%20%5B%3Cactor%3Eui%20developer%5D%0A%20%20%5Bui%20developer%5D-%3E%5Bui%20code%5D%0A%20%20%5Bui%20code%5D%3C-%3E%5Bjs-conflux-sdk%5D%0A%20%20%5Bjs-conflux-sdk%5D%3C-%3E%5Bconfluxportal-inpage%5D%0A%5D%0A%0A%5Bconfluxportal-background%7C%0A%20%20%5Bprovider-engine%5D%0A%20%20%5Bhooked%20wallet%20subprovider%5D%0A%20%20%5Bid%20store%5D%0A%20%20%0A%20%20%5Bprovider-engine%5D%3C-%3E%5Bhooked%20wallet%20subprovider%5D%0A%20%20%5Bhooked%20wallet%20subprovider%5D%3C-%3E%5Bid%20store%5D%0A%20%20%5Bconfig%20manager%7C%0A%20%20%20%20%5Brpc%20configuration%5D%0A%20%20%20%20%5Bencrypted%20keys%5D%0A%20%20%20%20%5Bwallet%20nicknames%5D%0A%20%20%5D%0A%20%20%0A%20%20%5Bprovider-engine%5D%3C-%5Bconfig%20manager%5D%0A%20%20%5Bid%20store%5D%3C-%3E%5Bconfig%20manager%5D%0A%5D%0A%0A%5Buser%5D%3C-%3E%5Bconfluxportal-ui%5D%0A%0A%5Buser%5D%3C%3A--%3A%3E%5Bweb%20dapp%5D%0A%0A%5Bconfluxportal-contentscript%7C%0A%20%20%5Bplugin%20restart%20detector%5D%0A%20%20%5Brpc%20passthrough%5D%0A%5D%0A%0A%5Brpc%20%7C%0A%20%20%5Bconflux%20blockchain%20%7C%0A%20%20%20%20%5Bcontracts%5D%0A%20%20%20%20%5Baccounts%5D%0A%20%20%5D%0A%5D%0A%0A%5Bweb%20dapp%5D%3C%3A--%3A%3E%5Bconfluxportal-contentscript%5D%0A%5Bconfluxportal-contentscript%5D%3C-%3E%5Bconfluxportal-background%5D%0A%5Bconfluxportal-background%5D%3C-%3E%5Bconfluxportal-ui%5D%0A%5Bconfluxportal-background%5D%3C-%3E%5Brpc%5D%0A
