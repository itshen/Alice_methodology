# 第九章：Skill 系统

> Skill 让用户定制 AI 行为，无需写代码。但设计一个好的 Skill 系统，比表面看起来复杂得多。

---

## 设计动机：在工具和 Prompt 之间的空白地带

做 Agent 产品时，用户有两类需求：

1. **"帮我做 X 这件具体的事"**（工具）：有明确输入输出，可以封装成函数
2. **"帮我按 Y 这个流程做事"**（工作方式）：是一套操作步骤和判断逻辑，很难变成函数

第二类需求，传统的解法是 Fine-tuning（让模型记住这套流程）或 Prompt Engineering（在 System Prompt 里写进去）。这两种方式的问题是：**用户无法自己修改**，改了需要重新部署。

Skill 系统是一个中间层：**用 Markdown 文件描述工作流程，让 LLM 在运行时读取并遵循，同时允许用户随时修改**。

---

## 业内类似方案对比

| 方案 | 代表实现 | 定义方式 | 触发机制 | 用户可修改 | 自动优化 |
|------|---------|---------|---------|---------|---------|
| **System Prompt 扩展** | 大多数 chatbot | 直接写进 System Prompt | 始终激活 | 需要开发介入 | 无 |
| **Custom Instructions** | ChatGPT Custom Instructions | 用户填写文本框 | 始终注入 | 可以 | 无 |
| **Plugin / 工具** | OpenAI Plugin、LangChain Tool | Python 函数 | 模型调用 | 需要开发技能 | 无 |
| **Workflow 模板** | n8n、Zapier | 可视化流程图 | 触发器 | 可以，但需要图形界面 | 无 |
| **Hermes Skills** | Hermes Agent | Markdown 文件 | 模型自行决定 | 可以 | 无自动闭环 |
| **Alice Skills** | Alice | Markdown + YAML frontmatter | 语义匹配 + 模型决策 | 可以 | 自动微调闭环 |

**各方案的根本差距：**

Custom Instructions 是最接近 Skill 的方案，但它是"全局激活"的：无论什么任务，那几行指令都在上下文里。Skill 的核心价值是**按需激活**：只有当任务匹配时，才把这个 Skill 的内容注入上下文。这让每个 Skill 可以很详细，而不用担心把所有 Skill 都塞满上下文。

Hermes 的 Skill 系统和 Alice 的最接近，但有一个关键缺失：**没有自动微调闭环**。用户用了一个 Skill，Alice 会在对话结束后分析这次执行效果，如果发现有可以改进的步骤，自动更新 SKILL.md 并备份旧版本。Hermes 靠模型"自觉"调用 skill_manage 保存改进，依赖性很强。

---

## Skill 解决什么问题

没有 Skill 系统，用户每次都需要重新告诉 Alice 怎么做某件事：

```
第一次："帮我写公众号文章，需要先分析风格，然后搭大纲，再逐段写..."
第二次：同样的话再说一遍
第三次：同样的话再说一遍
```

有了 Skill，用户只需要描述一次：

```markdown
---
name: content-creator
when_to_use: 用户想写文章/公众号/文案时
---
# 创作流程
1. 分析用户的写作风格偏好
2. 确认需求和受众
3. 搭建大纲（等用户确认）
4. 逐段创作...
```

之后 Alice 会自动识别触发时机，遵循这个流程。

---

## Skill 的格式

Skill 是 Markdown 文件，带有 YAML frontmatter：

```markdown
---
name: unique-skill-name          # 唯一标识（用于调用）
description: 一句话描述功能       # 给模型看的功能摘要
when_to_use: |                   # 何时触发（关键！）
  当用户说"帮我写"、"写个"...时使用
  不适用于：只是问问题时
allowed_tools:                   # 工具白名单（可选）
  - web_search
  - file_write
  - memory_read
disable_model_invocation: false  # 是否允许模型主动调用
version: "1.0"
---

# 正文：具体的操作指南
...
```

---

## 关键字段详解

**`when_to_use`** 是最重要的字段。

