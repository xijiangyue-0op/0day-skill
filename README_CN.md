<div align="center">

# 0day-skill

**多智能体白盒代码审计 · AI 子代理集群驱动**

[English](README.md) | 简体中文

[快速开始](#快速开始) · [工作原理](#工作原理) · [通杀六问](#通杀六问) · [档案示例](examples/finding-example.md)

</div>

> [!IMPORTANT]
> 仅限**授权安全测试**——渗透测试授权、CTF 竞赛、自有代码审计或防御研究。使用者须自行确保测试行为已获授权。

一套可移植的**提示词工程资产**（纯 Markdown、无可执行代码）：把任何支持「技能 + 子代理」的 AI 编码客户端——[ZCode](https://z.ai)、Claude Code 或任何兼容 `.agents` 目录结构的客户端——变成**多智能体白盒代码审计流水线**。

**它找什么**：未授权、可武器化的漏洞——SQL 注入、可 RCE 的文件上传、任意文件读取/下载、反序列化、命令/表达式/模板注入、SSRF——适用于任何源码或反编译产物（Java / PHP / Python / Go / Node / C#，含 .NET DLL、JAR、固件镜像）。

---

## 亮点

**🕷️ 杀伤链狩猎，而非模式匹配**
- 🤖 **总指挥 + 4 类子代理**——侦察 · sink 猎手 · 污点追踪 · 终审，每个都是全新隔离上下文
- 🔁 **反向污点追踪**——从每个危险汇聚点回溯到源，三态判定（CONFIRMED / BLOCKED / GAP）
- 🎯 **通杀六问**——未授权、默认部署成立、不依赖可轮换秘密，六问全过才立档
- 🧾 **前端触发点必附**——每条 finding 带确切 HTTP 请求与触发它的 UI 元素

**💾 状态在磁盘，会话是消耗品**
- 🗂️ **覆盖度矩阵**（模块 × 6 漏洞类）驱动可续审、无遗漏的探索
- ♻️ **每对话一次 /audit**——每轮换新对话；进度全在 `audit-state/`，从不在聊天历史里
- 📉 **实测防降智**——轮换会话把上下文压在 60-120K，长对话 190K+ 会衰减

**⚙️ 纪律编码进提示词**
- 🛡️ **对抗式复核**——verifier 子代理独立重判 tracer 结论 + 行号抽验
- 🚦 **饱和度信号**——流水线会告诉你静态分析何时穷尽，而不是无限空转
- 🔒 **报告门禁**——只有用户显式指令才生成报告，审计中绝不自动产出
- 🧩 **组合拳不计数**——只有经已计数根才可达的利用链记为危害放大证据，不虚报数字

## 工作原理

```
/audit  (一条命令——之后全自动)
 │
 ├─ 侦察子代理 ──────── 技术栈指纹 · 入口地图 · 鉴权覆盖 · 匿名可达面
 ├─ sink 猎手 ×N ────── 按模块扫危险 API → JSONL 候选
 ├─ 污点追踪 ────────── 反向 source→sink 回溯 → CONFIRMED / BLOCKED / GAP
 ├─ 终审 ────────────── 六问门槛 + 前端触发点 → findings/F-XXX.md
 │
 └─ 轮末：磁盘维护 → 饱和度报告 → 停等（等用户说"继续"）
```

**核心机制——状态在磁盘，会话是消耗品：**

```
会话 1: /audit → 跑完整一轮 → 写入 audit-state/ → 停等
会话 2: /audit → 读 state.json → 跳过已完成格子 → 续审
会话 3: /audit → 进入下一深度（depth-1 → 2 → 3 → …N）
```

| 做法 | 上下文峰值 | 判断力 | 推荐度 |
|---|---|---|---|
| 每轮换新对话 | ~60-120K | 全程稳定 | ✅ 推荐 |
| 同对话连跑多轮 | 190K+ | 后段衰减、幻觉风险上升 | ❌ 避免 |

<details>
<summary><b>为什么换对话能保住质量</b></summary>

单轮审计会累积技能加载文本 + 多次子代理派发任务书 + 结算摘要 + 状态文件读取——上下文自然涨到 190K+，此时指令遵循度与细节保持明显衰减（实测：重复引用早期结论、把已 BLOCKED 的链捞回来、数字对不上盘）。换新会话后主控只读裁剪过的 state.json 恢复——**进度零损失（全在磁盘），判断力完全重置**。

</details>

## 通杀六问

每条候选链必须过六问才能成为 finding：

| # | 问题 | 过滤什么 |
|---|---|---|
| 1 | 可达吗？ | 路由真实注册（注解 / 配置 / 约定式） |
| 2 | 鉴权吗？ | 对照侦察鉴权图核实过滤器覆盖，附 `file:line` 证据 |
| 3 | 可利用吗？ | payload 拼装、前置状态、利用前提 |
| 4 | **未授权** | 全链不需要任何用户/员工凭据 |
| 5 | **默认部署成立** | 厂商默认配置走通，无需管理员先犯低级错误 |
| 6 | **不依赖可轮换秘密** | 改硬编码口令杀不死的洞才算根（代码缺陷入口豁免） |

4~6 任一不过 → 记入 `chains.md` 作为**链路资产**（组合素材），绝不计数。**组合拳不计数**——只有经已计数根才可达的链是根的危害放大证据，不是新漏洞。

## 快速开始

```bash
# 1. 克隆
git clone https://github.com/xijiangyue-0op/0day-skill.git
cd 0day-skill

# 2. 安装（复制到用户级目录）
cp -r zcode/skills/*  ~/.agents/skills/
cp -r zcode/agents/*  ~/.agents/audit/agents/
cp -r zcode/commands/ ~/.agents/commands/

# 3. 打开目标源码目录为工作区，输入：
/audit
```

<details>
<summary><b>其他兼容客户端</b></summary>

任何支持「markdown 技能 + 子代理工具」的客户端都能跑：把 `zcode/skills/audit-orchestrator/SKILL.md` 作为主控系统提示词，`zcode/agents/*.md` 作为自包含任务书注入你的子代理派发器。

</details>

### 推荐环境

| 组合 | 客户端 | 模型 |
|---|---|---|
| **首选** | [ZCode](https://z.ai) | GLM-5.3 |
| 备选 | [DeepSeek Harness](https://www.deepseek.com) | deepseek-v4.1-flash |

两者技能体系同源（角色卡单源共享）；方法论与判定口径完全一致，仅派发层适配限流差异。

## 漏洞类别

<details>
<summary><b>六类白名单（点击展开）</b></summary>

| 编号 | 类别 | 编号 | 类别 |
|---|---|---|---|
| sqli | 未授权 SQL 注入 | deser | 各类反序列化 |
| upload | 可 RCE 的文件上传 | rce | 命令 / 代码 / 表达式 / 模板（SSTI）注入 |
| fileread | 任意文件下载 / 读取 / 穿越 | ssrf | 服务端请求伪造 |

黑名单（不立 finding）：XSS、CSRF、CRLF、点击劫持、纯信息泄露、弱口令、一切低危或强交互问题。

</details>

## 产物目录

```
audit-state/
├── state.json        # 覆盖度矩阵 + 索引——断点续审的单一事实源
├── recon.md          # 技术栈指纹 · 入口地图 · 鉴权覆盖 · 匿名可达面
├── findings/F-XXX.md # 通杀档案（污点链 + 触发点 + 六问表）
├── chains.md         # 链路资产（条件链 / 组合链，不计数）
├── pending.md        # 活待办：GAP · 二阶线索 · 下一轮队列
└── log.md            # 全时间线：格子完成 · 纠偏 · 轮次记录

audit-reports/审计报告-<系统>-<日期>.md   # 仅在用户显式授权后生成
```

真实档案示例见 [examples/finding-example.md](examples/finding-example.md)——污点链、触发请求、六问表、前端入口一应俱全。

<details>
<summary><b>实战纪律（已内置到提示词）</b></summary>

- 每条 finding 必附前端触发点；
- 容量截断强制闭环——只记 note 不开格 = 丢线索；
- 归并族 ≥10% 抽验 + 行号逐条亲验；
- 间接链（反射 / 配置驱动）只有被证明才是死胡同；
- 单轮上限（step ≥150 或 90 分钟必收官）；>30KB 读取必须裁剪；
- 主控永不代做子代理的格子（失败 → 重派 1 次 → 标回 pending）；
- 报告门禁：审计中绝不生成报告——仅响应用户显式指令。

</details>

## 目录结构

```
0day-skill/
├── zcode/                     # ZCode 安装包（编排技能 + 5 角色卡 + 3 命令）
│   ├── skills/audit-orchestrator/SKILL.md
│   ├── agents/{recon,sink-hunter,taint-tracer,verifier,report-writer}-agent.md
│   └── commands/{audit,audit-status,audit-report}.md
└── examples/
    └── finding-example.md     # findings/F-XXX.md 档案格式
```

## 许可

[MIT](LICENSE) — 仅限授权安全测试。

## ⚠️ 免责声明

本项目是**提示词工程资产**（纯 Markdown 文本），不包含任何可执行代码。使用者须自行确保测试行为已获授权，并对使用后果承担全部责任。作者不对任何滥用行为负责。

---

## 交流群

<div align="center">

**代码审计 skill 交流群 · Code Audit Skill Discussion Group**

有问题、建议或实战经验分享？欢迎加入交流群。

<img width="450" height="680" alt="代码审计skill交流群" src="https://github.com/user-attachments/assets/60bea1d5-be2e-4234-857e-9d4396733595" />

</div>
