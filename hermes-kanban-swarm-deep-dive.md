# Hermes Agent v0.16 Kanban Swarm：当 AI 代理学会像团队一样协作

> 你让一个 AI 代理写代码、做研究、发邮件，它都能搞定。但当你需要 **3 个代理并行调研、1 个评审员把关、1 个写手汇总**——而且这一切要在崩溃后能恢复、有人类能插手的会话中完成——传统的 `delegate_task` 就撑不住了。

Hermes Agent v0.16 的 Kanban Swarm 不是又一个任务队列。它是一个**基于 SQLite 的持久化多代理编排系统**，让不同 Profile（角色）的代理像 DevOps 团队一样通过看板协作，每个 Worker 是一个独立 OS 进程，每个交接在数据库中永久留痕。

这篇文章深入架构设计、Worker 生命周期、Swarm 拓扑和 CLI 实战，面向想理解多代理系统如何落地的 AI 开发者。

---

## 为什么需要 Kanban？`delegate_task` 不够吗？

先说结论：**`delegate_task` 是函数调用，Kanban 是工作队列**。

| 维度 | `delegate_task` | Kanban |
|------|----------------|--------|
| 形态 | RPC 调用（fork → join） | 持久化消息队列 + 状态机 |
| 父进程 | 阻塞直到子进程返回 | 创建后即忘（fire-and-forget） |
| 子进程身份 | 匿名子代理 | 命名 Profile，带持久化记忆 |
| 可恢复性 | 失败即丢失 | 阻塞 → 解阻塞 → 重跑；崩溃 → 回收 |
| 人工介入 | 不支持 | 随时 Comment / Unblock |
| 审计追踪 | 上下文压缩后丢失 | SQLite 永久存储 |
| 协作方式 | 层级式（调用者 → 被调用者） | 对等——任何 Profile 读写任何任务 |

`delegate_task` 适合父代理需要**短推理答案**的场景——一次函数调用等结果回来。Kanban 适合**跨代理边界、需要存活在重启之后、可能需要人工审核**的工作流。

**两者可以共存**：一个 Kanban Worker 在它自己的运行周期内完全可以调用 `delegate_task`。

---

## 架构全景

```
┌─────────────────────────────────────────────────────┐
│                   Dispatcher (gateway内)              │
│  每60s扫描 ready 任务 → 原子 claim → spawn Profile    │
└──────────┬──────────────────────────────┬────────────┘
           │                              │
           ▼                              ▼
┌──────────────────┐          ┌──────────────────────┐
│  kanban.db (WAL)  │          │  Workspace (scratch/  │
│  SQLite 持久化     │          │  dir/worktree)        │
│  任务/运行/事件    │          │  Worker 的工作目录      │
└──────────────────┘          └──────────────────────┘
           ▲                              ▲
           │                              │
┌──────────┴──────────────────────────────┴────────────┐
│                   三种访问面                           │
│                                                       │
│  CLI (hermes kanban …)  │  工 具 (kanban_show/…)    │
│  Dashboard (React SPA)  │  (Worker 内部使用)          │
└──────────────────────────────────────────────────────┘
```

### 核心组件

**Board** — 一个独立的任务队列，拥有自己的 SQLite DB（`~/.hermes/kanban.db`）、workspace 目录和 Dispatcher 循环。一个安装可以有多个 Board（每个项目一个）。

**Task** — 一行数据，含标题、正文、分配人（Profile 名）、状态（`triage → todo → ready → running → blocked → done → archived`）、可选的 tenant 命名空间和幂等键。

**Run** — Task 的**一次执行尝试**。Task 是逻辑工作单元，Run 是执行记录。多次失败后有完整的事故分析历史。

**Workspace** — Worker 的操作目录。三种类型：
- `scratch`（默认）——临时目录，完成后自动删除
- `dir:<path>`——共享目录（如 Obsidian 仓库），保留
- `worktree`——Git worktree，用于编码任务，保留

**Dispatcher** — 长期运行的循环，每 60 秒：回收过期 Claim、检测崩溃的 Worker、将 `ready` 任务原子性 Claim 并 Spawn 对应 Profile。默认嵌入在 Gateway 进程中运行。

### Worker 和 Orchestrator 的区别

```
Orchestrator Profile                 Worker Profile（我）
┌─────────────────────┐        ┌────────────────────────┐
│ 分解目标 → 创建子任务 │        │ kanban_show() 读取任务   │
│ 设置依赖关系 → 分配角色 │        │ cd $WORKSPACE 干活     │
│ 自己不干活            │        │ kanban_heartbeat()     │
└─────────┬───────────┘        │ kanban_complete()      │
          │                    └────────────────────────┘
          │ kanban_create / kanban_link
          ▼
┌───────────────────────────────────────────────────────┐
│          Worker 之间通过 Comment 交流                   │
│          Verifier 审核所有 Worker 输出                   │
│          Synthesizer 汇总成最终结果                      │
└───────────────────────────────────────────────────────┘
```

