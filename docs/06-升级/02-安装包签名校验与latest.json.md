---
title: 06-升级 · 02 安装包签名校验与 latest.json
---

> 预计阅读：9 分钟
> 适用版本：OpenClaw 2026.4.14+ · 最后审核：2026-04-25
> 前置：[00-npm升级兼容](./00-npm升级兼容.md) / [04-运维/02-升级流程](../04-运维/02-升级流程.md)
> 目标：看懂 OpenClaw 安装包更新链里的 `latest.json`、`.sig` 和公钥各自负责什么，知道买家侧怎么做最小签名校验

---

## 为什么升级链不该只看版本号

如果你只看到：

- “有新版本了”
- “下载链接没变”
- “SHA256 也对”

这还不够回答一个更关键的问题：

**这份元数据和安装包到底是不是发布者自己发出来的？**

`latest.json + detached signature + public key` 这条链解决的就是这个问题。

---

## 当前签名链里有哪几类文件

一条完整的买家侧升级校验链，通常会看到这些文件：

| 文件 | 作用 |
|---|---|
| `latest.json` | 最新版本元数据 |
| `latest.json.sig` | `latest.json` 的 detached signature |
| `openclaw-bundle-vX.Y.Z.zip` | 实际安装包 |
| `openclaw-bundle-vX.Y.Z.zip.sig` | ZIP 的 detached signature |
| `openclaw-bundle-vX.Y.Z.manifest.json` | 清单文件 |
| `openclaw-bundle-vX.Y.Z.manifest.json.sig` | 清单文件的 detached signature |
| `bundle-signing-public.pem` | 用来验签的公钥 |

这几份文件缺一不可，但职责不一样。

---

## 各自负责什么

### `latest.json`

回答的是：

- 最新版本号是什么
- 下载链接是什么
- 清单文件在哪
- SHA256 是多少
- 是否要求强制签名校验
- 这次版本的摘要和历史是什么

也就是“更新说明书”。

### `.sig`

回答的是：

- 这份文件是不是被对应私钥签出来的
- 中间有没有被篡改

也就是“来源和完整性证明”。

### `bundle-signing-public.pem`

回答的是：

- 你用哪把公钥去验证上面的签名

也就是“验签入口”。

---

## 为什么 `SHA256` 不等于签名链

很多人会误以为：

> 有 SHA256 就够了。

不够。因为如果攻击者能同时改：

- ZIP
- `latest.json`
- `sha256`

那你对出来的哈希仍然可能是“攻击者版本的正确哈希”。

签名链多做的一步是：

- 让 `latest.json` 自己也带签名
- 让 ZIP 和 manifest 也各带一个 detached signature
- 让买家侧只信任手里的公钥

这才是“元数据本身也被保护”。

---

## `latest.json` 里最值得看的字段

买家侧至少应该关心这些：

| 字段 | 含义 |
|---|---|
| `version` | 当前最新版本 |
| `downloadUrl` | ZIP 下载地址 |
| `manifestUrl` | 清单文件地址 |
| `sha256` | ZIP 摘要 |
| `downloadSignatureUrl` | ZIP 签名地址 |
| `manifestSignatureUrl` | 清单签名地址 |
| `publicKeyUrl` | 公钥地址 |
| `signatureAlgorithm` | 当前签名算法 |
| `signatureRequired` | 是否强制验签 |

如果 `signatureRequired=true`，就不能把验签当可选装饰。

---

## 买家侧最小手工验签

当前模板里已经提供了：

```bash
bash ~/.openclaw/scripts/verify-bundle-signature.sh --file FILE
```

最常见的两种用法：

```bash
# 验最新元数据
bash ~/.openclaw/scripts/verify-bundle-signature.sh \
  --file /tmp/latest.json \
  --signature /tmp/latest.json.sig \
  --public-key ~/.openclaw/scripts/bundle-signing-public.pem

# 验下载下来的 ZIP
bash ~/.openclaw/scripts/verify-bundle-signature.sh \
  --file /tmp/openclaw-bundle-vX.Y.Z.zip \
  --signature /tmp/openclaw-bundle-vX.Y.Z.zip.sig \
  --public-key ~/.openclaw/scripts/bundle-signing-public.pem
```

如果你不传 `--signature`，脚本默认会找同名 `.sig`。

---

## `--strict` 到底是什么意思

这个脚本有一个很实用的开关：

```bash
bash ~/.openclaw/scripts/verify-bundle-signature.sh --file FILE --strict
```

它的返回码语义大致是：

- `0`：验签通过
- `1`：验签失败
- `2`：缺少公钥 / 缺少签名 / 缺少 `openssl`（非 strict 模式下）

