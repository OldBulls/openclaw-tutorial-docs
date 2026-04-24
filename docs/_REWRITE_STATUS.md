# docs/ 重写状态追踪

这份文件现在不再记录“哪些章节还没补”，而是记录**当前仓库的真实完成状态**。

所有内容仍按下面这条流转定义理解：

`🔴 scaffold` → `🟡 partial` → `🟢 verified`

## 状态定义

- `🔴 scaffold`：自动生成草稿，命令 / 字段 / 路径未对真实环境验证，不能直接交付
- `🟡 partial`：结构可信，但细节还没完成真实环境核对
- `🟢 verified`：已基于真实 OpenClaw CLI、真实 `~/.openclaw/` 结构、当前仓库代码与自动化流程交叉验证

---

## 2026-04-24 当前结论

### docs/ 教程本体

- `33 章公开教程`：`🟢`
- `FAQ / 法律页 / 联系页 / 第三方致谢`：`🟢`
- `章节互链 / 目录导航 / 内部链接重写`：`🟢`
- `移动端文档可读性（表格 / 代码块 / 章节导航入口）`：`🟢`
- `07-实战` 新增 OpenClaw 能力总览、飞书交互卡片、模型配置、Gateway 链路、多 Agent 案例、运维 SOP、Prompt 库：`🟢`

### 档位 2 · templates/

- `shared/`：`🟢`
- `agents/`：`🟢`
- `skills/`：`🟢`
- `scripts/`：`🟢`
- `apply-openclaw-cli-hotfixes.template.mjs`：`🟢`
- `apply-hotfixes-usage.md` 口径已更新为“买家拿到稳定版 hotfix 脚本”：`🟢`

### 档位 2 · guided-install/

- `install.sh`：`🟢`
- `update.sh`：`🟢`
- `rollback.sh`：`🟢`
- `manifest / bundle-state` 协议：`🟢`
- `e2e-install`：`🟢`
- `e2e-update`：`🟢`

### 商业化与交付骨架

- `release-bundle.yml` 自动打包付费 bundle：`🟢`
- `buyer docs`（开箱 / 安装更新 / 售后 / 更新邮件模板）：`🟢`
- `上线当天清单 / 30 分钟执行单 / 10 行版 / 操作记录模板`：`🟢`
- `客服回复模板`：`🟢`

### site/ 站点

- `公开营销站`：`🟢`
- `法律 / 隐私 / 退款文案与当前产品口径一致`：`🟢`
- `/sign-in`、`/sign-up`：`🟢`
- `/account/downloads`、`/api/latest-bundle`：`🟢`
- `认证 / 支付 / 公开入口开关`：`🟢`
- `checkout / webhook / success` 的开关版正式骨架：`🟢`

---

## 当前已完成的关键收口

### 公开内容

- 全部章节已从早期草稿状态收口到可发布状态
- 新增 `07-实战`：最近新增能力总览、飞书交互卡片入门、周审卡片、回调处理器、验证与排障、模型中转排障、Gateway 链路、多 Agent 案例、日常运维、组件速查、AI Prompt 库
- 旧的 `.md` 内链已统一重写到真实站点路由
- 404 路由冲突已清理
- 移动端文档样式、表格、代码块、章节导航入口已做专门兼容处理

### 付费交付

- bundle 不再只是 `templates + guided-install`，而是带完整买家说明文档
- 买家不只会拿到 `install.sh`，还会拿到 `update.sh`
- 本地改过的文件不会被增量更新强行覆盖，而是生成 `.new`
- 付费更新期、交付方式、退款条件、下载权边界已经统一到一套口径

### 商业化上线准备

- `/account/downloads` 和 `/api/latest-bundle` 已接入长期骨架
- `/buy`、`/buy/success`、`/api/checkout`、`/api/webhook/creem` 已改成开关控制
- 入口显示与后端权限已拆成两组开关：
  - `COMMERCE_AUTH_ENABLED` / `COMMERCE_PAYMENTS_ENABLED`
  - `PUBLIC_COMMERCE_AUTH_ENABLED` / `PUBLIC_COMMERCE_PAYMENTS_ENABLED`

---

## 仍然故意保留为“待实际执行”的事项

这些不是“没做代码”，而是**等真实上线当天执行**：

- 拿到 Creem 邀请码
- 在 Cloudflare Pages 填真实 `CREEM_*`
- 在 Cloudflare Pages 填真实 `LATEST_BUNDLE_*`
- 打开认证 / 支付 / 入口显示开关
- 跑首单真实支付闭环验证

也就是说，当前剩余工作主要是**环境变量与实操执行**，不是继续补仓库骨架。

---

## 当前整体状态表

| 范围 | 状态 | 说明 |
|---|---|---|
| `docs/` 教程内容 | `🟢` | 已收口到当前可发布状态 |
| `templates/` | `🟢` | 模板、脚本、hotfix 稳定版可交付 |
| `guided-install/` | `🟢` | install/update/rollback 与状态协议已通 |
| `meta/` 商业化文档 | `🟢` | 上线、交付、售后、更新通知都已补齐 |
| `site/` 站点骨架 | `🟢` | 买家中心、支付闭环、入口开关已就位 |
| `真实支付开通` | `⏳` | 等 Creem 邀请码与真实环境变量 |

---

## 校对依据

本文件当前状态已对齐以下真实来源：

- 仓库当前 `main` 分支代码与最近收口提交
- `tests/e2e-install.sh`
- `tests/e2e-update.sh`
- `site/` 的 `npm run build`
- 当前 `.github/workflows/*.yml`

如果后续又做了大范围收口，请直接更新这份文件，不要再恢复“待新增”式清单。
