---
title: 04-运维 · 08 自定义脚本与 Cron 治理
---

> 预计阅读：14 分钟
> 适用版本：OpenClaw 2026.4.24 稳定基线，适合已经写了 `~/.openclaw/scripts/` 的用户
> 前置：[04-运维/00-健康检查](./00-健康检查.md) / [04-运维/01-日志与监控](./01-日志与监控.md)
> 本章回答：**为什么自定义运维脚本会把系统“修坏”？怎么避免 config 漂移、重复告警、后台任务互相打架？**

---

## 先说结论

OpenClaw 出问题时，很多锅不是 gateway 本体的，是你自己那层“自愈脚本 + cron”造成的：

- baseline 恢复脚本把新配置抹回旧值
- 告警脚本和 `auto` 入口双跑
- 后台 review 任务拿主会话当工作队列
- cron PATH 和交互 shell PATH 不一致，导致工具在 cron 里“凭空消失”

所以治理脚本时，核心不是“多写几个守护”，而是：

> **每个自动化动作都必须有明确所有权、边界和退出条件。**

---

## 最常见的四类事故

### 事故 1：baseline 覆盖生产配置

症状：

- 你明明刚把 `openclaw.json` 改好
- 半小时后又被改回旧值
- gateway 重启后回到老模型 / 老 dbPath / 老 fallback

根因通常是某个“修复脚本”拿 baseline 做整段覆盖。

### 事故 2：重复告警

症状：

- 同一类告警从两个入口同时发
- `alert_last_run` 冷却不可信
- 你搞不清到底是 `alert-notify` 还是 `openclaw-manage auto` 在报

### 事故 3：后台任务抢主会话

症状：

- WebUI 和飞书一起不回
- 日志里是 `session file locked`
- 背景任务越多，主控越容易卡

### 事故 4：cron 环境和手工环境不一致

症状：

- 手工跑脚本正常
- cron 跑就说 `lsof` / `openclaw` / 依赖工具不存在

根因是 cron 的 PATH 比交互 shell 短很多。

### 事故 5：gateway 服务路径漂移

症状：

- `openclaw --version` 正常
- launchd / systemd 里的 gateway 仍指向旧 nvm 或 npm 全局目录
- 重启后 `dist/index.js` 不存在，Control UI 打不开

根因通常是升级、重装 Node、清理 nvm 版本、切 npm prefix 后，没有同步修服务文件。

---

## 脚本治理的四条硬规则

### 规则 1：配置修复只允许“补缺省”，不允许“整段恢复”

危险做法：

```text
if current != baseline:
  config.memory = baseline.memory
```

安全做法：

```text
只补 baseline 里有、当前配置里缺的字段
显式保留用户最近修改过的字段
```

尤其是这几类生产字段，不能被 baseline 粗暴覆盖：

- `dbPath`
- `fallback`
- `primary model`
- `retrieval.rerank*`
- `hooks.allowConversationAccess`

### 规则 2：同一类动作只能有一个 cron owner

比如“告警检查”只能选一种：

- 要么独立 `alert-notify.sh` cron
- 要么 `openclaw-manage.sh auto` 内嵌告警

不要两条都开。

### 规则 3：后台任务不能默认复用主会话

像 background review、周审、批量修复这类动作，应该：

- 用自己的 `session-id`
- 有独立 lock
- 有独立 pending/running sentinel

否则很容易把 `agent:main:main` 锁死。

### 规则 4：cron 脚本要显式兜住 PATH 差异

不要默认假设 cron 能找到：

- `lsof`
- `openclaw`
- `node`
- `python3`

最稳的做法：

- 命令写绝对路径
- 或脚本里先探测 `/usr/sbin/lsof` 这类系统路径

### 规则 5：会重启 gateway 的脚本必须先检查服务路径

只要脚本会执行：

```text
openclaw gateway restart
launchctl kickstart
systemctl restart
```

就应该先确认服务文件里引用的 Node 和 OpenClaw 仍然存在。当前模板里的推荐入口是：

```bash
bash ~/.openclaw/scripts/repair-gateway-service.sh --check
```

如果检查失败，再显式执行：

```bash
bash ~/.openclaw/scripts/repair-gateway-service.sh --repair --restart
```

不要让高频 cron 在坏路径上反复重启，这只会制造更多日志噪音。

---

## 一套审计方法

先列 cron：

```bash
crontab -l
```

再找所有会动配置、重启服务、发送告警的脚本：

```bash
rg -n 'openclaw\\.json|gateway restart|gateway stop|launchctl|crontab|alert' \
  ~/.openclaw/scripts -g '*.sh' -g '*.mjs' -g '*.js'
```

然后逐个回答这 5 个问题：

1. 它会不会写 `openclaw.json`？
2. 它会不会重启 gateway？
3. 它是不是唯一 owner？
4. 它有没有 lock / cooldown / dry-run？
5. 它失败时会不会放大别的系统噪音？
6. 它重启 gateway 前有没有检查服务路径？

---

## 推荐的脚本分层

### 第 1 层：只读型

只负责读状态，不改系统：

- 健康检查
- 日志摘要
- 每日 digest
- 统计脚本

### 第 2 层：低风险修复式

只改可逆的小状态：

- 补缺省字段
- 清 stale lock
- 清 pending sentinel
- 重建软缓存

### 第 3 层：高风险动作

这类脚本必须显式触发或至少强保护：

- 恢复 baseline
- 批量改模型
- 改 secrets 来源
- 停 / 重启 gateway
- 清理活动会话索引

高风险动作不要藏在“日常 auto”里悄悄做。

---

## cron 设计建议

### 建议 1：把 `auto` 当“主编排器”，不是“垃圾桶”

`auto` 适合做：

- 健康检查调度
- monitor 节流
- maintenance 窗口控制

不适合做：

- 任意配置回滚
- 强侵入恢复
- 批量会话修理

### 建议 2：所有后台任务都要有“最近活跃”门槛

典型例子就是 background review。

不要单纯按时间：

```text
每 5 分钟 +1
累计到 5 就触发
```

更稳的是：

```text
最近 N 分钟有非 cron 活跃会话
才允许计数和触发
```

### 建议 3：所有重启都要区分“启动宽限期”

服务刚重启时 `/health` 不稳定很常见。  
不要一看到一两次失败就立刻再次重启。

---

## 一个可复用的最小 checklist

- [ ] 只有一条告警主链
- [ ] 所有会写配置的脚本都只补缺省，不整段恢复
- [ ] 高风险动作不放进高频 cron
- [ ] 后台任务不用主会话 id
- [ ] 有 lock 文件
- [ ] 有 cooldown
- [ ] 有 dry-run 或最小验证模式
- [ ] cron 依赖命令有绝对路径或 fallback
- [ ] gateway 重启前会检查 launchd / systemd 指向
- [ ] baseline 会随着已验证配置同步更新

---

## 一句话经验

> **OpenClaw 的自愈脚本如果没有边界，最后最需要修复的就是自愈脚本本身。**

脚本越多，越要先治理“谁负责什么”，而不是继续加新守护。
