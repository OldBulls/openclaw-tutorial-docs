---
title: 02-配置 · 02 Skills 选择
---

> 预计阅读：10 分钟
> 适用版本：OpenClaw 2026.4.24 稳定基线 · 最后审核：2026-05-02
> 本章回答：**OpenClaw 几十个 skill，我到底装哪些？哪些必装？哪些可有可无？**

---

## Skills 是什么、从哪来

**Skills** 是 OpenClaw 的能力扩展模块。类似浏览器插件 —— 装了就多一套工具、行为规则、记忆能力。

### 两个来源

| 来源 | 位置 | 更新方式 |
|---|---|---|
| 官方 / 社区 | ClawHub | `openclaw skills install <name>`（子命令是**复数** `skills`） |
| 本地开发 | `~/.openclaw/extensions/<name>` | `plugins.installs.<name>` 里写 `source: "path"` + `sourcePath`，gateway 启动时加载 |

### 两个配置层

| 层 | 配置在哪 | 作用域 |
|---|---|---|
| **plugins**（插件） | `openclaw.json` 的 `plugins` 节 | 全局行为（记忆引擎、上下文引擎、通道接入） |
| **skills**（技能） | `~/.openclaw/skills/` 目录 + agent 的 `workspace/skills/` | 具体任务能力（PDF 处理、视频剪辑、搜索） |

**区别**：

- **plugins 改变 OpenClaw 的核心行为**（如换记忆引擎）
- **skills 给 agent 增加具体工具**（如让 agent 会做 PPT）

---

## plugins：最关键的 slots 概念

`openclaw.json` 的 `plugins.slots` 是**槽位**指派：

```json
{
  "plugins": {
    "slots": {
      "memory": "memory-lancedb-pro",
      "contextEngine": "lossless-claw"
    }
  }
}
```

| slot | 当前值 | 作用 |
|---|---|---|
| `memory` | `memory-lancedb-pro` | 谁来做语义记忆 |
| `contextEngine` | `lossless-claw` | 谁来做会话内压缩 |

**换槽位** = 换核心行为。例如把 `memory` 换成别的记忆插件，整个 agent 的长期记忆机制都变。

**不要随便换**。槽位切换会涉及数据迁移，别人的方案在你场景未必好用。

---

## 强烈推荐（9 个必装）

作者 20+ 天实战里，下面这组是**最小可用集**。缺一个都会显著影响体验。

| 名字 | 类型 | 作用 | 不装的话 |
|---|---|---|---|
| `memory-lancedb-pro` | plugin (slot: memory) | 语义记忆（L2） | agent 每次都像金鱼，跨会话不记事 |
| `lossless-claw` | plugin (slot: contextEngine) | 会话内 DAG 压缩（L0） | 长对话必炸 context |
| `proactivity` | skill | 运行态推进（L1） | agent 只答不续，没"下一步"意识 |
| `self-improving` | skill | 稳定规则沉淀（L3） | 踩过的坑会反复踩 |
| `openclaw-lark` 或 `feishu` | plugin | 飞书通道 | 没法接飞书 |
| `openclaw-ops` | skill | 系统运维能力（gateway 状态、日志查看） | agent 不能自检 |
| `openclaw-workspace` | skill | 工作空间管理（memory/ 文件读写） | agent 不会写日记 |
| `skill-finder-cn` | skill | 中文场景下的技能匹配 | 英文 skill 匹配在中文对话里偏差大 |
| `delegate-task` | skill | `sessions_send` / `sessions_spawn` 能力 | 没法派发给其他 agent |

### 进一步解释

**`memory-lancedb-pro` 的 embedding 配置很关键**，作者用的：

```json
"embedding": {
  "provider": "openai-compatible",
  "baseURL": "https://api.siliconflow.com/v1",
  "model": "Qwen/Qwen3-Embedding-0.6B",
  "dimensions": 1024
}
```

配合 rerank（注意：rerank 字段在 `retrieval` 子对象里，**不是**顶层）：

```json
"retrieval": {
  "mode": "hybrid",
  "candidatePoolSize": 12,
  "vectorWeight": 0.7,
  "bm25Weight": 0.3,
  "minScore": 0.55,
  "hardMinScore": 0.6,
  "filterNoise": true,
  "rerank": "lightweight"
}
```

> 生产环境默认建议：`embedding` 继续走 SiliconFlow，`rerank` 先用 `lightweight`。这样能避开外部 rerank 的额外网络波动，稳定性通常比远程 cross-encoder 更好。如果后面你确认链路很稳，再单独把 `retrieval.rerank` 升到 `cross-encoder`。

