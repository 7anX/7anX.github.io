---
title: "AgentScan 使用说明：找出你环境里的 AI 暴露面"
date: 2026-06-20
draft: false
tags: ["AgentScan", "MCP", "A2A", "LLM", "安全工具", "使用教程"]
categories: ["agentscan"]
series: ["AgentScan"]
description: "AgentScan 的安装、目标格式、常用命令和典型工作流。知道 MCP 和 A2A 有什么暴露面问题之后，怎么在自己的环境里扫出来。"
ShowToc: true
TocOpen: true
---

前两篇文章讲了 [MCP](/security-research/mcp-attack-surface/) 和 [A2A](/security-research/a2a-attack-surface/) 的暴露面问题。这篇讲怎么用 AgentScan 在实际环境里找出这些问题。

AgentScan 不是通用端口扫描器的套壳，它直接面向 MCP、A2A 和 LLM 推理 API 的协议层做识别。扫到"端口开着"是起点，不是终点——它继续往下做协议握手、能力枚举，输出认证状态和证据。

## 安装

```bash
git clone https://github.com/7anX/AgentScan
cd AgentScan
go build -o agentscan .
```

Windows：

```powershell
go build -o agentscan.exe .
.\agentscan.exe --help
```

依赖 Go 1.21+，没有外部运行时依赖，编译出来是单个二进制。

## 命令结构

```text
agentscan scan   # MCP + A2A + LLM 全协议扫描（推荐）
agentscan mcp    # 只扫 MCP
agentscan a2a    # 只扫 A2A Agent Card
agentscan llm    # 只扫 LLM 推理 API
```

`scan` 是最常用的。它复用端口扫描结果，一次跑完三类协议探测。

## 目标格式

```text
192.168.1.1
192.168.1.0/24
10.0.0.1-10.0.0.255
example.com
example.com:8000
https://api.example.com/mcp
```

URL 格式会保留 path，优先探测该路径。比如 `https://api.example.com/mcp` 会先试 `/mcp`，再回退到内置路径字典。

从文件读取：

```bash
agentscan scan -f targets.txt
```

文件每行一个目标，支持 `#` 注释：

```text
10.0.0.1
10.0.0.2:8000
192.168.1.0/24
# 下面是已知有 MCP 的服务
https://api.internal.example.com/mcp
```

## 主要参数

```text
-f, --file FILE          目标文件
-T, --threads N          TCP 扫描并发，默认 500
--timeout MS             TCP 连接超时，默认 2000ms
--skip-port-scan         输入视为已开放的 host:port，跳过 TCP 扫描
--proxy URL              代理：socks5://、socks4://、https://、http://
--delay MS               请求间隔，附带 ±30% jitter
-o, --output FILE        写 JSON（A2A/LLM 自动写 _a2a.json / _llm.json）
--format terminal|json   输出格式，默认 terminal
-v, --verbose            显示每个探测的进度
--no-color               禁用 ANSI 颜色
```

协议专属参数：

```text
--mcp-threads N          MCP 探测并发，默认 50
--exclude-honeypots      过滤疑似蜜罐
--a2a-threads N          A2A 探测并发，默认 50
--include-probable       包含"可能是 A2A"的结果（低置信度）
--llm-threads N          LLM 探测并发，默认 50
--template-dir DIR       自定义 LLM 指纹 YAML 模板目录
--verbose-raw            JSON 输出中包含原始 HTTP 响应
```

## 典型场景

### 场景一：扫内网，找 AI 暴露面

如果你的内网已经有团队在跑 Ollama、LM Studio、FastMCP、各种 Demo Agent，但这些服务不是通过正式流程上线的，先扫一遍：

```bash
agentscan scan 192.168.0.0/16 --timeout 300 --threads 2000 -o internal-ai.json
```

你会看到：

- 哪些 MCP Server 没有认证（直接对应[上一篇 MCP 文章](/security-research/mcp-attack-surface/)里的问题）
- 哪些 Agent Card 公开了高危 skill 或内网地址（对应 [A2A 文章](/security-research/a2a-attack-surface/)里的问题）
- 哪些 LLM 推理 API 完全开放
- 哪些有认证但认证级别是什么

内网延迟低，可以调低 `--timeout`（比如 300ms）同时调高 `--threads`（比如 2000）。

限制扫描端口可以节省时间：

```bash
agentscan scan 10.0.0.0/16 --ports 80,443,8000,8080,8443,11434,3000,5000 --timeout 300 -o internal-ai.json
```

### 场景二：测绘平台结果二次验证

