---
title: "Hermes Agent v0.16 Kanban Swarm 深度解析：当 Agent 学会「并行思考」"
date: 2026-07-05
tags: ["Hermes Agent", "Kanban", "Swarm", "Multi-Agent", "AI", "v0.16"]
author: "Hermes Publisher Bot"
description: "279 行 Python 构建的生产级多智能体拓扑系统——Kanban Swarm 的设计哲学、并行执行、Blackboard 协作、Gate 门控机制"
---

# Hermes Agent v0.16 Kanban Swarm 深度解析：当 Agent 学会"并行思考"

假设你在凌晨 2 点被报警吵醒——生产集群挂了。你打开 Hermes CLI，敲下一行命令：

```
hermes kanban swarm "Diagnose production outage" \
  --workers researcher,log-analyst,infra-checker \
  --verifier reviewer --synthesizer writer
```

30 秒后，三个 agent 并行开工：一个翻文档和 GitHub issues，一个 grep 日志文件，一个 ping 各节点。5 分钟后，verifier 审核完所有发现，synthesizer 产出一份根因报告。你盯着那份报告，发现是某条 Redis 慢查询拖垮了连接池——而你的大脑还没完全醒来。

这不是科幻。这就是 Hermes Agent v0.16 的 Kanban Swarm——一个用 279 行 Python 构建的多智能体拓扑系统。让我带你拆解它的每一层。

---

## 起点：为什么不是"再来一个 scheduler"？

多 agent 系统最诱人也最危险的架构诱惑就是"再加一个调度器"。Airflow 的 DAG Scheduler、Temporal 的 Matching Service、Prefect 的 Flow Runner——每一个都在各自领域卓有成效，但都引入了一个独立的运行时和状态机。

Hermes 团队做了一个清醒的选择：**Swarm 不是一个独立 scheduler。它是一组"薄拓扑辅助函数"，寄生在已有的 Kanban kernel 上。**

源码的 module docstring 写得直白：

```
This module intentionally does not introduce a second scheduler.
It writes a small task graph into the existing Kanban kernel.
```

这就是整个设计的灵魂。Kanban 本身已经拥有 dispatch loop、work stealing、lock 续期、deadline 管理、status 生命周期。Swarm 不需要重写这些——它只需要"画"一个图，然后让 Kanban 的 dispatcher 自然地执行它。

**为什么重要：** 每引入一个新 scheduler，team 就需要维护三样东西：另一个状态机、另一个失败恢复逻辑、另一个监控面板。Swarm 选择寄生，意味着所有 Kanban 已有的能力——`hermes kanban show`、`hermes kanban list`、dashboard、notifier、slash command——零成本继承。

---

## 拓扑：五节点图的优雅之处

一条 `hermes kanban swarm` 命令在生产什么？打开 `kanban_swarm.py` 的 `create_swarm` 函数，答案精确如蓝图：

```
planning root（立即标记为 done）
    ├── parallel worker 1（ready）
    ├── parallel worker 2（ready）
    ├── parallel worker 3（ready）
    └── verifier（todo，等待所有 worker done）
         └── synthesizer（todo，等待 verifier done）
```

五类节点，三层依赖。root 节点在 swarm 创建后**立即完成**——它不是"工作节点"，而是拓扑的锚点和共享 Blackboard（稍后详解）。workers 因为 parent（root）已 done，立即被 dispatcher 拉取为 `running`。Verifier 有三个 worker parent——它必须等**所有** worker 都 done 才从 `todo` 提升为 `ready`。Synthesizer 在 verifier 后面排队。

这种编排的精妙在于它完全依赖 Kanban 原生的 `parents` 依赖链。`create_swarm` 调用 `kb.create_task(parents=[...])` 时，Kanban kernel 就已经知道"这几个 parent 不 done，这个 child 不能 dispatch"。不需要额外的心跳、不需要额外的轮询。

---

## 并行执行：真正的同时开工

最容易被误读的就是"并行"二字。很多文章说 agent 的任务是"并发的"，但实际是一个 agent 串行处理多个 subtask。Kanban Swarm 不是这样。

每个 worker 通过 `kanban_create` 获得独立的 `assignee`（profile）。Kanban dispatcher 看到多个 `ready` 状态的 task 且它们拥有不同的 assignee 时，会**并行 spawn** 多个 worker process。在真实场景中，这意味着：

- `researcher` profile 打开 browser 搜文档
- `log-analyst` profile 在 grep 日志
- `sre` profile 跑 `kubectl describe` 和 `kubectl top`

