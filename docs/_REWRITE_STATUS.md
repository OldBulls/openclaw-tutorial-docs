# docs/ 重写状态追踪

所有 docs/ 下的教程都要经历这个状态流转：

🔴 **scaffold** → 🟡 **partial** → 🟢 **verified**

## 状态定义

- 🔴 **scaffold**：MiniMax 自动生成的草稿，命令 / 工具名 / 代码片段**未经真实 CLI 验证**，必须重写
- 🟡 **partial**：结构与原则性论述可用，但具体细节仍需基于真实代码验证
- 🟢 **verified**：已基于真实 OpenClaw CLI、`~/.openclaw/shared/` 实际文件、个人 20+ 天记忆重写并实测

## 2026-04-18 二次 audit pass

全 22 章逐章重新对齐真实 `~/.openclaw/` 状态。本轮发现 / 修正：

- **新增 CLI 实测知识**：`openclaw subagents` 子命令受 `plugins.allow` 控制，缺 `subagents` 时报 `command is unavailable because plugins.allow excludes "subagents"` —— 已在 `05-排障/01-诊断流程.md` 加警告
- **02-配置/01-记忆系统.md** L312：`bootstrap-extra-files` 真实位置补全为 `hooks.internal.entries.bootstrap-extra-files`（原文模糊带过）
- **03-多Agent/02-共享规范.md** L31：dup typo 修正（`test-after-write、test-after-write` → `test-after-write、混淆写法禁用`）
- **04-运维/01-日志与监控.md** L15/L45：日志文件计数 74 → 76（真实 `ls ~/.openclaw/logs/ \| wc -l` 校准）
- **全 22 章导航清理**：过期 `（待写）` 标签全部删除（这些 cross-reference 的目标文件均已存在）—— 涉及 02-配置/00、02-配置/01、03-多Agent/00、03-多Agent/01、03-多Agent/02、01-入门/02、01-入门/03、04-运维/01、04-运维/02、04-运维/03 共 10 个文件
- **复核已确认**：feishu schema（connectionMode/webhookPath/dmPolicy/accounts.allowFrom/replyMode/footer/appSecret 三段）、launchd plist 双前缀（`ai.openclaw.*` 网关 / `com.openclaw.*` 维护）、hotfix 脚本三件套全部真实存在


## 当前文件清单

| 文件 | 状态 | 备注 |
|---|---|---|
| 00-先读我.md | 🟢 | 实测：阅读路径里的章节状态全部更新为 🟢；版本/最后更新日期校准 2026-04-18 |
| 01-入门/00-概述.md | 🟢 | 实测修正：L1 真实路径 `~/.openclaw/proactivity/memory.md`（全局共享 + 无 session-state.md）/ L2 单表 `memories.lance/` / L3 扁平 `YYYY-MM-DD-{话题}.md` / L4 真实产物 `graphify-out/` / gateway `run vs start` 双命令 + 架构图中的 taskflow 去掉（不是插件） |
| 01-入门/01-安装.md | 🟢 | 已基于真实 CLI 实跑验证：`openclaw setup`（非 init）、`openclaw models auth login`、`secrets.providers.*.source=file`（非 type）、`channels.feishu` 完整 schema（含 websocket connectionMode）、`openclaw gateway run/start/install/status` 子命令集 |
| 02-配置/01-记忆系统.md | 🟢 | 实测：L1 `proactivity/` 实际只有 `memory.md` + counter（没有 session-state.md）；L4 真实产物是 `graphify-out/graph.json` + `GRAPH_REPORT.md`（非 jsonl/ontology）；experience/ 无 main 子目录（只收子 agent） |
| 03-多Agent/00-协作拓扑.md | 🟢 | 实测：8 个 agent 的真实 `name` 字段（minsu=民宿助理 / reflector=记忆回顾 / shangfang=上房助手）+ main 真实 `model.primary=aigpt-openai/gpt-5.4`（5 级 fallback 链）+ `subagents.allowAgents` 真实顺序补 moltbook + `tools.alsoAllow` 补 lcm_describe |
| 05-排障/00-常见问题.md | 🟢 | 实测修正：fragments 计数路径 `memories.lance/data/`（不是 `*.lance/_versions/`）+ launchctl grep 要用 `grep -iE "openclaw\|claw"`（两套前缀）+ lossless-claw.contextThreshold 真实位置 `plugins.entries.lossless-claw.config.contextThreshold` |

## 待新增

### 01-入门/

- [x] 02-基础配置.md 🟢 实测：`bootstrap-extra-files` 真实位置 `hooks.internal.entries.bootstrap-extra-files`（enabled + paths[] 真实结构）+ shared/ 三文件确认（SOUL/TOOLS/AGENTS）
- [x] 03-验证.md 🟢 实测：`gateway status` 真实输出是 Service/Runtime/RPC probe 三段（不是列 agent id）；`/health` 端点真实返回 `{"ok":true,"status":"live"}`

### 02-配置/

