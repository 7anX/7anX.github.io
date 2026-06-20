---
title: "AgentScan 工作原理：MCP、A2A 与 LLM 指纹识别"
date: 2026-06-20
draft: false
tags: ["AgentScan", "MCP", "A2A", "LLM", "工具原理"]
categories: ["安全工具"]
series: ["AgentScan"]
description: "AgentScan 如何从端口扫描走到协议确认、能力枚举和报告生成。"
ShowToc: true
TocOpen: true
---

AgentScan 的核心思路是：端口开放不等于风险成立，AI 协议识别和能力枚举才是关键。

它把扫描拆成几层：

```text
目标解析
  -> TCP 端口扫描
  -> HTTP/HTTPS 候选过滤
  -> MCP / A2A / LLM 协议探测
  -> 只读枚举与证据整理
  -> terminal / JSON / HTML / TXT 输出
```

## 目标解析

AgentScan 支持 IP、CIDR、IP range、domain、host:port 和 URL。

域名输入会保留 hostname，用于 HTTPS SNI。URL 输入会保留 path，用于优先探测自定义挂载路径。

例如：

```text
https://api.example.com/custom/mcp
```

会优先探测 `/custom/mcp`，再回退到内置路径字典。

## HTTP 过滤

TCP 开放之后，AgentScan 会先做 HTTP 过滤：

- 判断服务是否能以 HTTP/HTTPS 连接。
- 根据端口推断 scheme，`443` 和 `8443` 默认按 HTTPS。
- 非标准端口 HTTP 失败时，会尝试 HTTPS。
- 记录 Server 和 Content-Type，用于候选优先级排序。

这一层可以显著减少后续协议探测压力。

## MCP 探测

MCP 需要同时处理两类传输：

- Streamable HTTP。
- HTTP+SSE legacy。

Streamable HTTP 会直接 POST initialize。HTTP+SSE legacy 会先建立 SSE GET 连接，解析 endpoint event，再向消息 endpoint POST initialize。

MCP 指纹不是简单字符串匹配，而是组合评分：

- `protocolVersion`。
- MCP 特有 capabilities，例如 `tools`、`resources`、`prompts`、`sampling`。
- `serverInfo.name`。
- JSON-RPC 结构。
- MCP headers。
- LSP capabilities 会被排除，避免把 LSP 当 MCP。

确认 no-auth 后，AgentScan 只执行只读枚举：

```text
tools/list
resources/list
resources/templates/list
prompts/list
```

它不会调用 `tools/call`，不会写远端资源。

## MCP 蜜罐信号

AgentScan 会记录疑似蜜罐信号：

- 非法协议版本被接受。
- 两次 initialize 返回相同 Session ID。

这些信号不会默认删除结果，但可以用 `--exclude-honeypots` 过滤。

## A2A 探测

A2A 的入口是 Agent Card。AgentScan 会尝试：

```text
/.well-known/agent-card.json
/.well-known/agent.json
/agent.json
```

如果用户输入了 URL path，也会基于该 path 推导 card 路径。

识别时会解析：

- `name`、`description`、`version`。
- `provider`。
- `capabilities`。
- `skills`。
- `supportedInterfaces`。
- `securitySchemes` / `security`。

对于 JSON-RPC 候选接口，AgentScan 会做轻量状态探测，判断：

- no-auth JSON-RPC reachable。
- auth-required。
- endpoint disabled / placeholder。
- private host advertised。
- cross-host interface。

## LLM 指纹模板

LLM 模块使用 YAML 模板。内置模板位于：

```text
pkg/llmtpl/templates/llm/
```

模板可以描述：

- 一阶段 HTTP matcher。
- 二阶段确认探测。
- 版本提取。
- 模型列表提取。
- 认证状态判断。
- negative matcher。
- 多框架歧义时的 priority。

最小模板：

```yaml
info:
  name: my-llm
  severity: high
  priority: 40
http:
  - method: GET
    path: /v1/models
    matchers:
      - 'status == 200 && json.has("data")'
models:
  - method: GET
    path: /v1/models
    extractor:
      part: body
      type: json_array
      json_path: data[*].id
auth:
  endpoint: /v1/models
  open_status: [200]
  auth_status: [401, 403]
```

使用自定义模板：

```bash
agentscan llm example.com --template-dir ./my-templates
```

同名 `info.name` 会覆盖内置模板。

## 输出设计

AgentScan 有三类输出：

- terminal：适合现场看。
- JSON：适合自动化和数据处理。
- HTML/TXT：适合报告和交付。

统一扫描会把 MCP、A2A、LLM 放在同一个 HTML 报告里，用 tab 分开展示。

JSON 保留 evidence 字段，让结果尽量可复现，而不是只输出“命中/未命中”。

## 安全边界

AgentScan 的设计原则是只读：

- MCP 不调用 `tools/call`。
- A2A 不创建任务。
- LLM 不发起推理请求。
- 不写远端资源。

它的定位是暴露面识别、资产盘点和授权研究，不是利用工具。
