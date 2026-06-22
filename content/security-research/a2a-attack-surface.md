---
title: "A2A 暴露面的安全问题"
date: 2026-06-20
draft: false
tags: ["A2A", "安全", "暴露面", "AI Agent", "Agent Card"]
categories: ["security-research"]
series: ["AI Agent 安全"]
description: "A2A Agent Card 设计带来的暴露面问题：什么信息不该放进卡片，为什么公开 JSON-RPC 比公开卡片更危险。"
ShowToc: true
TocOpen: true
---

A2A（Agent-to-Agent）是 Google 在 2025 年推出的协议，目标是让 AI Agent 之间能够互相发现、互相调用。它的核心机制是 Agent Card，一个描述 Agent 能力的 JSON 文档，挂在固定路径上供其他 Agent 读取。

A2A 的暴露面问题和 MCP 有相似之处，但也有自己的特点。核心区别在于：A2A 的设计里有一部分信息*本来就是要公开的*，但"哪些信息该公开"这条线很容易划错。

## A2A 协议基础

A2A 的发现机制基于 Agent Card，一个 JSON 文档，通常挂在：

```text
/.well-known/agent.json
/.well-known/agent-card.json
/agent.json
```

一张典型的 Agent Card：

```json
{
  "name": "DataPipeline Agent",
  "description": "Processes and transforms data pipelines",
  "version": "1.0.0",
  "url": "https://agent.example.com/a2a",
  "provider": {
    "organization": "Example Corp",
    "url": "https://example.com"
  },
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "run_pipeline",
      "name": "Run Pipeline",
      "description": "Execute a data pipeline by name"
    }
  ],
  "securitySchemes": {
    "bearerAuth": {
      "type": "http",
      "scheme": "bearer"
    }
  },
  "security": [{"bearerAuth": []}]
}
```

获取卡片之后，A2A Client 会向卡片里的 `url` 字段发起 JSON-RPC 请求来创建任务、查询状态。

## 暴露面问题

### 1. Agent Card 是公开发现的入口，但很多部署没意识到"公开"的代价

A2A 规范把 `/.well-known/agent.json` 设计成公开可读的路径，类比 `/.well-known/openid-configuration`。但这意味着任何人都能不带认证直接读取卡片内容。

当 Agent Card 被部署到内网 Agent 上时，如果服务被映射到公网（NAT、反向代理、云服务器配置错误），卡片就跟着暴露了。

### 2. 卡片里的 skills 直接暴露能力

skills 是 Agent 暴露给其他 Agent 的功能单元。问题在于 skill 的名称和描述往往直接反映了 Agent 背后挂着什么：

```json
"skills": [
  {"id": "query_internal_db", "name": "Query Internal Database"},
  {"id": "deploy_service", "name": "Deploy Microservice to K8s"},
  {"id": "access_user_records", "name": "Access User PII Records"},
  {"id": "send_internal_alert", "name": "Send Internal Alert"},
  {"id": "manage_secrets", "name": "Manage Secret Store"}
]
```

这些 skill 名字不是抽象描述，而是直接反映了 Agent 能做什么、后面连着什么系统。从攻击者角度看，这比 MCP tools/list 还直接——只要读一个 GET 请求，不需要任何认证，就拿到了完整的能力清单。

### 3. url 字段暴露内部 JSON-RPC 端点

Agent Card 里的 `url` 字段是 JSON-RPC 端点地址，A2A Client 用它来发任务。

问题是当 Card 公开但 JSON-RPC 没有保护时，任何人都可以：

1. 读 Card，拿到 `url`
2. 直接向这个 `url` 发 JSON-RPC 请求
3. 调用 skill、创建任务

更糟糕的情况：`url` 字段里写的是内网地址：

```json
{
  "url": "http://10.100.5.23:8080/a2a"
}
```

这就变成了 SSRF 辅助信息——攻击者已经知道有这个内网服务存在，以及它的地址和端口。

