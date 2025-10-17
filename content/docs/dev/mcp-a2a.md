---
title: "MCP 与 A2A"
weight: 1
date: 2025-10-14
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# MCP 与 A2A

## MCP

MCP 是一个开放协议,
提供了 LLM 应用与工具交互的标准协议.

可以将 MCP 比做 USB-C 接口.
显示器 / 电源 / U 盘都可以通过 USB-C 物理接口连接电脑.
相当于 MCP 为 AI 模型连接各种数据源和工具提供了标准化的接口.

MCP 遵循一个 client-server 架构，其中：
- Hosts 是 LLM 应用（如 Claude Desktop 或 IDEs），它们发起连接
- Clients 在 host 应用中与 servers 保持 1:1 的连接
- Servers 为 clients 提供上下文、tools 和 prompts

<div align="center">
<img id="mcp-architecture_auto_svg" src="/image/dev/mcp-a2a/mcp-architecture.svg" width="70%">
    <br>
    <div style="border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">MCP 架构</div>
	<!-- <img src="" >
    msdos -->
</div>

- Host 会为每个 Server 初始化对应的 Client.
- 之后 Host 会汇总所有 Server 的能力提供给 LLM.
- LLM 根据从中选取需要调用的工具进行调用.

> [!TIP]
> MCP 的传输协议用的是 JSON-RPC.

## A2A

A2A 是 Agent 之间交互的协议.
相当于某种 Agent 级别的 RPC.

<div align="center">
<img src="/image/dev/mcp-a2a/a2a.png" width="70%">
    <br>
    <div style="border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">A2A 架构</div>
	<!-- <img src="" >
    msdos -->
</div>

A2A 的调用流程大概如下:

{{< mermaid >}}
sequenceDiagram
    participant 客户端
    participant A2A 服务器
    participant 认证服务

    rect rgb(240, 240, 240)
    Note over 客户端, A2A 服务器: 1. 发现 Agent
    客户端->>A2A 服务器: 用 GET 请求 agent card eg: (/.well-known/agent-card)
    A2A 服务器-->>客户端: 返回 Agent Card
    end

    rect rgb(240, 240, 240)
    Note over 客户端, 认证服务: 2. 认证
    客户端->>客户端: 从 Agent Card 提取 securitySchemes 属性
    alt securityScheme 是 "openIdConnect"
        客户端->>认证服务: 通过 "authorizationUrl" 和 "tokenUrl" 请求 token
        认证服务-->>客户端: 返回 JWT 令牌
    end
    end

    rect rgb(240, 240, 240)
    Note over 客户端, A2A 服务器: 3. 使用 sendMessage API 发送请求
    客户端->>客户端: 从 Agent Card 提取 "url" 参数来发送请求
    客户端->>A2A 服务器: 用 POST 请求 /sendMessage (带有 JWT 令牌)
    A2A 服务器->>A2A 服务器: 处理消息并创建对应的任务
    A2A 服务器-->>客户端: 返回任务的响应
    end

    rect rgb(240, 240, 240)
    Note over 客户端, A2A 服务器: 4. 使用 sendMessageStream API 发送请求
    客户端->>A2A 服务器: 用 POST 请求 /sendMessageStream (带有 JWT 令牌)
    A2A 服务器-->>客户端: Stream: Task (Submitted)
    A2A 服务器-->>客户端: Stream: TaskStatusUpdateEvent (Working)
    A2A 服务器-->>客户端: Stream: TaskArtifactUpdateEvent (artifact A)
    A2A 服务器-->>客户端: Stream: TaskArtifactUpdateEvent (artifact B)
    A2A 服务器-->>客户端: Stream: TaskStatusUpdateEvent (Completed)
    end
{{< /mermaid >}}

> [!TIP]
> A2A 的传输协议可以有很多种, 例如 JSON-RPC, gRPC, HTTP+JSON.
