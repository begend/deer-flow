# Store 与 Checkpoint：定位、联系、区别与请求流解析

本文结合 DeerFlow 当前实现，系统说明 `store` 和 `checkpoint` 的定位，以及它们在具体请求中的分工。

## 1. 一句话理解

- `checkpoint`：保存线程在某一时刻的运行快照，用于状态恢复、历史读取、中断续跑。
- `store`：保存适合快速查询和列举的轻量业务记录，用于线程索引、元数据检索和展示投影。

如果把一个 thread 看成一条持续演进的会话：

- `checkpoint` 保存的是“运行现场”
- `store` 保存的是“可检索档案”

## 2. 核心定位

### 2.1 Checkpoint 的定位

在 DeerFlow 中，`checkpointer` 是传给 LangGraph agent 的状态持久化能力，核心职责是按 `thread_id` 保存和读取线程快照。

它主要负责：

- 保存当前线程的 `channel_values`
- 保存 checkpoint metadata
- 保存 `checkpoint_id` / `parent_checkpoint_id`
- 支持 history 遍历
- 支持 interrupt 后恢复
- 为后续 run 提供连续上下文

对应实现可以看：

- `backend/packages/harness/deerflow/agents/checkpointer/provider.py`
- `backend/app/gateway/routers/threads.py`
- `backend/packages/harness/deerflow/runtime/runs/worker.py`

在架构文档中，线程状态持久化也明确依赖 checkpointer：

- `backend/packages/harness/deerflow/HARNESS_ARCHITECTURE.md`

### 2.2 Store 的定位

`store` 是一个更通用的运行时存储抽象，偏向 namespace + key 的轻量业务数据存储。

在当前 DeerFlow 仓库里，它最明确的用途是：

- 保存线程索引记录
- 支持线程快速搜索和列举
- 保存适合列表展示的轻量字段
- 补充 Gateway 侧的业务检索能力

典型存储内容包括：

- `thread_id`
- `status`
- `created_at`
- `updated_at`
- `metadata`
- `values.title`

对应实现可以看：

- `backend/packages/harness/deerflow/runtime/store/provider.py`
- `backend/app/gateway/routers/threads.py`
- `backend/app/gateway/services.py`

## 3. 两者的联系

### 3.1 共用同一份后端配置

DeerFlow 里 `store` 和 `checkpointer` 共享 `config.yaml` 中的 `checkpointer` 配置段，因此通常会落到同一类后端：

- `memory`
- `sqlite`
- `postgres`

也就是说，虽然它们职责不同，但通常使用同一类持久化基础设施。

### 3.2 运行时会一起注入

Gateway 启动时会同时初始化：

- `app.state.checkpointer`
- `app.state.store`

兼容运行链路中，也会在执行前把二者都挂到 agent 上：

- `agent.checkpointer = checkpointer`
- `agent.store = store`

这说明二者在运行期是并列依赖，但不是同一概念。

### 3.3 Thread 是它们共同的关联单元

两者都围绕 `thread_id` 工作：

- `checkpoint` 按 `thread_id` 组织状态快照
- `store` 通常以 `thread_id` 作为记录 key

所以在业务层面它们描述的是同一条线程，但描述角度不同。

### 3.4 Store 经常是 Checkpoint 的投影层

很多时候，`store` 中的数据并不是独立真相，而是从 `checkpoint` 的状态中提取出来的轻量投影。

例如：

- 线程搜索时，`store` 不存在的旧线程会从 `checkpointer` 中扫描并懒迁移回来
- run 完成后，生成的 `title` 会从 checkpoint 读出，再同步回 `store`

所以可以把两者关系理解成：

- `checkpoint` 更接近底层状态真相源
- `store` 更接近面向查询的投影视图

## 4. 两者的区别

### 4.1 保存对象不同

`checkpoint` 保存的是运行快照，例如：

- messages
- title
- todos
- interrupts
- task 信息
- checkpoint 链关系

`store` 保存的是可检索记录，例如：

- 线程列表项
- 元数据摘要
- 展示字段
- 轻量 values 投影

### 4.2 目标不同

`checkpoint` 解决的是：

- 如何恢复线程状态
- 如何查看线程历史
- 如何支持中断续跑
- 如何让后续请求沿用之前上下文

`store` 解决的是：

- 如何快速列出所有线程
- 如何按 metadata 过滤线程
- 如何让列表页快速展示标题和摘要
- 如何维护 Gateway 层业务索引

### 4.3 读写方式不同

`checkpoint` 常见操作：

- `aget_tuple`
- `aput`
- `alist`

`store` 常见操作：

- `aget`
- `aput`
- `asearch`
- `adelete`

### 4.4 数据结构语义不同

`checkpoint` 天然有历史链和版本语义：

- 当前 checkpoint
- 父 checkpoint
- 历史 checkpoint 列表

`store` 更像当前记录表，通常是 upsert 最新结果，不强调完整历史链。

### 4.5 真相源级别不同

在当前实现里：