---

## 推荐按需选装

按你的业务场景选。

### 如果做**内容创作**

| skill | 场景 |
|---|---|
| `xiaohongshu-auto` | 小红书内容自动化 |
| `qiaomu-mondo-poster-design` | 海报设计 |
| `baoyu-post-to-wechat` | 公众号投递 |
| `markdown-converter` | MD 转各种格式 |
| `humanizer` | 让 AI 文案更像人话 |
| `word-docx` / `xlsx` / `pptx` / `pdf` | Office 文档处理 |

### 如果做**多媒体**

| skill | 场景 |
|---|---|
| `remotion-video-toolkit` | 程序化视频生成 |
| `seedance-prompt-designer` / `seedance-shot-design` | 镜头语言 |
| `nano-banana-pro` | 图生视频 |
| `video-download` / `video-edit` / `video-translate` / `video-understand` | 视频基础能力 |
| `text-to-speech` | TTS |
| `alicloud-ai-audio-cosyvoice-voice-clone` | 声音克隆（阿里云 CosyVoice） |

### 如果做**信息搜集 / 研究**

| skill | 场景 |
|---|---|
| `tavily-search-pro` | Tavily 搜索 |
| `multi-search-engine` | 多引擎聚合 |
| `web-access` | 浏览器抓取 |
| `atlas-cloud-ai-api` | Atlas Cloud 知识 API |

### 如果做**知识管理**

| skill | 场景 |
|---|---|
| `notion` | Notion 集成 |
| `obsidian` | Obsidian 集成 |
| `ontology` | 本体建模 |
| `summarize` | 自动摘要 |
| `clawddocs` | OpenClaw 文档内搜索 |

### 如果做**多 Agent / 团队协作**

| skill | 场景 |
|---|---|
| `agent-setup` | agent 引导 |
| `collab-setup` | 协作配置 |
| `context-flow` | 上下文流转 |
| `moltcn` | 社区互动（ moltbook 专属） |
| `background-review` | 背景审阅 |

### **高阶 / 慎装**

| skill | 注意事项 |
|---|---|
| `skill-creator` | 创建新 skill，非开发者慎装 |
| `skill-discovery` / `skill-vetter` | skill 生态工具，新手用不到 |
| `ui-ux-pro-max` | 重度 UI 设计场景，对小团队过度 |
| `visual-style` | 视觉风格体系，需配合设计师用 |

---

## 不推荐（踩过坑）

作者踩坑 / 评估后决定**不装**的：

| skill | 为什么不装 |
|---|---|
| 任何"一键同步所有配置"类 | 供应链风险，见 [05-排障/00-常见问题](../05-排障/00-常见问题.md) 的"安全红线" |
| 太新（< 30 天）且只有 1 个 maintainer 的 skill | 没经过充分测试，容易突然断维护 |
| 和 `memory-lancedb-pro` 功能重叠的 | 两个记忆 skill 会互相干扰语义库 |

---

## plugin 的"allow / load / entries / installs" 四层

容易混淆。看真实配置（从作者 `openclaw.json` 摘抄）：

```json
{
  "plugins": {
    "allow": ["memory-lancedb-pro", "lossless-claw", "acpx", "exa", "minimax", "google", "memory-core", "openclaw-lark", "feishu"],
    "load": ["..."],
    "slots": {
      "memory": "memory-lancedb-pro",
      "contextEngine": "lossless-claw"
    },
    "entries": {
      "openclaw-lark": { "enabled": true },
      "feishu": { "enabled": false }
    },
    "installs": {
      "memory-lancedb-pro": {
        "source": "path",
        "sourcePath": "/Users/<you>/.openclaw/extensions/memory-lancedb-pro",
        "installPath": "/Users/<you>/.openclaw/extensions/memory-lancedb-pro",
        "version": "1.1.0-beta.8",
        "resolvedName": "memory-lancedb-pro",
        "resolvedVersion": "1.1.0-beta.8",
        "installedAt": "2026-03-20T15:12:00+08:00"
      }
    }
  }
}
```

