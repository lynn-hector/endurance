# Endurance Agent 架构设计文档

## 1. 背景与目标

Endurance（永恒者号）是一个以 Go 语言开发的 Agent 项目，灵感来自电影《星际穿越》中人类为探索未来而建造的飞船。项目目标是构建一个可扩展、可协作、具备持久记忆的 Agent 系统，支持通过 WebSocket 与 Web 前端进行会话交互，并能自主完成复杂任务。

核心目标：
- 提供稳定的会话与任务完成能力
- 支持 Tool 与 Skill 的可扩展机制
- 支持 Subagent 的并行与分解调度
- 具备上下文管理与长期记忆能力
- 支持 Hook 机制以满足自定义需求

## 2. 关键决策

| 维度 | 决策 |
|------|------|
| LLM 协议 | OpenAI 兼容协议，原生 Function Calling（`tools` / `tool_calls`），Provider 层可抽象切换 |
| Tool | Go 代码内置，编译时注册到 ToolRegistry |
| Skill | 热插拔/动态加载，可组合多个 Tool，面向任务的高层抽象 |
| Memory | 本地 SQLite 持久化 + 异步同步到 Redis / 数据库 |
| Subagent | 并行工具调用 + 任务分解，由框架 DAG 调度与 LLM 动态编排结合 |
| Hook | LLM 请求前后、Tool 调用前后、WebSocket 消息收发前后、Session 生命周期事件全部支持 |
| 传输层 | 纯 WebSocket |
| 认证 | 第一版暂不实现 |

## 3. 整体架构与模块边界

采用 **混合核心 + 插件（Hybrid Core）** 架构：内核负责主循环和状态机，外围能力通过注册表和 Hook 点扩展。

```
┌─────────────────────────────────────────────────────────────┐
│  WebSocket Transport Layer (transport/ws)                   │
│  - 负责 HTTP upgrade、连接保活、心跳、断线重连协商            │
│  - 将字节流解析为 Protocol Message，反序列化后传给 Router   │
├─────────────────────────────────────────────────────────────┤
│  Protocol Layer (protocol)                                  │
│  - 定义 Endurance 自己的 JSON 消息协议                      │
│  - 帧类型：chat/request, chat/stream, event, error, ping    │
├─────────────────────────────────────────────────────────────┤
│  Router / Session Gateway (gateway)                         │
│  - 根据 session_id 路由消息到对应的 Agent 实例              │
│  - 负责新 Session 的创建与旧 Session 的恢复                 │
├─────────────────────────────────────────────────────────────┤
│  Agent Core (agent/core)                                    │
│  - 核心状态机 + 主 loop                                     │
│  - 显式依赖：Provider, ContextMgr, ToolRegistry, HookEmitter│
├─────────────────────────────────────────────────────────────┤
│  Subagent Scheduler (agent/subagent)                        │
│  - 管理子 agent 生命周期                                    │
│  - 提供两种模式：parallel_tool（并行工具）                  │
│                 task_decomposition（任务分解 DAG）          │
├─────────────────────────────────────────────────────────────┤
│  Tool Registry (agent/tool)                                 │
│  - 内置 Tool 的编译时注册表                                 │
│  - Tool 是 Go 函数，有明确的输入/输出 struct                  │
├─────────────────────────────────────────────────────────────┤
│  Skill Registry (agent/skill)                               │
│  - Skill 的动态加载/卸载                                    │
│  - Skill 声明自己依赖的 Tool 列表和参数 schema                │
├─────────────────────────────────────────────────────────────┤
│  Context Manager (agent/context)                            │
│  - 维护当前 turn 的 messages 数组                           │
│  - token 预算、滑动窗口截断、系统 prompt 注入                 │
├─────────────────────────────────────────────────────────────┤
│  Memory Layer (agent/memory)                                │
│  - 本地持久化（SQLite）                                     │
│  - 异步写 Redis / DB 的队列和同步器                         │
│  - 摘要生成（可选，MVP 可先只做原始消息存储）                 │
├─────────────────────────────────────────────────────────────┤
│  LLM Provider (llm/provider)                                │
│  - OpenAI 兼容接口                                          │
│  - 支持 tools 的原生 function calling                       │
└─────────────────────────────────────────────────────────────┘
```

**模块间依赖原则**：
- 内核层（Agent Core、Context Manager、Provider）只能互相通过 `interface` 调用。
- 外层（Transport、Gateway）依赖内核，但内核不感知外层。
- HookEmitter 作为横切关注点，被内核显式调用，但具体 Hook 逻辑由外部注册。