- `checkpoint` 更接近线程状态的主真相源
- `store` 更接近查询优化层和投影层

如果 `store` 丢失，一部分线程信息仍可以从 `checkpoint` 扫描恢复。

反过来如果 `checkpoint` 丢失，就无法正确恢复线程运行状态。

## 5. 结合具体请求分析

下面按典型请求说明两者如何配合。

---

## 5.1 创建线程：`POST /api/threads`

入口：

- `backend/app/gateway/routers/threads.py`

这个请求会同时做两件事：

1. 向 `store` 写入线程记录
2. 向 `checkpointer` 写入空 checkpoint

### Store 在这里做什么

它会保存线程索引项，使线程立刻能出现在 `/api/threads/search` 结果里。

典型字段：

- `thread_id`
- `status = idle`
- `created_at`
- `updated_at`
- `metadata`

### Checkpoint 在这里做什么

它会创建一个空的初始快照，使：

- `/api/threads/{thread_id}/state` 能立即工作
- 线程后续可以直接进入 run
- 线程有合法的状态起点

### 这个请求体现的本质

- `store` 负责“登记”
- `checkpoint` 负责“建状态容器”

---

## 5.2 搜索线程：`POST /api/threads/search`

入口：

- `backend/app/gateway/routers/threads.py`

这个接口采用三阶段逻辑：

1. 先查 `store`
2. 再扫 `checkpointer` 补漏
3. 最后过滤、排序、分页

### 为什么先查 Store

因为线程搜索更像“查目录”，适合轻量索引记录：

- 成本更低
- 可以快速按 metadata / status 过滤
- 不需要扫描整个 checkpoint 历史体系

### 为什么还要扫 Checkpoint

因为有些线程可能：

- 不是经由 `POST /api/threads` 创建
- 或早于 store 机制引入

这类线程在 `checkpoint` 中已经存在，但 `store` 中没有索引。

因此系统会：

- 从 `checkpointer.alist(None)` 扫描线程
- 补出缺失 thread
- 立刻回写 `store`

这说明：

- `store` 是快路径
- `checkpoint` 是兜底来源

### 这个请求体现的本质

- `store` 负责“快速列举”
- `checkpoint` 负责“补足真实存在但未索引的数据”

---

## 5.3 获取线程信息：`GET /api/threads/{thread_id}`

入口：

- `backend/app/gateway/routers/threads.py`

这个接口是“混合读取”：

- 元数据优先看 `store`
- 状态再由 `checkpoint` 推导

### Store 在这里做什么

提供：

- `created_at`
- `updated_at`
- `metadata`
- 部分轻量 `values`

### Checkpoint 在这里做什么

提供真实运行状态依据，例如：

- 当前是否 interrupted
- 是否存在 pending tasks
- 当前 channel_values

### 这个请求体现的本质

这是一个典型的：

- `store` 提供摘要
- `checkpoint` 提供真实状态

的组合型接口。

---

## 5.4 获取线程状态：`GET /api/threads/{thread_id}/state`

入口：

- `backend/app/gateway/routers/threads.py`

这个接口几乎完全依赖 `checkpoint`。

它直接从 `checkpointer.aget_tuple(...)` 中读取：

- `channel_values`
- `metadata`
- `checkpoint_id`
- `parent_checkpoint_id`
- `tasks`

### 为什么不主要依赖 Store

因为这个接口要回答的是：

- 当前 messages 是什么
- 当前有哪些 channel values
- 当前有没有 interrupt task
- 当前 checkpoint 链位置在哪里

这些都属于运行现场，不适合只靠索引记录来回答。

### 这个请求体现的本质

- 状态读取的核心来源是 `checkpoint`
- `store` 在此类请求中不是关键角色

---

## 5.5 创建运行：`POST /api/threads/{thread_id}/runs/stream`

入口：

- `backend/app/gateway/routers/thread_runs.py`
- `backend/app/gateway/services.py`
- `backend/packages/harness/deerflow/runtime/runs/worker.py`

这是最能体现两者配合关系的请求。

### 第一步：创建 RunRecord

`start_run(...)` 会先创建 run 记录，用于管理运行生命周期。

这一层本身不等于 checkpoint，也不等于 store，它属于 run 管理面。

### 第二步：确保线程可在 Store 中被搜索到

如果这个线程是通过“无状态 run 接口”触发的，而不是先显式创建线程，系统会补做一次 `store` upsert。

这样可以保证：

- 这个 thread 后续能出现在线程列表中

### 第三步：把 Store 和 Checkpoint 一起注入执行时上下文

执行前会把：

- `Runtime(context={"thread_id": ...}, store=store)`
- `agent.checkpointer = checkpointer`
- `agent.store = store`

一起挂到运行环境中。

### 执行中谁负责什么

#### Checkpoint 负责

- 持久化 thread state
- 保存本次运行产生的新状态
- 让下一次 run 可以延续这次状态
- 为 interrupt / history / resume 提供基础

#### Store 负责

