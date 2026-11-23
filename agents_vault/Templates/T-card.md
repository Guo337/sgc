---
id: "c-{yyyy}-{seq}"
layer: 2
type: concept          # concept|claim|procedure|observation|entity|feedback
project: 
domain: []
maturity: seed
provenance: manual     # manual|reflect|feedback
created: {{date}}
last-touched: {{date}}
summary: ""            # ≤50 token 自包含定义，必填
k: []                  # 父（上界）
v: []                  # 子（下界）
e: []                  # 同级边: {target, rel, weight}，rel 枚举见 schema.yaml
aliases: []            # 别名，供 MAP aliases 归一
---

# {{title}}

## 定义
<!-- 一句话自包含定义，不依赖上下文 -->

## 核心
<!-- 主体内容 -->

## 存疑
<!-- 不确定的部分如实保留，不得洗白为断言（结晶规则第4条） -->

## 应用
<!-- 在本库/项目中的具体用途 -->
