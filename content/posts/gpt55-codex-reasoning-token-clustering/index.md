---
title: "GPT-5.5 Codex 性能退化：推理 Token 聚类的隐性代价"
date: 2026-07-07
draft: false
tags:
  - GPT-5.5
  - Codex
  - LLM
  - 推理
  - AI
  - 性能分析
description: "深度解析 GPT-5.5 Codex 的推理 token 聚类如何导致复杂代码生成任务的性能退化，基于社区 500 次基准测试和 39 万条遥测数据。"
---

> 把同一个 prompt 发给 GPT-5.5 Codex，第一次出正常结果，第二次丢出明显不完整的错误答案。重试，又正常了。你以为撞上了什么间歇性幽灵 bug——但实际上，你遇到的是一个有精确数学结构的问题：**推理 token 被截断在 516/1034/1552 的固定阈值上**。

## 一个能复现异常的 prompt

先做一个实验。把下面这段话发给 GPT-5.5 Codex，关闭外部工具，让模型只靠推理回答：

```
Do not use external tools. A black bag contains candies with counts:
round apple 7, round peach 9, round watermelon 8; star apple 7,
star peach 6, star watermelon 4. Shape is distinguishable by touch
before drawing; flavor is not. What is the minimum number of candies
to draw to guarantee having apple and peach candies of different
shapes, i.e. round apple + star peach or round peach + star apple?
Give reasoning and final number.
```

正确答案是 **17**。但根据社区用户 `zuzululu` 在 Hacker News 上的 5 次 xhigh 模式重复测试，GPT-5.5 Codex 给出了 24、27、12、21、21——全部错误。更令人不安的是，每次运行的 `reasoning_output_tokens` 都精确落在 **516 token**，像被某种不可见的力量按了截断键。

这不是偶然。大规模遥测数据和成百上千次独立复现测试都指向同一个结论：GPT-5.5 Codex 的推理过程中存在一种称为 **Reasoning Token Clustering** 的系统性退化现象。

## 何为推理 Token 聚类？——技术原理

在大语言模型中，"推理 token"（reasoning token）是模型生成最终答案之前、内部思考阶段产生的 token——可以理解为模型"脑海中的草稿纸"。

正常情况下，推理 token 数量随任务复杂度自然变化：简单函数生成可能只需要 200 个，多文件重构可能需要 8000 个。但 GPT-5.5 Codex 完全不同：推理 token 强烈聚集在 **516、1034、1552、2070、2588**，间距精确为 **518 token**。

HN 用户 `tyingq` 提出了被社区广泛接受的假说：516 = 512 字节缓冲区 + 4 字节头，1034 = 516 + 512 + 4 字节头 + 2 字节链表指针，1552 = 1034 + 518。这看起来更像是**推理路由层的预算硬阈值**（budget cap），而非模型权重退化。

## 社区证据：39 万条遥测数据怎么说

OpenAI Codex GitHub Issue #30364 中，一位匿名用户提供了迄今为止最大规模的量化证据：**390,195 条响应记录、865 个会话**的遥测分析。核心数据如下：

| 指标 | 数值 |
|------|------|
| 分析的总响应记录 | 390,195 |
| GPT-5.5 精确 516 token 占比（≥516 的响应中） | **44.0%** |
| 非 GPT-5.5 精确 516 token 占比 | 1.3% |
| GPT-5.5 占所有精确-516 事件的比例 | **82.0%** |

44%——GPT-5.5 Codex 每两次 ≥516 token 的响应中，就有一次恰好截断在 516。对比：GPT-5.2 为 0.34%，GPT-5.3-codex 为 0.0%，GPT-5.4 为 19.8%。但更值得注意的是**月间趋势**：

| 月份 | 精确 516 / ≥516 | 平均推理 token |
|------|-----------------|---------------|
| 2026-02 | 0.11% | 268.1 |
| 2026-03 | 2.45% | 256.8 |
| 2026-04 | 4.25% | 228.7 |
| 2026-05 | **53.30%** | 106.9 |
| 2026-06 | 35.84% | 168.5 |

5 月峰值 53.3%，平均推理 token 从 2 月的 268 骤降至 106。6 月有所回升，但 35.84% 仍远高于年初水平。

Dev.to 作者 `onsen` 用基准数据集对比了 GPT-5 和 GPT-5.5 在五类任务上的准确率：

| 任务类型 | GPT-5 准确率 | GPT-5.5 准确率 | 退化 |
|---------|-------------|---------------|------|
| 单函数生成 | 94.2% | 93.8% | -0.4% |
| 简单 Bug 修复 | 96.1% | 95.7% | -0.4% |
| 多约束代码生成 | 87.1% | 79.3% | **-7.8%** |
| 递归算法设计 | 82.4% | 74.6% | **-7.8%** |
| 多文件重构 | 76.8% | 66.2% | **-10.6%** |

**简单任务几乎无退化，复杂任务退化显著**——这是推理 token 聚类区别于一般模型退化的关键特征：单函数生成不需要长推理链，516 token 足够；多文件重构需要跨文件追踪依赖，516 token 根本不够。

## 注意力干扰假说：聚类如何损害多步推理

在 chain-of-thought 推理中，每个推理 token 不仅是当前步骤的信息载体，还是后续步骤的注意力锚点。当推理链在 516 token 处被强硬截断，发生三种破坏：

