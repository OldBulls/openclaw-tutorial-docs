---
title: 07-实战 · OpenClaw 日常运维 SOP
---

> 目标：让 OpenClaw 长期稳定运行。不是出问题才看日志，而是每天、每周、升级前后都有固定检查动作。

---

## 一句话原则

OpenClaw 是常驻系统，不是一次性脚本。常驻系统要有：

- 健康检查
- 日志观察
- 备份策略
- 升级回滚
- 权限审计
- 成本监控

先用人工 SOP 跑顺，再自动化。

---

## 每日 3 分钟检查

```bash
openclaw gateway status
openclaw doctor
openclaw models status --json >/tmp/openclaw-models-status.json
 tail -n 80 ~/.openclaw/logs/gateway.err.log
```

看这些：

| 项目 | 正常状态 |
|---|---|
| gateway | running / reachable |
| doctor | 无阻断错误 |
| models | default 和 fallback 都可解析 |
| gateway.err.log | 没有持续重复错误 |
| 飞书 bot | 能收到消息并回复 |

如果只是偶发 warning，不要立刻大改配置；先记录时间和上下文。

---

## 每周 15 分钟检查

适合周审卡片前后做：

```bash
# 记忆候选是否过多
find ~/.openclaw -path '*memory*' -type f -maxdepth 6 2>/dev/null | head

# 日志大小
find ~/.openclaw/logs -type f -maxdepth 1 -print0 | xargs -0 ls -lh

# 最近错误
rg -n "error|failed|exception|timeout|unauthorized" ~/.openclaw/logs -S | tail -80
```

每周重点：

- 候选记忆有没有合并或清理
- 长期记忆有没有备份
- 飞书卡片按钮是否还有过期卡被点击
- fallback 是否频繁触发
- 某个 agent 是否持续报同一种错

---

## 升级前 SOP

升级前先备份，不要直接 `npm install -g` 后祈祷。

```bash
openclaw --version
openclaw doctor
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak.$(date +%Y%m%d-%H%M%S)
cp -R ~/.openclaw/agents ~/.openclaw/agents.bak.$(date +%Y%m%d-%H%M%S)
```

如果你有付费模板或自定义脚本，也备份：

```bash
cp -R ~/.openclaw/scripts ~/.openclaw/scripts.bak.$(date +%Y%m%d-%H%M%S)
cp -R ~/.openclaw/extensions ~/.openclaw/extensions.bak.$(date +%Y%m%d-%H%M%S)
```

---

## 升级后 SOP

```bash
openclaw --version
openclaw doctor
openclaw models list
openclaw gateway restart
openclaw gateway status
```

然后做 3 个冒烟测试：

1. CLI 问 main agent 一句简单问题
2. 飞书私聊 bot 一句简单问题
3. 如果有卡片系统，跑一次卡片 dry-run

```bash
python3 ~/.openclaw/scripts/sunday-memory-digest.py --dry-run 2>/tmp/digest.err || cat /tmp/digest.err
```

升级后不要马上删备份，至少保留一周。

---

## 日志轮转建议

日志过大后，排障会变慢，也容易把敏感内容留太久。简单策略：

```bash
mkdir -p ~/.openclaw/logs/archive
for f in ~/.openclaw/logs/*.log; do
  [ -f "$f" ] || continue
  size=$(wc -c < "$f")
  if [ "$size" -gt 10485760 ]; then
    gzip -c "$f" > "~/.openclaw/logs/archive/$(basename "$f").$(date +%Y%m%d).gz"
    : > "$f"
  fi
done
```

真实使用时建议写成脚本，并先 dry-run。

---

## 什么时候该重启 gateway

适合重启：

- 修改插件 enabled 状态
- 修改飞书 appSecret / encrypt key
- 升级 OpenClaw CLI 或插件
- 模型 provider 配置变更后运行异常
- gateway 日志显示插件初始化失败

不一定需要重启：

- 改文档内容
- 改候选记忆 Markdown
- 只跑一次 dry-run 脚本
- 单个模型请求 timeout

重启前先看状态，重启后再看状态：

```bash
openclaw gateway status
openclaw gateway restart
openclaw gateway status
```

---

## 运维告警分级

| 级别 | 例子 | 处理 |
|---|---|---|
| P0 | gateway 起不来、飞书完全不可用 | 立即回滚最近改动 |
| P1 | 主模型不可用但 fallback 可用 | 切模型或修 provider |
| P2 | 卡片按钮失败但文本可用 | 暂停卡片危险动作 |
| P3 | 周审卡片生成失败 | 手动跑 dry-run 排查 |
| P4 | 偶发 timeout | 观察，记录频率 |

不要把所有 warning 当 P0，否则系统维护会变成噪音。

---

## AI 辅助每日巡检 prompt

```text
以下是我今天的 OpenClaw 巡检摘要，敏感信息已脱敏。
请按 P0-P4 分级，并给出下一步操作。
要求：
1. 不要建议我大范围重装。
2. 先判断是否需要立即处理。
3. 给出最多 5 条命令。
4. 标出哪些问题可以观察。
```
