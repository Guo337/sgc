---
id: "c-2026-00017"
layer: 2
type: procedure
project: memory-lib
domain: [knowledge-engineering, retrieval]
maturity: seed
provenance: manual
created: 2026-11-13
last-touched: 2026-11-25
summary: 无 MCP 环境经 Local REST API 只读读库的速查：实测端点表+身份探针+只读白名单+密钥纪律+降级链
k: []
v: []
e:
  - { target: "[[SH-knowledge-engineering]]", rel: part_of, weight: 0.7 }
  - { target: "[[SPEC-READ]]", rel: derives_from, weight: 0.7 }
  - { target: "[[官方obsidianCLI命令速查]]", rel: contrasts_with, weight: 0.6 }
  - { target: "[[记忆通道使用纪律]]", rel: implements, weight: 0.6 }
aliases: ["REST读速查", "REST读库", "LocalRESTAPI读速查"]
---

# REST读库速查

## 定义

在**没有 obsidian MCP 工具表**的环境（如 VSCode/Copilot 全新子 Agent）里，经同一 Local REST API 插件的**裸 HTTP 只读端点**读库的速查。MCP 与 REST 是同一插件的两个面：有 MCP 走 MCP 工具，无 MCP 走本卡 REST；读的是同一个库、同一套语义（SPEC-READ 环境分流读通道，D30）。

## 核心

**实测端点表**（插件 5.3.1 / Obsidian 1.15.1，2026-11-13 本机逐一验证；base = `http://127.0.0.1:27123`，Bearer apiKey）：

| 能力 | 方法 + 路由 | 关键参数 | 返回 |
|------|------------|----------|------|
| 探活 | `GET /` | — | manifest（id/version/status） |
| 读原文 | `GET /vault/{file}` | — | text/markdown 原始字节 |
| 结构/版本 | `GET /vault/{file}` | Accept: `application/vnd.olrapi.document-map+json` | version token + 标题树 + frontmatterFields |
| 按节读取 | `GET /vault/{file}/heading/{H1}/{H2}/…` | 路径各段=**全标题逐字精确**（含括号/版本号长尾，先 document-map 取标题再拼；各段 URL 编码） | 该节 markdown（2026-11-20 活体 200：「角色」节=全文 2.9%；H1 含版本号→标题改则路径漂，失败先重取 map） |
| 列目录 | `GET /vault/{dir}/` | 尾斜杠 | `{files:[…]}` |
| 全文检索 | `POST /search/simple/` | **`?query=`+`contextLength`（query string，非 JSON body）** | `[{filename,score,matches:[{start,end,source}],context}]` |
| 结构化查询 | `POST /search/` | **Content-Type=`application/vnd.olrapi.jsonlogic+json`**，body=JsonLogic | `[{filename,result}]` |
| 标签 | `GET /tags/` | — | `{tags:[{name,count}]}` |
| 路由契约 | `GET /openapi.json` | — | 全路由 schema |

**两个易踩的形状坑（实测纠正，勿凭记忆）**：
- `search/simple` 是 **POST + `?query=`**，不是 GET、也不是 JSON body（GET 它返 404，JSON body 返 400 "A single '?query=' parameter is required"）。
- `/search/` 的 JsonLogic 须把 vendor type 放在 **Content-Type**（不是 Accept）；放错返 400 "Unknown or invalid Content-Type"。

**REST 与 MCP 的能力差（如实标注）**：
- REST **无独立 backlinks 端点**。MCP `vault_read` 原生返反链（App 实时索引），REST 裸端点没有；要入边须退官方 CLI `obsidian backlinks file=` 或离线扫 `[[wikilink]]`。
- REST 全文检索的 `context` 片段可能含 `\r\n`（插件切片行为）；`GET /vault/{file}` 读原文则与磁盘字节一致（身份探针据此 PASS）。