---

## Swarm 拓扑：并行 Worker + Verifier Gate + Synthesizer

v0.16 最引人注目的特性就是 `hermes kanban swarm` 命令——它一条命令创建完整的多代理协作拓扑：

```bash
hermes kanban swarm "设计多区域故障转移方案" \
  --worker researcher:"调研多区域故障转移方案" \
  --worker architect:"设计多区域架构" \
  --worker sre:"评估多区域方案" \
  --verifier reviewer \
  --synthesizer writer
```

**等效于手动创建以下任务图**：

```
                    ┌──── 根/黑板 (t_root) ────┐
                    │   目标、共享上下文、注释      │
                    └────────┬─────────────────┘
                             │
           ┌─────────────────┼──────────────────┐
           ▼                 ▼                  ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │  researcher   │ │  architect   │ │     sre      │
   │  并行调研      │ │  并行设计     │ │  并行评估     │
   └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
           └────────────────┼────────────────┘
                            ▼
                   ┌────────────────┐
                   │   verifier     │  ← 所有 Worker 完成后才激活
                   │  (reviewer)     │
                   │  审核所有输出    │
                   └───────┬────────┘
                           │
                           ▼
                   ┌────────────────┐
                   │  synthesizer   │  ← Verifier 通过后才激活
                   │  (writer)      │
                   │  汇总成最终稿件  │
                   └────────────────┘
```

### 关键设计决策

**为什么是 Verifier Gate 而不是自动合并？**

因为**信任但不验证**是危险的生产心态。在 Swarm 中，多个 Worker 并行产出的结果需要**一个独立的审核者 Profile**去检查质量、一致性和完整性。只有审核通过后，Synthesizer（写手）才开始汇总。这模拟了真实团队中的 PR Review → Merge 流程。

整个图的调度靠的是**Parent→Child 依赖链**：
- Verifier 的 Parent = 所有 Worker
- Synthesizer 的 Parent = Verifier

Dispatcher 只在**所有 Parent 都完成**后才将子任务从 `todo` 提升为 `ready` → 自动触发 Spawn。

---

## Worker 生命周期：从 Spawn 到 Complete

一个 Dispatcher-Spawned Worker 的典型完整生命周期：

```python
# 这是 Worker（AI 代理）内部的工具调用流程，不是人类运行的命令

# 第 1 步：读任务
kanban_show()
# 返回：标题、正文、父任务交接、历史尝试、注释线程、worker_context

# 第 2 步：切换到工作目录
# 终端命令：cd $HERMES_KANBAN_WORKSPACE

# 第 3 步：干活（调用终端/文件工具）
# ...

# 第 4 步：长时间操作时的心跳
kanban_heartbeat(note="完成 60%——4/8 个文件已转换")

# 第 5 步：继续干活
# ...

# 第 6 步：完成，带结构化交接
kanban_complete(
    summary="实现了 Token Bucket 限流器，添加了 14 个测试，全部通过",
    metadata={
        "changed_files": ["limiter.py", "tests/test_limiter.py"],
        "tests_run": 14,
        "verification": ["pytest tests/ -q"],
        "residual_risk": ["高并发下未做性能基准测试"]
    }
)
```

### 为什么重要：结构化交接

`summary` + `metadata` 是两个不同的传输通道：
- **summary**：人类可读的 1-3 句话，下游 Worker 和 Dashboard 直接展示
- **metadata**：机器可读的 JSON，下游 Worker 的 `kanban_show()` 能直接使用

下游的 Worker 读取父任务最近一次成功 Run 的 `summary` + `metadata` 作为 context。失败的 Worker 重新执行时，也能读到自己之前 Run 的 `outcome`、`summary`、`error`——避免重复走过已经失败的路径。

---

## CLI 实战：完整 Swarm 示例

### 初始化

```bash
# 一次性设置
hermes kanban init

# 确保 Gateway 运行（内嵌 Dispatcher）
hermes gateway start
```

### 创建 Swarm

```bash
hermes kanban swarm "深度解析 Hermes Agent v0.16 Kanban Swarm" \
  --worker researcher:"调研 Kanban Swarm 架构设计" \
  --worker writer:"撰写深度技术文章" \
  --verifier reviewer \
  --synthesizer writer
```

