---
name: audit-status
description: 查看审计进度与已发现漏洞（只读看盘，不启动新探索、不派子代理）。当用户问"审到哪了""有哪些漏洞了""看下审计进度"时使用。
user-invocable: true
---

# 审计进度查看（只读）

读取工作区根 `audit-state/`，只输出不探索（不派发任何子代理、不做新扫描、不修改任何文件）：

1. 覆盖度矩阵简表：每模块 × 六类的 done/pending/excluded 状态，当前 depth
2. findings 清单：每条一行——编号、类型、位置(file:line)、触发请求、前端位置、鉴权要求（独立通杀数单独计；组合链不计数，只列根引用）
3. pending.md 摘要：多少条缺环、各自缺什么
4. log.md 近 10 条记录
5. rules_learned：已确认的可达性规则（如"某过滤器排除 /api/*"）
6. 饱和度四信号当前状态（矩阵清零 / 候选枯竭 / pending 不可补 / 模式收敛）

audit-state/ 不存在时提示先用 `/audit <目标路径>` 初始化。发现维护缺失（pending 超 50KB、coverage 状态漂移）只提醒不代做——维护在审计轮次里执行。
