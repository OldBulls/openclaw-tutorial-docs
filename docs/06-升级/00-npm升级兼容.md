---
title: 06-升级 · 00 npm 升级兼容
---

# 06-升级 · 00 npm 升级兼容

> 预计阅读：12 分钟
> 本章回答：**升级 OpenClaw 之后为什么总出毛病，以及怎么把升级做成流水线**

---

## 升级为什么特别容易崩

OpenClaw 在快速迭代期。每次 `npm install -g openclaw` 可能带来：

| 变化 | 影响 |
|---|---|
| CLI 行为变（新命令 / 旧参数语义变） | 自动化脚本挂 |
| config schema 小变动 | openclaw.json 的旧字段被忽略或报警告 |
| 内部模块重构（dist 打包结构调整） | 对 dist 做过的补丁找不到锚点 |
| 默认值调整（如超时、缓存策略） | 原本好用的配置开始边缘问题 |

作者维护一份 `~/.openclaw/scripts/apply-openclaw-cli-hotfixes.mjs`，专门吸收这些"官方还没吸收但用户需要"的改动。

---

## hotfix 是什么

**定义**：对 OpenClaw CLI 的 dist 做**运行时补丁**的脚本。不是改源码（你没有源码），而是定位到 dist 里的某段代码，字符串替换。

**为什么需要**：

1. 有些行为调整作者**不想等官方发版**（如禁掉某个 bundled 的扩展以避免和本地插件冲突）
2. 有些新版本**意外破坏**了作者依赖的行为，需要回到老行为
3. 有些**小个性化**官方不会接受（如某个 toast 文案、某个默认值）

**反面问题**：

- dist 更新会让锚点失效
- 不同电脑 / 不同版本要不同的 hotfix 集合

---

## apply-openclaw-cli-hotfixes 的结构

```javascript
const targets = [
  { name: "config materialization",              locate: ..., patch: ... },
  { name: "plugin CLI bootstrap",                locate: ..., patch: ... },
  { name: "runtime plugin registry guard",       locate: ..., patch: ... },
  { name: "command bootstrap fast config routes", locate: ..., patch: ... },
  { name: "bootstrap fast config commands",       locate: ..., patch: ... },
  { name: "config CLI fast paths",                locate: ..., patch: ... },
  { name: "run-main fast config routes",          locate: ..., patch: ... },
  { name: "bundled Feishu disable guard",         locate: ..., patch: ... },
  { name: "status text security audit gating",    locate: ..., patch: ... },
  { name: "status text lightweight default path", locate: ..., patch: ... },
  { name: "model tool-input compat fallback",     locate: ..., patch: ... },
  { name: "assistant error ui compat fallback",   locate: ..., patch: ... },
  { name: "assistant runtime compat fallback",    locate: ..., patch: ... },
  { name: "embedded payload compat error suppression", ... },
  { name: "openclaw-lark rollback toast text",    locate: ..., patch: ... },
];
```

15+ 个补丁，大致分四类：

| 类 | 占比 | 说明 |
|---|---|---|
| **性能类** | ~30% | config fast paths、lightweight default path —— 让冷启快些 |
| **兼容类** | ~40% | model tool-input fallback、payload compat suppression —— 吸收 upstream breaking |
| **个性化** | ~20% | bundled Feishu disable guard、lark rollback toast —— 跟本地插件适配 |
| **守护类** | ~10% | runtime plugin registry guard、security audit gating —— 防炸逻辑 |

---

## 升级工作流（标准五步）

### 步骤 1：升级前打备份

```bash
# 备份整个 ~/.openclaw/（排除大文件）
bash ~/.openclaw/scripts/backup-memory-plugin-upgrade.sh
# 会生成 ~/Desktop/openclaw-backup-YYYYMMDD-HHMM.tar.gz

# 记录当前 CLI 版本
openclaw --version > /tmp/openclaw-version-before.txt
```

### 步骤 2：升级

```bash
npm install -g openclaw
openclaw --version
```

### 步骤 3：跑 hotfix（⚠️ 必做）

```bash
node ~/.openclaw/scripts/apply-openclaw-cli-hotfixes.mjs
```

**正常输出**：每个 target 一行 `✅ patched` 或 `= already patched`。

**异常输出**：`❌ failed: anchor not found` —— 就是锚点失效，进入下一节。

### 步骤 4：回归检查

```bash
bash ~/.openclaw/scripts/check-memory-plugin-upgrade.sh
# 脚本会检查：配置文件完整、关键 skill 存活、gateway 能起
```

### 步骤 5：重启 + 冒烟

```bash
openclaw gateway restart
# 等完全 ready
until bash ~/.openclaw/scripts/check-feishu-gateway-health.sh --summary 2>/dev/null | grep -q "^OK:"; do
  sleep 2
done

# 冒烟：在飞书发一条消息，走 01-入门/03-验证 的八维速通一遍
```

