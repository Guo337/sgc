---
id: "s-2026-00005"
layer: 2
type: summary
project: memory-lib
domain: [knowledge-engineering]
maturity: seed
provenance: manual
edge-policy: exempt
created: 2026-11-20
last-touched: 2026-11-20
summary: 联邦义务包：建库/复制/上提指针
---

# role-ke-federation（knowledge-engineering 联邦义务包）

> 你是建副库/上提的项目 Agent：不改主库，在自己工作区建库。

<!-- atlas:auto-start -->
- 通道：读 [[SPEC-READ]]「访问通道」节——探活→自启→降级前置（退离线须 Owner 批准 D33）；[[副库建库引导]]「第 0 步」节
- 写权限：主库只读不写，权限真值 [[SPEC-WRITE]]「写权限矩阵」节；副库写按「建库五步」节
- 建库五步：读 [[副库建库引导]]「建库五步」节——复制保 id+provenance:copied，裁剪孵化区引用
- id 命名空间：id-namespace 带本库前缀不续主库号，「建库五步」节第 5 步
- 上提：读 [[上提验收流程]]「六步」节——交件→验收→Owner 终审→换主库号
- 临时件：抽取件用完即删，[[SPEC-GOVERN]]「doctor 闸门」节运行方式段
- 诚实：[[表述精确性纪律]]「核心」节三律；复制卡存疑节原样保留不洗白（[[SPEC-WRITE]]「结晶规则」节第 4 条）；引导不清/卡壳如实报回
- 不越权：[[不越权纪律]]三律；不改主库、验收争议归 Owner，疑越权停呈裁
<!-- atlas:auto-end -->