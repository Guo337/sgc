---
id: "pit-2026-00012"
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
summary: Chromium 网络栈只读系统证书库、不认 NODE_EXTRA_CA_CERTS，本地自签 CA 未入库→TLS 握手 SELF_SIGNED_CERT_IN_CHAIN；仅回环服务可降明文 HTTP 绕过
e:
  - { target: "[[PIT-插件密钥入库泄漏]]", rel: co_occurs, weight: 0.6 }
  - { target: "[[PIT-obsidian裸名误调GUI版]]", rel: co_occurs, weight: 0.4 }
---

# ⚠️ 本地自签服务 TLS 信任失败与 loopback 明文降级

## Bug 类型

本地服务集成陷阱——客户端连不上一个用**自签 CA** 提供 HTTPS 的本机服务，报 `SELF_SIGNED_CERT_IN_CHAIN`，看似“服务坏了 / 证书无效”，实为**客户端信任链**问题；且不同客户端网络栈“如何注入信任”的行为不一致，容易把某个 env 变量误当通用解、在错误的层反复试。

## 根因

- **Electron/Chromium 网络栈只读操作系统证书库**，不认 Node/OpenSSL 的 `NODE_EXTRA_CA_CERTS`。基于 Chromium 的客户端（IDE 扩展宿主、Electron App 内的 fetch）做 TLS 校验时走系统信任库；本地自签 CA 若没进系统证书库，握手即被拒（`SELF_SIGNED_CERT_IN_CHAIN`）。
- 所以“设了 `NODE_EXTRA_CA_CERTS` 仍失败”不是变量没生效，而是**这条栈根本不读它**——用独立 Node 进程复现会成功（有该变量→TLS 通过 / 401，无→SELF_SIGNED），恰好把差异定位到“栈”而非“配置”。

## 修复模式

1. **先隔离“走哪条网络栈”**：独立 Node 复现（切换 `NODE_EXTRA_CA_CERTS` 对照）+ 导出 CA 验 SAN 是否含目标 host/IP（如 `127.0.0.1`）+ grep 客户端有无 `rejectUnauthorized` 开关 + 读**逐服务器**连接日志（常不在主日志里）。别一上来就判“证书无效 / 服务坏”。
2. **仅回环服务 → 降明文 HTTP 绕过 TLS 信任**：服务严格只听 `127.0.0.1`（不出网）时，启用其明文端口是可接受权衡——流量不离机、用户知情、bearer token 明文但不入库。这是**本地开发工具专用**降级。
3. **要 TLS 就把 CA 装进对应信任库**：Chromium 栈 → 操作系统证书库（不是 env 变量）；纯 Node → `NODE_EXTRA_CA_CERTS`。按栈选注入点，别跨栈套用。
4. **密钥处置**：token 磁盘到磁盘注入、掩码、绝不打印 / 入库；掩码须**递归嵌套对象**，不能只匹配顶层字段名。见 [[PIT-插件密钥入库泄漏]]。

## 适用边界

- 有效：基于 Chromium/Electron 网络栈的客户端连本机自签 HTTPS 服务、且服务仅回环监听。
- 无效：纯 Node/OpenSSL 客户端（认 `NODE_EXTRA_CA_CERTS`）；服务对公网暴露（此时明文降级**不可接受**，必须修 TLS 信任）。
- 可迁移教训：TLS“证书无效”类失败，先问“客户端走哪条网络栈、它的信任来源是什么”，别默认某个 env 变量通用；**严格回环**的本地服务，明文降级是低风险权衡，但硬边界是“流量绝不离机”。

## 来源

- log-2026-09-29-03 / log-2026-09-29-04（阶段B rB1/rB2）：接通 Obsidian Local REST API 插件 MCP 只读通道时，客户端扩展宿主 fetch 走 Chromium 栈、不认 `NODE_EXTRA_CA_CERTS`、插件自签 CA 不在系统库 → TLS 被拒；改用插件明文端口 `127.0.0.1:27123`（仅 loopback）绕过、token 明文 Bearer 被接受、读取管线跑通。log-04 已将本模式标为“可结晶”，此卡兑现。
- 与 [[PIT-插件密钥入库泄漏]] 同批（本机插件只读接入的信任与密钥处置）；同为“第三方本地工具集成坑”的还有 [[PIT-obsidian裸名误调GUI版]]。通道决策见 [[SPEC-READ]]（读取主走插件 MCP、明文 27123）。