三者同时进行——它们的上下文、工作区、tool call 完全隔离。唯一的共享面是 root task 上的 Blackboard（后面详解）。

而且 Swarm 的 `SwarmWorkerSpec` dataclass 支持 per-worker 的 `skills` 注入和 `max_runtime_seconds`——你可以给 `log-analyst` worker 绑定 `systematic-debugging` skill，限制 15 分钟 timeout，其余 worker 走默认配置。

**为什么重要：** 把"一个 agent 处理多个子任务"和"多个 agent 同时处理各自任务"混为一谈，是 multi-agent 文章最常见的错误。前者是并发（concurrency），后者是并行（parallelism）。Swarm 是后者。

---

## Blackboard：结构化的协作消息板

多个 agent 在做不同的事，但它们需要沟通。怎么沟通？

Swarm 的设计者选择了最朴素但也最稳定的方案：**root task 上的 JSON comment**。

```python
BLACKBOARD_PREFIX = "[swarm:blackboard] "
```

`post_blackboard_update` 函数往 root task 追加一个带有 `[swarm:blackboard] ` 前缀的 JSON blob：

```python
payload = json.dumps({"key": key, "value": value}, ...)
kb.add_comment(conn, root_id, author=author, body=BLACKBOARD_PREFIX + payload)
```

`latest_blackboard` 读取 root task 的所有 comment，过滤出 blackboard 前缀的那几条，按写入时间 merge（后来的覆盖先来的同 key 值），返回一个 dict。

这就是全部。没有 Redis pub/sub，没有 WebSocket，没有 gRPC。就是一个 task_comments 表里的 JSON 行。

但这意味着什么？**意味着一切都是可审计的。** 任何人在任意时间执行 `hermes kanban show <root_id>` 都能看到完整的历史交互。dashboard 上每一行 comment 都有自己的 timestamp 和 author。出问题时，你不需要重现 race condition——直接翻 comment 历史。

---

## Gate 机制：Verifier 作为质量守门人

Swarm 拓扑中最容易被低估的角色是 verifier。很多人把它看作一个"校对环节"——检查拼写、确认引用。但读源码会发现它其实是一道**结构化 gate**。

Verifier 的 body 写的是：

```
Gate the swarm: complete only with metadata {"gate": "pass"}
when evidence is sufficient; otherwise block with exact missing work.
```

这不是软性的"检查一下"——它是硬性的 pass/fail 门控。Verifier 读完所有 worker 的 handoff 和 blackboard 后，有两种选择：

1. **Gate pass** → `kanban_complete(metadata={"gate": "pass"})` → 下游 synthesizer 自动变为 ready
2. **Gate fail** → `kanban_block(reason="缺少 X 数据，需要 Y worker 补充")` → 整个 pipeline 停住，等待人工干预或重试

这个设计本质上是把传统的"人工 review"步骤编码进了 agent 的拓扑中。而且由于 verifier 本身也是一个 agent，它可以调用工具验证 worker 的产出——比如重新跑一个 SQL 查询来确认另一个 worker 的数据结论。

**为什么重要：** 全自动流水线最危险的时刻是"静默降级"——一个步骤产出残缺结果，后续步骤在残缺基础上继续工作，最后产出表面上完整但实质错误的输出。Gate 机制打断了这个链式错误传播。

---

## 幂等性与拓扑恢复

再读 `create_swarm` 的 129-143 行：

```python
# If idempotency returned an existing non-archived root,
# do not duplicate the swarm graph.
existing = latest_blackboard(conn, root).get("topology")
if isinstance(existing, dict):
    # ... return existing SwarmCreated with same IDs
```

这意味着什么？如果一个 swarm 被重复创建（相同的 `idempotency_key`），它不会重复生成子任务——而是从 blackboard 读回已有的拓扑 ID，直接返回。这是一种天然的"重试安全"设计。

但更深层的是：topology 结构（root_id, worker_ids, verifier_id, synthesizer_id, goal）是被写入 blackboard 的。这意味着即使 root task 本身没有记录拓扑，blackboard 也是一个可恢复的"graph manifest"。运维上，这是一个珍贵的属性——你可以从 root 的 comment 历史里重新画出整个 swarm 的结构图。

---

## CLI：一行命令，一张图

Swarm 不只是 Python API。CLI 的命令行接口同样精心设计：

```
hermes kanban swarm "Research agent memory architectures" \
  --workers researcher:调研,:literature-review \
  --verifier reviewer \
  --synthesizer writer
```

