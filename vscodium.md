# AI IDE 仅仅是 VSCode 的套壳版本吗？

当下，各类 AI IDE 如雨后春笋般涌现。一个普遍却片面的论调是：它们不过是 Visual Studio Code 的“套壳”版本。这种观点严重低估了构建一个合规、可控且体验独立的 AI IDE 所面临的工程技术挑战。事实是，直接基于官方开源版本 Code OSS 进行二次开发，会引入一系列从法律合规到技术自主性的致命问题。

## 重构微软依赖

### 遥感功能

VSCode 中存在 telemetryService 模块，主要用于收集用户的使用数据并发送到微软服务器。如果你不想你开发的 AI IDE 版本被微软收集使用数据，那么你需要对 VSCode 的源码进行修改，移除相关的遥感代码。

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

### Extensions

遵循 Visual Studio Marketplace 的[许可条款](https://aka.ms/vsmarketplace-ToU)，意味着你的产品将被牢牢绑定在微软生态内。任何未经授权的分发都可能引发法律纠纷。因此，彻底移除对 Marketplace 的依赖，并集成如 [Open-VSX](https://open-vsx.org/) 等开源替代品，是构建独立商业产品的法律底线。这也是 Cursor、Codeium 等主流 AI IDE 的共同选择。

## 品牌独立

### brand

在开发自己的 AI IDE 版本时，你还需要移除 Code OSS 版本中关于 VSCode 的品牌标识。你也不希望用户在使用你的 AI IDE 的时候，看到相关信息是 VSCode 吧？

这里的大部分标识都是在 i18n 的相关描述中吗，例如：

```typescript
// extensions/configuration-editing/src/configurationEditingMain.ts
{ label: 'workspaceFolder', detail: vscode.l10n.t("The path of the folder opened in VS Code") },
{ label: 'workspaceFolderBasename', detail: vscode.l10n.t("The name of the folder opened in VS Code without any slashes (/)") },
```

### 用户反馈

如果你不想你的 AI IDE 用户在遇到问题时，跳转到 VSCode 的 GitHub 仓库进行反馈，那么你还需要修改相关的“报告问题”链接。

```typescript
// src/vs/workbench/contrib/issue/browser/baseIssueReporterService.ts
this.issueReporterModel = new IssueReporterModel({...});
```

## 重构核心逻辑

### 云端同步

开发团队必须移除 Cloud 相关的代码逻辑，确保相关 Cloud 登录的功能完全被自身用户管理体系所取代。

```typescript
// src/vs/workbench/contrib/editSessions/browser/editSessionsStorageService.ts
this.registerSignInAction();
```

### Automatic Updates

你需要移除 VSCode 的自动更新功能以及检查更新的功能，并且实现你自己的更新机制。

```typescript
// src/vs/platform/update/common/update.config.contribution.ts
description: localize('updateMode', "Configure whether you receive automatic updates. Requires a restart after change. The updates are fetched from a Microsoft online service."),
```

## 其他

事实上，还有很多功能是需要修改的，例如：移除 Copilot、禁用 vscodedev 功能等。总之，如果你想开发一个真正意义上的 AI IDE，那么你需要对 VSCode 的源码进行深入的修改，而不仅仅是简单的套壳而已。

## 总结

事实上，上述提到的很多改动，在 VSCodium 项目中已经做了类似的处理。具体相关实现可以参考 [VSCodium](https://github.com/VSCodium/vscodium)。
****
然而该项目中的很多改动也并不完善，例如很多功能移除并不是删除代码，而是禁用。但是 VSCodium 项目为我们提供了一个有价值的起点，通过参考其改动，我们可以更好地理解如何将 VSCode 转变为一个独立的 AI IDE。希望本文的拆解，能为你的技术选型与架构设计提供有力的决策依据。