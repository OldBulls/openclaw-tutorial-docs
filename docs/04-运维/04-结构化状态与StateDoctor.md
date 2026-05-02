---
title: 04-运维 · 04 结构化状态与 State Doctor
---

> 预计阅读：10 分钟
> 适用版本：OpenClaw 2026.4.24 稳定基线 · 最后审核：2026-05-02
> 前置：[00-健康检查](./00-健康检查.md) / [01-日志与监控](./01-日志与监控.md)
> 目标：看懂 `~/.openclaw/state/` 这层结构化状态文件，知道什么时候该看 `runtime-status`、什么时候该跑 `state-doctor`、什么时候该做 `state-migrate`

---

## 为什么现在要专门看 `state/`

OpenClaw 长期运行后，最容易出现的不是“完全没日志”，而是：

1. 信息太散：状态在日志、命令输出、临时终端窗口里各放一份
2. 事故难复盘：更新失败后，过几小时就忘了当时到底报了什么
3. 飞书卡片和支持包口径不一致：你看的是一套，脚本打包的是另一套

`~/.openclaw/state/` 这层就是为了解决这件事：

- 命令先写结构化 JSON
- 再按同一份数据生成 Markdown 摘要
- 再让卡片、支持包、状态首页复用这些结果

这样排障时就不是“每次重新猜一遍”，而是沿着同一条状态链往回看。

---

## 这层目录里最重要的 7 份文件

| 文件 | 用途 | 先在什么场景看 |
|---|---|---|
| `runtime-status.json` / `.md` | 当前运行时快照 | 你想先确认 gateway、插件、自动化任务是否正常 |
| `next-steps.json` / `.md` | 当前最推荐的下一步动作 | install / update / self-heal 刚跑完 |
| `maintenance-queue-status.json` / `.md` | 维护队列计数与 backoff 状态 | 怀疑任务卡住、想看何时自动重试 |
| `index.json` / `README.md` | 聚合后的状态首页 | 想一页看总览，不想自己拼多个文件 |
| `state-doctor.json` / `.md` | state 目录一致性检查结果 | 怀疑 JSON 损坏、字段缺失、聚合没刷新 |
| `state-migrate.json` / `.md` | 旧状态文件补字段、迁 schema 的结果 | 从旧 bundle 升上来，或怀疑状态文件还是旧格式 |
| `events.jsonl` | 最近发生过的结构化事件时间线 | 想回看“刚才到底发生过什么” |

这 7 份就是买家侧最常用的状态主链。

---

## 它们之间怎么串起来

一条典型链路大致是这样：

```text
install / update / self-heal / resume
  ↓
写入 result.json
  ↓
刷新 runtime-status / next-steps / maintenance-queue-status
  ↓
聚合成 state/README.md + index.json
  ↓
state-doctor 做一致性校验
  ↓
append-state-event.sh 追加到 events.jsonl
  ↓
需要时再打支持包或发状态卡片
```

这也是为什么现在不要只盯着某一条 shell 输出看。很多主链结束后，真正稳定可复用的结果已经写进 `state/` 了。

---

## `runtime-status`：先回答“系统现在活没活着”

第一眼建议先看：

```bash
bash ~/.openclaw/scripts/runtime-status-report.sh
```

这份摘要关注的是**运行时当前状态**，而不是历史。

它通常会告诉你：

- `openclaw` CLI 版本
- gateway 当前状态
- main 机器人的 `open_id` 是否就绪
- `memory-lancedb-pro` / `lossless-claw` / `openclaw-lark` 的启用情况
- `bundle update watch` / `weekly review watch` / `maintenance queue` 的加载状态
- 维护队列的摘要计数

如果你只想先回答“系统到底还在不在正常工作”，这份是第一入口。

---

## `next-steps`：先回答“现在最该做哪一步”

很多人会在安装或更新后直接翻日志，这是低效的。更高 ROI 的做法是先看：

```bash
cat ~/.openclaw/state/next-steps.md
```

