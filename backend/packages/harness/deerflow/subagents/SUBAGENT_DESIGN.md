# Subagent（sub_agent）体系设计梳理（代码对齐）

本文聚焦 `deerflow/subagents/` 相关实现，系统梳理 Subagent 的核心结构体、执行链路、内置 subagent 设计与关键约束。

## 1. 定位与边界

Subagent 的定位是：通过内置工具 `task` 把复杂任务从 Lead Agent 分离到“独立上下文的子执行”中，以达到：

- 上下文隔离：子任务的对话不会污染主对话历史；
- 并行吞吐：一轮模型输出可以并行触发多个子任务（受硬上限约束）；
- 可控性：每个子任务有独立的 turn 上限与超时控制；
- 进度可见：子任务运行期间的 AIMessage 会被实时采集并以事件流返回。

非目标（当前实现不做的事情）：

- Subagent 不是独立的进程/服务；它是在同一进程内创建的 agent 实例，并借助线程池后台执行；
- Subagent 默认不允许递归再调用 `task`，防止套娃；
- Subagent 默认不走用户澄清链路（禁止 ask_clarification），避免子上下文把主流程打断到澄清模式。

## 2. 关键结构体与模块

### 2.1 SubagentConfig（“一个 subagent 类型”的静态配置）

文件：`deerflow/subagents/config.py`

- `name / description / system_prompt`：定义该 subagent 的身份、何时使用、行为约束；
- `tools: list[str] | None`：工具白名单；为 `None` 表示继承父工具集（之后仍会再做 deny 过滤）；
- `disallowed_tools: list[str]`：工具黑名单；默认包含 `task`（防递归），内置 subagent 还会额外禁掉 `ask_clarification/present_files`；
- `model: str`：支持 `"inherit"`，表示继承父 agent 的模型名；
- `max_turns: int`：子 agent 最大递归/turn 限制（会写入 `RunnableConfig["recursion_limit"]`）；
- `timeout_seconds: int`：后台执行超时（线程池级别）。

### 2.2 SubagentStatus / SubagentResult（“一次 subagent 执行”的运行态）

文件：`deerflow/subagents/executor.py`

- `SubagentStatus`：`PENDING → RUNNING → COMPLETED/FAILED/TIMED_OUT`
- `SubagentResult`：一次 task 的执行结果对象，承载：
  - `task_id`：通常使用 `tool_call_id`（便于串联）；
  - `trace_id`：用于分布式/日志串联（父子链路）；
  - `result / error`：终态输出；
  - `started_at / completed_at`：时间戳；
  - `ai_messages`：子任务运行中产生的完整 AIMessage（dict 化）列表，用于流式进度事件。

### 2.3 registry（内置 subagent 注册与可见性裁剪）

文件：`deerflow/subagents/registry.py`

- `BUILTIN_SUBAGENTS`：内置 subagent 的静态注册表（见 `deerflow/subagents/builtins/__init__.py`）
- `get_subagent_config(name)`：
  - 基于内置配置取出 `SubagentConfig`
  - 应用 `config.yaml` 中的 subagent 超时 override（`deerflow/config/subagents_config.py`）
- `get_available_subagent_names()`：
  - 基于 sandbox 安全策略，决定是否暴露 `bash` subagent（LocalSandboxProvider 默认禁 host bash）。

### 2.4 SubagentExecutor（执行引擎）

文件：`deerflow/subagents/executor.py`

职责：

- 基于 `SubagentConfig` + 父级上下文（模型名、sandbox、thread_data、thread_id、trace_id）创建子 agent；
- 过滤工具（allow/deny）；
- 支持同步 `execute()`（内部 `asyncio.run`）与后台 `execute_async()`；
- 全局任务表 `_background_tasks` + 两级线程池：
  - `_scheduler_pool`：调度后台任务、做超时等待与状态回填
  - `_execution_pool`：实际运行子 agent 执行逻辑（以便支持 timeout）
- 提供 `get_background_task_result / cleanup_background_task` 供 task 工具轮询与终态清理。

## 3. 内置 subagent（built-ins）

文件：`deerflow/subagents/builtins/*`

### 3.1 general-purpose

文件：`deerflow/subagents/builtins/general_purpose.py`

- 适用：复杂、多步、需要探索+修改的任务；
- 工具策略：
  - `tools=None`（继承父工具集）
  - `disallowed_tools=["task","ask_clarification","present_files"]`：
    - 防止递归 subagent；
    - 子上下文不触发澄清/交互；
    - 子上下文不直接“呈现文件”（交由主上下文负责）。

### 3.2 bash

文件：`deerflow/subagents/builtins/bash_agent.py`

- 适用：一串命令执行（git/build/test/deploy），把大量终端输出隔离；
- 工具策略：
  - 收敛到少量沙箱工具：`["bash","ls","read_file","write_file","str_replace"]`
  - `disallowed_tools=["task","ask_clarification","present_files"]`
  - `max_turns=30`（比通用 subagent 更小）
