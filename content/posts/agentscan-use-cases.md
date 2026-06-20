---
title: "AgentScan 使用手册：从内网盘点到互联网测绘"
date: 2026-06-20
draft: false
tags: ["AgentScan", "MCP", "A2A", "LLM", "安全工具"]
categories: ["安全工具"]
series: ["AgentScan"]
description: "AgentScan 的目标格式、常用命令、输出报告和典型扫描工作流。"
ShowToc: true
TocOpen: true
---

AgentScan 是一个面向 AI Agent 暴露面的扫描器。它不是单纯检查端口是否开放，而是继续识别 MCP、A2A Agent Card 和 LLM 推理 API，并输出认证状态、能力清单和证据。

## 快速安装

```bash
git clone https://github.com/7anX/AgentScan
cd AgentScan
go build -o agentscan .
```

Windows:

```powershell
go build -o agentscan.exe .
.\agentscan.exe --help
```

## 命令结构

```text
agentscan <command> [options] [target...]

Commands:
  scan    扫描所有已支持协议：MCP + A2A + LLM
  mcp     MCP-first 统一扫描
  a2a     只扫描 A2A Agent Card
  llm     只扫描 LLM 推理 API
```

`scan` 是最常用命令。它会先做端口扫描和 HTTP 过滤，再复用候选目标执行 MCP、A2A 和 LLM 探测。

## 目标格式

AgentScan 支持这些输入：

```text
10.0.0.1
10.0.0.0/24
10.0.0.1-10.0.0.255
api.example.com
api.example.com:8000
https://api.example.com/mcp
```

URL 会保留 scheme、port 和 path。比如 `https://api.example.com/mcp` 会优先探测 `/mcp`，这对反向代理或自定义挂载路径很有用。

从文件读取：

```bash
agentscan scan -f targets.txt
```

`targets.txt` 每行一个目标，支持 `#` 注释：

```text
10.0.0.1
10.0.0.2:8000
192.168.1.0/24
https://api.example.com/mcp
# comment
example.com
```

## 常用参数

```text
-t, --target TARGET       添加目标，可重复
-f, --file FILE           从文件读取目标
--ports LIST              逗号分隔端口，覆盖内置端口字典
--dict-dir DIR            加载自定义字典目录
--skip-port-scan          跳过 TCP 扫描，输入视为已开放 host:port
-T, --threads N           TCP 扫描并发，默认 500
--timeout MS              TCP 连接超时，默认 2000ms
--proxy URL               socks5/socks4/https/http 代理
--delay MS                请求延迟，带正负 30% jitter
--format terminal|json    输出格式
-o, --output FILE         写 JSON 文件
-v, --verbose             显示详细探测进度
```

协议相关参数：

```text
--mcp-threads N           MCP 探测并发
--exclude-honeypots       过滤疑似 MCP 蜜罐
--a2a-threads N           A2A 探测并发
--include-probable        包含 probable agent discovery
--llm-threads N           LLM 探测并发
--template-dir DIR        加载自定义 LLM YAML 指纹模板
--verbose-raw             JSON 中包含原始响应
```

## 场景一：内网 AI 资产盘点

```bash
agentscan scan 192.168.1.0/24 --timeout 300 --threads 2000 -o intranet-ai.json
```

内网延迟低，可以适当降低 `--timeout` 并提高 `--threads`。如果网段很大，建议先限制端口：

```bash
agentscan scan 10.0.0.0/16 --ports 80,443,8000,8080,11434 --timeout 300 -o intranet-ai.json
```

目标是快速找出：

- 没有认证的 MCP Server。
- 开放的 Ollama / vLLM / SGLang / LiteLLM。
- 公开的 Agent Card。
- Agent Card 中泄露的私网接口。

## 场景二：测绘平台结果二次验证

从 FOFA、Quake、Shodan、ZoomEye 导出 `host:port` 后，应该使用 `--skip-port-scan`：

```bash
agentscan scan -f exported.txt --skip-port-scan --mcp-threads 200 -o verified.json
```

这样可以省掉 TCP 扫描阶段，直接进入 HTTP 与协议层确认。

## 场景三：masscan + AgentScan

masscan 负责大范围端口发现，AgentScan 负责 AI 协议识别：

```bash
masscan 10.0.0.0/8 -p 80,443,8000,8080,11434 --rate 100000 -oL open_ports.txt
awk '/open/ {print $4 ":" $3}' open_ports.txt > targets.txt
agentscan scan -f targets.txt --skip-port-scan --timeout 3000 -o results.json
```

这适合互联网测绘、企业大网段资产发现和研究数据集清洗。

## 场景四：只看 LLM API

```bash
agentscan llm -f targets.txt --skip-port-scan --format json \
  | jq '.results[] | select(.auth_status=="open") | {url, framework, model_count}'
```

LLM 模块只做只读 HTTP 探测，不触发推理，也不会拉取模型。

## 场景五：自动化巡检

```bash
agentscan scan -f critical-ranges.txt --format json -o daily.json
```

可以把 JSON 接入定时任务、SIEM 或资产平台，关注：

- 新增 no-auth MCP。
- 新增 open LLM API。
- A2A JSON-RPC 无认证可达。
- Agent Card 暴露私网地址。
- 高危工具名或 skill 名。

## 输出文件

使用 `scan -o results.json` 时会生成：

```text
results.json        # MCP JSON
results_a2a.json    # A2A JSON
results_llm.json    # LLM JSON
agentscan-report-*/report.html
agentscan-report-*/report_en.html
agentscan-report-*/summary.txt
```

JSON 顶层结构统一是：

```json
{
  "version": "1.0",
  "summary": {},
  "results": []
}
```

HTML 报告适合人工查看，JSON 适合自动化处理。

## 自定义字典

`--dict-dir` 支持这些文件：

```text
mcp_ports.txt
mcp_paths.txt
mcp_paths_sse_legacy.txt
mcp_paths_auth.txt
a2a_ports.txt
a2a_paths.txt
llm_ports.txt
https_ports.txt
http_server_hints.txt
```

文件每行一个值，空行和 `#` 注释会被忽略。

示例：

```bash
agentscan scan -f targets.txt --dict-dir ./dicts-private
```