### 4. extensions 字段的过度暴露

A2A 规范允许扩展字段，一些实现里会把额外信息写进去：

```json
{
  "x-internal": {
    "deployRegion": "us-east-1",
    "serviceAccount": "agent-prod@project.iam.gserviceaccount.com",
    "dependsOn": ["postgres-prod", "redis-cache", "s3-bucket-internal"]
  }
}
```

这些扩展字段会把内部基础设施信息直接写进公开文档。

### 5. 无认证的 JSON-RPC 端点

Agent Card 的 `securitySchemes` 和 `security` 字段只是声明，不是保证。

扫描时常见的模式是：

- Card 里写了 `bearerAuth` 认证要求
- 但实际的 JSON-RPC 端点没有校验 token
- 或者只校验了 `tasks/send` 但没校验 `tasks/get` 和 `tasks/list`

这种不一致的认证实现比完全不认证更难发现，因为卡片声明了"我有认证"，但实际端点是开的。

### 6. 跨主机接口

有些 Agent Card 里会声明来自不同主机的 interfaces：

```json
{
  "additionalInterfaces": [
    {"transport": "grpc", "url": "grpc://internal-agent-grpc:50051"},
    {"transport": "websocket", "url": "ws://ws-agent.internal:9090/ws"}
  ]
}
```

这些内部地址暴露了整个 Agent 集群的网络拓扑，是很好的侦察材料。

## 攻击链：一个 GET 请求之后

A2A 暴露面的危险在于"入口门槛极低"。不需要做协议握手，不需要构造 JSON-RPC，一个 GET 请求，卡片就在那里。

**Step 1：找到 Agent Card**

```bash
curl -s http://10.0.8.77:8080/.well-known/agent.json
```

响应：

```json
{
  "name": "InfraAgent",
  "description": "Internal infrastructure automation agent",
  "version": "1.2.0",
  "url": "http://10.0.8.77:8080/a2a",
  "provider": {
    "organization": "Ops Team"
  },
  "capabilities": {"streaming": false},
  "skills": [
    {"id": "deploy_service", "name": "Deploy Service to Kubernetes"},
    {"id": "scale_deployment", "name": "Scale Kubernetes Deployment"},
    {"id": "exec_in_pod", "name": "Execute Command in Pod"},
    {"id": "get_secrets", "name": "Retrieve Secret from Vault"},
    {"id": "rotate_credentials", "name": "Rotate Service Credentials"}
  ],
  "securitySchemes": {
    "bearerAuth": {"type": "http", "scheme": "bearer"}
  },
  "security": [{"bearerAuth": []}]
}
```

`bearerAuth` 写在卡片里，看起来有认证。

卡片还给了另外两条信息：`url` 是 `http://10.0.8.77:8080/a2a`（内网地址，直接告诉攻击者 JSON-RPC 端点在哪），skills 名字直接写出了"操作 Kubernetes"、"从 Vault 取 secret"。

**Step 2：验证认证是否真实存在**

Card 声明了 bearerAuth，但端点有没有真正校验？不带 token 发一个请求：

```bash
curl -s -X POST http://10.0.8.77:8080/a2a \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tasks/send","params":{"id":"task-001","message":{"role":"user","parts":[{"type":"text","text":"list pods in default namespace"}]}}}'
```

响应：

```json
{
  "result": {
    "id": "task-001",
    "status": {"state": "working"},
    "artifacts": []
  }
}
```

没有 401，没有 403。卡片里的认证声明是文档，不是保护。

**Step 3：调用高危 skill**

任务创建成功，轮询状态直到完成：

```bash
curl -s -X POST http://10.0.8.77:8080/a2a \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tasks/get","params":{"id":"task-001"}}'
```

响应：