## 4. 协议层设计（Protocol）

WebSocket 帧采用统一的 JSON 信封结构：

```json
{
  "type": "chat/request",
  "id": "uuid-v4",
  "session_id": "sess-xxxxx",
  "payload": { ... }
}
```

### 4.1 帧类型定义

| 帧类型 | 方向 | 说明 |
|--------|------|------|
| `chat/request` | Client → Server | 用户发送的新消息 |
| `chat/stream` | Server → Client | LLM 的增量文本或 tool_call 片段 |
| `chat/delta` | Server → Client | 打字机式流式文本片段 |
| `event/tool_start` | Server → Client | 某个 tool 开始执行 |
| `event/tool_end` | Server → Client | tool 执行完成，携带结果摘要 |
| `event/subagent_spawn` | Server → Client | 子 agent 被创建 |
| `event/subagent_done` | Server → Client | 子 agent 结束，返回结果 |
| `error` | Server → Client | 运行时错误 |
| `ping` / `pong` | 双向 | 心跳 |

### 4.2 关键约定

- 所有 Server → Client 的消息都必须带 `id`（可以是 request 的 id 或新生成的 correlation id），方便前端做请求-响应关联。
- `chat/stream` 携带 `done: true` 标记一次 LLM 调用的结束。
- `error` 帧带 `code` 和 `message`，`code` 采用字符串形式（如 `tool_timeout`、`provider_error`）。

## 5. Core Loop 与状态机

### 5.1 主循环流程

Agent Core 维护一个状态机，单次用户消息的处理流程如下：

```
1. 接收到 chat/request
2. 将用户消息写入 Context Manager
3. 调用 Memory Layer 加载相关历史/摘要（如有）
4. 组装 messages + tools 列表
5. Hook: OnLLMRequest
6. 请求 Provider
7. Hook: OnLLMResponse
8. 解析 LLM 响应：
   a. 若为纯文本 → 直接返回给前端，结束本轮
   b. 若为 tool_calls → 继续步骤 9
9. 对 tool_calls 进行路由：
   - 普通 Tool → ToolRegistry 同步/异步执行
   - Subagent 相关 Tool → Subagent Scheduler 处理
10. 收集所有 tool_results
11. 将 assistant message + tool_results 写回 Context Manager
12. 循环回到步骤 4（再次请求 LLM）
13. 直到 LLM 返回纯文本或达到最大循环次数/ token 上限
```

### 5.2 状态机

Agent 实例在其生命周期内有以下状态：

- `idle`：待机，等待用户输入
- `thinking`：已向 LLM 发起请求，等待响应
- `acting`：正在执行 tool_calls
- `streaming`：正在向前端发送流式内容
- `paused`：被外部中断（如用户 cancel）
- `error`：遇到不可恢复错误

状态转换由 Core 内部严格管理，状态变化时触发 Session Hook。

### 5.3 循环保护

- **最大 turn 数**：默认 15 次 LLM 调用，超过则自动终止并返回错误提示
- **Token 预算上限**：Context Manager 负责监控，超过时触发截断或报错
- **超时控制**：单次 LLM 调用最大 60s，单个 Tool 最大 30s（可配置）

## 6. Tool 与 Skill 机制

### 6.1 Tool

Tool 是 Agent 最原子的能力单元，以 Go 代码内置。

```go
type Tool interface {
    Name() string
    Description() string
    Schema() json.RawMessage // JSONSchema for arguments
    Execute(ctx context.Context, input json.RawMessage) (Result, error)
}
```

所有内置 Tool 在 `init()` 或主函数启动时注册到全局 `ToolRegistry`。

示例内置 Tool：
- `read_file`：读取本地文件
- `write_file`：写入本地文件
- `bash`：执行 shell 命令
- `grep`：代码搜索
- `http_get` / `http_post`：发起 HTTP 请求

### 6.2 Skill

Skill 是面向任务的高层能力，可以动态加载/卸载，运行时组合多个 Tool。

```go
type Skill interface {
    Name() string
    Description() string
    RequiredTools() []string
    Execute(ctx context.Context, task SkillTask, toolCaller ToolCaller) (SkillResult, error)
}
```

Skill 的加载方式（MVP 阶段可先选其一）：
- **配置式**：通过 JSON/YAML 定义 Skill 的输入输出 schema 和内部 Tool 调用链（类似工作流）。
- **代码式**：加载 Go plugin（或 WASM，复杂度高，后期再考虑）。

