---
id: "c-2026-00008"
layer: 2
type: procedure
project: memory-lib
domain: [knowledge-engineering]
maturity: seed
provenance: reflect
created: 2026-09-28
last-touched: 2026-11-20
summary: 官方 obsidian CLI 第一方通道命令速查：读/写/探活命令 + Windows UTF-8 前置 + eval 禁令
k: []
v: []
e:
  - { target: "[[SH-knowledge-engineering]]", rel: part_of, weight: 0.7 }
  - { target: "[[SPEC-READ]]", rel: derives_from, weight: 0.6 }
  - { target: "[[SPEC-WRITE]]", rel: derives_from, weight: 0.5 }
  - { target: "[[记忆通道使用纪律]]", rel: implements, weight: 0.6 }
aliases: ["obsidianCLI速查", "CLI命令速查"]
---

# 官方obsidianCLI命令速查

## 定义

官方第一方 `obsidian` CLI（D19/D23 写入主通道、读取兜底；D29 读取主改插件 MCP，IPC 连 App）的常用命令、Windows 编码前置与安全禁令速查。

## 核心

reflect 编译自候选结晶（来源：log-2026-09-26-01、log-2026-09-27-01）：

- **读**：`obsidian read file=`、`search query=`、`backlinks file=`、`unresolved`（只报断链目标）；
- **写**：`append file= content=`、`create name= content= template=`、`property:set name= value= file=`（写入须 App 运行，`silent` 防自动打开，多行用 `\n`）；
- **探活**：`obsidian version`（先探活再用，返回非零/超时则 CLI 不可用；**退离线须 Owner 批准，D33**，不得自行降级）；
- **Windows 前置**：消费 CLI 输出前设 `[Console]::OutputEncoding=[Text.Encoding]::UTF8`（+ `$env:PYTHONUTF8=1`）；
- **禁令**：`obsidian eval`（任意 JS）一律不用（SPEC-WRITE 安全 A）。

## 存疑

- 命令名/参数随 obsidian CLI 版本可能变（本机实测 v1.13.7）；以 SPEC-READ/SPEC-WRITE 访问·写入通道节为准，本卡是速查快照。

## 应用

本库**写入**（append/create/property:set）优先走此 CLI（byte-exact、不重造平行工具链，B 决策）；**读取/查询**（链接图/全文/反链/断链）主走插件 MCP（D29，见 [[SPEC-READ]]），CLI 读命令（search/read/backlinks/unresolved）为兜底；App 未运行先尽自启义务，CLI 仍不可用时**退 Python/grep 离线须 Owner 批准（D33）**。