```json
{
  "result": {
    "id": "task-001",
    "status": {"state": "completed"},
    "artifacts": [
      {
        "parts": [
          {
            "type": "text",
            "text": "NAME\t\t\t\tREADY\tSTATUS\tRESTARTS\n------\napplication-pod-7d9f\t1/1\tRunning\t0\npayment-service-4b2c\t1/1\tRunning\t0\nauth-service-8e1a\t1/1\tRunning\t0\n..."
          }
        ]
      }
    ]
  }
}
```

已经在操作 Kubernetes 集群。继续发任务调用 `get_secrets`：

```bash
curl -s -X POST http://10.0.8.77:8080/a2a \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tasks/send","params":{"id":"task-002","message":{"role":"user","parts":[{"type":"text","text":"get secret db-credentials from vault"}]}}}'
```

---

整个过程：一个 GET + 三个 POST，没有任何凭证，没有利用任何漏洞。

和 MCP 相比，A2A 的起点门槛更低——MCP 需要先做 initialize 握手确认服务存在，A2A 一个 GET 就能把能力清单、内网地址、JSON-RPC 端点全部拿到。卡片设计本身是为了"方便发现"，这个目标和"安全部署"之间的矛盾，在很多实现里没有被认真处理。

## 和 MCP 暴露面的对比

| 维度 | MCP | A2A |
| --- | --- | --- |
| 发现方式 | 需要做 HTTP 探测 + 路径枚举 + 协议握手 | GET `/.well-known/agent.json` 一个请求 |
| 能力信息获取 | 需要 `tools/list`（no-auth 才行） | 卡片直接包含 skills，不需要认证 |
| 认证设计 | 规范未强制，完全可选 | 规范有认证框架，但卡片获取本身无认证 |
| 内网地址泄露 | 通过 SSE endpoint event 路径泄露 | 通过 Card `url` / `additionalInterfaces` 直接写明 |
| 主要风险 | 工具被调用，数据被读取 | 信息侦察，JSON-RPC 端点被直接访问 |

A2A 的暴露面在发现阶段比 MCP 低门槛——一个 GET 请求就能拿到 Agent 的完整能力声明。但真正的风险在后面：能不能直接调用 JSON-RPC 端点。

## 实际情况

用搜索引擎搜 `"/.well-known/agent.json"` 或 `"skills" "securitySchemes"` 能找到一些公开索引的 Agent Card。多数是 Demo 或开放 API，但也有一些看起来是误暴露的内部 Agent——skill 名称里能看出是内部业务系统。

## 怎么检查

用 [AgentScan](https://github.com/7anX/AgentScan) 扫内网：

```bash
agentscan a2a 10.0.0.0/16 -o a2a-results.json
```

它会检查：

- Agent Card 是否可公开读取
- skills 和 interfaces 内容
- JSON-RPC 端点是否无认证可达
- Card 里是否包含私网地址
- 是否存在扩展字段信息泄露

具体探测逻辑见《[AgentScan 工作原理](/agentscan/agentscan-architecture/)》。

## 修复建议

1. **JSON-RPC 端点加认证**：Card 声明了认证要求，端点也要真正校验，不能只声明不实现。
2. **Card 里不要写内网 URL**：`url` 和 `additionalInterfaces` 里只写公网可访问的地址，内网地址不要出现在公开文档里。
3. **skills 名称和描述不要暴露内部系统名**：面向公开 Agent 发现的 skill 描述应该是功能描述，不是内部实现细节。
4. **不要把内部 Agent 的 Card 路径暴露到公网**：内网 Agent 应该只在内网可访问，防火墙规则要配对。
5. **审查 extensions 字段**：扩展字段里不要写 serviceAccount、内部 IP、依赖服务名等信息。

---

*上一篇：[MCP 暴露面的安全问题](/security-research/mcp-attack-surface/)*  
*下一篇：[AgentScan 使用说明](/agentscan/agentscan-use-cases/)，如何用 AgentScan 扫出这些问题。*
