---
id: "c-2026-00018"
layer: 2
type: procedure
project: memory-lib
domain: [knowledge-engineering, retrieval, memory-governance]
maturity: seed
provenance: manual
created: 2026-11-13
last-touched: 2026-11-20
summary: VSCode/Copilot 经 mcp-config.json tools 数组把 obsidian 6 只读白名单升级为配置级硬约束（子 Agent 继承），附 MCP/REST 实测手感对比
k: []
v: []
e:
  - { target: "[[SH-knowledge-engineering]]", rel: part_of, weight: 0.7 }
  - { target: "[[SPEC-READ]]", rel: derives_from, weight: 0.7 }
  - { target: "[[REST读库速查]]", rel: contrasts_with, weight: 0.7 }
  - { target: "[[记忆通道使用纪律]]", rel: implements, weight: 0.6 }
aliases: ["MCP白名单", "MCP配置级过滤", "obsidian MCP注册", "MCP使用手感"]
---

# MCP配置级白名单与使用手感

## 定义

在 **VSCode/Copilot** 环境把 obsidian 插件 MCP 注册进 harness，并用原生 `tools` 数组把「6 只读白名单」从 Qoder 式的纯自律升级为**配置级硬约束**（子 Agent 继承）的操作速查 + MCP/REST 真实使用手感对比。与 [[REST读库速查]]（无 MCP 环境）成对：有 MCP 走本卡，无 MCP 走 REST 卡（SPEC-READ 环境分流 D30/D31）。

## 核心

**注册落点（官方核实，勿凭记忆）**：
- Copilot harness 原生读 **`~/.copilot/mcp-config.json`**（`mcpServers` 格式，全局）或工作区根 **`.mcp.json`**。
- **不直接读** `.vscode/mcp.json` / profile `mcp.json`——那是旧 Local harness 的 `servers` 格式（需 forwarding，不可假设生效）。
- 本机配置（零字面密钥）：

```json
{
  "mcpServers": {
    "obsidian": {
      "type": "http",
      "url": "http://127.0.0.1:27123/mcp/",
      "headers": { "Authorization": "Bearer ${OBSIDIAN_API_KEY}" },
      "tools": ["search_simple","search_query","vault_read","vault_list","vault_get_document_map","tag_list"]
    }
  }
}
```

**配置级白名单实证（D31，本机 2026-11-13）**：
- 插件 MCP 端点 `POST /mcp/`（Streamable HTTP，`Mcp-Session-Id` 会话头）`tools/list` **暴露 20 工具含 14 写**——**过滤不在插件侧**。
- `tools` 数组列 6 只读 → Copilot **主 Agent 工具表只有 6 只读**，14 写工具被挡在表外（无法调用一个不存在的工具）。
- **子 Agent 继承**：general-purpose 探针实测 **6 只读 / 0 写**，vault_list 调通。官方文档未 guarantee 子 Agent 继承，本机实测确认。
- 对比 Qoder：IDE 丢弃 `includeTools`、20 工具全暴露 → 只能自律（[[log-2026-09-29-04]]）。**VSCode 能把只读白名单做成硬约束，Qoder 不能**。

**密钥纪律**：`OBSIDIAN_API_KEY` 经 `setx` 设用户环境变量（从插件 data.json **in-memory 读、零回显零落盘**）；配置用 `${OBSIDIAN_API_KEY}` 引用，**字面密钥绝不进配置/库/git**。`setx` 只对新进程生效 → 设后须**重启 VSCode**。

**MCP vs REST 实测手感（同一插件两面，本机计时）**：

| 维度 | 插件 MCP | 插件 REST（rest_read.py） |
|------|----------|---------------------------|
| 读卡 | `vault_read` **一次给全文+frontmatter+links+backlinks+unresolvedLinks+stat** | `GET /vault/{file}` 只给原文字节；反链/links 须另取 |
| 反链（入边） | **原生 backlinks**（App 实时索引，含刚落盘文件） | **无端点**，退官方 CLI `obsidian backlinks` 或离线扫 |
| 定向读 | `vault_read` targetType=heading/block/frontmatter **只回目标段**（省 token） | 无，须读全文再切 |
| 结构化查询 | `search_query` JsonLogic 一次过滤（实测 project== 返 107 文件） | `POST /search/` 同语义，须 vendor Content-Type |
| 全文检索 | `search_simple` 结构化 JSON | `POST /search/simple/?query=`（POST 非 GET） |
| 延迟 | 工具调用直达（无进程启动） | 每次起 Python 进程 ~120–185 ms（probe/read/search/list 实测） |
| 编码坑 | 结构化 JSON，无 cp936 坑 | 内部 UTF-8 正确；**Windows 经 subprocess/here-string 传中文 argv 可能丢字成 `?`**（见存疑） |

**手感结论**：MCP 是「更好的眼睛」——一次调用拿全事实（尤其反链+定向读），多跳读顺 backlinks 直达，省 token 省往返；REST 是无 MCP 环境的等价兜底，能力差主要在反链与定向读。**两者都只读、都喂事实给薄护栏，不是引擎**（主文本约束弱脚本约束）。

**按需自启 App（Owner 授权，仅限 Obsidian + 本目录，D31）**：REST/MCP 需 App 运行而 App 关着时，Agent 可自启 Obsidian + 探活（含身份探针），用完按需留/关。**实测（2026-11-15）**：`Start-Process Obsidian.exe` 生效；`Start-Process Obsidian.com` 在 PowerShell 下**不拉起进程**（.com 是 CLI 入口非 GUI 启动器）。冷启动分阶段 ~25-30s：App 进程 → REST 200 → CLI 命令面就绪。自启后 harness MCP 客户端可能缓存旧失败状态需重载窗口重连（CLI/REST 即时可用）。

## 存疑

- 子 Agent 继承白名单是**本机实测**（general-purpose 探针），官方文档未 guarantee；换 harness/版本须重验，勿假设普遍成立。
- `tools` 白名单是 harness 侧过滤；**插件侧仍暴露 20 工具**——若换不守 `tools` 的 harness，写工具会重新可达，自律红线仍须守（能力≠许可）。
- Windows 中文路径编码：rest_read.py **本身 UTF-8 正确**（直接 CLI `read "中文路径"` 读通、进程内 unicode 读通）；但经 **PowerShell here-string `@'...'@` 注入中文字面量到 python stdin** 时实测丢字成 `?`（测试注入层 artifact，非脚本 bug）。子 Agent 传中文路径宜直接 CLI 调或 unicode 转义，勿经 here-string 中转。
- 延迟数字为本机单次实测（~120–185 ms REST 含进程启动），非统计均值；MCP 工具调用延迟未单独计时。
- 本机走 loopback 明文 HTTP 27123（用户知情选择）；TLS 27124 自签证书路线未走（[[PIT-本地自签CA信任与回环明文降级]]）。

## 应用

VSCode/Copilot 接 obsidian MCP：写 `~/.copilot/mcp-config.json`（tools 列 6 只读 + `${ENV}` 密钥）→ setx 环境变量 → 重启 VSCode → 验工具表只 6 只读 + 子 Agent 继承。读库优先 MCP（反链/定向读强），无 MCP 退 [[REST读库速查]]。与 [[记忆通道使用纪律]]（七律：先认人/守白名单/净字节）、[[SPEC-READ]]（A.6 两档白名单权威）配套。