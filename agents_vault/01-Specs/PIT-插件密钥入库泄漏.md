---
id: "pit-2026-00009"
layer: 3
type: pitfall
trigger: on-domain
scope: "knowledge-engineering"
project: memory-lib
domain: [knowledge-engineering, obsidian-behavior]
maturity: seed
provenance: manual
created: 2026-09-28
last-touched: 2026-09-28
summary: Obsidian 插件把密钥写进 .obsidian/plugins/<id>/data.json；.gitignore 若只排除 workspace.json 而保留其余配置，git add -A 会把密钥提交入库
e:
  - { target: "[[PIT-gitignore子目录锚定失效]]", rel: co_occurs, weight: 0.6 }
---

# ⚠️ 插件 data.json 密钥入库泄漏

## Bug 类型

配置缺失导致密钥泄漏——`.gitignore` 只忽略了 `vault/.obsidian/workspace.json` 等易变项，未忽略 `vault/.obsidian/plugins/`；安装 Local REST API 等插件后，含 API Key 的 `data.json` 与大体积 `main.js` 变成未跟踪文件，一次 `git add -A` 就把密钥提交进 Git 历史（且历史一旦含密钥即视为泄漏，须轮换）。

## 根因

Obsidian 插件的运行时数据（含密钥，如 Local REST API 的 `apiKey`）写在 `<vault>/.obsidian/plugins/<id>/data.json`。本库 `.gitignore` 原策略是"保留 .obsidian 其余配置（插件/主题/热键等可复现设置）"，只排除 workspace.json/cache，于是 plugins 整个目录落入可提交范围。密钥文件与可复现配置混在同一目录、未被单独排除。是 [[PIT-gitignore子目录锚定失效]] 的同族延伸（都属 .obsidian 忽略覆盖范围问题）。

## 修复模式

1. `.gitignore` 忽略整个 `vault/.obsidian/plugins/`（`main.js` 可重装、`data.json` 含密钥，都不该入库）；
2. "装了哪些插件"由已跟踪的 `community-plugins.json` 复现，无需跟踪插件本体；
3. 用 `git check-ignore -v` 验证命中 + `git status` 确认 plugins 目录不再冒出来；
4. 密钥只经环境变量或插件配置 in-memory 使用，绝不写入任何层级、绝不打印到日志/终端。

## 适用边界

- 有效：vault 纳入 git 版本控制、且安装了会把密钥写进 data.json 的插件（Local REST API 等）；
- 无效：vault 不入 git，或插件不写密钥到配置目录；
- 可迁移教训：任何把第三方运行时数据（可能含密钥）放进"被跟踪配置目录"的工具，都要为该目录单独加忽略规则，别默认"配置=可提交"。

## 来源

- log-2026-09-28-06（批次4）：用户装 Local REST API 插件后，`git status` 显示 plugins 目录未跟踪且未忽略；加 `vault/.obsidian/plugins/` 规则后 `git check-ignore -v` 命中、`git status` 清零。
- 扩展 [[PIT-gitignore子目录锚定失效]]（pit-2026-00004）。