**纪律（随文本传递，子 Agent 冷启动须守）**：
1. **先认人**：读前先 `GET /vault/00-ROOT.md` 算 sha256 对磁盘——不符即插件答的是另一个同名库，绝不采信（撞名事故同款守卫）。
2. **只读白名单**：只 GET + 两个 search POST（皆为读）；**绝不** POST/PATCH/PUT/DELETE 到 `/vault/`，绝不 `command_execute`/`eval`。REST 14 个写端点可达 ≠ 许可（能力≠许可）。
3. **密钥纪律**：apiKey 从插件 `data.json` in-memory 读，**绝不打印/落盘/入库**（真安全红线；data.json 已 gitignore）。REST 读 `/vault/` 须带 `Authorization: Bearer <apiKey>` 头（根端点 `GET /` 探活不需鉴权，故探活 200 ≠ 读可用）。**外来审核者取只读 key 属合规动作**（只读用途+in-memory+不入库三条满足即可，无需另批；或经 `rest_read --key-from-plugin` 代取）；不取 key 时官方 CLI 是合规替代通道，不是降级失败。
4. **降级链（D33/D33-b）**：REST 不可达（App/插件未运行）→ 先尽自启义务 → 官方 CLI；CLI 也不可用 → **呈报 Owner 批准后才许离线 grep**，不得自行降级；不硬失败、不归咎用户（运行时状态非用户过错），但未获批准不擅自磁盘读知识。**例外（D33-b）**：非交互只读审计子 Agent 无 Owner 回路时，**Reader 权限**（审核者承载，略次于 Owner）可批准只读离线降级（四条全满足，见 [[role-ke-auditor]] / [[SPEC-READ]] D33-b）。
5. **结构化优先**：REST 返 JSON，无 CLI 的 cp936 解码坑（D10 是 CLI 的坑，REST 没有）。
6. **副本分析口径**（Owner 2026-11-22 裁，corr-102）：经合规通道（MCP/REST/CLI）取回的库内容**副本**（如导出到库外 scratch 的文本），对其用 grep/view 等本地工具分析属**工具面**操作，不算通道绕过——通道纪律约束的是**原件读取**；副本分析仍须守临时件用完即删。库外文件（社区源码/Audit 底稿）本就不在通道纪律范围。

**现成助手**：`90-Meta/scripts/rest_read.py`（真值源=脚本卷 [[rest_read]]，用时经 [[vol_extract]] 抽取；[[rest_read#VERSION|v0.2.1]]，纯标准库、只读、英文注释）封装上表 + 身份探针 + 降级，子命令 `probe/read[--map|--heading]/list/search/tags`，`--self-test` 15 规则离线自检（含 CRLF 探针回归）。**按节读取**（v0.2.0）：`read <file> --heading H1 --heading H2 …` 每层一个 `--heading`、每段全标题逐字精确（段内 `/` 自动编码 %2F，含斜杠标题无歧义）；标题写错时 rc=1 且 stderr 附**可用标题树**（喂事实不猜 fuzzy），照树改参重试即可；与 `--map` 互斥。**参数顺序**：全局 `--vault <库根>` 须放在子命令**之前**（如 `rest_read.py --vault <库根> probe`），放后面会被 argparse 拒绝。它是"更好的眼睛"（喂事实给薄护栏），**不是引擎**——无缓存/索引/语义路由/写路径。

## 存疑

- 端点形状随插件版本可能变（本机 5.3.1 实测）；升级后须重探 `GET /openapi.json` 复核，勿假设不变。
- `/search/` JsonLogic 仅验证了 `frontmatter.project==` 一例；复杂逻辑（glob/or/嵌套）未逐一实测。
- REST 无 backlinks 是当前插件能力差，未来版本若新增须更新本卡；反链多跳目前仍依赖 CLI/离线。
- 本卡经验来自本机 loopback 明文 HTTP（用户知情选择，仅 127.0.0.1）；TLS 27124 路线因自签证书信任问题未走（见 log-2026-09-29-04）。

## 应用

子 Agent 冷启动读库：先 `rest_read.py probe` 认人 → `search`/`read` 取知识 → 反链退 CLI。与 [[官方obsidianCLI命令速查]]（CLI 兜底）、[[记忆通道使用纪律]]（七律）、[[SPEC-READ]]（环境分流读通道权威）配套。

## 变更记录

- 2026-11-22（独立审计轮二，corr-101）：纪律 3 补 Bearer 头/外来审核者取只读 key 合规路径；纪律 4 补 D33-b Reader 权限例外。
- 2026-11-22（审计处置，corr-102）：纪律 6 副本分析口径（Owner 裁：合规通道取回的副本用本地工具分析属工具面，不算通道绕过）。
- 2026-11-20（处置批一，corr-83）：端点表补「按节读取」行（活体 200 实证）。
- 2026-11-25（外部反馈修复，corr-114）：助手行同步 rest_read v0.2.1——身份探针磁盘侧 `newline=""` 保原始 CRLF 字节，修 Windows autocrlf 检出下探针假 MISMATCH（ansys 副库重测回报，sgc#1）；自测 14→15 规则（IDENTITY_CRLF 回归）。
- 2026-11-20（工具刀，corr-85）：助手行同步 rest_read v0.2.0（`--heading` 按节读取封装、自测 8→14 规则、404 喂标题树），用法要点入核心节。