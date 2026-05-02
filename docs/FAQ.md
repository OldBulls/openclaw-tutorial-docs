---
title: FAQ · 高频问题速查
description: OpenClaw 实战手册高频问题速查，涵盖安装、升级、记忆、Agent 协作、gateway 和付费模板常见疑问。
---

> 这里只放**最高频**问题。详细排障流程看 [05-排障/](./05-排障/00-常见问题.md)。
> 提 issue 前先翻这里 + 用 ⌘F 搜关键词。

**导航**：[📖 教程目录](./00-先读我.md) · [05-排障 · 常见问题（详版）](./05-排障/00-常见问题.md) · [05-排障 · 诊断流程](./05-排障/01-诊断流程.md)

## 快速跳转

- [🚀 入门 / 安装](#-入门--安装) — Q1–Q6
- [🧠 记忆系统](#-记忆系统) — Q7–Q10
- [🤝 多 Agent](#-多-agent) — Q11–Q14
- [🚪 飞书 / 渠道](#-飞书--渠道) — Q15–Q17
- [🔧 升级 / 运维](#-升级--运维) — Q18–Q24

---

## 🚀 入门 / 安装

### Q1：完全没用过 OpenClaw，从哪开始读？
[00-先读我.md](./00-先读我.md) → [01-入门/00-概述.md](./01-入门/00-概述.md)。先花 5 分钟跑一遍 hello world，再回头看为什么。

### Q2：本教程和 OpenClaw 官方文档什么关系？
官方文档讲**怎么用**，本教程讲**作者怎么用 + 真实踩过的坑**。两份配合看。

### Q3：装 OpenClaw 报 Node 版本错？
要 Node 22.10+。`nvm install 22 && nvm use 22` 后重装。

### Q4：教程哪些章节适用我的版本？
看每章顶部的「适用版本」字段。当前稳定基线是 OpenClaw 2026.4.24，见 [先读我的版本说明](./00-先读我.md#版本说明)。

### Q5：付费档位 2 卖什么？
可直接 install 的模板包（42 篇公开教程配套的 SOUL / AGENTS / 维护脚本骨架）+ 1v1 答疑。详见 [chengzhen.vip](https://chengzhen.vip)。

### Q6：为什么模板不直接跟 npm 最新 OpenClaw？
因为模板不是只依赖 OpenClaw core，还依赖飞书插件、memory-lancedb-pro、本地补丁、gateway 服务脚本和买家安装链。当前稳定基线是 `2026.4.24`，要升更高版本先做兼容性评估。

---

## 🧠 记忆系统

### Q7：`memory_recall` 默认搜哪个 scope？
**按调用方 agent 的可访问 scope 过滤**，不是「只搜 main」。跨 agent 查必须显式传 `scope` 参数。详见 [02-配置/01-记忆系统 §5.3](./02-配置/01-记忆系统.md)。

### Q8：LanceDB fragments 爆涨（>500）怎么办？
跑 `~/.openclaw/scripts/run-weekly-maintenance.py` 的 step A（compact）。如果维护脚本本身没跑，看 [05-排障/00-常见问题](./05-排障/00-常见问题.md) 找「维护脚本静默失效」。

### Q9：agent A 写的记忆，agent B 能读到吗？
**默认不能**。要么写到 `global` scope，要么 B 显式传 `scope: 'A'` 且有授权。

### Q10：lossless-claw 压缩和 safeguard compaction 什么区别？
lossless-claw 是 65% 触发的智能 DAG 压缩（保细节）；safeguard 是兜底。两档关系看 [02-配置/01-记忆系统 §3.2](./02-配置/01-记忆系统.md)。

---

## 🤝 多 Agent

### Q11：什么时候用 `sessions_send`，什么时候用 `sessions_spawn`？
- 已有 session 继续聊 → `sessions_send`
- 新建持续对话 → `sessions_spawn(mode="session")`
- 一次性任务 → `sessions_spawn(mode="run")`
- 详见 [03-多Agent/00-协作拓扑 §委派](./03-多Agent/00-协作拓扑.md)

### Q12：派发失败 / 沉默丢弃怎么排？
查三处：① `allowAgents` 白名单 ② 目标 agent 是否启动 ③ gateway 是否 `^OK:`。详见 [05-排障/01-诊断流程](./05-排障/01-诊断流程.md)。

### Q13：sub-agent 能直接派给同级 sub-agent 吗？
**不能**。必须回报 main，由 main 决定是否派下一个。原因：避免死循环和上下文污染。

### Q14：agent 数量多少合适？
≤ 7 个。详见 [03-多Agent/00-协作拓扑 §原则 4](./03-多Agent/00-协作拓扑.md)。

---

## 🚪 飞书 / 渠道

### Q15：飞书 callback 收不到消息？
按这个顺序查：① gateway 状态 `^OK:` ② callback URL 配对 ③ encrypt key 一致 ④ 看 `~/.openclaw/logs/gateway.err.log`。详见 [02-配置/03-飞书深度配置](./02-配置/03-飞书深度配置.md)。

### Q16：发飞书卡片报 `Receiver_id_not_exist`？
绝大多数是 user_id_type 用错。`open_id` 和 `user_id` 不能混用。

### Q17：gateway 重启后立刻发卡片失败？
gateway 启动有 warming 阶段，必须轮询到 `^OK:` 再发。代码里：

```bash
until openclaw gateway status | grep -q '^OK:'; do sleep 2; done
```

---

## 🔧 升级 / 运维

### Q18：升级 OpenClaw 后 hotfix 全失配？
两个常见成因：① dist 哈希轮换（用 glob 替代死写文件名）② upstream 插了代码（缩短锚点）。详见 [04-运维/02-升级流程 §hotfix 锚点失效](./04-运维/02-升级流程.md)。

### Q19：每周维护脚本要不要自动跑？
要。launchd 或 cron 都行，但**别放 `/tmp/`**。脚本本身放 `~/.openclaw/scripts/`，编排器 step 缺失必须 `return False`（不能静默 skip）。

### Q20：备份多久跑一次？
- 每天：daily-digest 检查
- 每周：维护脚本（compact / consolidate / graphify rebuild）
- 升级前：必备份，详见 [04-运维/02-升级流程](./04-运维/02-升级流程.md)

### Q21：教程跟着 OpenClaw 升级吗？
跟踪，但不盲升。上游有新版本会先做兼容性评估；只有 OpenClaw core、飞书链路、memory-lancedb-pro、本地补丁和买家安装链都验证通过，才会提升模板基线。当前稳定基线仍是 `2026.4.24`。

### Q22：Control UI 打不开，提示 assets not found 怎么办？
先别反复重装。多数是 gateway 服务文件仍指向旧 nvm / npm 路径。执行：

```bash
bash ~/.openclaw/scripts/repair-gateway-service.sh --repair --restart
bash ~/.openclaw/scripts/runtime-status-report.sh
```

详见 [06-升级/03-CLI与Gateway版本统一](./06-升级/03-CLI与Gateway版本统一.md)。

### Q23：买家端在线更新为什么要先 review 再 approve？
`review` 负责下载并验证 `latest.json` 和签名，发现签名缺失或失败会进入 `blocked`；`approve` 才真正下载 ZIP 和更新。这样可以避免只看版本号就覆盖本地文件。

### Q24：memory-lancedb-pro 明明补了 15 秒，为什么还显示 5 秒？
看两处：源码补丁和 `openclaw.plugin.json` 清单默认值。当前模板会同时校验 `autoRecallTimeoutMs` 是否统一到 15000ms；如果只改源码没改清单，状态报告仍可能显示旧默认值。

---

## 还没解？

1. [05-排障/00-常见问题（详版）](./05-排障/00-常见问题.md) —— 完整问题树
2. [05-排障/01-诊断流程](./05-排障/01-诊断流程.md) —— 系统性排障方法
3. [开 issue](https://github.com/OldBulls/openclaw-tutorial-docs/issues/new) —— 描述 + 命令输出，越具体越快

---

**导航**：[📖 教程目录](./00-先读我.md)
