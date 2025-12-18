# 手把手带你写 AI IDE

## IDE 选型

就目前来看，市面上主流的 AI IDE 全部都是基于 VSCode 开发的，那么我们也选择基于 VSCode 开发。

首先，我们假设你已经非常熟悉 VSCode 的 service-based 的架构，以及熟悉 VSCode 的源码，并且你已经搭建好了 VSCode 的开发环境。

由于直接在 VSCode 项目上进行开发会遇到如下几个问题：

1. Microsoft 的版权问题
2. Telemetry 的问题
3. Extensions Marketplace 的问题

这里我们借助开源的 [VSCodium](https://github.com/VSCodium/vscodium) 来解决这些问题。

VSCodium 是一个由社区维护的脚本库，其作用是帮助用户构建禁用 Telemetry、具有社区驱动默认配置的、并且使用 [open-vsx.org](https://open-vsx.org/) 的 VSCode 版本。

## 技术栈选择

由于 VSCode 的源码是基于原生的 HTML 进行开发的，那么我们开发的选型有如下几种：

| 技术栈 | 优点 | 缺点 |
| ------ | ---- | ---- |
| 原生 HTML + CSS + JS/TS | 与 VSCode 源码兼容性好 | 可维护性差，开发效率低 |
| React/Vue | 组件化开发，社区生态丰富 | 额外引入框架，增加包体积，diff 算法复杂 |
| SolidJS(或其他响应式框架) | 响应式编程，开发效率高，包体积小 | 学习成本较高，社区生态不如 React/Vue 丰富 |


这里我推荐是用 SolidJS 进行开发，原因如下：

- 如果选择原生 HTML 开发，效率太低，并且在当下大部分前端开发者已经习惯了组件化开发，直接使用原生 HTML 进行开发会大大降低开发效率。导致用人成本较高。
- 如果选择 React/Vue 进行开发，虽然组件化开发效率高，且社区生态丰富，但是额外引入框架会导致包体积增加，且 diff 算法复杂，必然会影响效率。

所以折中选择 SolidJS，既具备响应式编程的优势，开发效率高，且包体积小。