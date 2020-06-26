# 发布指南

当发布ConfluxPortal的新版本时，我们按照如下步骤进行：

## 概述

下图对我们的设计、开发和发布流程进行了详细的概述。
构建ConfluxPortal是一项社区的工作，该工作流程中的许多步骤都邀请了外部贡献者的参与。所有的QA工作、代码审查及发布新的版本是由ConfluxPortal的核心团队成员完成的。

<img width="664" alt="mm-dev-process"
src="https://user-images.githubusercontent.com/1016190/56308059-36906000-60fb-11e9-8e61-6655bca0c54f.png">


## 参与

我们需要确保在部署前，满足一定的标准：

- 在一周的早些时候进行部署，为紧急应对难以预见的漏洞缺陷赢得时间。
- 在一天中的早些时候进行部署，原因同上。
- 确保支持团队内至少有一名成员保持“当值”状态，以便通过支持系统获取用户新遇到或发现的问题。
- 在可能的情况下，采用增量推广的方案：先向少量用户推送，随后逐步向更多的用户推送。

## 递增版本 & 变更日志

通过使用命名规范 `Version-vX.Y.Z` 创建分支可以实现版本的自动递增，其中 `X`， `Y`，以及 `Z` 都是数字。分支应当从主分支的基础上创建。[可以在GitHub上创建分支](https://help.github.com/en/articles/creating-and-deleting-branches-within-your-repository)

一旦创建了一个版本分支，在CircleCI上的构建将会针对该版本创建一个Pull Request并对应用清单和变更日志版本进行变更。

## 为较为敏感的变动做准备

如果新的发行版本存在较为敏感的变化，在发布前无法完全核实，请遵循[敏感发行协议](./sensitive-release-cn.md).

## 构建

当我们在 `develop` 分支上进行开发时，我们的生产版本则会在 `master` 分支上进行维护。

对于每一次的Pull request请求，@ConfluxBot都会针对这个新的Pull request请求进行构建和评论，所以在 `develop` 分支上更新版本后，可以针对 `master` 分支开启一个Pull request请求，一旦该请求被审查与合并，你就可以下载这些新的构建以便进行发布。

## 发布

1. 发布到Chrome商店。
2. 访问[Chrome开发者仪表盘](https://chrome.google.com/webstore/developer/dashboard?authuser=2)。
3. 发布到[Firefox附加组件中心](http://addons.mozilla.org/en-us/firefox/addon/ether-metamask)。
4. 发布到[Opera商店](https://addons.opera.com/en/extensions/details/metamask/)。
5. 发布到[Github](https://github.com/Conflux-Chain/conflux-portal/releases) 页面。
6. 运行 `yarn announce` 脚本，并在公开场合发布这一公告。

## 修补程序差异

我们的 `develop` 分支通常还没有进行完整的测试以确保质量，所以我们会认为他依然还处于不稳定的状态。

因此，当在生产环境中需要进行紧急修改时，其Pull request请求应：

- 把它描述为一个热修复补丁。Describe it as a hotfix.
- 使用热修复补丁标记。
- 应针对 `master` 分支进行提交。

此外，版本和变更日志应当从 `master` 分支上进行修改，并在随后合并到 `develop` 分支以使得两个分支恢复同步的状态。通过将版本/变更日志放入针对 `master` 分支的Pull Request请求内，可以进一步节省时间，因为我们会依靠@ConfluxBot的帮助在合并前运行测试。