这份摘要会把当前动作压成一句最短建议，例如：

- 当前动作是 `install` 还是 `update`
- `.new` 文件是不是需要人工合并
- 占位符是不是还没填完
- main 机器人的 `open_id` 是否就绪
- 现在优先执行哪条命令

也就是说，它不是“全部信息”，而是“当前最小下一步”。

---

## `maintenance-queue-status`：先回答“任务是不是堵住了”

如果你怀疑更新、自修复或维护动作没有继续推进，直接看：

```bash
bash ~/.openclaw/scripts/maintenance-queue-status.sh
```

它会告诉你：

- `pending / ready / delayed / processing / done / failed` 的数量
- 下一次自动重试时间
- 当前正在处理的任务
- 最近失败的任务和最近错误

这里最关键的不是总数，而是 **`delayed` 和 `failed`**：

- `delayed > 0`：一般是 backoff 中，不一定真挂了
- `failed > 0`：说明已经有明确失败任务，需要人工看

如果 `runtime-status` 说“自动化已加载”，但队列里长期堆了很多 `failed`，问题通常不在 launchd / cron，而在具体任务本身。

---

## `state-dashboard`：把多份结果收成一页

如果你不想自己在几个 JSON 文件之间跳来跳去，直接看：

```bash
cat ~/.openclaw/state/README.md
```

这页是 `state-dashboard.sh` 聚合出来的，重点包括：

- 当前动作
- gateway 状态
- 当前最推荐下一步
- 自动化任务状态
- 维护队列摘要
- 最近结果文件是否存在
- 最近失败事件
- 最近支持包路径

它适合“先总览，再下钻”。  
不适合拿它替代原始文件做精确比对。

---

## `state-doctor`：检查 state 目录自己有没有坏

最容易忽略的一点是：排障时，**状态文件本身也可能坏**。

这时要跑：

```bash
bash ~/.openclaw/scripts/state-doctor.sh
```

这份检查主要做 4 类事：

1. JSON 是否有效  
   例如 `runtime-status.json` 被截断、手工改坏、写入失败
2. 聚合是否完整  
   例如 `index.json` 没带上 `runtime_status` 或 `next_steps`
3. 队列摘要是否一致  
   例如 `runtime-status` 里的队列计数和 `maintenance-queue-status` 对不上
4. `events.jsonl` 是否有坏行  
   例如某一行不是合法 JSON，后续聚合就会出错

它的输出是 `PASS / WARN / FAIL` 三档：

- `PASS`：字段或结构正常
- `WARN`：缺文件、值异常、但还没到直接阻断
- `FAIL`：文件损坏或关键聚合失效，应该优先处理

如果 state-doctor 已经 `FAIL`，不要再相信基于这些状态文件做出来的卡片和支持包结果。

---

## `state-migrate`：旧 bundle 升上来时补齐 schema

旧版本 bundle 升上来，常见情况不是“完全没有文件”，而是：

- 老 JSON 缺 `schema_version`
- `next-steps.json` 缺 `action`
- `events.jsonl` 是旧格式，没有统一字段

这时先跑：

```bash
bash ~/.openclaw/scripts/state-migrate.sh --write
```

它做的事很保守：

- 只补默认字段
- 尽量不覆盖已有值
- 对事件流只追加 `schema_version`
- 记录迁移了多少文件、多少事件、哪些失败

它不是“修一切”的大锤。  
如果 JSON 已经坏到不能解析，`state-migrate` 会报失败，不会假装迁好了。

---

## `events.jsonl`：排障时间线比截图靠谱

很多故障其实不是“看不懂命令”，而是“记不清顺序”。  
这时 `events.jsonl` 非常有用：

```bash
tail -n 20 ~/.openclaw/state/events.jsonl
```

每一行通常会记录：

- 什么时候发生
- 动作是什么
- 状态是 success / warn / failed
- 简短消息
- 对应结果文件路径
- 是否生成了支持包