- [x] 00-多Agent架构.md 🟢 实测：纯方法论文档无可疑 CLI 字段，`agents.list[]` + `subagents.allowAgents` 字段 example 与真实 openclaw.json 一致
- [x] 02-Skills选择.md 🟢 实测：`openclaw skills` 子命令是**复数**（install/search/info/update/check/list 全套）+ `plugins` 四层（allow/load/entries/installs）非三层 + `installs.<name>` 自动补齐 resolvedVersion/installedAt
- [x] 03-飞书深度配置.md 🟢 实测：`openclaw plugins install`（复数）+ `tools.alsoAllow` 是 per-agent（在 `agents.list[].tools.alsoAllow` 下，非根级）+ 作者真实 35 个 feishu_* 工具清单已对齐

### 03-多Agent/

- [x] 01-委派机制.md 🟢 实测：subagents 上限（maxConcurrent=8/maxSpawnDepth=2/maxChildrenPerAgent=8）+ `memoryFlush` 在 `agents.defaults.compaction.memoryFlush` 真实位置确认 + `sessions_send/spawn` + `subagents list/steer/kill` 全部对着 `shared/TOOLS.md` 确认
- [x] 02-共享规范.md 🟢 实测：`bootstrap-extra-files` 配置位置改为真实 `hooks.internal.entries.bootstrap-extra-files`（`enabled` + `paths[]` 相对路径）+ 三个 shared 文件真实行数（SOUL 163 / AGENTS 217 / TOOLS 147）

### 04-运维/（全新）

- [x] 00-健康检查.md 🟢 实测：fragments 计数路径 `memories.lance/data/`（不是 `_versions/`）+ 阈值 500⚠️ / 1000🚨 对齐 daily-digest.py + 两套 plist 前缀（gateway/watchdog 用 ai.* / digest/maintenance 用 com.*）说明
- [x] 01-日志与监控.md 🟢 实测：`~/.openclaw/logs/` 真实 76 文件 75MB（2026-04-18 二次 audit 校准），日志文件名清单全部真实存在
- [x] 02-升级流程.md 🟢 实测：三个升级脚本路径全部真实存在（`backup-memory-plugin-upgrade.sh` / `restore-memory-plugin-upgrade.sh` / `apply-openclaw-cli-hotfixes.mjs`）
- [x] 03-备份与恢复.md 🟢 实测：`backup-config-to-github.sh` / `backup-shared-to-github.sh` / `backup-workspace-to-github.sh` 真实存在；L5 重建路径修正为 `data/graphify/openclaw-memory/graphify-out/`

### 05-排障/

- [x] 01-诊断流程.md 🟢 实测：C1/C2 LanceDB 路径修正为 `memories.lance/data/`（真实 387 fragments）+ launchctl grep 改为 `-iE "openclaw|claw"` 抓两套前缀 + 阈值对齐 daily-digest.py 500/1000

### 06-升级/（全新）

- [x] 00-npm升级兼容.md 🟢 实测：`apply-openclaw-cli-hotfixes.mjs` / `check-memory-plugin-upgrade.sh` / `check-feishu-gateway-health.sh` / `restore-memory-plugin-upgrade.sh` 全部真实存在；npm 包名 `openclaw`（非 `@openclaw/cli`）
- [x] 01-breaking-change-应对.md 🟢 实测：npm 包名 `openclaw` 已修；三种 fork 出路与作者 2026-02 至 2026-04 真实经验对齐

---

## templates/（档位 2 付费内容）

### shared/  整体状态：🟢（端到端冒烟 e2e 已通过，commit aeafc57 / 32325ed）
- [x] SOUL.template.md 🟢 家族通用核心原则骨架（工具叙事 / 三查 / 收口三段论 / 语言规则 / 自主记忆）
- [x] AGENTS.template.md 🟢 派发决策树 + 派发前确认 + 协作机制 + 五层记忆 + 启动顺序 + 收口规则
- [x] TOOLS.template.md 🟢 渠道工具链 / 系统工具 / 路径别名 / Shell 规范 / 升级脚本 / Proactive 原则
- [x] USER.template.md 🟢 称呼 + 识别 ID + allowFrom + 身份变更触发（已脱敏）
- [x] USER_PROFILE.template.md 🟢 沟通风格 / 工作节奏 / 决策风格 / 领域知识 / 禁区 / 偏好演化记录
- [x] MEMORY.template.md 🟢 家族共享记忆索引骨架（配置/运维/协作/偏好/踩坑 5 类）
- [x] IDENTITY.template.md 🟢 家族身份 / 价值主张 / 人设基调 / 家族边界

### agents/  整体状态：🟢（apply.sh 业务 agent ID 注入 + 冒烟生成正常）
- [x] main/ 🟢 SOUL + IDENTITY（统一入口 + 记忆中枢 + 派发路由）
- [x] planner/ 🟢 SOUL + IDENTITY（只规划不执行 / 强制输出格式 / 四原则）
- [x] reflector/ 🟢 SOUL（单一职责 / 只 SOUL 设计 / 四工作规则 / 触发节律）
- [x] 业务-agent-骨架/ 🟢 SOUL + IDENTITY（职责边界 / 业务知识 / 决策守则 / 记忆策略）

