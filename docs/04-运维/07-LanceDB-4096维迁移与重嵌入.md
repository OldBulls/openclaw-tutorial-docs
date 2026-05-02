---
title: 04-运维 · 07 LanceDB 4096 维迁移与重嵌入
---

> 预计阅读：14 分钟
> 适用版本：memory-lancedb-pro 1.1.0-beta.10 / OpenClaw 2026.4.24 稳定基线 · 最后审核：2026-05-02
> 前置：[02-配置/01-记忆系统](../02-配置/01-记忆系统.md) / [04-运维/03-备份与恢复](./03-备份与恢复.md)
> 本章回答：**把 embedding 从 1024 维切到 4096 维时，为什么会报 `Vector dimension mismatch`？怎么迁移旧库而不丢记忆？**

---

## 先说结论

如果你把 embedding 模型从旧维度升级到新维度，比如：

- `Qwen/Qwen3-Embedding-0.6B / 1024`
- 升到 `Qwen/Qwen3-Embedding-8B / 4096`

真正的问题通常不是：

- 芯片不支持
- LanceDB 版本太旧

而是：

> **你当前这张 LanceDB 表是按旧维度建出来的。**

所以 runtime 一启动就会看到：

```text
Vector dimension mismatch: table=1024, config=4096
```

---

## 这类迁移的正确姿势

不要原地硬改。正确流程是：

1. 新建一个 **新的 dbPath**
2. 用新 embedding 重新生成所有向量
3. 写入新库
4. 校验数量和 ID 集合
5. 再把生产流量切过去

---

## 为什么不建议原地改

LanceDB 里已经落盘的 `vector` 列维度是固定的。  
把配置从 `1024` 改成 `4096` 之后：

- 旧表里的每一行向量还是 `1024`
- 新请求返回的是 `4096`

两边不一致，store 初始化时就会直接拒绝工作。

---

## 迁移前检查

### 1. 记录当前模型和维度

```bash
openclaw config get plugins.entries.memory-lancedb-pro.config.embedding
openclaw config get plugins.entries.memory-lancedb-pro.config.dbPath
```

### 2. 先看错误是不是维度错配

```bash
rg -n 'Vector dimension mismatch|table=1024|config=4096' ~/.openclaw/logs/gateway.err.log
```

### 3. 先做 dry-run 思维模型

你要同时有两套目录：

```text
旧库：~/.openclaw/memory/lancedb-pro
新库：~/.openclaw/memory/lancedb-pro-qwen3-emb8b-4096
```

旧库先保留，不要删。

---

## 生产迁移步骤

### 第 1 步：切新 dbPath

在 `openclaw.json` 里把 memory 插件改到新路径：

```json
"plugins": {
  "entries": {
    "memory-lancedb-pro": {
      "config": {
        "embedding": {
          "model": "Qwen/Qwen3-Embedding-8B",
          "dimensions": 4096
        },
        "dbPath": "~/.openclaw/memory/lancedb-pro-qwen3-emb8b-4096"
      }
    }
  }
}
```

### 第 2 步：停 gateway

```bash
openclaw gateway stop
```

迁移期间不要让在线流量继续往旧库和新库混写。

### 第 3 步：先确认旧库能读

```bash
openclaw memory-pro reembed \
  --source-db ~/.openclaw/memory/lancedb-pro \
  --dry-run
```

如果 dry-run 就报错，先别正式迁。

### 第 4 步：正式重嵌入

```bash
openclaw memory-pro reembed \
  --source-db ~/.openclaw/memory/lancedb-pro \
  --skip-existing
```

这一步做的事情是：

- 读旧库里的 `text/category/scope/metadata`
- 用你当前配置的新 embedding 重新生成向量
- 写入新库

### 第 5 步：校验数量

```bash
openclaw memory-pro stats
```

如果你要更严，自己写一段脚本对比：

- 旧库总条数
- 新库总条数
- ID 集合是否一致

注意：如果旧库里本来就有重复 ID，那么“总条数”可能不完全相等，但 **唯一 ID 集合** 才是关键。

### 第 6 步：启动 gateway

```bash
openclaw gateway start
openclaw gateway status
```

然后确认日志里不再出现维度错配：

```bash
rg -n 'Vector dimension mismatch' ~/.openclaw/logs/gateway.err.log
```

---

## 如果你有脚本会把 dbPath 写回旧值

这在生产上非常常见。  
你的脚本、baseline 或恢复逻辑可能还以为旧路径是：

```text
~/.openclaw/memory/lancedb-pro
```

这时可以加一个过渡兜底：

1. 保留旧 1024 库目录到别的名字
2. 把旧路径做成指向新库的符号链接

示例：

```bash
mv ~/.openclaw/memory/lancedb-pro \
   ~/.openclaw/memory/lancedb-pro.legacy-1024-YYYYMMDD-HHMM

ln -s ~/.openclaw/memory/lancedb-pro-qwen3-emb8b-4096 \
      ~/.openclaw/memory/lancedb-pro
```

这样就算某些脚本暂时还把 `dbPath` 写回旧名字，实际也还是落到新库。

---

## 怎么判断“是版本问题”还是“只是旧表维度问题”

看这两个证据：

### 情况 A：版本 / 安装不兼容

表现：

- 插件根本起不来
- LanceDB 模块加载失败
- `darwin-x64` / 原生模块报错

### 情况 B：只是旧表维度不匹配

表现：

- 插件能加载
- 新库能建
- 但只要继续读旧表，就报：

```text
table=1024, config=4096
```

这种情况下，不要先升级 LanceDB，先迁库。

---

## 实战经验

### 经验 1：`reembed` 是数据迁移，不是配置切换

很多人只改了：

- `embedding.model`
- `embedding.dimensions`

但没做 reembed。  
这不叫迁移，只叫把生产配置改坏。

### 经验 2：升级前先保留旧库快照

因为你后面可能要回答：

- 新 embedding 检索质量到底有没有变好？
- 哪条记忆是不是在迁移中丢了？

没有旧库对照，你只能猜。

### 经验 3：迁移后别忘了重跑 memory upgrade / 体检

如果你还有 legacy memories，没有升级成新 metadata 结构，迁完向量后最好顺手统一。

---

## 最小 checklist

- [ ] 新 embedding 模型和 dimensions 已确定
- [ ] `dbPath` 已切到新目录
- [ ] `reembed --dry-run` 通过
- [ ] `gateway` 已停
- [ ] 正式 `reembed` 完成
- [ ] 新旧库条数 / ID 集合已核对
- [ ] 已确认不再报 `Vector dimension mismatch`
- [ ] 已确认在线流量切到新库
- [ ] 旧库仍保留，可回看

---

## 一句话经验

> **4096 维迁移本质上是一次“新库重建 + 向量重嵌入”，不是单纯改配置。**

只改 `dimensions` 而不迁库，等于主动制造事故。
