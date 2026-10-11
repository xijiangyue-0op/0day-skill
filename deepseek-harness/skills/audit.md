---
name: audit
description: 白盒代码审计总控流程（多轮深度版）。当用户要求对某个代码库做整体安全审计、深挖漏洞、不知道从哪开始审计、或对已审过的目标说"继续审计/接着审"时使用。它不执行审计本身，而是编排分类审计会话与深度轮次，通过 audit-state/ 磁盘状态实现断点续审与无限深度探索。
user-invocable: true
---

# 白盒代码审计总控流程（多轮深度版）

核心思想：**状态在磁盘，会话是消耗品**。每轮审计把进度写入 `audit-state/`，上下文膨胀/会话结束都不丢进度；任何会话说"继续审计"即从 state.json 无缝续跑。一个会话只做一个轮次，做完汇报并停下，用户决定是否继续（禁止无限迭代）。

## 状态目录 audit-state/ 与 state.json

```
audit-state/
├── state.json        # 覆盖度矩阵+索引（schema 见下）
├── recon.md          # 侦察情报：技术栈/入口地图/防护覆盖/匿名可达面
├── findings/         # 通杀档案 F-XXX.md（六问全过）
├── chains.md         # 链路资产：条件链（缺哪问+可组合根）与组合链（不计数）
├── pending.md        # 活待办：GAP+二阶线索+下一轮队列（>50KB 触发归档）
│   ├── pending-archived/
│   └── log.md        # 时间线：格子完成/黑名单丢弃/轮次记录
```

state.json 结构：

```json
{
  "target": "目标绝对路径",
  "stack": {"lang": "", "framework": "", "version": ""},
  "depth": 1,
  "updated": "ISO时间",
  "modules": [{"name": "src/.../controller", "entries": ["GET /api/..."], "auth": "匿名|需登录|未知"}],
  "coverage": [{"module": "...", "class": "sqli", "status": "pending|in_progress|traced|done|excluded", "attempts": 0, "derived_from": null, "note": ""}],
  "rules_learned": ["AuthFilter.java:45 排除 /api/* → 该前缀全部匿名可达"],
  "next_round": ["公共污点根全传播"],
  "findings": ["F-001"]
}
```

## 每轮会话的固定流程

1. **恢复状态**：读 `audit-state/state.json`
   - 不存在 → 本会话就是 depth-1：先跑 `/audit-recon`（或 recon 任务书）产出 `audit-state/recon.md` + state.json 初始化
   - 存在 → 读取 coverage / next_round，跳过全部 done 格子，恢复 pending 格子
2. **执行本轮格子**：对未完成格子逐类审计（分类技能全文即任务书），串行优先、至多 2 个子代理并行（API 限流：并发≤5、每分钟≤20 次，并行只是把等待叠起来，收益有限）；中大目标的格子级派发用 `~/.agents/audit/agents/` 角色卡流水线（与 ZCode 共用单源，见 audit-swarm）。派了子代理主控就不读大段业务代码：只进 ≤15 行摘要，抽验不合格退回子代理重做，禁止自己重干
3. **轮末收尾（自动执行，不等指令）**：
   - 新发现与线索写入 state.json 的 next_round
   - 覆盖度矩阵更新（每格：module × class × status，词表 pending/in_progress/traced/done/excluded）
   - 会话 todo 列表全部标 completed（UI 残留 pending 会被误读为"任务没跑完"）
   - 汇报并停下：**必须按「汇报格式」小节逐项输出**（覆盖度/新发现通杀表/链路资产/待验证/饱和度/下一轮提议）——**列"已落盘文件清单"不算汇报，禁止以此代替**；汇报前重读 state.json，覆盖度等数字一律以磁盘为准；等用户说"继续"才进下一轮
4. **报告门禁**：审计阶段绝不生成最终报告，只有用户显式要求时才跑 `/audit-report`

## 六类漏洞白名单（编号固定）

| 编号 | 类别 |
|---|---|
| sqli | 未授权 SQL 注入 |
| upload | 可 RCE 的文件上传 |
| fileread | 任意文件下载/读取/路径穿越 |
| deser | 各类反序列化 |
| rce | 命令执行/代码执行/表达式注入/模板 SSTI |
| ssrf | 服务端请求伪造 |

黑名单（发现只记一行 log，不立 finding）：XSS、CSRF、CRLF、点击劫持、纯信息泄露、弱口令、一切低危或强交互问题。

## 硬性纪律

- 每条发现必附**前端触发点**（触发请求/操作路径 + 前端位置；纯接口注明"可直接构造请求"）
- **通杀六问**决定一条链是"通杀发现"还是"链路资产"（记 chains.md）：①未授权？②默认部署成立？③不依赖可轮换秘密（管理员改口令/密钥即失效的入口不算，但以 SQL 注入等代码缺陷为入口的不在此列）？④不依赖员工账号？——加漏洞成立三问（可达/鉴权/可利用）。**组合拳不计数**：经通杀根组合后未授权的链登记为组合链（危害放大证据），不作为独立通杀计数
- 一切结论基于实读代码，引用 `文件:行号`；污点链未闭环进 pending.md
- sink 搜索保持全量（条件链是组合链素材），通杀筛选只发生在立档判定

## 深度轮次

- **depth-1**：六类标准 sink 定向搜索（任务书：分类技能全文）
- **depth-2**：二阶污点（DB 回读再入 sink）、配置注入面（配置写原语→框架睡眠能力，如 appSettings 写→反序列化框架激活）、净化/绕过、冷门入口
- **depth-3**：依赖反编译（版本可疑时）、反射/动态调用链、上传解析差异
- **depth-N**：自创角度；每条发现的涉及文件/调用者/同模式代码派生新格子
- **depth 递进规则**：轮次由用户说"继续"推进，与对话次数无关。depth-1 矩阵未清零时，"继续"= 挖剩余格子/next_round 队列（depth 不变）；矩阵清零且下一轮提议被用户认可后才 depth+1
- 停止标准（四信号，报满即建议停）：矩阵清零 / 候选枯竭 / pending 不可补（需运行时验证）/ 模式收敛（连续一轮零新发现）

## 汇报格式（轮末必报）

```
[depth N] 覆盖度: done X / total Y
新发现（独立通杀）: F-001 [sqli|通杀] file:line | 触发点 | 匿名
链路资产: M 条（组合链 K 条根=F-XXX 不计数；缺项分布）
待验证: M 条
饱和度: X/4 —— 说"继续"进入下一轮
```