- 为运行时提供可选通用存储依赖
- 保证线程在 Gateway 层可搜索
- 在 run 完成后同步 `title` 等轻量字段

### Run 完成后的标题同步

当前实现里，TitleMiddleware 产生的标题先写进 agent state，也就是先落到 `checkpoint` 中。

随后 Gateway 会在 run 结束后：

1. 从 checkpoint 读出最终 `title`
2. 再同步到 `store.values.title`

这样 `/api/threads/search` 可以立刻显示正确标题。

### 这个请求体现的本质

- `checkpoint` 负责“运行状态连续性”
- `store` 负责“运行结果在检索面上的可见性”

---

## 5.6 无状态运行：`POST /api/runs/stream`

入口：

- `backend/app/gateway/routers/runs.py`

这个接口表面上叫“无状态”，但内部并不是真的没有 thread。

逻辑是：

- 如果请求里已有 `thread_id`，直接复用
- 否则自动生成一个新的 `thread_id`

之后依然会进入同样的 `start_run(...)` 流程。

这意味着：

- 真正的状态延续仍由 `checkpoint` 承担
- 真正的 thread 可发现性仍由 `store` 负责补登记

所以它只是“调用方不必先手动创建 thread”，不是系统内部没有 thread。

---

## 5.7 人工更新状态：`POST /api/threads/{thread_id}/state`

入口：

- `backend/app/gateway/routers/threads.py`

这个请求本质上是：

1. 读取当前 checkpoint
2. 合并新的 `values`
3. 写入新的 checkpoint

### 为什么主写入对象是 Checkpoint

因为用户在这里改的不是“线程目录信息”，而是线程运行状态本身。

比如：

- 修改 channel values
- 从某个状态继续
- 写入人工干预后的结果

这些都必须进入 checkpoint 链，才能真正影响后续运行。

### Store 在这里做什么

只有当更新里包含对列表展示有价值的字段，例如 `title`，才会顺带同步到 store。

也就是说：

- 主写入对象是 `checkpoint`
- `store` 只做摘要同步

### 这个请求体现的本质

- “状态变更”属于 checkpoint
- “列表展示字段刷新”属于 store

---

## 5.8 获取线程历史：`POST /api/threads/{thread_id}/history`

入口：

- `backend/app/gateway/routers/threads.py`

这个请求完全依赖 `checkpoint`。

因为历史请求要返回：

- 每个 checkpoint 的 ID
- parent checkpoint
- 当时的 metadata
- 当时的 values
- 当时的 next tasks

这些天然属于快照链语义，而不是索引表语义。

### 这个请求体现的本质

- history 是 `checkpoint` 的天然能力
- `store` 不适合承担状态历史链

---

## 5.9 取消运行：`POST /api/threads/{thread_id}/runs/{run_id}/cancel`

入口：

- `backend/app/gateway/routers/thread_runs.py`
- `backend/packages/harness/deerflow/runtime/runs/manager.py`
- `backend/packages/harness/deerflow/runtime/runs/worker.py`

这个请求特别能体现 checkpoint 的重要性。

系统支持两种取消语义：

- `interrupt`
- `rollback`

其中：

- `interrupt`：停止执行，但保留当前 checkpoint
- `rollback`：理论上回到运行前的 checkpoint

worker 在 run 开始时会先记录 `pre_run_checkpoint_id`，就是为了 rollback 使用。

虽然当前 rollback 仍是未完全实现的 TODO，但设计方向已经很清楚：

- 取消和回滚的核心依赖是 `checkpoint`
- `store` 在这里并不负责状态回退

### 这个请求体现的本质

- 复杂状态控制语义压在 `checkpoint`
- `store` 不承担状态回滚职责

## 6. 用一句工程化的话总结

如果从系统设计角度概括：

- `checkpoint` 是线程运行状态的主持久化层
- `store` 是面向查询和展示的辅助索引层

或者更具体一点：

- `checkpoint` 回答“这个 thread 现在到底处于什么状态，以及之前经历了哪些状态”
- `store` 回答“系统里有哪些 thread，它们的摘要信息是什么，怎么快速查出来”

## 7. 适合用哪个的判断标准

当你面对一类数据时，可以这样判断：

### 应该进入 Checkpoint 的数据

如果它的作用是：

- 恢复线程
- 延续上下文
- 支持中断恢复
- 支持历史追踪
- 决定后续运行行为

那它应该主要进入 `checkpoint`。

### 应该进入 Store 的数据

如果它的作用是：

- 快速列举
- 快速过滤
- 列表展示
- 元数据检索
- 业务索引

那它更适合进入 `store`。

## 8. 最终结论

在 DeerFlow 当前实现中，二者不是替代关系，而是分层协作关系：

- `checkpoint` 负责线程状态真相
- `store` 负责线程查询投影

因此可以把它们理解为：

- `checkpoint` = execution state snapshot layer
- `store` = queryable metadata/index layer

这也是为什么很多请求里二者会同时出现：

- 一个负责“状态正确”
- 一个负责“查询高效”