### scripts/  整体状态：🟢（15 个 .template + 1 个 usage 全部 parse 通过、commit 32325ed README 重写按场景分组）

**Gateway 健康（2 个）**
- [x] health-check.template.sh 🟢 fast/strict 双模式 + err.log 致命错扫描 + 退出码触发 launchd 恢复
- [x] check-feishu-gateway-health.template.sh 🟢 token 有效性自检 + 静默测试消息

**日志 / 报告（2 个）**
- [x] rotate-logs.template.sh 🟢 大小轮转 + 30 天压缩归档 + 磁盘告警
- [x] daily-digest.template.sh 🟢 7 项指标采集 + 三档色分级 + launchd plist 配置示例

**Hotfix（2 个）**
- [x] apply-openclaw-cli-hotfixes.template.mjs 🟢 15 锚点 + dist 哈希动态识别（已脱敏 MiniMax 模型名）
- [x] apply-hotfixes-usage.md 🟢 hotfix 机制 / 锚点识别思路 / 三种失配场景排查 / 扩展方式

**备份矩阵（3 个）**
- [x] backup-config-to-github.template.sh 🟢 脱敏 push openclaw.json + git pull --rebase 保护
- [x] backup-shared-to-github.template.sh 🟢 shared/ 脱敏 push 独立 repo
- [x] backup-workspace-to-github.template.sh 🟢 workspace/MEMORY.md 脱敏 push 独立 repo

**Plugin 升级保护（3 个）**
- [x] backup-memory-plugin-upgrade.template.sh 🟢 升级前 tar.gz 全量快照
- [x] restore-memory-plugin-upgrade.template.sh 🟢 dry-run 默认 / --apply 真覆写
- [x] check-memory-plugin-upgrade.template.sh 🟢 6 步回归（version/health/CLI 延迟/patch/慢调用/启动诊断）

**周维护编排（4 个）**
- [x] weekly-maintenance.template.sh 🟢 shell 版四步编排 + step 不静默 skip
- [x] run-weekly-maintenance.template.py 🟢 python 版四步编排（与 shell 版二选一）
- [x] consolidate-memory.template.py 🟢 跨 agent memory/ 按主题归并 MEMORY.md 索引
- [x] memory-archive.template.py 🟢 陈旧 weekly-merge 归档 yearly archive 两阶段原子

### skills/  整体状态：🟢（plugins 三件 JSON + 4 篇说明全部针对真实 schema 校准）
- [x] README.md 🟢 plugins vs skills 命名澄清 + 应用流程 + 红线
- [x] plugins-allow.template.json 🟢 白名单 + 选型建议注释
- [x] plugins-slots.template.json 🟢 memory / contextEngine slot 样例
- [x] plugins-entries.template.json 🟢 9 插件配置块，secret 统一走 source=file 不 inline
- [x] skills-by-scenario.md 🟢 5 场景组合（最小/研究/内容/知识/协作）+ 对照表
- [x] troubleshooting.md 🟢 快速自检 + 5 类错误 + 4 层诊断 + 特殊场景

---

## guided-install/（档位 2 付费内容）  整体状态：🟢（端到端冒烟通过 + manifest JSON 协议 + 自底向上 rmdir）

- [x] install.sh 🟢 主编排（detect → prompt → apply + 二次确认 + manifest written_files JSON 数组 + 后续步骤指引）
- [x] detect.sh 🟢 环境探测（CLI / 家目录 / 已有 agent / gateway / 磁盘 / node / 时区）
- [x] prompt.sh 🟢 问答式采集（GATEWAY_PORT 默认对齐 18789 真实 OpenClaw 默认值）
- [x] apply.sh 🟢 模板应用（KEYS 数组 + awk 字面替换 + 业务 agent 注入 AGENT_ID + plist + credentials README）
- [x] rollback.sh 🟢 manifest.written_files 精确删除 + 自底向上 rmdir 空目录 + 旧 manifest 向后兼容 warn
- [x] 冒烟 ①：detect 识别 8 个已有 agent / apply 生成 19 文件 / AGENT_ID 注入正常
- [x] 冒烟 ②：mock vars + 全 templates → apply 39 文件 → install 写入 42 + manifest → rollback 精确删 42 + rmdir 8 空目录

### 2026-04-18 端到端冒烟（②）

mock vars + 全 templates → apply.sh 生成 39 个 stage 文件（含 launchd plist + credentials/README 自动追加）→ 模拟 install.sh 写入 42 个 + manifest.written_files JSON 数组 → rollback 精确删除 42 + rmdir 8 个空目录 + 归档 manifest。

产物语法：11 个 .sh / 1 个 .mjs / 3 个 .py / 3 个 .json / 1 个 .plist 全部 parse 通过。

业务占位符：22 种 → 63 处归入 `.PLACEHOLDERS_TODO.md`（DOMAIN / TONE / BUSINESS_RULE / TASK_SUMMARY / 等，设计如此用户填）。

templates/ 整体可视为 🟢（生成路径完整、产物可解析、占位符归集准确）。