全过 → 升级成功。
任何一步挂 → 回滚（下一节）。

---

## 锚点失效怎么处理

**症状**：hotfix 脚本报 `anchor not found for target "X"`。

**根因** 两种：

### 成因 1：dist 哈希轮换

OpenClaw 打包会把大 chunk 命名成 `chunk-<hash>.js`。哈希变了，**硬编码路径**的 locate 函数找不到文件。

**作者的脚本怎么扛**：用 `findTopLevelDistFile([锚点 A, 锚点 B])` 这种**按内容查找**的方法，而不是按文件名。这样即使文件名变了，内容里的锚点字符串大概率还在。

**如果你的 hotfix 硬编码了文件名**：改成按内容查找。

### 成因 2：upstream 在锚点附近插代码

作者的锚点是一段字符串（如 `"function materializeRuntimeConfig(config, mode) {"`）。官方在附近插新代码，**锚点字符串可能裂开**或变成多段。

**解决**：

1. 打开 dist 找到实际代码，看锚点附近变成什么样了
2. 要么**缩短锚点**（留最稳定的一段），要么**换锚点**（换附近另一段不变的字符串）
3. 写到 hotfix 脚本里，重新跑

**⚠️ 升级频繁（每周一次）的用户**：建议把 hotfix 脚本单独开一个 git repo，每次失败都 commit 一次，方便回溯。

---

## 回滚

### 完整回滚（推荐新手）

```bash
bash ~/.openclaw/scripts/restore-memory-plugin-upgrade.sh --apply
# 从最近的 backup tar.gz 恢复
```

然后降级 CLI：

```bash
npm install -g openclaw@<旧版本号>
```

### 只回滚 dist（高阶）

如果你**只想回滚补丁**，而 `~/.openclaw/` 其他部分不动：

```bash
# hotfix 每次跑都会把改动前的原文件备份到 ~/.openclaw/backups/openclaw-cli-hotfixes/<时间戳>/
ls -t ~/.openclaw/backups/openclaw-cli-hotfixes/ | head -3   # 找最近一次
# 把整个时间戳目录里的文件覆写回 dist
ts=$(ls -t ~/.openclaw/backups/openclaw-cli-hotfixes/ | head -1)
for f in ~/.openclaw/backups/openclaw-cli-hotfixes/$ts/*; do
  cp "$f" ~/.local/lib/node_modules/openclaw/dist/$(basename "$f")
done
```

如果想直接跑 hotfix 但不写备份，加 `--no-backup`（不推荐，除非磁盘极紧）。

> ⚠️ **不要**用 `rm -rf ~/.local/lib/node_modules/openclaw/` 再重装 —— 这样会丢当次 hotfix 备份的对照基线，复盘"改动了什么"会很麻烦。

---

## 什么时候可以"退出"某个 hotfix

**判据**：如果某个 patch 是为了**绕开官方 bug**，定期查官方 release notes：

```
官方已修 → hotfix 脚本里把该 target 注释掉，重新跑
```

如果某个 patch 是**纯个性化**（如自定义文案），那就永远留着，无需"退出"。

---

## 升级节奏建议

| 升级频率 | 合适用户 |
|---|---|
| 每次新版立刻升 | 愿意踩坑反馈的早期用户 |
| 每月一次 | 大多数个人 / 小团队 |
| 每季度一次 | 生产环境稳定优先 |

**作者自己的节奏**：新版出来后**观望 3-5 天**（看 GitHub Issues 有没有明显炸），然后升，升完立刻跑 hotfix + 冒烟。

---

## 升级前检查清单

```
[ ] 已打备份（backup-memory-plugin-upgrade.sh）
[ ] 已记录当前版本号
[ ] 当前 doctor 全绿（升级前就不绿的别升，先修）
[ ] 已确认不在关键任务时段（升级会中断 gateway 1-2 分钟）
[ ] 本次升级的 changelog 已读（看有没有 breaking）
[ ] hotfix 脚本最新（如果你自己维护了本地改动，git pull 一下）
```

---

## 下一步

- [01-breaking-change-应对](./01-breaking-change-应对.md) —— 遇到 breaking 时的诊断流程
- [04-运维/02-升级流程](../04-运维/02-升级流程.md) —— 升级的运维侧（plist 调整、备份自动化）

---

> **本章准确性保证**
> hotfix target 清单直接来自 `~/.openclaw/scripts/apply-openclaw-cli-hotfixes.mjs` 真实代码（2026-04-17 版本，15 个 targets）。回滚 / 回归脚本路径已对 `~/.openclaw/scripts/` 核对。分类与占比为作者主观归纳，可能因用户场景不同而不同。