1. **连贯性断裂**：模型失去将前置推理结果保持在注意力窗口中的能力。多文件重构中，文件 A 的接口变更无法传递到文件 B 的调用方修改。
2. **结论跳跃**：截断迫使模型从部分推理直接跳到最终答案。糖果问题中，模型正确枚举了前几个 case，但尚未排完所有条件就被迫输出——于是给出 24 而非 17。
3. **回溯失效**：截断后的 token 失去了对早期步骤的注意力链接。即使部分推理路径是完整的，模型也无法回到前几步修正一条错误的推理分支——因为截断恰好发生在它需要回溯的关键节点上。

这也解释了为何 516 token 短路径的答案并非纯随机错误：它们常在局部步骤上看似合理，但全局逻辑错误——正是截断式推理的典型症状。

## 实用对策：社区已验证的四条缓解路径

截至 2026 年 7 月，OpenAI 未就此事发表官方声明。但社区已经独立找到并验证了多条缓解路径。

### 方案一：移除 `## Intermediary updates` 系统提示（彻底修复）

多个独立验证者（`nsingh2`、`zuzululu`、`nickalaso`）发现，Codex 系统提示中的 `## Intermediary updates` 章节是导致短路的直接原因。该章节指导模型通过 `commentary` 通道产生中间更新，但模型将其误解为"应该提前终止推理"的信号。

移除该章节后，`nsingh2` 报告所有测试运行成功返回正确结果，`zuzululu` 评价"现在 GPT-5.5 的感觉完全回到了一个半月前"。

**操作方式**：从 Codex 的 `models.json` 中提取基础系统提示，删除 `## Intermediary updates` 章节，重新构建自定义 prompt。适合愿意付出配置成本的高级用户。

### 方案二：降级到 GPT-5.4 high（最省力）

用户 `matco11` 报告，GPT-5.4 high 在 3 周内保持 100% 可靠——因为它不受 516 token 截断影响。

**操作方式**：对于复杂多约束任务，显式指定 `model=gpt-5.4` + `model_reasoning_effort=high`。牺牲一部分"最新模型"的吸引力，换取结果可靠性。

### 方案三：temperature 调整 + 两阶段验证（低成本防御）

- 降低 temperature 到 0.2–0.4，减少 516 token 截断发生频率
- prompt 中要求"逐步推理，每一步写中间结论"，有缓解作用
- 两阶段验证：先让模型生成，再让它检查是否遗漏约束

### 方案四：使用检测钩子（被动防御）

`bentoner` 开发了 [codex-516-hook](https://github.com/bentoner/codex-516-hook)——一个 Codex CLI 钩子，检测到响应被截断在 516 token 时自动发出警告，让你能立即重试而不是盲目信任一个可能是错的答案。

你也可以用下面的 Python 脚本自行检测本地 Codex 日志中的推理 token 分布：

```python
import os, glob, re
import matplotlib.pyplot as plt

vals = []
codex_dir = os.path.expanduser("~/.codex")
for f in glob.glob(os.path.join(codex_dir, "**", "*"), recursive=True):
    if os.path.isfile(f):
        try:
            s = open(f, "r", encoding="utf-8", errors="ignore").read()
            vals += [int(x) for x in re.findall(
                r'"reasoning_output_tokens"\s*:\s*(\d+)', s)]
        except Exception:
            pass

plt.hist(vals, bins=200, range=(0, 5000),
         weights=[100 / len(vals)] * len(vals))
plt.xlabel("reasoning_output_tokens")
plt.ylabel("%")
plt.title("Your Codex Reasoning Token Distribution")
plt.show()
```

如果你的分布图在 516/1034/1552 附近出现了柱子，说明你也被这个 bug 影响了。

## 行业启示："智能倒退"的系统性风险

GPT-5.5 Codex 的推理 token 聚类揭示了一个被低估的风险：**能力提升与推理退化可能来自同一个优化动作**。社区推测，516 token 预算阈值是 OpenAI 为推理成本优化引入的路由规则——将预算上限设为 512 字节（+4 字节头）以减少计算开销。如果成立，这意味着降低成本的优化，意外牺牲了复杂任务的推理深度。

对依赖 LLM 的工程团队的三点启示：

1. **不要假设"新版本一定更好"**。GPT-5.5 在简单任务上的 94% 准确率不差，但如果你 80% 的场景是多约束代码生成，那 7.8% 的退化是实打实的回归。
2. **建立自己的退化检测流程**。不是靠"感觉变差了"，而是靠量化 benchmark。上面的 Python 脚本是起点；更系统性的做法是在 CI/CD 中集成一组代表性 prompt，每次 API 更新后自动跑并对比。
3. **区分"模型退化"和"平台退化"**。GPT-5.5 的权重本身可能没有退化——问题出在 Codex 的系统提示层。这意味着你的测试要覆盖实际使用的"包装层"，而非仅针对 API。

---

*本文基于 OpenAI Codex GitHub Issue #30364（390,195 条遥测记录）、Dev.to 社区基准数据集（5 类任务对比）、Hacker News 讨论帖（151 条评论，多用户独立复现）、Pi 项目 Issue #6278 等多个独立来源的交叉验证。所有量化数据均可追溯到原始讨论帖中的实验记录。*
