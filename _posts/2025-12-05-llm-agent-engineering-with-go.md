---
layout: post
title: LLM Agent 工程实践：用 Go 实现 ReAct 模式的工具调用 Agent
categories: [Go, AI]
description: 从零实现一个支持工具调用的 LLM Agent，包括 ReAct 循环、工具注册、多轮对话和错误恢复。
keywords: LLM, Agent, ReAct, Tool Calling, Golang
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

上一篇写了 RAG 系统，RAG 解决了"基于知识回答问题"，但很多场景需要 Agent 真正去**执行操作**——查数据库、调接口、读文件。这篇记录如何用 Go 实现一个生产级别的 LLM Agent。

## 什么是 ReAct

ReAct 的核心循环很简单：**Reason（思考） → Act（行动） → Observe（观察）**，不断循环直到任务完成。

```
用户：帮我查一下订单 ORD-2025001 的物流状态

思考：用户想查询订单的物流信息，我需要先查订单详情，再查物流状态
行动：调用 get_order(order_id="ORD-2025001")
观察：{order_id: "ORD-2025001", status: "shipped", tracking_no: "SF1234567"}

思考：拿到了运单号，需要查物流详情
行动：调用 get_logistics(tracking_no="SF1234567")
观察：{status: "in_transit", location: "北京分拨中心", update_time: "2025-11-20 14:30"}

思考：已经拿到了所有信息，可以回答用户了
最终回答：订单 ORD-2025001 已发货，运单号 SF1234567，当前在北京分拨中心，更新时间 2025-11-20 14:30。
```

## 核心抽象

### Tool 接口

```go
type Tool struct {
    Name        string          `json:"name"`
    Description string          `json:"description"`
    Parameters  json.RawMessage `json:"parameters"` // JSON Schema
    Execute     ToolFunc        `json:"-"`
}

type ToolFunc func(ctx context.Context, params json.RawMessage) (string, error)
```

每个 Tool 有名字、描述、参数 schema（给 LLM 看的）和执行函数。参数用 `json.RawMessage` 传递，各工具内部自己反序列化。

### Agent 结构

```go
type Agent struct {
    llmClient    LLMClient
    tools        map[string]Tool
    systemPrompt string
    maxSteps     int // 最大循环次数，防止死循环
}

type LLMClient interface {
    Chat(ctx context.Context, messages []Message, tools []Tool) (*Response, error)
}

type Response struct {
    Content   string      // 文本回复
    ToolCalls []ToolCall  // 工具调用请求
}

type ToolCall struct {
    ID       string          `json:"id"`
    Name     string          `json:"name"`
    Arguments json.RawMessage `json:"arguments"`
}
```

## Agent 主循环

核心代码其实不长：

```go
func (a *Agent) Run(ctx context.Context, userMessage string) (string, error) {
    messages := []Message{
        {Role: "system", Content: a.systemPrompt},
        {Role: "user", Content: userMessage},
    }

    for step := 0; step < a.maxSteps; step++ {
        resp, err := a.llmClient.Chat(ctx, messages, a.toolList())
        if err != nil {
            return "", fmt.Errorf("llm chat at step %d: %w", step, err)
        }

        // 如果 LLM 没有调用工具，说明任务完成
        if len(resp.ToolCalls) == 0 {
            return resp.Content, nil
        }

        // 把 LLM 的回复（包含工具调用）加入对话历史
        messages = append(messages, Message{
            Role:      "assistant",
            Content:   resp.Content,
            ToolCalls: resp.ToolCalls,
        })

        // 执行每个工具调用
        for _, tc := range resp.ToolCalls {
            result, err := a.executeTool(ctx, tc)
            if err != nil {
                // 工具执行失败，把错误信息告诉 LLM，让它决定怎么处理
                result = fmt.Sprintf("工具执行失败: %v", err)
            }

            messages = append(messages, Message{
                Role:       "tool",
                Content:    result,
                ToolCallID: tc.ID,
            })
        }
    }

    return "", fmt.Errorf("agent reached max steps (%d) without completing", a.maxSteps)
}

func (a *Agent) executeTool(ctx context.Context, tc ToolCall) (string, error) {
    tool, ok := a.tools[tc.Name]
    if !ok {
        return "", fmt.Errorf("unknown tool: %s", tc.Name)
    }

    // 超时控制：单个工具调用最多 30 秒
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    return tool.Execute(ctx, tc.Arguments)
}
```