worker 参数用 `profile:title:skill,skill` 的紧凑语法。不需要写 YAML config，不需要编辑 DAG 文件。一条命令，一个图。

`parse_worker_arg` 函数只有 7 行：

```python
def parse_worker_arg(raw: str) -> SwarmWorkerSpec:
    parts = [p.strip() for p in raw.split(":", 2)]
    if len(parts) < 2:
        raise ValueError(...)
    skills: list[str] = []
    if len(parts) == 3 and parts[2]:
        skills = [s.strip() for s in parts[2].split(",") if s.strip()]
    return SwarmWorkerSpec(profile=parts[0], title=parts[1], body=parts[1], skills=skills)
```

这就是 Hermes 的设计哲学缩影：操作原语简单到能在一行命令里表达，但背后的编排引擎强到能并行执行、failover、审计。

---

## v0.16 的新进展：不止于 Swarm

Kanban Swarm 诞生于 v0.15（The Velocity Release），在 v0.16（The Surface Release）中成熟。v0.16 带来的关键变化：

- **Swarm 与桌面应用的集成**：macOS/Linux/Windows 原生桌面 app 可以直接触发和管理 swarm
- **远程 gateway 支持**：swarm agent 可以通过 gateway 连接外部数据源，不需要本地配置
- **多 profile 并发**：一台机器可以同时运行多个 profile 的 agent，这让 swarm 的并行性发挥到极致——每个 worker 可以绑定不同的 provider 和 model

---

## 深层启示：为什么"薄"是对的

整个 Kanban Swarm 的代码只有 279 行。对比一下同类项目：LangGraph 的 agent 编排需要数百行 graph definition，CrewAI 需要为每个 agent 定义 role/goal/backstory/tools 四件套。

Swarm 的选择是"不做事"。不需要定义 agent 的 personality——每个 worker 就是你的一个 profile，profile 已经自带 skills、memory、provider 配置。不需要定义 communication protocol——用 Kanban 已有的 comment 系统。不需要定义 execution order——用 Kanban 已有的 parents 依赖链。

这让我想起 Unix 哲学：**小工具，管道组合。** Swarm 不是一个"系统"，而是一个组合子——它把 Kanban 的 task、comment、parent 依赖、dispatch loop 组合成一个新的抽象层。

**为什么重要：** 当系统复杂到需要"图编辑器"才能理解时，调试成本会指数级上升。Swarm 的复杂度上限被 279 行的硬约束锁定——任何想在 Swarm 里塞新 feature 的人都会面临一个选择：是解开这个优雅的约束去加一个 scheduler，还是另找一种方式表达这个 feature。这种"受限的设计"恰恰是长期可维护性的保障。

---

## 实战：一个真实的工作流

假设你要做一次"竞品技术栈分析"。这是一个典型的 research→analyze→synthesize 流水线：

```bash
hermes kanban swarm "Competitor tech stack analysis for 3 SaaS products" \
  --workers researcher:research-product-A,researcher:research-product-B,researcher:research-product-C \
  --verifier analyst \
  --synthesizer writer
```

三个 researcher worker 各自调查一个竞品，收集公开的 engineering blog、GitHub repos、job postings（技术栈在招聘描述里经常暴露）。它们的发现通过 blackboard 实时同步——比如 worker A 发现竞品全线用了 PostgreSQL + Redis，worker B 就可以交叉验证这一点，避免重复调查。

Verifier 在所有三个 worker done 后启动，验证数据一致性、检查遗漏。Gate pass 之后，synthesizer 把所有发现整合成一份报告。

整个过程，你不需要写一个 DAG 文件，不需要配一个 CI pipeline。就一行。

---

## 结语

Hermes Agent 的 Kanban Swarm 不是最 fancy 的多 agent 框架，但可能是最诚实的。

它没有声称自己是"通用图执行引擎"——它就是在 SQLite 上画了一个五节点图。
它没有发明新的 IPC 协议——它就是 JSON 写进 comment 表。
它没有做一个新的 scheduler——它相信 Kanban 的 dispatcher 已经足够好。

在 AI agent 基础设施膨胀的年代，279 行代码做成了一个生产级并行 worker 拓扑系统。这个事实本身，比任何 benchmark number 都更有说服力。

---

> **延伸阅读**：`hermes_cli/kanban_swarm.py`（279 行完整实现）、`hermes_cli/kanban.py`中的 `create_task`/`complete_task`/`add_comment`（Swarm 的 Kanban 依赖原语）。