**推荐 MVP 阶段采用配置式**：Skill 的 schema 定义在 YAML 中，Agent 加载后将其封装成一个伪 Tool 注入到 LLM 的 `tools` 列表中。这样 LLM 无需感知 Skill 和 Tool 的区别，统一走 function calling。

Skill 的 YAML 示例：

```yaml
name: web_search_and_summarize
description: 搜索网络并总结结果
required_tools: [http_get, write_file]
workflow:
  - tool: http_get
    input:
      url: "{{task.url}}"
  - tool: write_file
    input:
      path: "/tmp/summary.txt"
      content: "{{step_0.result}}"
```

MVP 阶段可以不支持复杂模板引擎，只做简单的顺序调用链。

## 7. Subagent 调度

Subagent 由 `SubagentScheduler` 统一管理，提供两种模式：

### 7.1 Parallel Tool 模式

当 LLM 在一次响应中返回多个独立的 `tool_calls` 时，Subagent Scheduler 判断其依赖关系（LLM 不会显式声明依赖，所以默认认为同批次 tool_calls 是独立的，除非参数存在明显的数据依赖），并将其分发到多个 goroutine 中并行执行。

执行完成后，统一收集结果返回给 Core Loop。

### 7.2 Task Decomposition 模式

当 LLM 返回一个特殊的内置 tool call（如 `create_subagent`）时，触发任务分解：

```json
{
  "name": "create_subagent",
  "arguments": {
    "goal": "分析项目中的 API 路由",
    "required_tools": ["read_file", "grep"],
    "parent_context_summary": "用户要求重构路由代码"
  }
}
```

SubagentScheduler 的行为：
1. 创建一个新的 `Agent` 实例（或复用池中的实例）
2. 为其分配独立的 `ContextManager`，但设置 `parent_session_id`
3. 将目标传给子 agent，启动其独立的 Core Loop
4. 子 agent 执行完毕后，结果返回给父 agent
5. 如果多个子 agent 之间存在依赖关系，由 Scheduler 维护 DAG 并依次调度

**父子关系**：子 agent 可以访问父 agent 的只读 context 摘要，但不能直接修改父 agent 的内存。这样可以实现安全隔离。

## 8. Context、Session 与 Memory

### 8.1 Session

Session 是用户与 Agent 一次完整对话的持久化单元。

```go
type Session struct {
    ID            string
    CreatedAt     time.Time
    UpdatedAt     time.Time
    Metadata      map[string]any
    ParentID      string // 若有 subagent，则指向父 session
}
```

`SessionGateway` 负责：
- WebSocket 连接建立时，根据 `session_id` 查找或创建 Session
- 当连接断开时，保持 Agent 实例存活一段时间（如 5 分钟），允许重连恢复
- 超时后回收 Agent 实例，但 Session 数据已持久化，下次连接可冷恢复

### 8.2 Context Manager

Context Manager 是 agent 运行时的"工作区"，负责维护当前 turn 的完整 messages 数组。

```go
type ContextManager interface {
    AddMessage(msg Message) error
    GetMessages() []Message
    Trim(maxTokens int) error
    GenerateToolList() []ToolDefinition // 根据当前加载的 Skill 动态生成 tools 列表
}
```

Context Manager 的关键职责：
- **滑动窗口截断**：当 messages 总 token 数接近模型上限时，按 FIFO 或重要性策略移除旧消息
- **系统 Prompt 注入**：在每次组装请求时，注入当前系统级指令（如时间、环境变量、可用 Skill 列表）
- **Tool 列表生成**：根据当前已加载的 Skill 和基础 Tool，组合成最终传给 LLM 的 `tools` 字段

### 8.3 Memory Layer

Memory Layer 负责跨 turn 甚至跨 session 的历史信息持久化。

```go
type MemoryStore interface {
    SaveTurn(sessionID string, turn TurnRecord) error
    LoadHistory(sessionID string, limit int) ([]TurnRecord, error)
    SearchLongTerm(query string, topK int) ([]MemoryChunk, error) // 可选，MVP 可空实现
}
```

**三层存储策略**：
1. **内存缓存**：当前 session 的最近 N 条消息直接保留在 Context Manager 中，最快速度访问
2. **本地 SQLite**：所有消息按 session 表归档，进程重启后可恢复
3. **异步远端同步**：一个后台 worker 周期性地（或事件驱动地）将 SQLite 中的增量数据同步到 Redis 和远端数据库

**同步逻辑**：为避免阻塞主 loop，同步操作必须异步化。可采用一个轻量级的本地消息队列（如 Go channel + 后台 goroutine）积批同步。

