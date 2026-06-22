---
title: "AgentScan 工作原理：如何识别 MCP、A2A 和 LLM"
date: 2026-06-20
draft: false
tags: ["AgentScan", "MCP", "A2A", "LLM", "工具原理", "指纹识别"]
categories: ["agentscan"]
series: ["AgentScan"]
description: "AgentScan 的扫描流水线：从端口扫描到协议握手、指纹评分、能力枚举，以及 MCP、A2A、LLM 各自的探测逻辑。"
ShowToc: true
TocOpen: true
---

知道 [MCP](/security-research/mcp-attack-surface/) 和 [A2A](/security-research/a2a-attack-surface/) 有什么暴露面问题之后，接下来的问题是：怎么系统地找出来。这篇讲 AgentScan 是怎么做的。

## 流水线概览

```text
目标解析
  ↓
TCP 端口扫描
  ↓
HTTP/HTTPS 可达性过滤
  ↓
协议探测（MCP / A2A / LLM 并行）
  ↓
只读能力枚举
  ↓
报告生成（terminal / JSON / HTML）
```

每一层的输出是下一层的输入。端口扫描过滤掉不可达的目标，HTTP 过滤过滤掉非 HTTP 服务，协议探测在剩余候选上做指纹识别。

## 目标解析

支持格式：IP、CIDR、IP range、domain、host:port、URL。

```text
192.168.1.1
10.0.0.0/24
10.0.0.1-10.0.0.50
example.com
example.com:8000
https://api.example.com/mcp
```

URL 输入会保留 scheme、port 和 path。`https://api.example.com/mcp` 会：

1. 优先探测 `/mcp` 路径
2. 在 HTTPS 上做 SNI（用 hostname）
3. 如果 `/mcp` 没命中，再试内置路径字典

这对反向代理或自定义挂载路径很有用——如果你知道服务挂在哪，可以直接告诉 AgentScan。

## TCP 端口扫描

用 Go 标准库做并发 TCP 连接，默认并发 500，超时 2000ms。

端口字典是分协议的：

- MCP 端口字典（框架默认端口、常见开发端口）
- A2A 端口字典
- LLM 端口字典（Ollama `11434`、vLLM `8000` 等）

`scan` 命令会取三个字典的并集。`--skip-port-scan` 跳过这一层，把输入直接当作已开放的 host:port。

## HTTP/HTTPS 过滤

TCP 开放之后，AgentScan 做一次 HTTP 连接尝试：

- 根据端口推断 scheme：`443`、`8443` 默认 HTTPS；其他端口先试 HTTP，失败再试 HTTPS
- 记录 `Server` 和 `Content-Type` 响应头
- 过滤掉无法以 HTTP 连通的端口

这一层的目的是在进入协议探测之前减少候选数量。一个开放的 22 端口（SSH）不需要进 MCP 探测。

## MCP 探测

这是 AgentScan 里最复杂的部分。原因是 MCP 有两套传输协议，而且指纹识别不是简单字符串匹配。

### 两种传输

**Streamable HTTP**（2025-03-26 规范）：

直接 POST initialize 到候选路径：

```http
POST /mcp HTTP/1.1
Content-Type: application/json
Accept: application/json, text/event-stream

{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"agentscan","version":"1.0"}}}
```

响应可以是 JSON 或 SSE 流（规范都允许）。

**HTTP+SSE legacy**（2024-11-05 规范）：

分两步：

1. GET `<sse_path>`，建立 SSE 长连接，等待 `endpoint` event
2. 从 event data 里拿到 POST 路径，再 POST initialize
3. 响应通过 SSE 流推回来

```
GET /sse
data: /messages/?session_id=abc123
    ↓
POST /messages/?session_id=abc123
    ↓
SSE stream: {...initialize response...}
```

AgentScan 对每个候选路径并发尝试两种传输，优先返回 no-auth 命中。

### 指纹评分

找到响应之后，AgentScan 不是做字符串匹配，而是组合评分：

| 信号 | 分值 |
| --- | --- |
| `result.protocolVersion` 存在 | +0.2 |
| `result.capabilities` 含 MCP 特有 key（`tools`、`resources`、`prompts`、`sampling`） | +0.3 |
| `result.serverInfo.name` 存在 | +0.1 |
| `Mcp-Session-Id` 或 `Mcp-Protocol-Version` 响应头 | 兜底确认 |

LSP（Language Server Protocol）排除机制：如果 capabilities 里出现 LSP 特有 key（`textDocumentSync`、`hoverProvider` 等），直接返回 0，避免把 LSP 当 MCP。

原因是 MCP 和 LSP 都是 JSON-RPC，响应结构类似，如果没有 LSP 排除，会有误报。

