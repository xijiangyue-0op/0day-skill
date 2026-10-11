# Audit Orchestrator — 多智能体白盒代码审计框架

一套可移植的提示词工程资产：把支持「技能 + 子代理」的 AI 编码客户端（ZCode、Claude Code 等兼容 .agents 目录结构的客户端）变成**多智能体白盒代码审计流水线**——总指挥派发侦察/搜索/污点追踪/终审四级子代理，以磁盘状态实现断点续审与无限深度探索。

> 实战背景：该框架已在 7 个真实目标上完成审计并经双引擎交叉复审（.NET WinForms 胖客户端、Java Struts/Spring、PHP 设备固件、Perl 邮件网关、Java 安平平台），累计产出 80+ 独立通杀漏洞。所有规则均来自实战复盘迭代，非理论设计。

## 核心理念

1. **状态在磁盘，会话是消耗品** — 审计进度全量落盘 `audit-state/`，会话崩溃/超时/压缩零损失，任何新会话一句话续审。
2. **有限空间，无限探索** — 单个智能体上下文有限，但覆盖度矩阵（模块 × 漏洞类）的每个格子永远由全新子代理接手；深度轮次（depth-1/2/3…N）无上限，推进与停止由用户发起。
3. **总指挥不干粗活** — 派发、验收、语义裁定、轮末维护归主控；一切代码阅读归子代理。主控上下文只进摘要，不进代码转储。
4. **双引擎互补** — 第一引擎全程审计（广度+穷举），第二引擎复审（判卷：共同根基复核、最新判定验证、近失链重判、覆盖面抽验）。

## 支持的漏洞类别（六类白名单）

| 编号 | 类别 | 编号 | 类别 |
|---|---|---|---|
| sqli | 未授权 SQL 注入 | deser | 各类反序列化 |
| upload | 可 RCE 的文件上传 | rce | 命令/代码执行/表达式注入/模板 SSTI |
| fileread | 任意文件下载/读取/路径穿越 | ssrf | 服务端请求伪造 |

黑名单（不立 finding）：XSS、CSRF、CRLF、点击劫持、纯信息泄露、弱口令、一切低危或强交互问题。

## 通杀六问（finding 立档硬门槛）

前置三问（漏洞成立性）：路由注册 / 鉴权覆盖 / 可利用拼装。

通杀三问（决定立档还是链路资产）：
4. **未授权** — 触发全程不需要任何员工/用户凭据；
5. **默认部署成立** — 厂商默认配置下走通，无需管理员先犯低级错误；
6. **不依赖可轮换秘密** — 管理员改口令即掐断的入口不算根，但以代码缺陷（如未授权 SQLi）为 DB 写入口的链不在此列。

判定：4~6 全过 → `findings/F-XXX.md`（标注通杀）；缺任一问 → `chains.md` 链路资产（注明缺哪问 + 可组合的通杀根）。**组合拳不计数**：与通杀根组合后整体未授权的链只作根的危害放大证据。

## 快速开始

```bash
# 1. 安装（复制到用户级目录）
cp -r zcode/skills/* ~/.agents/skills/
cp -r zcode/agents/*   ~/.agents/audit/agents/
cp -r zcode/commands/*  ~/.agents/commands/

# 2. 打开目标源码目录为工作区，输入
/audit
```

### 通用（其他兼容客户端）

任何支持「markdown 技能 + 子代理工具」的客户端都能跑：把 `zcode/skills/audit-orchestrator/SKILL.md` 作为主控系统提示词，`zcode/agents/*.md` 作为子代理任务书模板按需注入即可。

## 每轮流程

```
/audit
 └─ 恢复状态：读 audit-state/state.json（存在→跳过 done 格子续审；不存在→recon 侦察建矩阵）
 └─ 执行格子：sink-hunter（扇出搜索，JSONL 候选）→ taint-tracer（三态判定 CONFIRMED/BLOCKED/GAP）
 │            → 通杀预筛（六问 4~6）→ verifier（六问+前端触发点，立档 findings/F-XXX.md）
 └─ 轮末自动维护：pending 瘦身归档（>50KB 触发）/ coverage 规范化 / 队列固化 next_round
 └─ 汇报并停等：覆盖度 / 新通杀表 / 链路资产 / 待验证 / 饱和度 X/4 —— 等用户说"继续"才进下一深度
```

**饱和度四信号**（报满即建议停止静态深挖）：矩阵清零 / 候选枯竭 / pending 不可补（需运行时或缺失件）/ 模式收敛（连续一轮零新通杀）。

## 产物目录（审计后生成）

```
audit-state/
├── state.json        # 覆盖度矩阵 + 索引（断点续审的单一事实源）
├── recon.md          # 技术栈 / 入口地图 / 鉴权覆盖 / 匿名可达面 / 框架睡眠能力
├── findings/F-XXX.md # 通杀档案（污点链 + 前端触发点 + 六问表）
├── chains.md         # 链路资产（条件链/组合链，不计数）
├── pending.md        # 活待办：GAP / 二阶线索 / 下一轮队列（>50KB 自动归档）
├── pending-archived/ # 历史批次归档
└── log.md            # 全时间线（格子完成 / 纠偏 / 黑名单丢弃 / 轮次记录）

audit-reports/审计报告-<系统>-<日期>.md   # 仅在用户显式授权后生成
```

## 双引擎复审（强烈建议）

第一引擎审计完成后，第二引擎（建议换更强模型）只读结论层判卷：

1. **共同根基复核** — 多条 finding 共享的技术前提（鉴权链定论、会话机制），前提翻则成片塌方；
2. **最新判定验证** — 刚立档/刚降级的 finding 最热乎也最脆弱；
3. **近失链重判** — chains.md 里"缺第 N 问"的条件链是历史翻案率最高处；
4. **覆盖面抽验** — excluded 理由 / 饱和度是否报早 / 计数口径。

模板见 [docs/review-prompt-template.md](docs/review-prompt-template.md)。

## 目录结构

```
audit-orchestrator/
├── zcode/                          # ZCode 安装包（编排技能 + 5 角色卡 + 3 命令）
│   ├── skills/audit-orchestrator/SKILL.md
│   ├── agents/{recon,sink-hunter,taint-tracer,verifier,report-writer}-agent.md
│   └── commands/{audit,audit-status,audit-report}.md
├── docs/
│   └── review-prompt-template.md   # 复审提示词模板（判卷）
└── examples/
    └── finding-example.md          # findings/F-XXX.md 档案示例
```

## 实战规则（已内置到提示词）

- 每条 finding 必附**前端触发点**（触发请求 + 前端位置；纯接口注明可直接构造）；
- 容量截断强制闭环：sink-hunter 截断必须当场开衍生格子，只记 note = 丢线索；
- 归并族 ≥10% 抽验 + 代表链行号逐条亲验；
- 间接链纪律：反射/配置驱动调用不是死胡同，"对象池无出站 ≠ deser 格完成"；
- 单轮规模上限（step ≥ 150 / 时钟 > 90 分钟必收官）、>30KB 全文读取必须裁剪；
- 主控永不代做子代理的格子（失败 → 重派 1 次 → 标回 pending）；
- 报告门禁：审计阶段绝不生成报告，仅响应用户显式指令。

## 许可

[MIT](LICENSE) — 仅限授权测试用途（渗透测试授权、CTF、自有代码审计、防御研究）。
