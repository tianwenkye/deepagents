# 状态流转与管理分析

## 1. 概述

Deep Agents 的状态管理建立在 LangGraph 的 StateGraph 之上。每个中间件通过 `state_schema` 扩展全局状态，LangGraph 的 reducer 机制确保状态更新的正确合并。状态流转贯穿 Agent 的整个生命周期。

## 2. 全局状态结构

```mermaid
classDiagram
    class AgentState {
        +messages: list~AnyMessage~
        +todos: list~Todo~
        +structured_response: ResponseT
    }

    class FilesystemState {
        +files: dict~str, FileData~
    }

    class SummarizationState {
        +_summarization_event: SummarizationEvent
    }

    class SkillsState {
        +skills_metadata: list~SkillMetadata~
    }

    class MemoryState {
        +memory_contents: dict~str, str~
    }

    class AsyncSubAgentState {
        +async_tasks: dict~str, AsyncTask~
    }

    AgentState <|-- FilesystemState
    AgentState <|-- SummarizationState
    AgentState <|-- SkillsState
    AgentState <|-- MemoryState
    AgentState <|-- AsyncSubAgentState
```

### 状态字段分类

| 字段 | 中间件 | Reducer | 传播性 |
|------|--------|---------|--------|
| `messages` | LangGraph 内置 | 消息追加 | 跨节点传播 |
| `files` | FilesystemMiddleware | `_file_data_reducer`（支持删除） | 跨节点传播 |
| `todos` | TodoListMiddleware | 覆盖 | 跨节点传播 |
| `_summarization_event` | SummarizationMiddleware | 覆盖（PrivateStateAttr） | 不传播到父代理 |
| `skills_metadata` | SkillsMiddleware | 覆盖（PrivateStateAttr） | 不传播到父代理 |
| `memory_contents` | MemoryMiddleware | 覆盖（PrivateStateAttr） | 不传播到父代理 |
| `async_tasks` | AsyncSubAgentMiddleware | `_tasks_reducer`（合并） | 跨节点传播 |

## 3. 状态流转全景

```mermaid
flowchart TD
    subgraph "Agent 启动阶段"
        INIT["LangGraph 初始化 State"]
        INIT --> BA1["MemoryMiddleware.before_agent()<br/>加载 AGENTS.md → memory_contents"]
        INIT --> BA2["SkillsMiddleware.before_agent()<br/>扫描技能目录 → skills_metadata"]
    end

    subgraph "LLM 调用循环"
        WMC["Middleware.wrap_model_call()"]
        WMC --> WMC1["Memory → 注入 memory 到 System Prompt"]
        WMC --> WMC2["Skills → 注入 skills 列表到 System Prompt"]
        WMC --> WMC3["Filesystem → 注入工具描述 + 过滤工具"]
        WMC --> WMC4["SubAgent → 注入 task 工具描述"]
        WMC --> WMC5["Summarization → 截断/摘要消息"]
        WMC --> WMC6["Patch → 修复悬空工具调用"]

        WMC1 & WMC2 & WMC3 & WMC4 & WMC5 & WMC6 --> LLM["LLM 调用"]
        LLM --> RESP["ModelResponse"]
    end

    subgraph "工具执行阶段"
        RESP --> TOOL["工具调用执行"]
        TOOL --> T1["文件操作 → files 更新"]
        TOOL --> T2["task 工具 → 子代理 state 隔离"]
        TOOL --> T3["compact 工具 → _summarization_event 更新"]
        TOOL --> T4["async 工具 → async_tasks 更新"]
    end

    T1 & T2 & T3 & T4 --> CHECKPOINT["LangGraph Checkpoint<br/>(持久化状态快照)"]
    CHECKPOINT --> WMC
```

## 4. 关键 Reducer 机制

### 4.1 files 字段 — 支持删除的字典合并

```python
def _file_data_reducer(left, right):
    """合并文件更新，支持通过 None 值删除"""
    if left is None:
        return {k: v for k, v in right.items() if v is not None}
    result = {**left}
    for key, value in right.items():
        if value is None:
            result.pop(key, None)  # None 表示删除
        else:
            result[key] = value     # 覆盖/新增
    return result
```

```mermaid
graph LR
    subgraph "现有 files"
        A["/file1.txt: 'hello'"]
        B["/file2.txt: 'world'"]
    end

    subgraph "更新"
        C["/file2.txt: None (删除)"]
        D["/file3.txt: 'new'"]
    end

    subgraph "合并结果"
        E["/file1.txt: 'hello'"]
        F["/file3.txt: 'new'"]
    end

    A --> E
    B -->|"被删除"| X[""]
    C --> X
    D --> F
```