这会创建 5 个任务：
1. **t_root**（已完成）——共享黑板，Swarm 上下文
2. **t_researcher**（ready）——研究 Kanban 架构、内部设计
3. **t_writer**（ready）——撰写深度技术文章
4. **t_verifier**（todo，父任务=t_researcher+t_writer）——审核所有输出
5. **t_synthesizer**（todo，父任务=t_verifier）——汇总成最终稿件

### 查看进度

```bash
# 实时看板事件流
hermes kanban watch

# 列表视图
hermes kanban list

# 统计
hermes kanban stats

# 看某个任务的完整上下文
hermes kanban show t_researcher

# 查看运行历史
hermes kanban runs t_researcher
```

### Dashboard 可视化

相比 CLI 的文本列表，`hermes dashboard` 的 Kanban 标签页提供完整的看板视图：

- 每 Status 一列：triage / todo / ready / running / blocked / done
- **Per-Profile Lanes**：Running 列内按 Profile 分组
- **拖拽改状态**：card 拖到另一列即更新
- **侧边抽屉**：点击 card 显示可编辑标题、描述、依赖图、状态操作、运行历史
- **WebSocket 实时更新**：所有变化（CLI、Dashboard、Worker 工具）实时反射

---

## Verifier Gate 的审核模式

Verifier（审核者）Profile 拿到任务后应该做什么？不是简单说"好"或"不好"，而是给出结构化的 Gate 决策：

### Pass 模式

```python
kanban_complete(
    summary="审核通过。两个 Worker 产出完整：researcher 提交了架构分析，writer 提交了技术文章",
    metadata={
        "gate": "pass",
        "worker_reviews": [
            {"task_id": "t_researcher", "verdict": "pass", "notes": "架构分析充分"},
            {"task_id": "t_writer", "verdict": "pass", "notes": "文章覆盖全面"}
        ]
    }
)
```

### Fail 模式

```python
kanban_block(
    reason="审核不通过：writer 的文章缺少 Kanban Swarm CLI 命令的代码示例（gate=fail; missing_work: writer 需要补充 CLI swarm 命令示例）"
)
```

当 Verifier Block 后，有问题的 Worker（t_writer）会留在 `blocked` 状态。人工在 Dashboard 或 CLI 上 Unblock 后，Worker 重跑时会读到之前 Block 的 `reason`——直接定位要修改的部分。

### 类型化 Block

Kanban v0.16 引入了**类型化 Block**，Dispatcher 根据 Block Kind 做不同处理：

| Kind | 含义 | Dispatcher 行为 |
|------|------|----------------|
| `dependency` | 等待另一个任务 | 回到 todo，父任务完成后自动 promote |
| `needs_input` | 需要人类决策 | 停在 blocked，等待 Unblock |
| `capability` | 当前 Worker 做不了 | 停在 blocked，建议重新分配 |
| `transient` | 临时故障 | 停在 blocked，人工恢复 |
| 通用（null） | 默认 | 停在 blocked |

相同的 Block Kind 重复出现达到阈值（默认 2 次）→ 自动跳到 triage，打破 Unblock↔Re-block 死循环。

---

## Orchestrator 模式：分解目标，不亲自干

最佳实践是有一个专门的 Orchestrator Profile，**只做任务分解和分配，不亲自实现**。它的工具集可以限制为只有 `kanban`、`gateway`、`memory`，让它物理上不能干活：

```yaml
# orchestrator Profile 的 config.yaml 片段
toolsets:
  default:
    - kanban
    - gateway
    - memory
```

一个典型的 Orchestrator 回合：

```python
# 用户目标："写一篇关于分布式系统的博文"

kanban_create(
    title="调研分布式共识算法进展",
    assignee="researcher-a",
    body="关注 Raft / Paxos 变体 / EPaxos，2024-2026 论文"
)  # → t_r1

kanban_create(
    title="调研分布式事务实践",
    assignee="researcher-b",
    body="关注 2PC/Saga/ TCC，生产环境案例"
)  # → t_r2

kanban_create(
    title="汇总成博文初稿",
    assignee="writer",
    parents=["t_r1", "t_r2"],  # 两个研究都完成后才 promote
    body="技术博文，2000+ 字，面向后端开发者"
)  # → t_w1

kanban_complete(
    summary="分解为 2 个并行研究任务 → 1 个写作任务，依赖已链接"
)
```

---

## Run 系统：一次任务，多次尝试

一个 Task 可能有多次 Run。每个 Run 单独记录，便于事后审计：

