---
id: "s-2026-00004"
layer: 2
type: summary
project: memory-lib
domain: [knowledge-engineering]
maturity: seed
provenance: manual
edge-policy: exempt
created: 2026-11-20
last-touched: 2026-11-20
summary: 知识消费者义务包：只读白名单/通道分流/探针粒度/GAP 诚实指针
---

# role-ke-consumer（knowledge-engineering 消费者义务包）

> 你是知识消费者：只读用知识，不写库。逐条读指针目标节；真值在宿主。

<!-- atlas:auto-start -->
- 只读白名单：读 [[SPEC-READ]]「红线」节 A.6——只调只读工具，绝不写 /vault/
- 通道分流：读 [[SPEC-READ]]「访问通道」节——探活→路由→降级前置（退离线须 Owner 批准 D33）；[[REST读库速查]]「核心」节
- 读范围：[[SPEC-READ]]「入口与路由规则」节路由优先忌盲扫；「装配预算」节 L3/L2/L1 上限
- 探针粒度：匹配 title/子串/别名非语义，见 [[副库建库引导]]「验证」节；无知识诚实报 GAP
- 不越权：[[不越权纪律]]三律；只读不写、降级须批准（D33），疑越权停呈裁
- 不写库：写入权限见 [[SPEC-WRITE]]「写权限矩阵」节；域清单 → [[SH-knowledge-engineering]]
<!-- atlas:auto-end -->