# AI IDE 仅仅是 VSCode 的套壳版本吗？

当下，各种 AI IDE 层出不穷。大部分都认为目前的 AI IDE 仅仅是 VSCode 的套壳版本而已。这个观点可以说是片面的，因为事实上来说，直接使用 VSCode 开源仓库进行编译打包的 Code OSS 版本是无法直接作为 AI IDE 二开基础的。

## 遥感功能

VSCode 中存在 telmetryService 模块，主要用于收集用户的使用数据并发送到微软服务器。如果你不想你开发的 AI IDE 版本被微软收集使用数据，那么你需要对 VSCode 的源码进行修改，移除相关的遥感代码。

移除该功能并不是简简单单的删除几个文件那么简单，你需要移除相关模块代码，还需要移除相关的设置，还需要移除启动命令中 `telemetry-level` 的设置。

```typescript
// src/vs/server/node/serverServices.ts
const initialTelemetryLevelArg = environmentService.args['telemetry-level'];
let injectedTelemetryLevel: TelemetryLevel = TelemetryLevel.USAGE;
// Convert the passed in CLI argument into a telemetry level for the telemetry service
if (initialTelemetryLevelArg === 'all') {
    injectedTelemetryLevel = TelemetryLevel.USAGE;
} else if (initialTelemetryLevelArg === 'error') {
    injectedTelemetryLevel = TelemetryLevel.ERROR;
} else if (initialTelemetryLevelArg === 'crash') {
    injectedTelemetryLevel = TelemetryLevel.CRASH;
} else if (initialTelemetryLevelArg !== undefined) {
    injectedTelemetryLevel = TelemetryLevel.NONE;
}
```

## Extensions

根据 Visual Studio Marketplace 的[使用条款](https://aka.ms/vsmarketplace-ToU)，你只能在 Visual Studio 的产品和服务中使用相关功能。如果你不想你的 AI IDE 还没有发布就因为使用了 Marketplace 的扩展而被微软起诉，那么你需要移除 VSCode 中对 Marketplace 的支持。

当然，你也可以选择用 [open-vsx](https://open-vsx.org/) 作为替代。这也是大部分当前的 AI IDE 选择的方案，例如 Cursor、Trae 等。


## brand

在开发自己的 AI IDE 版本时，你还需要移除 Code OSS 版本中关于 VSCode 的品牌标识。你也不希望用户在使用你的 AI IDE 的时候，看到相关信息是 VSCode 吧？

这里的大部分标识都是在 i18n 的相关描述中吗，例如：

```typescript
// extensions/configuration-editing/src/configurationEditingMain.ts
{ label: 'workspaceFolder', detail: vscode.l10n.t("The path of the folder opened in VS Code") },
{ label: 'workspaceFolderBasename', detail: vscode.l10n.t("The name of the folder opened in VS Code without any slashes (/)") },
```

## Report

如果你不想你的 AI IDE 用户在遇到问题时，跳转到 VSCode 的 GitHub 仓库进行反馈，那么你还需要修改相关的“报告问题”链接。

```typescript
// src/vs/workbench/contrib/issue/browser/baseIssueReporterService.ts
this.issueReporterModel = new IssueReporterModel({...});
```

## Cloud

你还需要移除 VSCode 中的 Cloud 相关登录功能，你也不想你的 AI IDE 是用微软账户登录吧？

```typescript
// src/vs/workbench/contrib/editSessions/browser/editSessionsStorageService.ts
this.registerSignInAction();
```

## Automatic Updates

你需要移除 VSCode 的自动更新功能以及检查更新的功能，并且实现你自己的更新机制。

```typescript
// src/vs/platform/update/common/update.config.contribution.ts
description: localize('updateMode', "Configure whether you receive automatic updates. Requires a restart after change. The updates are fetched from a Microsoft online service."),
```

## 其他

事实上，还有很多功能是需要修改的，例如：移除 Copilot、禁用 vscodedev 功能等。总之，如果你想开发一个真正意义上的 AI IDE，那么你需要对 VSCode 的源码进行深入的修改，而不仅仅是简单的套壳而已。

## 总结

上述提到的很多改动，其实在 VSCodium 项目中已经做了类似的处理。具体相关实现可以参考 [VSCodium](https://github.com/VSCodium/vscodium)，但该项目中的很多改动也并不完善，例如很多功能移除不够彻底，仅仅是禁用。如果你想开发一个合规且独立的 AI IDE，那么你需要对 VSCode 的源码进行深入的研究和修改。希望本文能为你提供一些有价值的参考。