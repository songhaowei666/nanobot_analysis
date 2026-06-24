# nanobot 源码解读文档

> 本仓库是对 [HKUDS/nanobot](https://github.com/HKUDS/nanobot) 的源码解读与学习笔记整理。
>
> nanobot 是一个开源、超轻量的个人 AI Agent，核心代码小巧易读，同时内置了 WebUI、聊天渠道、工具调用、记忆、MCP、模型路由、自动化与部署等面向真实长期任务的完整能力。

---

## 📌 关于 nanobot

| 项目 | 链接 |
|---|---|
| 官方仓库 | [github.com/HKUDS/nanobot](https://github.com/HKUDS/nanobot) |
| 官方文档 | [nanobot.wiki](https://nanobot.wiki/docs/latest/getting-started/nanobot-overview) |
| PyPI 包名 | `nanobot-ai` |
| 许可证 | MIT |

### 一句话介绍

**nanobot** = 小而美的 AI Agent 运行时，让你真正拥有自己的 Agent：持久化工作流、多渠道聊天、模型自由切换、MCP 扩展、记忆与自动化，全部集成在一个可读性很高的 Python 代码库中。



## 📚 本仓库文档目录

| 文档 | 主题 | 状态 |
|---|---|---|
| [`nanobot_loop_state_machine.md`](./nanobot_loop_state_machine.md) | `AgentLoop` 事件驱动状态机深度解析 | ✅ 已完成 |

---

## 🔍 已完成解读：`AgentLoop` 状态机

nanobot 的 `AgentLoop`（位于 `nanobot/agent/loop.py`）没有把一次用户请求简单变成一次 LLM 调用，而是实现了一个**事件驱动的有限状态机（FSM）**，将单轮用户请求的生命周期拆成 7 个清晰阶段：

```text
RESTORE  → COMPACT → COMMAND ──dispatch──→ BUILD → RUN → SAVE → RESPOND → DONE
             │                              │
             └────────shortcut──────────────┘
```

| 状态 | 主要职责 |
|---|---|
| **RESTORE** | 加载 session、处理媒体附件、崩溃恢复 |
| **COMPACT** | 准备历史摘要，控制上下文长度 |
| **COMMAND** | 斜杠命令路由与 shortcut，减少不必要 LLM 调用 |
| **BUILD** | 组装 LLM 上下文：历史、系统提示、工具上下文 |
| **RUN** | 调用 LLM、执行 tool call 循环、收集结果 |
| **SAVE** | 正式持久化历史、清理临时状态、记录延迟指标 |
| **RESPOND** | 组装最终 outbound 消息并发送给用户 |

### 设计亮点

- **崩溃恢复**：`RESTORE` + runtime checkpoint 修复工具调用中断或 assistant 未回复的残局。
- **上下文压缩**：`COMPACT` / `BUILD` 阶段通过 token 阈值触发历史摘要，避免 prompt 无限增长。
- **命令 shortcut**：`COMMAND` 阶段拦截 `/new`、`/image` 等命令，直接返回而不调用 LLM。
- **持久化与回复解耦**：`RUN` 只写 checkpoint，`SAVE` 才写入正式历史，避免污染。
- **长任务自动续杯**：当 `max_iterations` 触发且存在活跃 `sustained goal` 时，自动构造虚拟用户消息开启新一轮。

详细内容请阅读：[`nanobot_loop_state_machine.md`](./nanobot_loop_state_machine.md)

---


## 🤝 说明

本仓库为个人学习整理，所有源码引用与描述均基于 nanobot 官方仓库对应版本。如发现解读有误，欢迎提交 Issue 或 PR 指正。



