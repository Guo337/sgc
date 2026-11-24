---
id: "proj-{yyyy}-{slug}"
type: project-spec
project: ""              # 所属项目名（多项目隔离，D13）
owner: ""                # 内容权威（项目/人名）——本项目只校验结构不审内容
status: ACTIVE           # ACTIVE / SUPERSEDED（SUPERSEDED 须补 supersedes 指针）
created: {{date}}
last-touched: {{date}}
summary: ""              # ≤50 token 自包含：本规格管什么流程
---

# {规格名}

> 项目流程规格：内容权威=项目 Owner，对 TTL runtime 零权威（NON-AUTHORITY，见 [[01-AUTHORITY]] §2）。
> 可就地演进（修订走项目区自己的约定，建议留 changelog 节）；不走本库 corr 流程。
> 落点 `06-Project/specs/`；须在 [[00-INDEX]]（06-Project 入口）登记链接（PROJ_ENTRY）。

## 流程

<!-- 规定项目流程怎么走：步骤/角色/触发条件 -->

## 库内指针

<!-- 本规格引用的库内记忆（[[卡]]/[[SPEC-*]]），链接须可达（BROKEN_LINK 照判） -->

## changelog

<!-- 可选：就地演进留痕（日期+改了什么+为什么）；项目区不强制 corr -->