### 4.2 async_tasks 字段 — 任务合并

```python
def _tasks_reducer(existing, update):
    merged = dict(existing or {})
    merged.update(update)
    return merged
```

### 4.3 PrivateStateAttr — 私有字段

使用 `PrivateStateAttr` 注解的字段**不会传播到父代理**：

```python
class SkillsState(AgentState):
    skills_metadata: NotRequired[Annotated[list[SkillMetadata], PrivateStateAttr]]

class MemoryState(AgentState):
    memory_contents: NotRequired[Annotated[dict[str, str], PrivateStateAttr]]
```

## 5. 子代理间的状态流转

### 5.1 主代理 → 子代理（调用时）

```python
# 过滤排除字段
_EXCLUDED_STATE_KEYS = {
    "messages", "todos", "structured_response",
    "skills_metadata", "memory_contents"
}

subagent_state = {
    k: v for k, v in runtime.state.items()
    if k not in _EXCLUDED_STATE_KEYS
}
subagent_state["messages"] = [HumanMessage(content=description)]
```

### 5.2 子代理 → 主代理（返回时）

```python
def _return_command_with_state_update(result, tool_call_id):
    # 仅保留非排除字段 + 最后一条消息
    state_update = {
        k: v for k, v in result.items()
        if k not in _EXCLUDED_STATE_KEYS
    }
    message_text = result["messages"][-1].text.rstrip()
    return Command(update={
        **state_update,
        "messages": [ToolMessage(message_text, tool_call_id=tool_call_id)],
    })
```

```mermaid
sequenceDiagram
    participant Main as 主 Agent State
    participant Tool as task 工具
    participant Sub as 子 Agent State

    Note over Main: {messages, files, todos, skills_metadata, ...}

    Main->>Tool: task(description, subagent_type)
    Tool->>Tool: 过滤排除字段
    Tool->>Sub: {files, ...其他共享字段, messages: [HumanMessage]}

    Note over Sub: 子代理独立执行<br/>可能修改 files 等

    Sub-->>Tool: {messages: [...], files: {...}, ...}
    Tool->>Tool: 提取最后消息 + 过滤排除字段
    Tool-->>Main: Command(update={messages: [ToolMessage], files: {...}})
```

## 6. 状态持久化

```mermaid
graph TB
    subgraph "持久化机制"
        CP["LangGraph Checkpointer"]
        ST["LangGraph Store"]
    end

    subgraph "状态类型"
        S1["Agent State<br/>(messages, files, todos, ...)"]
        S2["跨线程持久数据<br/>(StoreBackend)"]
    end

    S1 -->|"每个 superstep 结束<br/>自动快照"| CP
    S2 -->|"主动读写"| ST

    subgraph "恢复场景"
        R1["用户中断后恢复"]
        R2["Human-in-the-loop 审批后恢复"]
        R3["跨 turn 恢复对话"]
    end

    CP --> R1 & R2 & R3
```

**Checkpointer** 保存每个 superstep 结束时的完整状态快照，支持：
- 对话中断后恢复
- Human-in-the-loop 审批后继续
- 时间旅行调试

## 7. Backend 层的状态读写

StateBackend 通过 LangGraph 的内部机制直接读写状态：

```python
class StateBackend(BackendProtocol):
    def _read_files(self) -> dict[str, Any]:
        config = self._get_config()
        read = config["configurable"][CONFIG_KEY_READ]
        return read("files", fresh=False) or {}

    def _send_files_update(self, update: dict[str, Any]) -> None:
        config = self._get_config()
        send = config["configurable"][CONFIG_KEY_SEND]
        send([("files", update)])
```

```mermaid
sequenceDiagram
    participant Tool as 文件工具
    participant SB as StateBackend
    participant Pregel as LangGraph Pregel
    participant State as Agent State

    Tool->>SB: write("/hello.txt", "world")
    SB->>Pregel: _read_files() via CONFIG_KEY_READ
    Pregel->>State: 读取 files channel
    State-->>SB: current files dict
    SB->>SB: 检查文件是否已存在
    SB->>Pregel: _send_files_update() via CONFIG_KEY_SEND
    Pregel->>State: 写入 files channel (排队)
    Note over Pregel: 写入在下一个 superstep 边界生效
    SB-->>Tool: WriteResult(path="/hello.txt")
```
