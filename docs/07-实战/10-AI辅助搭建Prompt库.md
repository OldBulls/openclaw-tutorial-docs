---
title: AI 辅助搭建 Prompt 库
---

> 目标：给你一套“让 AI 帮你搭 OpenClaw”的分阶段 prompt。重点是拆任务、设边界、先 dry-run，而不是一次性生成所有代码。

---

## 使用原则

和 AI 协作搭 OpenClaw 时，最容易犯的错是：

```text
帮我搭一个完整 OpenClaw 多 agent + 飞书 + 记忆 + 卡片系统。
```

这个 prompt 太大，AI 容易生成看似完整但不可维护的东西。正确做法是分阶段，每阶段都有明确产物和验证命令。

---

## 阶段 1：设计职责边界

```text
我想用 OpenClaw 搭一个多 agent 系统，业务场景是【填写】。
请设计 main、planner、worker、reflector 的职责边界。
要求：
1. 每个 agent 最多 5 条职责。
2. 每个 agent 写清楚“不该做什么”。
3. 给出工具权限建议。
4. 给出 shared/roles.md 草稿。
5. 不要生成安装脚本。
```

验收：你能清楚说出每个 agent 为什么存在。

---

## 阶段 2：写 shared 规则

```text
基于上面的职责设计，帮我生成 OpenClaw shared 目录的规则文档。
文件包括：roles.md、glossary.md、style-guide.md、safety.md、workflow.md。
要求：
1. 内容简洁，每个文件不超过 80 行。
2. 规则可被多个 agent 共用。
3. safety.md 必须包含禁止自动执行的动作。
4. 不要包含真实密钥、手机号、客户信息。
```

验收：规则能被人读懂，不像 prompt 垃圾场。

---

## 阶段 3：设计记忆候选格式

```text
帮我设计 OpenClaw 记忆候选 Markdown 格式。
要求：
1. 分为高价值、待审阅、低价值三层。
2. 每条候选包含 hash、agent、标题、内容、来源、建议目标文件、风险。
3. 给出 5 条示例。
4. 给出解析时应校验的字段。
```

验收：候选能被脚本稳定解析，不依赖自然语言猜测。

---

## 阶段 4：生成卡片 JSON

```text
基于这个候选记忆 JSON，帮我生成飞书 CardKit 2.0 周审卡片。
要求：
1. header 显示日期和状态。
2. body 包含三层统计、热门主题、折叠明细。
3. 按钮包括：合并高价值、合并选中、取消、回滚预览。
4. 每个按钮 value 必须包含 action、task_id、tier、hashes、card_meta.issued_at。
5. 卡片超过 30KB 时给出缩减策略。
6. 先解释结构，再输出 JSON。
```

验收：`json.tool` 能解析，按钮 value 结构正确。

---

## 阶段 5：写发送脚本

```text
帮我写 Python 脚本发送飞书 CardKit 2.0 卡片。
要求：
1. 支持 --dry-run。
2. 从环境变量读取 app_id、app_secret、receive_id。
3. 先创建 card，拿到 card_id 后再发送消息。
4. 失败时输出 stage、HTTP status、响应摘要。
5. 日志必须脱敏，不打印完整 token。
6. 给出本地验证命令。
```

验收：dry-run 不访问网络，真实发送失败时能看出失败阶段。

---

## 阶段 6：写回调 handler

```text
帮我写飞书卡片 action handler。
要求：
1. 只允许白名单 action。
2. 不允许 eval、sh -c、执行用户传入 command。
3. 校验 task_id、issued_at、过期时间。
4. 危险动作需要管理员或二次确认。
5. 每次点击写 JSONL 审计日志。
6. 给出至少 8 个单元测试 case。
```

验收：未知 action 会拒绝，过期卡片不会执行危险动作。

---

## 阶段 7：写验证脚本

```text
帮我写一个 verify-weekly-card.sh。
要求检查：
1. Python 脚本语法。
2. Node handler 语法。
3. 卡片 dry-run 能生成 JSON。
4. 卡片 JSON 可解析。
5. 按钮 value 包含 card_meta。
6. 合并脚本 --dry-run 能运行。
7. 回滚脚本 --dry-run 能运行。
8. handler 单元测试通过。
```

验收：每次上线前一条命令跑完。

---

## 阶段 8：让 AI 分析日志

```text
下面是 OpenClaw 的脱敏日志。
请按入口、gateway、插件、agent、模型、飞书卡片六层分析。
要求：
1. 判断最可能的失败断点。
2. 给出最多 3 个原因。
3. 给出下一步命令。
4. 不要让我提供完整 token、appSecret、open_id。
```

验收：AI 输出的是下一步动作，而不是泛泛解释。

---

## 反向 prompt：让 AI 自查

每次 AI 生成代码后，追加：

```text
请自查你刚才的方案：
1. 是否存在泄露密钥风险？
2. 是否执行了用户传入的任意命令？
3. 是否缺少 dry-run？
4. 是否缺少失败日志？
5. 是否有不可回滚的状态变更？
6. 是否依赖不存在的 OpenClaw 字段或路径？
请列出问题并修正。
```

这一步很有用，尤其是 handler 和脚本类代码。

---

## 最小推进节奏

一天内建议只完成一层：

| 天数 | 目标 |
|---:|---|
| Day 1 | agent 职责 + shared 规则 |
| Day 2 | 记忆候选格式 + 解析 dry-run |
| Day 3 | 卡片 JSON + 本地验证 |
| Day 4 | 真实飞书发送 |
| Day 5 | 回调 handler + 单元测试 |
| Day 6 | 合并 / 回滚 dry-run |
| Day 7 | 全链路 smoke test |

不要把 Day 1 到 Day 7 压成一个 prompt。
