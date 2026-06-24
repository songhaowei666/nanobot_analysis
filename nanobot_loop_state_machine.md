# nanobot AgentLoop 状态机深度解析

> 基于 `nanobot/agent/loop.py` 源码分析，聚焦 `AgentLoop` 中的事件驱动状态机实现。

---

## 1. 状态机概览

`AgentLoop` 不直接调用一次 LLM 就回复用户，而是把一次用户请求的生命周期拆成了一个**有限状态机（FSM）**。核心循环位于 `loop.py:1277`：

```python
while ctx.state is not TurnState.DONE:
    handler_name = f"_state_{ctx.state.name.lower()}"
    handler = getattr(self, handler_name, None)
    event = await handler(ctx)
    next_state = self._TRANSITIONS.get((ctx.state, event))
    ctx.state = next_state
```

### 1.1 状态定义

```python
class TurnState(Enum):
    RESTORE = auto()
    COMPACT = auto()
    COMMAND = auto()
    BUILD = auto()
    RUN = auto()
    SAVE = auto()
    RESPOND = auto()
    DONE = auto()
```

### 1.2 状态转移表

```python
_TRANSITIONS = {
    (TurnState.RESTORE, "ok"):     TurnState.COMPACT,
    (TurnState.COMPACT, "ok"):     TurnState.COMMAND,
    (TurnState.COMMAND, "dispatch"): TurnState.BUILD,
    (TurnState.COMMAND, "shortcut"): TurnState.DONE,
    (TurnState.BUILD, "ok"):       TurnState.RUN,
    (TurnState.RUN, "ok"):         TurnState.SAVE,
    (TurnState.SAVE, "ok"):        TurnState.RESPOND,
    (TurnState.RESPOND, "ok"):     TurnState.DONE,
}
```

整体流转：

```text
RESTORE → COMPACT → COMMAND ──dispatch──→ BUILD → RUN → SAVE → RESPOND → DONE
              │                              │
              └────────shortcut──────────────┘
```

---

## 2. 每个状态函数详解

### 2.1 RESTORE：入口准备与崩溃恢复

**位置**：`nanobot/agent/loop.py:1364`

**职责**：每轮处理的入口准备阶段，负责加载 session、预处理用户输入、发布回合开始事件，以及**修复上次崩溃遗留的不完整历史**。

主要工作：

1. **处理媒体附件**：如果有 `msg.media`，提取文档文本或引用非图片附件。
2. **日志记录**：`Processing message from {channel}:{sender_id}`。
3. **确保 session 存在**：`self.sessions.get_or_create(ctx.session_key)`。
4. **发布回合开始事件**：`session_turn_started` 供 WebUI/监控订阅。
5. **持久化 workspace scope**：仅对 WebSocket 渠道生效。
6. **崩溃恢复**：
   - `_restore_runtime_checkpoint()`：修复工具调用到一半中断的残局。
   - `_restore_pending_user_turn()`：修复只保存了用户消息、assistant 未回复就中断的情况。

返回 `"ok"`，进入 `COMPACT`。

---

### 2.2 COMPACT：历史压缩准备

**位置**：`nanobot/agent/loop.py:1400`

**职责**：在构造 LLM 上下文前，尝试获取已有历史摘要，控制 prompt 长度。

```python
async def _state_compact(self, ctx: TurnContext) -> str:
    ctx.session, pending = self.auto_compact.prepare_session(ctx.session_key)
    ctx.pending_summary = pending
    return "ok"
```

- 从内存热缓存或 `session.metadata["_last_summary"]` 中读取摘要。
- 真正的压缩（生成摘要）发生在后台 `check_expired` 或 `BUILD` 阶段的 token 超限时。
- 每条用户消息都会执行，但通常只是快速查一下，不会真正做压缩。

返回 `"ok"`，进入 `COMMAND`。

---

### 2.3 COMMAND：命令路由与 shortcut

**位置**：`nanobot/agent/loop.py:1405`

**职责**：检查用户输入是否是斜杠命令（如 `/new`、`/image`），决定走 shortcut 还是正常 LLM 流程。

```python
async def _state_command(self, ctx: TurnContext) -> str:
    raw = ctx.msg.content.strip()
    cmd_ctx = CommandContext(...)
    result = await self.commands.dispatch(cmd_ctx)
    if result is not None:
        ctx.outbound = result
        if raw.lower() != "/new":
            # 提前保存用户消息和命令回复到 session
            ctx.user_persisted_early = self._persist_user_message_early(...)
            ctx.session.add_message("assistant", result.content, _command=True)
            self.sessions.save(ctx.session)
        return "shortcut"
    return "dispatch"
```