也就是说：

- 非 strict：缺材料时会返回“无法校验”
- strict：缺材料也算失败

如果 `latest.json` 已明确要求 `signatureRequired=true`，买家侧就应该按 strict 心智处理。

---

## 公钥通常从哪里读

当前模板默认会优先找：

```text
~/.openclaw/scripts/bundle-signing-public.pem
~/.openclaw/bundle-signing-public.pem
```

这也是为什么公开文档里一直建议：

- 公钥跟安装包一起给到买家侧
- 私钥永远只留在发布者一侧

不要把“我自己从聊天窗口复制一段公钥文本”当成正式验签流程。

---

## 自动更新链现在已经会做哪些检查

如果作者开启了签名链，买家侧现在通常会做两层自动校验：

1. `check-bundle-update.py`  
   优先校验 `latest.json.sig`
2. `approve-bundle-update.sh`  
   下载 ZIP 后优先校验 `zip.sig`

也就是说，公开文档现在建议的顺序是：

```text
先信任元数据
  ↓
再信任具体安装包
```

而不是跳过 `latest.json` 直接信 ZIP。

---

## 一条手工校验的最小流程

如果你想人工抽检一次，最小流程如下：

```bash
# 1. 下载元数据、签名、公钥
curl -L -o /tmp/latest.json https://downloads.example.com/bundle/latest.json
curl -L -o /tmp/latest.json.sig https://downloads.example.com/bundle/latest.json.sig
curl -L -o /tmp/bundle-signing-public.pem https://downloads.example.com/bundle/bundle-signing-public.pem

# 2. 验 latest.json
bash ~/.openclaw/scripts/verify-bundle-signature.sh \
  --file /tmp/latest.json \
  --signature /tmp/latest.json.sig \
  --public-key /tmp/bundle-signing-public.pem \
  --strict
```

通过后，再从 `latest.json` 里取 ZIP 和 manifest 地址继续校验。

---

## 什么时候要特别警惕

下面这些情况都应该停下来：

1. `signatureRequired=true`，但本地没有公钥
2. `latest.json` 有签名字段，但对应 `.sig` 不存在
3. ZIP 的 SHA256 对得上，但 `.sig` 验不过
4. 公钥来源和作者给你的安装包来源不一致
5. 你手工改过 `latest.json` 后还想继续拿它去验签

这几种都不是“小告警”，而是应该先阻断更新。

---

## 红线：不要这样做签名校验

### ❌ 不要只看 ZIP 的 SHA256

哈希只能证明“你下载到的是某个固定文件”，不能证明这个文件就是作者发的那份。

### ❌ 不要在 `signatureRequired=true` 时忽略验签失败

这时候再继续更新，等于主动绕过安全闸门。

### ❌ 不要从未知来源替换公钥

公钥本身也是信任链的一部分。来路不明的公钥没有意义。

### ❌ 不要改了 `latest.json` 还拿它验签

任何修改都会让 detached signature 失效，这是设计本意，不是 bug。

---

## 验证清单

| 检查项 | 怎么验 |
|---|---|
| `latest.json` 可访问 | 能正常下载最新元数据 |
| 元数据有签名字段 | `downloadSignatureUrl / manifestSignatureUrl / publicKeyUrl / signatureAlgorithm` 不为空 |
| 公钥能读到 | `bundle-signing-public.pem` 在本地可访问 |
| 手工验签可通过 | `verify-bundle-signature.sh --strict` 返回 0 |
| 自动更新链会验 | `check-bundle-update` 和 `approve-bundle-update` 不会跳过签名链 |
| `signatureRequired` 语义明确 | 需要强制时不会被当成“只是提醒” |

---

## 下一步

- [04-运维/02-升级流程](../04-运维/02-升级流程.md) —— 真正执行升级的运维侧顺序
- [04-运维/05-支持包与脱敏排障](../04-运维/05-支持包与脱敏排障.md) —— 验签异常时怎么把证据打给作者
- [07-实战/00-最近新增能力总览](../07-实战/00-最近新增能力总览.md) —— 升级之外，继续看更完整的实战链路

---

> **本章准确性保证**
> 本章对齐了当前仓库里的 `verify-bundle-signature.sh`、bundle 签名配置说明以及买家侧更新链对签名字段的实际使用方式，未包含作者发布后台或站点运营系统实现细节。

---

**导航**：[← breaking change 应对](./01-breaking-change-应对.md) · [📖 目录](../00-先读我.md) · [→ 07-实战 · 最近新增能力总览](../07-实战/00-最近新增能力总览.md)
