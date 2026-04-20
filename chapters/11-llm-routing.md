# 第十一章：模型路由

> 用户换了模型，Agent 的行为应该无缝延续。

---

## 为什么需要路由层

不同的 LLM 提供商有不同的 API 格式：

- **OpenAI 兼容**：`/v1/chat/completions`，工具调用用 `function_calling`
- **Anthropic 原生**：不同的请求格式，工具调用用 `tool_use` 块
- **Gemini**：Google 自己的格式，工具调用又不一样
- **本地模型**：OpenAI 兼容格式，但通常不支持工具调用

如果 Agent 循环直接调用这些 API，代码会充满条件分支，难以维护。更糟的是，每次支持新的 Provider，都需要改动核心循环代码。

**路由层的职责**：对外暴露统一接口，对内按 Provider 类型分发到对应适配器。

---

## 统一接口设计

```
接口：
  llm.stream(messages, tools, modelId) → AsyncGenerator<StreamChunk>

输出的 StreamChunk 类型（全部 Provider 统一）：
  text              → 模型输出的文字 token
  reasoning         → 思考内容（CoT 模型，如 Kimi）
  tool_call_delta   → 工具调用的 JSON 参数（流式积累中）
  tool_call         → 完整的工具调用（参数积累完成）
  usage             → Token 用量统计
  done              → 流结束
  error             → 错误
```

Agent 循环只消费 `StreamChunk`，不知道也不关心底层是哪个 Provider。

---

## 四个适配器

### OpenAI 兼容适配器

最通用的适配器，覆盖：
- OpenAI
- 阿里云 DashScope（Qwen 系列）
- DeepSeek
- Moonshot（Kimi）
- LM Studio（本地模型）
- 任何 OpenAI 兼容 API

处理两种工具调用方式：
- **原生 Function Calling**（模型支持时）
- **XML 格式模拟**（模型不支持时，见下文）

### Anthropic 原生适配器

专门处理 Claude 系列的原生 API 格式（`tool_use` 块、`thinking` 字段等）。

### Gemini 适配器

处理 Google Gemini 的特有格式。

---

## XML 工具调用模拟

不支持原生工具调用的模型（本地模型、部分小参数量模型），用 XML 格式来模拟：

```
在系统提示里加入：
  "调用工具时，使用以下 XML 格式输出：
   <tool_call>
   <name>工具名</name>
   <input>{"参数": "值"}</input>
   </tool_call>"

然后解析模型输出里的 XML 块，转换为标准的 tool_call 事件
```

**权衡**：
- 质量下降（模型需要"想着"输出 XML，而不是原生支持）
- 但让所有模型都能工作，哪怕没有原生工具支持

这是"可降级"原则的体现：功能质量下降，但不崩溃。

---

## Provider 配置

每个 Provider 需要声明：

```
id              唯一标识
name            显示名
apiType         API 类型（openai_compatible / anthropic / gemini）
baseUrl         API 端点
supportsTools   是否支持原生工具调用（false 时用 XML 模拟）
maxContextTokens  最大上下文长度
isOverseas      是否是海外服务（影响代理配置）
```

---

## 多场景模型配置

Agent 的不同功能对模型有不同需求：

| 场景 | 需求 | 推荐策略 |
|------|------|---------|
| 主对话 | 最高质量 | 最好的可用模型 |
| 记忆提取 | 便宜、够用 | 低价模型 |
| 权限分类 | 最快 | 低延迟模型 |
| 图像生成 | 专门能力 | 多模态模型 |
| 代码生成 | 代码能力强 | 代码专项模型 |

不同场景用不同模型可以大幅降低成本，同时保证主要体验不受影响。

```
设置示例：
  defaultModelId: qwen3-plus          ← 主对话
  memoryModelId: qwen3-mini           ← 记忆提取（比主对话便宜 10 倍）
  permissionClassifierModelId: ...    ← 权限分类（追求低延迟）
```

---

## 代理配置

国内用户访问 OpenAI、Anthropic、Google 等海外服务需要代理。但配置"所有流量走代理"会让国内服务（阿里云等）也通过代理，增加延迟。

三种代理模式：

```
all           所有 LLM 请求走代理
overseas_only 只有 isOverseas=true 的 Provider 走代理
off           不走代理
```

`overseas_only` 是最常用的：国内服务直连，海外服务走代理，两不耽误。

---

## CoT 模型的特殊处理

部分模型（Kimi/Moonshot 等）在流式响应中包含 `reasoning_content`（思考过程）。

需要特殊处理：
1. 把 `reasoning_content` 保留在 assistant 消息里（某些模型 API 要求，否则质量下降）
2. 可选地把思考内容展示给用户（"蛐蛐"功能）
3. 思考内容的格式可能触发前端的特殊渲染（比如折叠显示）

---

## LLM 调用日志

每次 LLM 调用都应该记录：

```
caller      谁发起的（主对话、记忆提取、压缩、权限分类、子 Agent）
modelId     使用的模型
promptTokens / completionTokens  用量
durationMs  耗时
sessionId   会话标识
```

`caller` 字段特别重要，它让你能在日志里看出大量 token 消耗究竟来自哪个功能，帮助识别成本问题。

---

*上一章：[自进化架构](10-self-evolution.md) · 下一章：[安全架构](12-security.md)**