这段代码的关键设计：
1. **错误不中断循环**：工具失败时把错误作为观察结果传给 LLM，让它决定是重试还是换个方案
2. **最大步数限制**：防止 LLM 陷入死循环
3. **超时控制**：每个工具调用都有独立超时

## 工具注册

```go
func NewOrderAgent(llm LLMClient, orderSvc OrderService, logisticsSvc LogisticsService) *Agent {
    agent := &Agent{
        llmClient: llm,
        tools:     make(map[string]Tool),
        maxSteps:  10,
        systemPrompt: `你是一个订单助手。你可以查询订单信息和物流状态。
请根据用户的需求调用合适的工具获取信息，然后给出清晰的回答。
如果工具调用失败，可以尝试换一种方式或者告知用户。`,
    }

    agent.RegisterTool(Tool{
        Name:        "get_order",
        Description: "根据订单号查询订单详情，包括商品、金额、状态等",
        Parameters: json.RawMessage(`{
            "type": "object",
            "properties": {
                "order_id": {"type": "string", "description": "订单号"}
            },
            "required": ["order_id"]
        }`),
        Execute: func(ctx context.Context, params json.RawMessage) (string, error) {
            var p struct {
                OrderID string `json:"order_id"`
            }
            if err := json.Unmarshal(params, &p); err != nil {
                return "", fmt.Errorf("invalid params: %w", err)
            }
            order, err := orderSvc.GetOrder(ctx, p.OrderID)
            if err != nil {
                return "", err
            }
            result, _ := json.Marshal(order)
            return string(result), nil
        },
    })

    agent.RegisterTool(Tool{
        Name:        "get_logistics",
        Description: "根据运单号查询物流轨迹",
        Parameters: json.RawMessage(`{
            "type": "object",
            "properties": {
                "tracking_no": {"type": "string", "description": "物流运单号"}
            },
            "required": ["tracking_no"]
        }`),
        Execute: func(ctx context.Context, params json.RawMessage) (string, error) {
            var p struct {
                TrackingNo string `json:"tracking_no"`
            }
            if err := json.Unmarshal(params, &p); err != nil {
                return "", fmt.Errorf("invalid params: %w", err)
            }
            info, err := logisticsSvc.GetLogistics(ctx, p.TrackingNo)
            if err != nil {
                return "", err
            }
            result, _ := json.Marshal(info)
            return string(result), nil
        },
    })

    return agent
}

func (a *Agent) RegisterTool(t Tool) {
    a.tools[t.Name] = t
}
```

## 踩过的坑

### 1. LLM 编造工具参数

LLM 有时候会瞎编参数，比如用户说"查一下我的订单"，LLM 直接编一个 `order_id: "12345"` 出来。

**解决方案：** 在 system prompt 里明确要求"如果缺少必要信息，向用户追问，不要猜测"。

```
重要：如果用户没有提供必要的参数（如订单号），你必须向用户询问，绝对不能自己编造。
```

### 2. 工具返回太多数据

查数据库返回了一个几百行的大 JSON，直接塞进对话历史，context window 很快就满了。

**解决方案：** 对工具返回结果做截断和摘要：

```go
func (a *Agent) executeTool(ctx context.Context, tc ToolCall) (string, error) {
    result, err := a.tools[tc.Name].Execute(ctx, tc.Arguments)
    if err != nil {
        return "", err
    }

    // 如果结果太长，截断并提示
    if len(result) > 4000 {
        result = result[:4000] + "\n...(结果已截断，共 " + strconv.Itoa(len(result)) + " 字符)"
    }
    return result, nil
}
```

更好的做法是让工具本身支持分页和字段选择，但快速迭代阶段先截断兜底。

### 3. 死循环：LLM 反复调同一个工具

遇到过 LLM 反复调查询接口但每次都用不同的参数格式，导致一直失败一直重试。

**解决方案：** 加步数限制 + 重复检测：