模型通过 `when_to_use` 判断当前对话是否应该激活这个 Skill。写得越精确，触发准确率越高。

```
好的 when_to_use：
  "当用户明确说'帮我写公众号'、'写篇文章'、'写个文案'时使用。
  不适用于：用户只是在询问写作技巧、修改已有文章。"

差的 when_to_use：
  "写作相关"
```

**`allowed_tools`** 实现最小权限原则。

当 Skill 激活时，工具列表被限制为白名单。这防止了 Skill 误用其他工具（比如一个写作 Skill 意外调用了文件删除工具）。

**`disable_model_invocation`** 控制主动性。

- `false`（默认）：模型可以主动决定激活这个 Skill
- `true`：只能被显式调用（比如通过 `skill_invoke` 命令），模型不会自动触发

对于高权限的 Skill（比如会写入大量文件的工作流），设为 `true` 更安全。

---

## 加载优先级：用户 vs 项目

Skill 从两个位置加载：

```
~/.alice/skills/              ← 用户全局 Skill（适用所有项目）
{workdir}/.alice/skills/     ← 项目专属 Skill（只在当前项目生效）
```

**同名时，项目 Skill 覆盖用户 Skill**。

这让用户可以在某个项目里自定义不同的行为，比如全局的 `code-review` Skill 是通用的，但这个项目的 `code-review` Skill 要求特别检查安全漏洞。

---

## Skill 激活的运行时机制

当模型决定激活某个 Skill，会调用 `skill_invoke_xxx` 工具（每个 Skill 自动生成一个对应工具）。

工具执行时：
1. 把 Skill 的正文内容注入为一条 `system` 消息
2. 如果有 `allowed_tools`，把工具列表限制为白名单
3. 后续的对话轮次都在这个约束下进行

**Skill 的正文如何影响 AI 行为**：

Skill 正文出现在 system 消息里，模型会把它当作当前任务的操作指南。如果写得足够详细（每步该做什么、该调什么工具、遇到什么情况该如何判断），模型的执行质量会大幅提升。

---

## Skill 与工具的区别

| 维度 | 工具 | Skill |
|------|------|-------|
| 本质 | 代码实现的能力 | 给 AI 的操作手册 |
| 创建者 | 开发者 | 任何用户（写 Markdown 即可）|
| 执行方式 | 代码直接执行 | AI 阅读后自行执行 |
| 工具调用 | 本身是工具 | 激活后，AI 调用其他工具 |
| 适合场景 | 原子能力 | 多步流程、最佳实践 |

Skill 是工具的"上层编排"，告诉 AI 应该按什么顺序、用什么工具来完成任务，本身不直接执行任何操作。

---

## Skill 自动改进

每次对话使用了某个 Skill 后，异步分析执行效果：

1. 提取与此 Skill 相关的对话片段
2. 用 LLM 分析：这次执行中，有没有明显不对或可以更好的地方？
3. 如果有改进建议，通过 UI 展示（不自动修改）
4. 用户确认后，应用改进到 Skill 文件

这实现了"Skill 越用越好"，改进过程由用户确认，不由 AI 自动执行。

---

## 子 Agent 的角色 Skill

子 Agent 的角色系统（researcher、developer、writer 等）也是通过 Skill 实现的。

每个角色对应一个内置 Skill（`agent-researcher.md`、`agent-developer.md` 等），定义了该角色的操作规范：
- 优先使用哪些工具
- 如何组织输出格式
- 遇到某类问题如何处理

创建子 Agent 时，角色对应的 Skill 自动激活，规范子 Agent 的行为。

---

## Skill vs MCP vs 内置工具：选哪个

| 场景 | 推荐方案 |
|------|---------|
| 需要访问新的外部系统 | MCP Server |
| 需要规范化某类工作流程 | Skill |
| 需要新的原子能力（无法通过现有工具组合实现）| 内置工具 |
| 需要领域专家的最佳实践 | Skill |
| 需要可重复的多步任务 | Skill |

**原则**：能用 Skill 解决的，不要做成工具；能用工具组合解决的，不要做成内置工具。

---

*下一章：自我进化。*
