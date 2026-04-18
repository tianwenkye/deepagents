# 多 Agent 协作模块分析

## 1. 概述

Deep Agents 支持三种类型的子代理协作：

| 类型 | 中间件 | 执行方式 | 适用场景 |
|------|--------|---------|---------|
| **同步子代理 (SubAgent)** | `SubAgentMiddleware` | 阻塞式，在当前进程内运行 | 独立的复杂任务，需要上下文隔离 |
| **预编译子代理 (CompiledSubAgent)** | `SubAgentMiddleware` | 阻塞式，使用预构建的 Runnable | 自定义图结构，非标准 Agent |
| **异步子代理 (AsyncSubAgent)** | `AsyncSubAgentMiddleware` | 非阻塞式，远程 Agent Protocol 服务器 | 长时间运行，远程部署 |

## 2. 总体架构

```mermaid
graph TB
    MAIN["主 Agent<br/>(create_deep_agent)"]
    
    subgraph "同步子代理 (via task tool)"
        direction TB
        SA1["general-purpose<br/>通用子代理<br/>(自动创建)"]
        SA2["researcher<br/>研究子代理"]
        SA3["code-reviewer<br/>代码审查子代理"]
    end

    subgraph "异步子代理 (via async tools)"
        direction TB
        AA1["remote-analyzer<br/>远程分析"]
        AA2["remote-builder<br/>远程构建"]
    end

    MAIN -->|"task tool<br/>(阻塞等待)"| SA1
    MAIN -->|"task tool"| SA2
    MAIN -->|"task tool"| SA3
    MAIN -->|"start_async_task<br/>(立即返回 task_id)"| AA1
    MAIN -->|"start_async_task"| AA2

    SA1 & SA2 & SA3 -->|"返回单条结果"| MAIN
    AA1 & AA2 -->|"check_async_task<br/>查询结果"| MAIN
```

## 3. 同步子代理 — SubAgentMiddleware

### 3.1 子代理规格定义

```python
class SubAgent(TypedDict):
    name: str           # 唯一标识
    description: str    # 描述（用于主 Agent 决策何时委派）
    system_prompt: str  # 子代理的系统提示
    tools: NotRequired[...]    # 工具集（默认继承主 Agent）
    model: NotRequired[...]    # 模型（默认继承主 Agent）
    middleware: NotRequired[...] # 中间件
    interrupt_on: NotRequired[...] # 人机交互配置
    skills: NotRequired[...]   # 技能路径
```

### 3.2 子代理构建流程

```mermaid
sequenceDiagram
    participant CDA as create_deep_agent
    participant SAM as SubAgentMiddleware
    participant CA as create_agent

    CDA->>CDA: 解析 subagents 列表
    CDA->>CDA: 分离 SubAgent / CompiledSubAgent / AsyncSubAgent

    loop 对每个 SubAgent
        CDA->>CDA: resolve_model(spec.model)
        CDA->>CDA: 构建子代理中间件栈:
        Note right of CDA: TodoListMiddleware<br/>FilesystemMiddleware<br/>SummarizationMiddleware<br/>PatchToolCallsMiddleware<br/>SkillsMiddleware (可选)<br/>AnthropicPromptCachingMiddleware<br/>HumanInTheLoopMiddleware (可选)
        CDA->>CA: create_agent(model, system_prompt, tools, middleware)
        CA-->>CDA: compiled graph (Runnable)
    end

    CDA->>CDA: 添加默认 general-purpose 子代理<br/>(如果用户未自定义)
    CDA->>SAM: SubAgentMiddleware(subagents=inline_subagents)
```

### 3.3 task 工具调用流程

```mermaid
sequenceDiagram
    participant LLM as 主 Agent LLM
    participant TT as task 工具
    participant SA as 子代理 Graph
    participant MAIN as 主 Agent State

    LLM->>TT: task(description="...", subagent_type="general-purpose")
    TT->>TT: _validate_and_prepare_state()
    Note right of TT: 过滤排除字段:<br/>messages, todos, structured_response,<br/>skills_metadata, memory_contents
    TT->>TT: 创建子代理初始状态<br/>{...filtered_state, messages: [HumanMessage(description)]}
    TT->>SA: subagent.invoke(subagent_state)
    SA->>SA: 子代理独立执行
    SA-->>TT: result dict (含 messages)
    TT->>TT: _return_command_with_state_update()
    Note right of TT: 提取最后一条消息文本<br/>过滤排除字段<br/>构建 Command(update={...})
    TT-->>MAIN: Command(update={messages: [ToolMessage(result)], ...files...})
```

### 3.4 子代理默认中间件栈

```python
# graph.py 中构建子代理中间件
subagent_middleware = [
    TodoListMiddleware(),
    FilesystemMiddleware(backend=backend),
    create_summarization_middleware(subagent_model, backend),
    PatchToolCallsMiddleware(),
]
if subagent_skills:
    subagent_middleware.append(SkillsMiddleware(backend=backend, sources=subagent_skills))
subagent_middleware.extend(spec.get("middleware", []))
subagent_middleware.append(AnthropicPromptCachingMiddleware(unsupported_model_behavior="ignore"))
```