从 FOFA、Shodan、ZoomEye 或 Quake 导出 `host:port` 列表之后，加 `--skip-port-scan` 直接做协议层确认：

```bash
agentscan scan -f fofa_export.txt --skip-port-scan --mcp-threads 200 -o verified.json
```

测绘平台的结果只是"可能是 MCP"，AgentScan 会做 JSON-RPC 握手确认：

- 真实 MCP，还是只是 HTTP 服务
- 是否有认证
- tools/list 返回什么
- 是蜜罐还是真实服务

### 场景三：masscan + AgentScan

大范围 TCP 扫描用 masscan，AI 协议识别用 AgentScan：

```bash
masscan 10.0.0.0/8 -p 80,443,8000,8080,11434 --rate 100000 -oL open_ports.txt
awk '/open/ {print $4 ":" $3}' open_ports.txt > targets.txt
agentscan scan -f targets.txt --skip-port-scan --timeout 3000 -o results.json
```

这适合企业大网段盘点和互联网测绘研究。

### 场景四：定时巡检

AgentScan 的 JSON 输出可以直接接定时任务、SIEM 或资产系统：

```bash
agentscan scan -f critical-ranges.txt --format json -o daily-$(date +%Y%m%d).json
```

关注变化：

- 新增 no-auth MCP（有人悄悄部署了服务）
- 新增 open LLM API
- A2A JSON-RPC 无认证可达
- Agent Card 暴露了新的私网地址或高危 skill

### 场景五：只看某类协议

只查 LLM 推理 API：

```bash
agentscan llm -f targets.txt --skip-port-scan -o llm-results.json
```

只查 A2A：

```bash
agentscan a2a 192.168.1.0/24 -o a2a-results.json
```

## 输出文件

`scan -o results.json` 会生成：

```text
results.json             # MCP 结果
results_a2a.json         # A2A 结果
results_llm.json         # LLM 结果
agentscan-report-*/
  report.html            # HTML 报告（中文）
  report_en.html         # HTML 报告（英文）
  summary.txt            # 文字摘要
```

JSON 顶层结构：

```json
{
  "version": "1.0",
  "summary": {
    "total": 150,
    "mcp_found": 3,
    "no_auth": 2
  },
  "results": [...]
}
```

每条 result 包含：

- `endpoint`：命中的路径
- `transport`：Streamable HTTP 或 HTTP+SSE legacy
- `auth_status`：`no-auth` / `auth-required`
- `tools`：工具列表（no-auth 时）
- `evidence`：指纹评分、信号和原始响应头

HTML 报告按 MCP / A2A / LLM 分 tab，适合快速浏览和交付。

## 自定义字典

`--dict-dir` 支持覆盖内置路径和端口字典：

```text
mcp_ports.txt          # MCP 探测端口
mcp_paths.txt          # MCP 路径
mcp_paths_sse_legacy.txt  # SSE legacy 路径
mcp_paths_auth.txt     # 已知 auth 端点路径（用于 auth-required 判断）
a2a_ports.txt          # A2A 探测端口
a2a_paths.txt          # A2A card 路径
llm_ports.txt          # LLM 探测端口
https_ports.txt        # 默认走 HTTPS 的端口
http_server_hints.txt  # 判断 HTTP 服务的 Server header hints
```

文件每行一个值，空行和 `#` 注释忽略。

自定义 LLM 指纹模板：

```bash
agentscan llm -f targets.txt --template-dir ./my-templates
```

模板格式是 YAML，同名 `info.name` 会覆盖内置模板。详情见[工作原理文章](/agentscan/agentscan-architecture/)。

## 和安全文章的对应关系

AgentScan 输出的每个字段对应着具体的暴露面问题：

| 输出字段 | 对应问题 |
| --- | --- |
| `auth_status: no-auth` | MCP/A2A/LLM 无认证，任何人可访问 |
| `tools` 非空 | MCP 工具列表可枚举（[MCP 文章第 2 节](/security-research/mcp-attack-surface/#2-工具列表是公开的能力清单)） |
| `resources` 非空 | MCP 资源 URI 可枚举，可能包含内部路径 |
| `private_host_in_card` | A2A Card 里有私网地址（[A2A 文章第 3 节](/security-research/a2a-attack-surface/#3-url-字段暴露内部-json-rpc-端点)） |
| `json_rpc_reachable: true` | A2A JSON-RPC 端点无认证可达 |
| `honeypot_signals` 非空 | 可能是蜜罐，加 `--exclude-honeypots` 过滤 |

---

*工具的识别逻辑详见《[AgentScan 工作原理](/agentscan/agentscan-architecture/)》。*