分支：

- **命中 shortcut 命令**：直接生成回复，保存 session，返回 `"shortcut"` → 跳到 `DONE`。
- **未命中命令**：返回 `"dispatch"` → 进入 `BUILD`。

这是减少不必要 LLM 调用、提升响应速度的关键设计。

---

### 2.4 BUILD：LLM 上下文组装

**位置**：`nanobot/agent/loop.py:1430`

**职责**：把用户消息、历史、系统提示、工具上下文等全部组装成 LLM 能消费的 `ctx.initial_messages`。

主要工作：

1. **按 token 压缩历史**：`maybe_consolidate_by_tokens`（非临时会话）。
2. **设置工具上下文**：当前 channel、chat_id、session_key 等。
3. **启动 message tool 新回合**。
4. **获取 session 历史**：`ctx.session.get_history(...)`，限制 max_messages / max_tokens。
5. **构建初始消息**：`ctx.initial_messages = self._build_initial_messages(...)`。
6. **提前持久化用户消息**：`_persist_user_message_early()`，防止 LLM 调用中崩溃丢失输入。
7. **构建回调**：`on_progress`（进度通知）、`on_retry_wait`（重试等待通知）。

返回 `"ok"`，进入 `RUN`。

---

### 2.5 RUN：核心 LLM 执行

**位置**：`nanobot/agent/loop.py:1477`

**职责**：真正调用 LLM，执行 tool call 循环，收集完整结果。

```python
async def _state_run(self, ctx: TurnContext) -> str:
    if ctx.visible_run_started_at is None:
        ctx.visible_run_started_at = time.time()
    await self._runtime_events().run_status_changed(..., "running", ...)
    result = await self._run_agent_loop(
        ctx.initial_messages,
        on_progress=ctx.on_progress,
        on_stream=ctx.on_stream,
        on_retry_wait=ctx.on_retry_wait,
        session=ctx.session,
        pending_queue=ctx.pending_queue,
        tools=ctx.tools,
        ...
    )
    final_content, tools_used, all_msgs, stop_reason, had_injections = result
    ctx.final_content = final_content
    ctx.all_messages = all_msgs
    ctx.stop_reason = stop_reason
    ctx.had_injections = had_injections
    await turn_continuation.maybe_continue_turn(ctx)
    return "ok"
```

关键点：

- `_run_agent_loop` 把消息发给 provider，处理流式输出、tool call、工具执行、重试等。
- 结果暂存在 `ctx` 中，**不会直接写入 `session.messages`**。
- `maybe_continue_turn(ctx)`：如果 tool call 到上限但 sustained goal 还没完成，会把续杯消息放入 `pending_queue`。

返回 `"ok"`，进入 `SAVE`。

#### 为什么不直接保存历史？

RUN 期间只保存 **runtime checkpoint**（到 `session.metadata`），用于崩溃恢复。正式历史在 SAVE 写入，因为：

- 需要拿到完整结果后才能清洗（去重、截断 tool 结果、过滤 runtime 上下文）。
- 避免把不完整的中间状态写入历史，污染后续 LLM 上下文。
- 保持 RUN 的职责单一。

---

### 2.6 SAVE：持久化与清理

**位置**：`nanobot/agent/loop.py:1511`

**职责**：把 LLM 执行结果正式写入 session，并清理临时状态。

主要工作：

1. **准备保存边界**：`turn_continuation.prepare_save_boundary(ctx)`。
2. **空回复兜底**：如果 `final_content` 为空，填充默认消息。
3. **计算延迟**：`ctx.turn_latency_ms`。
4. **保存历史**：`_save_turn(ctx.session, ctx.all_messages, ctx.save_skip, ...)`。
5. **记录延迟指标**：`record_turn_latency`。
6. **文件上限与后台压缩**：`enforce_file_cap` + 调度 `maybe_consolidate_by_tokens`。
7. **清理恢复标记**：`_clear_pending_user_turn`、`_clear_runtime_checkpoint`。
8. **落盘**：`self.sessions.save(ctx.session)`。

返回 `"ok"`，进入 `RESPOND`。

---

### 2.7 RESPOND：组装最终回复

**位置**：`nanobot/agent/loop.py:1550`

**职责**：根据本轮结果组装最终 outbound 消息，发给用户/渠道。

