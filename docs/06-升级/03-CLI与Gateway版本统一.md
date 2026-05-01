---
title: 06-升级 · 03 CLI 与 Gateway 版本统一
---

> 预计阅读：12 分钟
> 适用版本：OpenClaw 2026.4.24 稳定基线，尤其适合自己装过多份 OpenClaw 的环境
> 前置：[04-运维/02-升级流程](../04-运维/02-升级流程.md) / [05-排障/01-诊断流程](../05-排障/01-诊断流程.md)
> 本章回答：**为什么 `openclaw config validate` 和实际 gateway 提示会互相打脸？为什么明明配了一个键，CLI 说不认识，gateway 却又要求你设置它？**

---

## 这是什么问题

最典型的症状有三类：

- CLI 校验说某个配置键非法，但 gateway 日志里明确提示你要设置这个键
- `openclaw --version` 和 gateway 实际运行逻辑对不上
- 你改完配置后，CLI 看起来通过了，服务重启后却像没读到新能力
- 浏览器访问 `http://127.0.0.1:18789/`，提示 `Control UI assets not found`
- `openclaw gateway start` 看似成功，但 launchd / systemd 实际指向一个已经不存在的 Node 或 OpenClaw 路径

这通常不是“你改错了配置”，而是 **CLI 和 gateway 服务跑的不是同一版 OpenClaw**。

---

## 为什么会发生

在 macOS 本地环境里，OpenClaw 常见有两套安装来源：

| 来源 | 常见路径 |
|---|---|
| 用户级 npm / nvm | `~/.local/bin/openclaw`、`~/.nvm/...` |
| Homebrew / Cellar / 全局 Node | `/usr/local/bin/openclaw`、`/usr/local/Cellar/...` |

如果：

- 终端默认 `openclaw` 走的是旧版本
- launchd 启 gateway 走的是另一套新版本

你就会遇到“CLI schema 旧、runtime 行为新”的错配。

更隐蔽的一种情况是：shell 里的 `openclaw` 已经修好，但系统服务文件仍写死旧路径，例如：

```text
~/.nvm/versions/node/vX.Y.Z/lib/node_modules/openclaw/dist/index.js
```

只要这个目录被 nvm 清掉、全局 npm 包被重装到别的位置，gateway 就会继续尝试执行一个不存在的 `dist/index.js`。这时你在终端里手工运行 `openclaw` 可能正常，服务却仍然起不来。

---

## 先判断是不是版本错配

### 1. 看 shell 里到底会命中哪个 `openclaw`

```bash
which -a openclaw
ls -l "$(command -v openclaw)"
openclaw --version
```

### 2. 看 gateway 服务实际跑哪一个二进制

```bash
openclaw gateway status
```

重点看输出里的：

- `Command: ...`
- `Service file: ...`
- `Config (cli)` / `Config (service)`

如果 `Command` 里指向的不是你 shell 默认 `openclaw` 那套目录，就已经是强信号。

### 3. 看 warning 是否属于“新 runtime 认得、旧 CLI 不认得”的类型

典型例子：

```text
typed hook "agent_end" blocked because non-bundled plugins must set
plugins.entries.<plugin>.hooks.allowConversationAccess=true
```

这类提示说明 **runtime 已经支持更细粒度的 hook policy**。
如果此时旧 CLI 还说：

```text
Unrecognized key: "allowConversationAccess"
```

那基本就是双版本错配。

---

## 修复原则

### 原则 1：先统一入口，再改配置

不要一边用旧 CLI 改配置，一边让新 gateway 跑服务。
先选定一套“主版本”，然后：

- shell 入口指向它
- gateway 服务也指向它

### 原则 2：只保留一个“主解释器”

你可以同时保留历史安装副本，但 **不能让它们同时承担生产入口**。

推荐做法：

- 要么统一到 `~/.local/bin/openclaw`
- 要么统一到 Homebrew / Cellar

总之，CLI 和服务必须是一套。

---

## 一套可复用的修复流程

### 第 1 步：找出你要保留的新版本入口

例如：

```bash
/usr/local/Cellar/node/25.8.1_1/bin/openclaw --version
~/.local/bin/openclaw --version
```