这份时间线最适合回答：

- 这次失败之前做过什么
- 失败后脚本有没有自动打支持包
- 最近一次 `self-heal` / `resume` 到底是否成功

当 `events.jsonl` 超过上限后，旧记录还会归档到 `events-archive/`。这比手工翻终端历史稳定得多。

---

## 推荐的读取顺序

多数场景下，按这个顺序最快：

1. `runtime-status.md`
2. `next-steps.md`
3. `maintenance-queue-status.md`
4. `state-doctor.md`
5. `events.jsonl`

也就是：

```text
系统现在状态
  ↓
现在先做哪一步
  ↓
后台队列堵没堵
  ↓
这些状态文件自己有没有坏
  ↓
最近到底发生过什么
```

如果你一上来就去翻 200 行日志，通常是在跳过更高信号的入口。

---

## 常用命令清单

```bash
# 1. 刷新运行时快照
bash ~/.openclaw/scripts/runtime-status-report.sh --write

# 2. 刷新维护队列摘要
bash ~/.openclaw/scripts/maintenance-queue-status.sh --write

# 3. 刷新聚合首页
bash ~/.openclaw/scripts/state-dashboard.sh --write

# 4. 做一致性检查
bash ~/.openclaw/scripts/state-doctor.sh --write

# 5. 旧 schema 补字段
bash ~/.openclaw/scripts/state-migrate.sh --write

# 6. 追最近事件
tail -n 20 ~/.openclaw/state/events.jsonl
```

---

## 红线：不要这样用 state

### ❌ 不要手改 `result.json` 伪装“成功”

这些文件会被 dashboard、飞书卡片和支持包复用。手改之后，你看到的是“假的绿灯”。

### ❌ 不要把 `README.md` 当唯一事实来源

它是聚合页，适合先看，不适合当最终精确证据。要精确比对，回到对应的原始 JSON。

### ❌ 不要跳过 `state-doctor`

怀疑状态异常时，先确认是“系统坏了”还是“状态文件坏了”。两者修法完全不同。

### ❌ 不要让 `events.jsonl` 无限增长

时间线是排障资产，但不是无限日志池。要保留归档策略，不要把它变成第二个 `gateway.log`。

---

## 验证清单

| 检查项 | 怎么验 |
|---|---|
| `runtime-status` 能生成 | `bash ~/.openclaw/scripts/runtime-status-report.sh --write` 返回 0 |
| `maintenance-queue-status` 能生成 | `bash ~/.openclaw/scripts/maintenance-queue-status.sh --write` 返回 0 |
| 聚合首页存在 | `~/.openclaw/state/README.md` 和 `index.json` 都存在 |
| `state-doctor` 能跑通 | `bash ~/.openclaw/scripts/state-doctor.sh` 输出 PASS/WARN/FAIL |
| 旧状态可迁移 | `bash ~/.openclaw/scripts/state-migrate.sh --write` 能写出结果 |
| 时间线可追 | `tail ~/.openclaw/state/events.jsonl` 能看到最近动作 |

---

## 下一步

- [05-支持包与脱敏排障](./05-支持包与脱敏排障.md) —— 状态文件会怎样被打进支持包
- [06-每周维护编排](./06-每周维护编排.md) —— 结构化状态如何配合周维护
- [05-排障/01-诊断流程](../05-排障/01-诊断流程.md) —— 真出故障时的系统化排查顺序

---

> **本章准确性保证**
> 本章对齐了当前模板里的 `runtime-status-report.sh`、`maintenance-queue-status.sh`、`state-dashboard.sh`、`state-doctor.sh`、`state-migrate.sh` 和 `append-state-event.sh` 的实际字段与行为，所有路径均按 `~/.openclaw/` 脱敏表达。

---

**导航**：[← 备份与恢复](./03-备份与恢复.md) · [📖 目录](../00-先读我.md) · [→ 支持包与脱敏排障](./05-支持包与脱敏排障.md)