```python
async def _state_respond(self, ctx: TurnContext) -> str:
    if ctx.suppress_response:
        ctx.outbound = None
        return "ok"
    ctx.outbound = self._assemble_outbound(
        ctx.msg,
        ctx.final_content,
        ctx.all_messages,
        ctx.stop_reason,
        ctx.had_injections,
        ctx.on_stream,
        turn_latency_ms=ctx.turn_latency_ms,
    )
    if ctx.ephemeral and ctx.outbound is not None:
        ctx.outbound.metadata["_stop_reason"] = ctx.stop_reason
    return "ok"
```

- 如果 `suppress_response=True`（如内部续杯），不发消息。
- 否则构造 `OutboundMessage`，包含 content、latency_ms、_streamed 等 metadata。
- 临时会话额外附加 `_stop_reason`。

返回 `"ok"`，进入 `DONE`。

---

## 3. 如何启动下一轮

### 3.1 用户触发

```text
用户发新消息 → channel → MessageBus → consume_inbound() → _dispatch_message() → 新 TurnContext
```

### 3.2 内部自动续杯（continuation）

当 `_state_run` 中 `stop_reason == "max_iterations"` 且存在活跃 sustained goal 时：

1. `turn_continuation.maybe_continue_turn(ctx)` 构造一条虚拟用户消息。
2. 消息放入 `ctx.pending_queue`，并设置 `ctx.suppress_response = True`。
3. 当前 turn 正常结束（SAVE → RESPOND → DONE）。
4. `_dispatch_message` 在 `finally` 中把 `pending_queue` 剩余消息重新 `publish_inbound` 到总线。
5. 主循环消费该消息，开启新一轮，但因为 `metadata["_internal_continuation"] == True`，不会重复保存用户输入。

最多续杯 `_MAX_GOAL_CONTINUATION_ROUNDS = 12` 次。

---

## 4. 与 LangGraph 的异同

### 相同点

- 都是状态机/图模型：节点（状态）+ 边（转移）。
- 都支持条件分支（nanobot 用 `event`，LangGraph 用 `add_conditional_edges`）。
- 都支持循环执行。
- 都有持久化/checkpoint 思想。

### 不同点

| 维度 | nanobot AgentLoop | LangGraph |
|---|---|---|
| 实现 | 手写轻量状态机 | 框架级图引擎 |
| 状态数量 | 固定 7 个 + DONE | 任意 |
| 执行方式 | 严格串行 | 支持并行、子图 |
| 状态定义 | `TurnState` 枚举 + `_state_*` 方法 | `StateGraph` + `TypedDict` |
| 转移定义 | 静态字典 `_TRANSITIONS` | `add_edge` / `add_conditional_edges` |
| 依赖 | 无额外框架 | 依赖 LangChain |
| 定位 | 聊天机器人生命周期 | 通用复杂 agent 工作流 |

nanobot 的状态机可以看作 **LangGraph 理念的一个极简内嵌版**，专门为单轮/多轮对话生命周期定制。

---

## 5. 为什么设计得这么复杂

核心原因：nanobot 不只是一个“把用户消息转发给 LLM”的聊天机器人，而是一个需要处理多种复杂场景的 agent 框架。

| 设计目标 | 对应状态/机制 |
|---|---|
| 用户输入预处理（媒体、文档） | `RESTORE` |
| 崩溃恢复 | `RESTORE` + checkpoint |
| 上下文长度管理 | `COMPACT` + `BUILD` 压缩 |
| 命令 shortcut，减少 LLM 调用 | `COMMAND` |
| 完整的 LLM/tool 执行 | `RUN` |
| 持久化与回复解耦 | `SAVE` + `RESPOND` |
| 长任务自动续杯 | `RUN` + `turn_continuation` |
| 可观测性（WebUI、指标） | 各阶段 `runtime_events` |

状态机让每阶段职责清晰、可追踪、可扩展，同时为崩溃恢复、续杯、命令 shortcut 等能力提供了自然的实现结构。

---

## 6. 总结

nanobot 的 `AgentLoop` 通过事件驱动的状态机，把一个用户请求的生命周期拆成 7 个清晰阶段：

```text
RESTORE   → 加载 session、处理媒体、崩溃恢复
COMPACT   → 准备历史摘要
COMMAND   → 路由斜杠命令
BUILD     → 组装 LLM 上下文
RUN       → 调用 LLM / 执行 tools
SAVE      → 持久化历史、清理状态
RESPOND   → 发送最终回复
```

这种设计牺牲了一定的简单性，换来了：

- 清晰的职责分离
- 可靠的崩溃恢复
- 可控的上下文长度
- 灵活的命令 shortcut
- 长任务自动续杯
- 完善的运行时事件与监控

对于需要长期运行、处理复杂工具调用、支持多渠道和多会话的 agent 框架而言，这是一种合理且必要的架构选择。