| 键 | 含义 | 不配会怎样 |
|---|---|---|
| `allow` | 白名单，只有在这的插件才能加载 | 不在 allow 里，直接被拒绝加载 |
| `load` | 显式加载顺序（有依赖关系时用） | 一般让 OpenClaw 自动推导即可 |
| `slots` | `memory` / `contextEngine` 等核心槽位指派给哪个插件 | 核心行为回退到默认 |
| `entries.<name>.enabled` | 每插件开关 | `false` 即使在 allow 里也不启用 |
| `installs.<name>` | 记录每个插件的真实安装源和版本（path / npm / clawhub） | 不配就默认从 ClawHub 按名取 |

> `installs.<name>` 里的 `resolvedVersion` / `installedAt` 是 `openclaw plugins install` 自动写入的，你**不用手写**；手写只填 `source` + `sourcePath` 即可，其余由 CLI 补齐。

**作者的典型配置**（`openclaw-lark` 启用、`feishu` 禁用）：

- 两个都在 `allow` 里（预留切换空间）
- `openclaw-lark.enabled = true`（当前用）
- `feishu.enabled = false`（备用，不激活）

这种"双插件并存 + 软开关"的做法详见 [03-飞书深度配置](./03-飞书深度配置.md)。

---

## 装 skill 的基本流程

```bash
# 1. 搜 + 看详情（可选）
openclaw skills search <keyword>
openclaw skills info <skill-name>

# 2. 装（子命令是**复数** skills）
openclaw skills install <skill-name>

# 3. 启用（有些 skill 装完默认启用，有些要手动 enable）
#    看该 skill 的 README，或直接在 ~/.openclaw/skills/<name>/ 找 config

# 4. 重启 gateway
openclaw gateway restart

# 5. 等 OK（见 ../05-排障/00-常见问题.md 第 2 节）
until bash ~/.openclaw/scripts/check-feishu-gateway-health.sh --summary 2>/dev/null | grep -q "^OK:"; do
  sleep 2
done

# 6. 跑 doctor，确认 skill 注册进来
openclaw skills check

# 7. 验证：在飞书问 agent "你现在会做什么"，或者直接让它调用新 skill 的工具
```

---

## skill 冲突诊断

两个 skill 功能重叠时 agent 会混乱。症状：

- 同一个任务，两次调用走了不同的工具路径
- agent 自己都不确定该用哪个
- memory_recall 出现重复或相互矛盾的结果

排查：

```bash
# 列出所有装了的 skill
ls ~/.openclaw/skills/

# grep 关键字，找功能重叠的
ls ~/.openclaw/skills/ | grep -iE 'search|memory|pdf'
```

发现重叠 → 留一个、禁用另一个（改 skill 自己的 config 里的 `enabled` 或从 allow 里拿掉）。

---

## skill 升级

```bash
# 单个升级
openclaw skills update <skill-name>
# 或全量升级
openclaw skills update --all
```

**升级前同样要备份**（见 [06-升级/00-npm升级兼容](../06-升级/00-npm升级兼容.md) 的备份章节）。某些 skill 升级会带 breaking schema change。

---

## 验证清单

装完核心 9 个 skill 后，验证：

| 检查项 | 怎么验 |
|---|---|
| lossless-claw 工作 | 问 agent"你现在 context 用了多少" |
| memory-lancedb-pro 工作 | 让 agent "记住我喜欢喝冰美式"，/reset 后再问 |
| openclaw-lark 工作 | 飞书回复正常 + footer 带 metrics |
| delegate-task 工作 | 让 main 派发给 planner（如果你已配多 agent） |
| openclaw-ops 工作 | 问 agent"gateway 状态怎么样" |
| openclaw-workspace 工作 | 让 agent 写一条记忆到 `memory/YYYY-MM-DD.md` |

六项全绿 → 核心 skill 集装到位。

---

## 下一步

- [01-记忆系统](./01-记忆系统.md) —— 核心 skill（memory/context）的深度配置
- [03-飞书深度配置](./03-飞书深度配置.md) —— openclaw-lark / feishu 双插件机制
- [03-多Agent/02-共享规范](../03-多Agent/02-共享规范.md) —— `shared/` 下的通用 TOOLS.md 怎么组织

---

> **本章准确性保证**
> 推荐 skill 清单基于作者 `~/.openclaw/skills/` 的真实安装集（2026-04-17，47 个 skill）。plugin slots / entries / installs 配置来自 `~/.openclaw/openclaw.json` 的真实 plugins 节。embedding / rerank 参数为生产环境实测可用组合。

---

**导航**：[← 记忆系统](./01-记忆系统.md) · [📖 目录](../00-先读我.md) · [→ 飞书深度配置](./03-飞书深度配置.md)
