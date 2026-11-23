---
id: "pit-2026-00004"
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
summary: 仓库根≠vault根时，.gitignore 含斜杠的模式锚定到仓库根，匹配不到子目录 vault/.obsidian/，忽略失效
e:
  - { target: "[[PIT-嵌套wikilink识别]]", rel: co_occurs, weight: 0.4 }
---

# ⚠️ .gitignore 子目录 vault 锚定失效

## Bug 类型

配置失效——根 `.gitignore` 的忽略模式匹配不到子目录 vault 内的目标，`.obsidian/workspace.json` 等未被忽略，每次 `git status` 冒出来。

## 根因

`.gitignore` 中**含斜杠的模式**（如 `.obsidian/*`）相对于该 `.gitignore` 文件所在位置（= 仓库根）锚定，只匹配仓库根下的 `.obsidian/`，**匹配不到子目录** `vault/.obsidian/`。本库仓库根 = 工作区根、vault 在其子目录，故根 `.gitignore` 的 `.obsidian/*` 对 `vault/.obsidian/` 无效（`git check-ignore` 返回空为证）。

## 修复模式

1. vault 在子目录时，模式带 vault 前缀：`vault/.obsidian/workspace.json` 等；
2. 用 `git check-ignore -v <path>` 验证生效（命中会打印匹配的模式与来源行）；
3. 有意保留可提交的配置（app.json/plugins）与须忽略的（workspace.json 等易变项）分别处理。

## 适用边界

- 有效：**仓库根 ≠ vault 根**（vault 在子目录）时；
- 无效：vault 即仓库根，则 `.obsidian/*` 直接生效、无需前缀；
- 可迁移教训：gitignore 含斜杠模式是"相对 .gitignore 位置锚定"，嵌套项目结构下须显式带路径前缀。

## 来源

- log-2026-09-27-02（外部评审 #7）：`git check-ignore` 返回空证实锚定失效，加 `vault/` 前缀后 `workspace.json` 正确忽略（`!!` 验证）。
- 与 [[PIT-嵌套wikilink识别]] 同属 Obsidian 工程坑。