假设你确认要保留的是：

```text
/usr/local/Cellar/node/25.8.1_1/bin/openclaw
```

### 第 2 步：把常用 shell 入口指到它

```bash
ln -sfn /usr/local/Cellar/node/25.8.1_1/bin/openclaw ~/.local/bin/openclaw
hash -r
openclaw --version
```

如果你的 PATH 里 `~/.local/bin` 在前面，这一步就够了。

### 第 3 步：重新校验配置

```bash
openclaw config validate
```

此时如果你要用新版本引入的配置键，CLI 才能和 runtime 说同一种语言。

### 第 4 步：重启 gateway

```bash
openclaw gateway restart
openclaw gateway status
```

### 第 5 步：验证 warning 是否消失

```bash
rg -n 'typed hook "agent_end" blocked' ~/.openclaw/logs/gateway.log
```

看最后一条 warning 的时间是否停在修复前。

---

## 当前模板里的自愈脚本

如果你使用的是当前模板，优先用脚本修，不要手工编辑 launchd / systemd 文件：

```bash
bash ~/.openclaw/scripts/repair-gateway-service.sh --repair --restart
bash ~/.openclaw/scripts/runtime-status-report.sh
```

这条修复链会做几件事：

- 找到当前可用的 `openclaw` 命令
- 解析它背后的全局 npm 安装目录
- 检查 `dist/index.js` 是否真实存在
- 修复 macOS launchd 或 Linux systemd 里的 gateway 启动路径
- 重启 gateway 并重新做健康检查

在 `v1.0.10` 之后，如果全局 `openclaw` 命令或 `dist/index.js` 已经不存在，修复脚本会优先按 `RUNTIME_BASELINE.json` 里的 OpenClaw 基线补装，再继续修服务路径。当前稳定基线是：

```text
openclaw_core = 2026.4.24
```

也就是说，修复脚本的目标不是“装 npm 上最新的 OpenClaw”，而是先恢复到模板验证过的 runtime。

---

## 什么时候不要继续升级

如果你刚升级到更高版本后出现：

- 飞书回复明显变慢
- Control UI 打不开
- memory-lancedb-pro 超时或补丁校验异常
- gateway 服务路径反复指向失效目录

先回到模板基线，不要继续追更。正确顺序是：

```bash
# 1. 修回 gateway 服务路径
bash ~/.openclaw/scripts/repair-gateway-service.sh --repair --restart

# 2. 看当前 runtime 状态
bash ~/.openclaw/scripts/runtime-status-report.sh

# 3. 再决定是否执行升级评估
openclaw --version
```

只有当 CLI、gateway、插件补丁和自检都统一后，升级评估才有意义。

---

## 实战里最容易踩的坑

### 坑 1：只看 `openclaw --version`

这只能说明 **当前 shell** 命中的版本。
不代表 launchd 服务也在跑同一套。

### 坑 2：以为“配置通过校验”就等于“服务会按这个能力运行”

不是。
只有当 **校验 schema 和服务 runtime 来自同一套版本**，这件事才成立。

### 坑 3：升级 CLI 之后，没同步 baseline / 自愈脚本

如果你的自定义脚本会回写 `openclaw.json`，而 baseline 还是旧版配置，它可能把新字段再抹掉。

升级后要顺手检查：

- `openclaw.json.bak`
- 任何会 restore baseline 的脚本
- 任何会批量写配置的 cron / 自愈脚本

### 坑 4：以为重装 npm 包会自动修系统服务

不一定。`npm install -g openclaw@...` 只改变 npm 安装目录里的文件，不保证 launchd / systemd 的启动命令一起改。
升级后必须检查：

```bash
openclaw gateway status
bash ~/.openclaw/scripts/runtime-status-report.sh
```

如果服务文件仍指向旧路径，继续用 `repair-gateway-service.sh --repair --restart` 收口。

---

## 一句话经验

> **先统一 CLI 和 gateway 的版本，再谈配置键、hook policy 和运行时行为。**

只要这一步没做，后面所有“为什么配置不生效”的排障都可能是伪问题。
