# 第八章：能力扩展：MCP 协议

> 如果 Agent 的能力完全内置，它就是一个封闭的系统。

---

## 内置工具的局限

内置工具（读文件、执行命令、搜索网络）是 Agent 的基础能力，但很快就会遇到需要访问更多外部系统的需求：

- 操作 WPS 笔记
- 查询公司内部数据库
- 调用 Slack / 企业微信 API
- 访问特定的数据分析平台

如果这些都需要写进 Agent 的代码里，Agent 会变成一个无底洞：永远有新的外部系统需要支持，永远有外部系统的 API 在变化，永远有"我想用 XXX 但 Agent 不支持"的抱怨。

**MCP（Model Context Protocol）**是 Anthropic 提出的开放协议，让 Agent 可以在运行时动态发现和调用外部能力，而不需要修改 Agent 本身。

---

## MCP 的工作模型

```
MCP Server（外部服务）           Alice（MCP 客户端）
    │                                  │
    │  暴露工具列表：                    │
    │  - create_note                   │
    │  - search_notes                  │  listTools()
    │  - update_note                   │ ←────────────
    │                                  │
    │                                  │  包装为内置工具格式
    │                                  │  mcp_wpsnote_create_note
    │                                  │
    │  LLM 发起工具调用                 │
    │     ←──────────────────────────  │
    │                                  │  callTool()
    │  执行并返回结果                   │ ─────────────→
    │
```

从 Agent 的视角，MCP 工具和内置工具看起来完全一样：都是工具列表里的一个工具，都有名称、描述、输入格式，都返回 ToolResult。

唯一的区别是名称前缀（`mcp_{serverId}_{toolName}`）用于路由。

---

## 工具命名规范

MCP 工具名必须有明确的命名空间，防止不同 Server 的同名工具冲突：

```
格式：mcp_{serverId}_{toolName}

例子：
  wpsnote Server 的 create_note → mcp_wpsnote_create_note
  xsct-bench Server 的 list_models → mcp_xsct-bench_list_models
```

命名时需要规范化特殊字符（`-` 等转换为 `_`），保证工具名合法。

---

## 传输协议

MCP 支持多种传输方式：

| 传输方式 | 适用场景 | 推断规则 |
|---------|---------|---------|
| `stdio` | 本地进程（命令行程序）| 有 `command` 字段 |
| `sse` | 远程服务（旧格式）| URL 结尾含 `/sse` |
| `streamable_http` | 远程服务（新格式）| 有 `url` 字段 |

大多数情况下不需要显式指定，按规则自动推断。

---

## 懒连接：不阻塞启动

MCP Server 的启动可能很慢（特别是需要网络连接的远程服务）。如果在 Agent 启动时同步等待所有 MCP Server 连接，会严重拖慢启动速度。

正确做法：**懒连接**。

```
register(config)  → 只保存配置，不建立连接
    ↓ 第一次调用该 Server 的工具时
ensureConnected()  → 建立连接，缓存工具列表
    ↓ 后续调用
直接使用已建立的连接
```

连接失败不影响 Agent 整体启动，只影响该 Server 的工具不可用。

---

## 工具描述截断

MCP Server（特别是通过 OpenAPI 生成的）可能给工具提供非常长的描述（15-60KB 也不罕见）。

如果不截断，50 个工具的描述可以轻松占用 100 万 token 以上，让模型的推理空间几乎为零。

解决方案：对工具描述做长度限制（约 2048 字符），超出截断。

**影响**：模型对工具的了解程度下降。
**替代方案**：鼓励 MCP Server 的作者写更精炼的工具描述。

---

## 超时与大结果处理

MCP 工具调用可能很慢（远程服务、数据库查询等）。必须有超时保护：

- 调用超时（约 30 秒）：超时后报错返回，不无限等待
- 结果大小限制（约 200KB）：超过则截断，防止大结果填满上下文

---

## 断线重连

网络问题会导致 MCP 连接断开。监听连接断开事件，几秒后自动重连：

```
连接断开事件
    ↓ 5 秒后
尝试重连
    ↓ 成功
更新工具缓存
    ↓ 失败
标记为不可用，等待下次尝试
```

---

## Alice 作为 MCP Server

MCP 是双向的。Alice 不只是 MCP 客户端，也可以作为 MCP Server，把自己的能力（对话、bash 执行、记忆访问）暴露给其他 MCP 客户端。

这让 Alice 可以被集成进其他工具链（比如 IDE 插件、CI/CD 系统），作为一个"有权限管控的 AI 执行引擎"被调用。

---

## MCP 与权限

MCP 工具默认使用最保守的权限设置：

- `isReadOnly: false`（保守假设：可能有副作用）
- `requiresPermission: true`（需要权限检查）
- `isConcurrencySafe: false`（保守假设：不能并行）

用户可以在权限规则里显式放行特定的 MCP 工具：

```yaml
permissions:
  - action: allow
    tool: "mcp_wpsnote_*"  # 允许 wpsnote 的所有工具
```

---

## 工具发现的质量问题

MCP 让"有更多工具"变得容易，但"模型知道什么时候用哪个工具"仍然是挑战。

工具过多会：
1. 降低模型的 Tool Choice 准确率
2. 增加每次请求的 token 成本（工具定义要发给模型）

解决方向：
- MCP Server 作者写高质量的 `when_to_use` 描述
- 按场景动态加载工具（而不是把所有工具都塞给模型）
- 用 ToolSearch 工具（让模型先搜索可用工具再决定用哪个）

---

*上一章：[权限系统](07-permission.md) · 下一章：[Skill 系统](09-skills.md)**
