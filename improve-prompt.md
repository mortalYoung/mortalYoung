# Improve Prompt

基于 [AI 应用的三大核心](./ai-app-cores.md) 中提到的 **AI 应用的质量取决于模型质量，上下文质量和提示词质量** 观点。

本文详细阐述了 **提示词质量** 的提升方法。

## 六个方面 ✨

### 角色描述 🧑‍💼

角色描述是指在提示词中明确指定模型所扮演的角色或身份。

这种方法可以帮助模型更好地理解任务的背景和要求，从而生成更符合预期的回答。

例如在 Claude Code 的 System Prompt 中，其明确说明了模型的角色：

```markdown
You are Claude Code, Anthropic's official CLI for Claude.
```

> 💡 TIPS：角色描述应尽量简洁明了，避免过于复杂或冗长的信息，以免影响模型的理解和生成效果。

> 🌐 TIPS: 据说提示词用英文效果更好。


### 任务描述 🎯

任务描述是指在提示词中明确指定模型需要完成的具体任务或目标。

例如：

```markdown
You are an interactive CLI tool that helps users with software engineering tasks. Use the instructions below and the tools available to you to assist the user.
```

### 上下文信息 📚

上下文信息是指在提示词中提供与任务相关的背景信息或上下文，以帮助模型更好地理解任务。

上下文信息是非必需的，在某些情况下可以省略，但提供足够的上下文信息通常会提高模型的生成效果。


### 约束条件 🚦

约束条件是指在提示词中明确模型在生成回答的时候需要遵守的规则或限制。

例如：
```markdown
# Code style
- IMPORTANT: DO NOT ADD ***ANY*** COMMENTS unless asked
```

### 回答格式 📝

回答格式是指在提示词中明确指定模型生成回答的格式或结构。

例如：
```markdown
If the user asks for help or wants to give feedback inform them of the following: 
- /help: Get help with using Claude Code
- To give feedback, users should report the issue at https://github.com/anthropics/claude-code/issues
```

### 例子 🧩

例子是非必须的，但提供例子可以帮助模型更好地理解任务和预期的回答格式。

例如：
```markdown
<example>
user: what is 2+2?
assistant: 4
</example>
```


## Example 🌟

这里举个例子，比如我现在有一个客服系统，我需要为其开发 Agent 相关功能，使其成为智能客服系统。

那么根据上述六个方面，首先我需要定义模型的角色为：

```markdown
你是一个 xxx 客服系统的虚拟助手。
```

接着，定义智能体的任务：

```markdown
# responsibilities
- IMPORTANT: 帮助用户解决他们在使用我们产品时遇到的问题。
- IMPORTANT: 回答用户的常见问题，提供产品信息，并引导用户完成特定的操作。
```

> 🔔 大写字母是具有强调意图，在约束或规范中通过大写字母来强调重要性。

> 🏷️ 模型对 Markdown 的**结构化语法**可以会更加敏感。

然后，提供一些上下文的信息：
```markdown
# context
- xxx 客服系统是一个基于人工智能的智能客服平台，旨在为用户提供快速、准确的在线支持和服务。
```

接下来是我理解提示词中比较重要的约束条件的部分：
```markdown
# constraints
## Tone
- 你的语气应该是平易近人，友好和乐于助人。
- 不要（NEVER）使用技术术语或复杂的语言，除非用户明确要求。
- 不要（NEVER）表现出不耐烦或冷漠的态度。
- 回答的内容必须（MUST）简洁明了，不超过 200 字，除非用户要求更详细的信息。
- 回答必须（MUST）基于提供的上下文信息，并与用户的问题直接相关。

<example>
user: 你能告诉我如何使用这个客服系统吗？
assistant: 当然可以！请问你具体想了解哪个功能呢？
</example>

<example>
user: 帮我查询 xxx 产品
assistant: 好的，查询结果为 xxxx。
</example>

## Response Format
- 如果你需要获取帮助文档，请使用 `/help` 格式
- 如果你想要调用客服系统的 API，请使用 `[use the xxx tool to do something](#toolcall-xxx)`
```

> ⚠️ 除非是明确的 `NEVER` 和 `MUST`，针对比较复杂的场景建议是更详细的描述执行流程。

上述例子中的提示词是比较简单的，在实际的场景中，通常还需要考虑更多的约束细节以实现更复杂的交互。

例如，为了实现 CoT 推理，可以在提示词中加入推理过程的描述，例如：

```markdown
# reasoning
- 在回答用户问题之前，先进行必要的推理和分析。
- 需要明确每一步(STEP BY STEP)推理的依据和逻辑。
```

## 总结 🏁

通过对提示词的六个方面进行优化，可以显著提升 AI 应用的性能和用户体验。明确的角色描述、任务描述和上下文信息能够帮助模型更好地理解用户需求，而合理的约束条件和回答格式则确保了生成内容的质量和一致性。最后，提供具体的例子可以进一步引导模型生成更符合预期的回答。