### 3.5 状态隔离与共享

```mermaid
graph TB
    subgraph "主 Agent State"
        MS["messages"]
        MF["files"]
        MT["todos"]
        MSM["skills_metadata"]
        MM["memory_contents"]
    end

    subgraph "子代理 State (独立上下文)"
        SS["messages: [HumanMessage(task_description)]"]
        SF["files (继承自主代理)"]
        ST["todos (独立)"]
    end

    MS -.->|"排除"| SS
    MF -.->|"继承"| SF
    MT -.->|"排除"| SS
    MSM -.->|"排除"| SS
    MM -.->|"排除"| SS

    SF -->|"工具调用结果写回"| MS
```

**排除字段：** `_EXCLUDED_STATE_KEYS = {"messages", "todos", "structured_response", "skills_metadata", "memory_contents"}`

## 4. 异步子代理 — AsyncSubAgentMiddleware

### 4.1 异步子代理规格

```python
class AsyncSubAgent(TypedDict):
    name: str           # 唯一标识
    description: str    # 描述
    graph_id: str       # 远程服务器上的 Graph 名称
    url: NotRequired[str]      # Agent Protocol 服务器 URL
    headers: NotRequired[dict] # 请求头
```

### 4.2 异步子代理生命周期

```mermaid
stateDiagram-v2
    [*] --> Running: start_async_task
    Running --> Success: 执行完成
    Running --> Error: 执行出错
    Running --> Cancelled: cancel_async_task
    Running --> Running: update_async_task<br/>(中断当前 run, 创建新 run)

    Success --> [*]
    Error --> [*]
    Cancelled --> [*]

    note right of Running
        可通过 check_async_task 查询状态
        可通过 list_async_tasks 批量查询
    end note
```

### 4.3 异步工具集交互流程

```mermaid
sequenceDiagram
    participant LLM as 主 Agent LLM
    participant TOOLS as Async Tools
    participant SDK as LangGraph SDK
    participant SERVER as Remote Agent Protocol Server

    Note over LLM,SERVER: 1. 启动任务

    LLM->>TOOLS: start_async_task(description, subagent_type)
    TOOLS->>SDK: client.threads.create()
    SDK->>SERVER: POST /threads
    SERVER-->>SDK: thread_id
    TOOLS->>SDK: client.runs.create(thread_id, graph_id, input)
    SDK->>SERVER: POST /threads/{id}/runs
    SERVER-->>SDK: run_id
    TOOLS-->>LLM: "Launched async subagent. task_id: xxx"
    Note right of LLM: 主 Agent 立即返回控制

    Note over LLM,SERVER: 2. 用户主动查询

    LLM->>TOOLS: check_async_task(task_id)
    TOOLS->>SDK: client.runs.get(thread_id, run_id)
    SDK->>SERVER: GET /threads/{id}/runs/{id}
    SERVER-->>TOOLS: Run{status: "success"}
    TOOLS->>SDK: client.threads.get(thread_id)
    SDK->>SERVER: GET /threads/{id}
    SERVER-->>TOOLS: {values: {messages: [...]}}
    TOOLS-->>LLM: {status: "success", result: "..."}

    Note over LLM,SERVER: 3. 更新任务

    LLM->>TOOLS: update_async_task(task_id, message)
    TOOLS->>SDK: client.runs.create(thread_id, graph_id, input, multitask_strategy="interrupt")
    SDK->>SERVER: POST /threads/{id}/runs (interrupt existing)
    SERVER-->>TOOLS: new run_id
    TOOLS-->>LLM: "Updated async subagent. task_id: xxx"
```

### 4.4 异步任务状态追踪

```python
class AsyncTask(TypedDict):
    task_id: str          # 等同于 thread_id
    agent_name: str       # 子代理类型名称
    thread_id: str        # 远程线程 ID
    run_id: str           # 当前执行 ID
    status: str           # running / success / error / cancelled
    created_at: str       # ISO-8601
    last_checked_at: str  # 上次查询时间
    last_updated_at: str  # 上次状态变更时间
```

任务信息持久化在 Agent State 的 `async_tasks` 字段中，使用自定义 reducer 合并：

```python
def _tasks_reducer(existing, update):
    merged = dict(existing or {})
    merged.update(update)
    return merged
```

## 5. 子代理配置示例

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=[my_custom_tool],
    subagents=[
        # 同步子代理 — 自动构建
        {
            "name": "researcher",
            "description": "Deep research agent for complex topics",
            "system_prompt": "You are a research specialist...",
            "tools": [search_tool, read_tool],
        },
        # 预编译子代理 — 使用自定义图
        {
            "name": "custom-processor",
            "description": "Custom processing pipeline",
            "runnable": my_custom_graph,
        },
        # 异步子代理 — 远程部署
        {
            "name": "remote-builder",
            "description": "Remote build service",
            "graph_id": "build_agent",
            "url": "https://my-deployment.langsmith.dev",
        },
    ],
)
```