- 可见性：仅当 host bash 允许时才暴露（LocalSandboxProvider 默认禁）。

## 4. 核心执行链路（从 Lead Agent 到 Subagent）

这条链路由三层共同完成：Lead Agent（启用与治理）→ task 工具（编排与轮询）→ SubagentExecutor（执行与采集）。

### 4.1 启用 subagent 能力（工具与 prompt）

文件：`deerflow/tools/tools.py`

- 只有 `get_available_tools(..., subagent_enabled=True)` 才会把 `task` 工具注入到工具集中；
- 子 agent 获取工具时强制 `subagent_enabled=False`（防止递归）。

文件：`deerflow/agents/lead_agent/prompt.py`

- `apply_prompt_template(subagent_enabled=True)` 会注入 `<subagent_system>` 段落：
  - 强调“分解 → 并行 task → 汇总”
  - 强调每轮最多 N 个 `task`（硬限制）。

### 4.2 并发治理：SubagentLimitMiddleware（硬截断）

文件：`deerflow/agents/middlewares/subagent_limit_middleware.py`

- Hook 点：`after_model`
- 行为：统计最后一个 AIMessage 中 `tool_calls` 里 `name == "task"` 的数量，超过上限则截断多余 calls，并回写更新后的 AIMessage。
- 上限：默认 3（并 clamp 到 [2,4]），来源于 runtime configurable `max_concurrent_subagents`。

### 4.3 触发：task 工具（入口、继承父上下文、轮询与事件流）

文件：`deerflow/tools/builtins/task_tool.py`

核心步骤：

1) 校验 subagent 类型 + 安全策略（bash 可用性）
2) 解析 subagent config，并叠加 override：
   - 注入 skills prompt 片段到子 agent system_prompt
   - 覆盖 max_turns（如果传参）
3) 从 runtime 提取父上下文：
   - `sandbox_state / thread_data / thread_id`
   - `parent_model`（来自 runtime.config.metadata）
   - `trace_id`（用于串联日志）
4) 构建子 agent 工具集：
   - `get_available_tools(model_name=parent_model, subagent_enabled=False)`
5) 创建 `SubagentExecutor` 并后台执行：
   - `task_id = executor.execute_async(prompt, task_id=tool_call_id)`
6) 后端轮询状态并推送事件：
   - `task_started`
   - `task_running`（每次发现新增 `ai_messages`，逐条推送）
   - `task_completed / task_failed / task_timed_out`
7) 终态清理：
   - `cleanup_background_task(task_id)` 删除全局任务表条目，防止内存堆积。

### 4.4 执行：SubagentExecutor（创建子 agent、执行、采集与超时）

文件：`deerflow/subagents/executor.py`

关键点：

- Agent 创建：
  - 模型：`create_chat_model(name=_get_model_name(config, parent_model), thinking_enabled=False)`
  - middleware：复用 runtime middlewares（subagent 版本，见 `build_subagent_runtime_middlewares`）
  - `system_prompt=config.system_prompt`
  - `state_schema=ThreadState`
- 初始 state：
  - 仅注入 `HumanMessage(task)`
  - 透传 `sandbox_state / thread_data`（用于 sandbox 路径与能力）
- 实际执行（async）：
  - `agent.astream(..., stream_mode="values")` 获取实时 state chunk
  - 每个 chunk 抽取最后一个 `AIMessage`，dict 化后追加到 `result.ai_messages`（去重）
- 最终结果抽取：
  - 从 final state 里倒序找到最后一个 `AIMessage`，把其 `content` 规整成字符串作为 `result.result`
- 后台执行与超时：
  - `execute_async()` 把任务记录到 `_background_tasks`
  - `_scheduler_pool` 提交调度任务
  - 调度任务再提交 `_execution_pool` 实际执行，并以 `Future.result(timeout=timeout_seconds)` 等待
  - 超时则标记 `TIMED_OUT` 并 best-effort cancel future

## 5. 安全与隔离策略总结

- 防递归：工具装配时对子 agent 强制 `subagent_enabled=False`，配置层面默认 deny `task`；
- 防打断：内置 subagent deny `ask_clarification`，避免子上下文触发澄清工作流；
- 防越权：LocalSandboxProvider 默认禁 host bash，进而隐藏 `bash` subagent，且 tools 装配也会过滤 host bash 工具；
- 并发硬限：prompt 软提示 + `SubagentLimitMiddleware` 硬截断 + 线程池容量共同约束。

## 6. 扩展点（新增 subagent 的方式）

当前 subagent 类型来自内置注册表 `BUILTIN_SUBAGENTS`。新增一个 subagent 的最小改动通常是：

1) 在 `deerflow/subagents/builtins/` 增加一个 `SubagentConfig`
2) 在 `deerflow/subagents/builtins/__init__.py` 注册到 `BUILTIN_SUBAGENTS`
3) 根据安全策略决定是否在 `get_available_subagent_names()` 做可见性裁剪

如果希望支持“可配置注册”（非硬编码），需要引入额外的配置加载与动态构建逻辑（当前实现未提供）。

