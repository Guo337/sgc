---
id: "pit-2026-00010"
layer: 3
type: pitfall
trigger: on-domain
scope: "knowledge-engineering"
project: memory-lib
domain: [knowledge-engineering, obsidian-behavior]
maturity: seed
provenance: manual
created: 2026-09-28
last-touched: 2026-09-29
summary: Obsidian Local REST API 的 frontmatter PATCH（及 UI 编辑器）把整个 YAML 从 inline 重排为 block 风格并剥引号，只认 inline 的离线解析器会把字段读成空
e:
  - { target: "[[PIT-插件密钥入库泄漏]]", rel: co_occurs, weight: 0.5 }
  - { target: "[[PIT-obsidian裸名误调GUI版]]", rel: co_occurs, weight: 0.5 }
---

# ⚠️ frontmatter 被重排为 block 风格致离线解析静默读空

## Bug 类型

格式重排导致静默数据丢失——经 Obsidian Local REST API 的 frontmatter PATCH 写入后，整个 YAML 块从 inline 风格（如 reflect-to 写成方括号内联列表）被重排为 block 风格（字段名后换行、值改成缩进短横线列表），且标量引号被剥离。只认 inline 的离线解析器把该字段捕获为空串，读成空列表。

## 根因

插件用 YAML 序列化器整体重写 frontmatter，而非外科式只改目标字段。本库约定 inline 列表 + 引号标量，`parse_inline_list` 只认方括号内联写法；遇到 block 风格时 `fm_get` 的正则在字段名那一行只捕获到空串。更糟：离线 backfill 若此时把字段名那一行替换为新的 inline 列表，下面缩进的短横线项会变成孤儿行，导致 YAML 损坏 + 旧值丢失。一次 REST backfill 实测 `git diff --numstat` 得 18 增 6 删（单字段追加绝不该动 24 行）即为证。

## 修复模式

1. 写入走**逐字节精确**的通道：离线原子写，或官方 CLI `create`（实测字节精确保留 inline frontmatter，见 [[PIT-obsidian裸名误调GUI版]] 的全路径启动器要求）；REST 降为只读通道（仅 GET document-map 验连通 + 取 ifMatch version token），`--apply` 明确拒绝；
2. 离线 backfill 加 block-style 守卫：目标字段值为空且下一行是缩进短横线项时 fail-loud 拒绝，而非静默把项变孤儿；
3. 集成第三方"定向修改"API 前，用可逆实测（apply 后看 `git diff`、再 `git checkout` 回滚）验其真实字节行为，别信表面 `exit 0`。

## 适用边界

- 有效：任何经 Obsidian UI 编辑器 / Local REST API / 外部 YAML 工具改写 frontmatter 的场景（三者都可能重排为 block 风格）；
- 无效：全程只用本库离线工具写 frontmatter（始终保持 inline）；
- 可迁移教训：第三方"只改一个字段"的 API 可能整体重排序列化格式；解析器要对格式漂移 fail-loud，集成前必须可逆实测字节行为。

## 来源

- log-2026-09-28-06（批次4）：REST backfill `--apply` 可逆实测，`git diff` 显示 inline 转 block + 剥引号（18 增 6 删），值虽加进但格式全变；`git checkout` 零残留复原。
- 与 [[PIT-插件密钥入库泄漏]] 同批（批次4 集成 Local REST API）发现。
