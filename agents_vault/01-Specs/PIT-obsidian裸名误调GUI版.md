---
id: "pit-2026-00011"
layer: 3
type: pitfall
trigger: on-domain
scope: "knowledge-engineering"
project: memory-lib
domain: [knowledge-engineering, obsidian-behavior]
maturity: seed
provenance: manual
created: 2026-09-29
last-touched: 2026-09-29
summary: 裸名 obsidian 经 Windows CreateProcess 补 .exe 误启 225MB GUI 版，致 property:set 等命令 rc=-1；须用 shutil.which 解析全路径启动器 Obsidian.com
e:
  - { target: "[[PIT-frontmatter重排为block风格]]", rel: co_occurs, weight: 0.5 }
---

# ⚠️ obsidian 裸名经 CreateProcess 误调 GUI 版致 CLI 命令 rc=-1

## Bug 类型

调用陷阱导致的假崩溃 + 误判——在 Windows 上用 Python `subprocess.run(["obsidian", ...])` 传**裸名**时，`property:set`/`property:read`/`search:context` 等"冒号命名空间 + 参数"命令一律返回 `rc=4294967295`（即 -1）、无 stdout/stderr，看起来像"命令本身在此环境不可用"。而 `version`/`create`（正文）/`append`/`search`/`daily:path`/`properties` 却正常，令误判更像真的（部分命令可用）。

## 根因

`Obsidian` 安装目录同时存在两个可执行体：**22KB 的控制台 CLI 启动器 `Obsidian.com`** 与 **225MB 的 GUI 主程序 `Obsidian.exe`**。Windows `CreateProcess` 对裸名默认优先补 `.exe`，于是 Python 裸名 argv 启动的是 GUI 版；GUI 版对这些 CLI 子命令的参数解析失败即返回 -1。而 `shutil.which`/cmd.exe 按 `PATHEXT`（`.COM` 先于 `.EXE`）选中的是正确的 `.com` 启动器——所以 `shell=True`（经 cmd）能跑通、裸名 argv 跑不通，差异被误当成"引号/冒号"问题，实为**调错了二进制**。

## 修复模式

1. **禁止裸名 argv**：用 `shutil.which("obsidian")` 解析**全路径**再传给 subprocess（`PATHEXT` 会选中 `.com`）；若解析结果落在 `.exe`，优先取其同名 `.com` 兄弟文件。全路径 `.com` + argv 列表 = 无 shell、免注入、可靠。
2. **写后回读校验**：CLI 写入后从磁盘回读、与意图内容逐字节比对，不符即 fail-loud，绝不信未校验的写入。
3. **黑盒崩溃先怀疑调用层**：`rc=-1` 且部分同类命令可用时，先排除"调错二进制/参数解析"，再谈"命令不可用"；查官方文档 + 加对照命令（如另一个冒号命令）快速定位。

## 适用边界

- 有效：Windows 平台经 Python/程序化 subprocess 调 obsidian CLI（裸名 → `.exe` 陷阱）；
- 无效：macOS/Linux（通常单一二进制，无 `.com`/`.exe` 同名分歧）；在真实终端手敲 `obsidian`（走 `PATHEXT` 选中 `.com`，正常）；
- 可迁移教训：同名不同扩展的多二进制环境下，"命令崩溃"可能纯属解析到了错误的可执行体；程序化调用应显式解析全路径，别依赖裸名。

## 来源

- log-2026-09-29-01（批次4 延伸 wA2/wA3）：8 轮黑盒实测一度误判"property:set 不可用、CLI 写不了 frontmatter"，经用户提示查官方文档 + 加对照实验，`Get-ChildItem` 证 `Obsidian.com`(22KB) vs `Obsidian.exe`(225MB)，全路径 `.com` argv 后 create/append/property:set 全 rc=0，误判推翻。
- 与 [[PIT-frontmatter重排为block风格]] 同属"Obsidian CLI 写入双坑"；命令清单见 [[官方obsidianCLI命令速查]]。