```go
// 在主循环中加：
if step > 0 && isRepeatedToolCall(messages, tc) {
    messages = append(messages, Message{
        Role:       "tool",
        Content:    "你已经重复调用了相同的工具。请换一种方式或直接回答用户。",
        ToolCallID: tc.ID,
    })
    continue
}
```

### 4. 并发工具调用

有些 LLM 会在一次回复中返回多个 tool call（比如同时查订单和查用户信息）。串行执行太慢，并行执行要注意错误处理：

```go
// 并发执行工具调用
g, toolCtx := errgroup.WithContext(ctx)
toolResults := make([]Message, len(resp.ToolCalls))

for i, tc := range resp.ToolCalls {
    i, tc := i, tc
    g.Go(func() error {
        result, err := a.executeTool(toolCtx, tc)
        if err != nil {
            result = fmt.Sprintf("工具执行失败: %v", err)
        }
        toolResults[i] = Message{
            Role:       "tool",
            Content:    result,
            ToolCallID: tc.ID,
        }
        return nil // 工具失败不中断其他工具
    })
}

g.Wait() // 错误已经在上面处理了，这里不会返回 error
messages = append(messages, toolResults...)
```

## 对话记忆管理

多轮对话时 messages 会越来越长。简单的做法是滑动窗口：

```go
func (a *Agent) trimMessages(messages []Message) []Message {
    if len(messages) <= a.maxMessages {
        return messages
    }

    // 保留 system prompt + 最近 N 轮对话
    trimmed := []Message{messages[0]} // system prompt
    trimmed = append(trimmed, messages[len(messages)-a.maxMessages+1:]...)
    return trimmed
}
```

更高级的做法是对早期对话做摘要压缩，但对于大多数工具调用场景，滑动窗口够用了。

## 可观测性

生产环境必须有日志和 trace，否则出了问题完全没法排查：

```go
func (a *Agent) Run(ctx context.Context, userMessage string) (string, error) {
    traceID := generateTraceID()
    log := slog.With("trace_id", traceID, "user_message", userMessage)

    for step := 0; step < a.maxSteps; step++ {
        log.Info("agent step", "step", step, "message_count", len(messages))

        resp, err := a.llmClient.Chat(ctx, messages, a.toolList())
        if err != nil {
            log.Error("llm call failed", "step", step, "error", err)
            return "", err
        }

        for _, tc := range resp.ToolCalls {
            start := time.Now()
            result, err := a.executeTool(ctx, tc)
            duration := time.Since(start)

            log.Info("tool executed",
                "step", step,
                "tool", tc.Name,
                "duration_ms", duration.Milliseconds(),
                "error", err,
                "result_length", len(result),
            )
        }
    }
    // ...
}
```

每次 LLM 调用和工具执行都记录下来，线上排查问题时能还原完整的推理链路。

## 整体架构

最终我们的 Agent 服务结构：

```
HTTP/gRPC API
     ↓
  AgentRouter（根据意图路由到不同 Agent）
     ↓
  ┌─────────────────────┐
  │  OrderAgent          │ ← 工具: get_order, get_logistics, cancel_order
  │  KnowledgeAgent      │ ← 工具: rag_search (对接 RAG 系统)
  │  DataAnalysisAgent   │ ← 工具: query_db, generate_chart
  └─────────────────────┘
     ↓
  LLM Client（统一的模型调用层，支持重试、降级、负载均衡）
     ↓
  Tool Registry（工具注册中心，运行时可增删工具）
```

每个 Agent 有自己的 system prompt 和工具集，AgentRouter 根据用户意图做第一步分发。Router 本身也是一个 LLM 调用，Prompt 里列出所有 Agent 的职能描述。

## 总结

Agent 的核心逻辑不复杂——就是一个 LLM 调用 + 工具执行的循环。但工程化需要关注：

1. **错误恢复**：工具失败不能让整个 Agent 崩溃，把错误信息反馈给 LLM
2. **安全边界**：最大步数、单工具超时、结果截断
3. **可观测性**：每一步都要有日志和 trace
4. **Prompt 工程**：明确告诉 LLM 什么时候该追问、什么时候该停止

Agent 开发最花时间的不是写代码，而是调 Prompt 和处理各种 edge case。
