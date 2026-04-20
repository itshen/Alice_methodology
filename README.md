<div align="center">
  <img src="assets/luoxiaoshan.png" width="120" style="border-radius: 50%;" />

  <h1>Alice 工程方法论 Wiki</h1>
  <p><em>一个桌面 AI Agent 背后的设计哲学与工程范式</em></p>

  <p>
    🌐 <a href="https://alice.fans?from=methodology">alice.fans</a> &nbsp;·&nbsp;
    <a href="https://github.com/itshen/">GitHub @itshen</a> &nbsp;·&nbsp;
    作者：洛小山
  </p>

  <img src="https://img.shields.io/badge/license-MIT-blue.svg" />
  <img src="https://img.shields.io/badge/Agent-工程方法论-purple.svg" />
  <img src="https://img.shields.io/badge/Anthropic-对标研究-orange.svg" />
</div>

---

## 这是什么

这本书是我在构建 [Alice](https://alice.fans?from=methodology) 过程中系统整理的工程方法论。

Alice 是一款全平台可用的桌面 AI Agent，支持自我进化、多层记忆、多 Agent 协作，并拥有完整的角色人设。

内容是一套**经过生产验证的设计范式**，关于如何思考 Agent 的状态、工具、记忆、权限、多 Agent 协作，以及如何把这些拼成一个可靠的产品。

每一章不只讲 Alice 怎么做，还会对标 **Anthropic 官方的工程思路**，说清楚我们哪里一致、哪里走了不同的路，以及为什么。

---

## 适合谁读

<table>
<tr>
<td width="50%">

**工程师 / 研究者**
- 在构建 AI Agent 系统的开发者
- 研究 LLM 应用工程化的同学
- 对多 Agent 协作、上下文工程感兴趣的人

</td>
<td width="50%">

**产品 / 创业者**
- 想理解 Agent 产品设计逻辑的 PM
- 正在做 AI 原生产品的创业者
- 对 Agent 人格化、活人感感兴趣的设计师

</td>
</tr>
</table>

---

## 章节目录

### 一、哲学与架构

| 章节 | 主题 | 核心问题 |
|------|------|---------|
| [序章](chapters/00-preface.md) | 为什么要写这本书 | Agent 的真正复杂度在哪里？ |
| [第一章](chapters/01-philosophy.md) | 五大设计哲学 | 什么是做 Agent 之前要想清楚的事？ |
| [第二章](chapters/02-architecture.md) | 整体架构 | 一个可靠的 Agent 系统长什么样？ |

### 二、核心机制

| 章节 | 主题 | 核心问题 |
|------|------|---------|
| [第三章](chapters/03-agent-loop.md) | Agent Loop | 一次任务的完整执行流程是什么？ |
| [第四章](chapters/04-tool-system.md) | 工具系统 | 如何设计 AI 能可靠调用的工具？ |
| [第五章](chapters/05-context-memory.md) | 上下文与记忆 | 如何让 Agent 记得「正确的事」？ |

### 三、协作与扩展

| 章节 | 主题 | 核心问题 |
|------|------|---------|
| [第六章](chapters/06-multi-agent.md) | 多 Agent 协作 | 多个 Agent 如何分工而不混乱？ |
| [第七章](chapters/07-permission.md) | 权限系统 | 如何在自动化与安全之间找到平衡？ |
| [第八章](chapters/08-mcp.md) | MCP 协议 | 如何用标准接口扩展 Agent 的能力边界？ |
| [第九章](chapters/09-skills.md) | Skill 系统 | 如何让 Agent 学会用户的工作方式？ |

### 四、进化与安全

| 章节 | 主题 | 核心问题 |
|------|------|---------|
| [第十章](chapters/10-self-evolution.md) | 自进化架构 | 如何让 Agent 在用户许可下变得更好？ |
| [第十一章](chapters/11-llm-routing.md) | 模型路由 | 如何统一管理多个模型服务商？ |
| [第十二章](chapters/12-security.md) | 安全架构 | 如何保护用户数据和系统边界？ |
| [第十三章](chapters/13-observability.md) | 可观测性 | 如何看清一个 Agent 在做什么？ |

### 五、Prompt 与范式

| 章节 | 主题 | 核心问题 |
|------|------|---------|
| [第十四章](chapters/14-prompts.md) | Prompt 工程 | 如何构建分层的系统提示词体系？ |
| [第十五章](chapters/15-engineering-patterns.md) | 工程范式 | 哪些设计思路可以迁移到所有 Agent 项目？ |
| [特别章](chapters/16-alive-agent.md) | 活人感设计 | 用做游戏的方式做 AI，会有什么不同？ |

### 附录

- [附录](chapters/appendix.md)：核心概念词典、架构决策清单

---

## 与 Anthropic 官方思路的对标

这本书的一个重要线索是：**我们做了什么、Anthropic 怎么想的、两者有何不同**。

涉及到的 Anthropic 官方文章：

- [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)（2024.12）— Agent Loop、工具设计、并行化
- [Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — 上下文工程
- [How We Built Our Multi-Agent Research System](https://www.anthropic.com/engineering/built-multi-agent-research-system) — 多 Agent 架构
- [Code Execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) — MCP 协议实践

---

## 关于作者

<table>
<tr>
<td width="160">
<img src="assets/luoxiaoshan.png" width="150" style="border-radius: 12px;" />
</td>
<td>

**洛小山**

大厂产品总监，连续多年研究 AI Agent 工程化，做了十年游戏，把做游戏的方式带进了 AI 产品设计。

Alice 是我持续打磨的产品，这里记录的是背后完整的工程思考。

- 🌐 [alice.fans](https://alice.fans?from=methodology)
- 🐙 [github.com/itshen](https://github.com/itshen/)

</td>
</tr>
</table>

---

## 赞赏 & 联系

<table>
<tr>
<td align="center" width="200">
<img src="assets/qrcode_sponsor.jpg" width="160" style="border-radius: 8px;" /><br/>
<em>赞赏码</em>
</td>
<td align="center" width="200">
<img src="assets/qrcode_luoxiaoshan.jpg" width="160" style="border-radius: 8px;" /><br/>
<em>公众号</em>
</td>
<td>

如果这本书对你有帮助，欢迎：

- ⭐ 给仓库点个 Star
- 扫码关注公众号，获取更多 AI 工程化内容
- 扫码赞赏，支持持续更新

Alice 下载地址：[alice.fans](https://alice.fans?from=methodology)

</td>
</tr>
</table>

---

## License

MIT License · Copyright (c) 2026 Miyang Tech (Zhuhai Hengqin) Co., Ltd.