```bash
hermes kanban runs t_abcd

#   #  OUTCOME      PROFILE     ELAPSED   STARTED
#   1  blocked      worker-a       12s     2026-04-27 14:02
#       → BLOCKED: 需要决定限流键策略
#   2  completed    worker-a        8m      2026-04-27 15:18
#       → 实现了 Token Bucket，按 user_id 键控
```

### 为什么重要

Run 系统解决了**调试多代理系统的核心痛点**：
- 可以回答"第二次尝试改了什么？"
- 失败的 Worker 可以读到自己的前一次 `summary` + `error`，避免重走死路
- 下游 Worker 只读父任务**最近一次成功 Run**的交接数据

---

## 8 种协作模式

Hermes Kanban 用同一套原语支持 8 种不同的协作模式，无需新特性：

| 模式 | 结构 | 示例 |
|------|------|------|
| **P1 扇出** | N 个兄弟任务，同角色 | "并行调研 5 个方向" |
| **P2 流水线** | 角色链：侦察 → 编辑 → 写作 | 每日简报组装 |
| **P3 投票/仲裁** | N 个兄弟 + 1 个聚合器 | 3 个研究员 → 1 个审核者选择 |
| **P4 长期日志** | 同 Profile + 共享目录 + Cron | Obsidian 仓库维护 |
| **P5 人工介入** | Worker 阻塞 → 用户评论 → 解阻塞 | 模糊决策需人工 |
| **P6 @提及** | 从正文中内联路由 | @reviewer 看看这个 |
| **P7 线程执行域** | 网关线程中的 /kanban | 按项目的网关线程 |
| **P8 舰队耕作** | 一个 Profile，N 个主体 | 50 个社交媒体账号 |

Swarm 机制专门为 P1（扇出）+ P3（仲裁）+ P2（流水线）的组合而生。

---

## 从消息网关驱动看板

你不需要坐在终端前操作看板。从 Telegram、Discord、Slack 等任何已连接的 Gateway 平台，通过 `/kanban` 斜杠命令就能操作：

```
你> /kanban create "翻译 README 到日语" --assignee linguist
bot> ✅ 创建 t_9fc1a3 (ready, assignee=linguist)
     （已订阅——t_9fc1a3 完成或阻塞时会通知你）

… ~5 分钟后 …
bot> ✓ t_9fc1a3 已完成 (linguist)
     翻译了 README.md，保存到 docs/ja/README.md
```

**自动订阅**：从 Gateway 创建任务时，源会话自动订阅该任务的完成/阻塞事件。不用手动轮询。

**/kanban bypasses 运行保护**：即使当前会话的 AI 正在推理中，`/kanban` 命令也能立即执行——不会排队等待。因为看板存在 `~/.hermes/kanban.db` 中，不在正在运行的代理状态里。

---

## 配置要点

```yaml
# config.yaml
kanban:
  dispatch_in_gateway: true        # Dispatcher 嵌入 Gateway
  dispatch_interval_seconds: 60    # 扫描间隔
  dispatch_stale_timeout_seconds: 14400  # 4h 无 heartbeat 即回收
  failure_limit: 2                 # 连续失败 N 次后自动 Block
  max_in_progress: 5               # 同时最大运行任务数
  max_in_progress_per_profile: 2   # 每个 Profile 同时最多 N 个任务
  auto_decompose: true             # triage 中的任务自动分解
  auto_decompose_per_tick: 3       # 每 tick 最多分解 3 个任务
  auto_promote_children: true      # 子任务自动从 todo → ready
```

---

## 总结

Hermes Agent v0.16 的 Kanban Swarm 解决了一个核心工程问题：**当多代理协作需要持久化、可恢复、可审计时，什么样的架构才是对的？**

答案不是更复杂的代理间通信协议，而是一个简单的 SQLite 表 + 一个明确的 Worker 生命周期协议 + 一个可审核的 Gate 机制。

- **Worker 不 Shell 到 `hermes kanban`**——它们通过原生 Python 工具接口读写数据库
- **Dispatcher 不猜 Worker 在做什么**——Worker 显式调用 `kanban_heartbeat`、`kanban_complete`、`kanban_block`
- **Verifier 不信任任何 Worker 的自评**——它独立检查，Check Pass 前 Synthesizer 不会启动
- **Human 永远能介入**——在任何平台 `/kanban comment` 或 `/kanban unblock`，都实时生效

这套模式对 AI 开发者的启示是：**当你构建多代理系统时，把编排层和推理层解耦**。编排层（Dispatcher + Board）只关心状态机和调度，推理层（Worker Profile）只关心干活。中间用持久化数据库连接，而不是进程内消息传递。

这样，即使一个 Worker 进程崩溃了——看板记得一切，Dispatcher 会重新拉起它，它读到自己之前的工作，继续前进。