阈值 0.35。低于阈值的响应不报告。

### no-auth 判断

状态码 200 且指纹评分过阈值 → no-auth。

4xx 响应会做进一步分析：

- 有 `Mcp-Session-Id` / `Mcp-Protocol-Version` 响应头 + 401/403 → auth-required
- 响应体含 `"jsonrpc"` 且有 `"error"` 字段 + 401/403 → auth-required
- 其他 4xx → 不报告

这区分了"MCP Server 在但需要认证"和"不是 MCP 服务"。

### 只读枚举

确认 no-auth 后，执行：

```text
tools/list
resources/list
resources/templates/list
prompts/list
```

不调用 `tools/call`，不写远端资源。AgentScan 是只读的。

### 蜜罐信号

两个场景会记录 honeypot signal：

1. 发送无效协议版本（随机字符串），服务器依然接受
2. 两次 initialize 返回相同 Session ID

这些信号不过滤结果，用 `--exclude-honeypots` 才过滤。

## A2A 探测

A2A 的探测比 MCP 简单，核心是 GET Agent Card，然后做多维度评分。

### 获取 Agent Card

优先尝试：

```text
/.well-known/agent.json
/.well-known/agent-card.json
/agent.json
```

如果用户输入了 URL path，会基于该 path 推导 card 路径（比如 `/agents/my-agent` 会尝试 `/agents/my-agent/.well-known/agent.json`）。

### 指纹评分

解析 JSON 之后做多维度评分，信号包括：

- `name` 字段存在（Agent 名称）
- `url` 字段是有效 URL
- `skills` 数组非空
- `capabilities` 对象存在
- `securitySchemes` / `security` 字段
- 响应 `Content-Type` 是 `application/json`
- 路径是 `.well-known` 路径

有负面信号会跳过：比如响应里同时有明显的非 A2A 标志（比如 OpenID 配置字段）。

置信度阈值两档：

- `confirmed`（≥ 0.65）：直接报告
- `probable`（≥ 0.45）：只在 `--include-probable` 时报告

### JSON-RPC 可达性检查

拿到 Card 里的 `url` 字段之后，AgentScan 做轻量状态探测：

- 向 `url` 发一个最小 JSON-RPC 请求
- 判断响应是 200 no-auth、4xx auth-required、连接失败、还是端点关闭

同时检查：

- 私网地址泄露（Card 里的 `url` 或 `additionalInterfaces` 包含 RFC 1918 地址）
- 跨主机接口（Card 里声明了不同主机的接口地址）

这两类检查对应 [A2A 文章](/security-research/a2a-attack-surface/) 里讲的内网地址泄露问题。

## LLM 推理 API 探测

LLM 模块用 YAML 模板驱动，内置覆盖了主流框架：Ollama、vLLM、SGLang、TGI、llama.cpp、Xinference、LiteLLM、FastChat、LocalAI、LM Studio、LMDeploy。

### 模板结构

```yaml
info:
  name: ollama
  severity: high
  priority: 50

http:
  - method: GET
    path: /api/tags
    matchers:
      - 'status == 200 && json.has("models")'

models:
  - method: GET
    path: /api/tags
    extractor:
      part: body
      type: json_array
      json_path: models[*].name

auth:
  endpoint: /api/tags
  open_status: [200]
  auth_status: [401, 403]
```

模板可以定义：

- 一阶段 HTTP matcher（判断是不是这个框架）
- 二阶段确认探测（减少误报）
- 版本提取
- 模型列表提取
- 认证状态判断
- negative matcher（排除相似但不是的服务）
- priority（多框架歧义时谁优先）

### 自定义模板

```bash
agentscan llm example.com --template-dir ./my-templates
```

同名 `info.name` 会覆盖内置模板。如果有内部 LLM Gateway 或私有推理框架，可以写自己的模板。

## 输出

三种格式：

**terminal**：适合现场看，带 ANSI 颜色，按协议分块输出，no-auth 高亮。

**JSON**：机器可读，包含完整 evidence 字段（指纹信号、HTTP 状态、响应头），适合自动化处理和二次分析。

**HTML/TXT**：适合报告和交付，`scan` 命令自动生成，按 MCP/A2A/LLM 分 tab。

JSON 的 evidence 字段设计目标是让结果可复现——不只输出"命中"，还输出是什么信号让它命中，支持人工二次确认。

---

*前置阅读：[MCP 暴露面的安全问题](/security-research/mcp-attack-surface/) · [A2A 暴露面的安全问题](/security-research/a2a-attack-surface/)*  
*如何用 AgentScan 扫你的环境：[使用说明](/agentscan/agentscan-use-cases/)*