MVP 阶段可以先只实现 SQLite 持久化，Redis/DB 同步用单个后台 goroutine 做简单轮询。

## 9. Hook 机制

Hook 是系统的横切扩展点，允许第三方代码在关键生命周期节点插入自定义逻辑（日志、审计、限流、修改 payload 等）。

### 9.1 Hook 点定义

```go
type HookEmitter interface {
    OnLLMRequest(ctx context.Context, req *LLMRequest) error
    OnLLMResponse(ctx context.Context, resp *LLMResponse) error
    OnToolCallBefore(ctx context.Context, call *ToolCall) error
    OnToolCallAfter(ctx context.Context, call *ToolCall, result *ToolResult) error
    OnWSMessageIn(ctx context.Context, msg *ProtocolMessage) error
    OnWSMessageOut(ctx context.Context, msg *ProtocolMessage) error
    OnSessionStateChange(ctx context.Context, sessionID string, from, to AgentState)
}
```

### 9.2 注册与执行方式

- Hook 通过 `HookRegistry` 注册，支持注册多个 Hook，按注册顺序链式执行。
- 如果某个 Hook 返回 error，根据场景决定是中断流程还是继续（例如 `OnLLMRequest` 返回 error 应该中断请求，`OnSessionStateChange` 返回 error 只记录日志）。
- 为了不影响主 loop 性能，耗时 Hook 建议内部自行 goroutine 化，但同步 Hook 可以阻塞以修改参数。

### 9.3 使用场景示例

- **日志 Hook**：记录每次 LLM 请求的耗时和 token 消耗
- **审计 Hook**：记录哪些 Tool 被调用、调用了什么参数
- **限流 Hook**：在 `OnLLMRequest` 中检查配额，超限则返回 error
- **消息改写 Hook**：在 `OnWSMessageOut` 中统一给消息加上服务端时间戳

## 10. 数据流与错误处理

### 10.1 典型数据流（单次用户请求）

```
Browser (WebSocket)
  │ chat/request
  ▼
Transport Server ──► Gateway ──► Agent Core
                                      │
                                      ▼
                              Context Manager (加载历史/组装 messages)
                                      │
                                      ▼
                              Hook: OnLLMRequest
                                      │
                                      ▼
                              LLM Provider
                                      │
                                      ▼
                              Hook: OnLLMResponse
                                      │
                              ┌───────┴───────┐
                              ▼               ▼
                          纯文本响应      tool_calls
                              │               │
                              ▼               ▼
                         流式返回前端    ToolRegistry / Subagent Scheduler
                                              │
                                              ▼
                                         Hook: OnToolCallBefore/After
                                              │
                                              ▼
                                         收集结果后重新进入 LLM 循环
```

### 10.2 错误分级处理

| 错误级别 | 说明 | 处理策略 |
|----------|------|----------|
| `recoverable` | 如 LLM 临时超时、Tool 网络抖动 | 自动重试 1-2 次，若仍失败则降级返回错误摘要 |
| `user_visible` | 如 Tool 参数不合法、文件不存在 | 不中断 loop，将错误信息作为 tool_result 返回给 LLM，让它自行修正 |
| `fatal` | 如 Provider API Key 无效、内部状态损坏 | 终止当前 loop，返回 `error` 帧给前端，并记录日志 |
| `session_fatal` | 如 Session 数据丢失 | 断开 WebSocket，告知前端需要重新创建 Session |

**错误传播原则**：
- Tool 执行中的错误尽量通过 `ToolResult` 的 `Error` 字段返回，不要直接 panic 或终止 loop
- LLM provider 层面的错误（如 429、500）由 Core Loop 内的重试逻辑处理
- 所有 fatal 级别错误都需要通过 `error` 帧推送到前端，并带上可读的 `code`

## 11. MVP 范围建议

为了尽快验证核心循环，MVP 阶段建议：

1. **优先实现模块**：WebSocket Transport → Protocol → Agent Core → Context Manager → LLM Provider（OpenAI）→ 2-3 个基础 Tool（如 `bash`、`read_file`）
2. **延后实现**：Skill 的高级工作流模板、Memory 的向量检索、Subagent 的 DAG 分解、复杂 Hook
3. **Skill 的妥协方案**：MVP 中 Skill 可以先作为简单的配置式 wrapper，把多个 Tool 封装成一个新的 schema 供 LLM 调用
4. **Memory 的妥协方案**：MVP 中只做 SQLite 的 messages 持久化，Redis/DB 同步用后台 goroutine 轮询

---

*本文档为 Endurance Agent 的架构设计初稿，后续将根据实现反馈持续迭代。*
