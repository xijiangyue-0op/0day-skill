---
description: 启动或继续对目标的白盒代码审计（多智能体流水线，断点续审）
argument-hint: "[目标路径] [可选: 聚焦某模块/漏洞类]"
---

对 $ARGUMENTS 执行白盒代码审计，必须加载并遵循 `audit-orchestrator` 技能：

1. 工作区根查 `audit-state/state.json`：存在 → 读状态恢复续审，绝不重扫已完成的格子；不存在 → 初始化状态目录并派 recon-agent 侦察。
2. 建覆盖度矩阵（模块 × 六类），按派发协议跑 sink-hunter（并行扇出）→ taint-tracer（回溯）→ verifier（三问+前端触发点）。
3. 只找六类：未授权SQL注入 / 可RCE文件上传 / 任意文件下载读取 / 反序列化 / RCE / SSRF。XSS/CSRF/CRLF 等低危或强交互问题一律不立 finding。
4. 每条 finding 必须含前端触发点（触发请求 + 前端位置或"纯接口可直接构造请求"）。
5. 轮末收官：先执行轮末自动维护（pending 瘦身归档 / coverage 规范化 / 队列固化进 state.json.next_round），然后按汇报格式输出（覆盖度 / 新 findings / pending / 饱和度信号 / 下一轮提议）并停下——等用户说"继续"或再跑 /audit 才进入下一深度。单次 /audit = 单轮有界工作量，禁止自动无限续跑。

**硬性提醒：不生成任何提交报告，报告阶段只等用户显式指令或 /audit-report 命令。**
