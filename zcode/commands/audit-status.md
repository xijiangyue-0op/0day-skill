---
description: 查看审计进度与已发现漏洞（只读，不启动新探索）
---

读取工作区根 `audit-state/`，输出（不派发任何子智能体、不做新探索）：

1. 覆盖度矩阵简表：每模块 × 六类的 done/pending/excluded 状态，当前 depth。
2. findings 清单：每条一行——编号、类型、位置(file:line)、触发请求、前端位置、鉴权要求。
3. pending.md 摘要：多少条缺环、各自缺什么。
4. log.md 近 10 条记录。
5. rules_learned：已确认的可达性规则（如"某过滤器排除 /api/*"）。

若 audit-state/ 不存在，提示先用 `/audit <目标路径>` 初始化。
