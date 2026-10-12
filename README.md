# 0day-skill

English | [简体中文](README_CN.md)

AI 多智能体白盒代码审计。一条命令 `/audit`，自动派发子代理侦察、搜漏洞、追污点链、终审立档，产出可直接提交的漏洞档案。

> ⚠️ 仅限授权测试使用。

## 装个就能跑

```bash
git clone https://github.com/xijiangyue-0op/0day-skill.git
cp -r 0day-skill/zcode/skills/*  ~/.agents/skills/
cp -r 0day-skill/zcode/agents/*  ~/.agents/audit/agents/
cp -r 0day-skill/zcode/commands/ ~/.agents/commands/
```

打开目标源码目录，输入 `/audit`。

推荐 ZCode + GLM-5.3（备选 DeepSeek Harness + deepseek-v4.1-flash）。

## 它找什么

SQL 注入 / 文件上传 RCE / 任意文件读取 / 反序列化 / RCE / SSRF——只报**未授权**的，每条附触发请求。

## 怎么用

```
新对话输入 /audit → 自动跑一轮 → 停
关掉对话，再开新的输入 /audit → 接着上次的进度跑
重复直到它说"审完了"
```

**每轮必须换新对话**——进度存在磁盘上不丢，长对话会让模型降智。

跑完说"写报告"，生成 `audit-reports/审计报告.md`。

## 产出

```
audit-state/findings/F-XXX.md   # 漏洞档案（污点链+触发点+六问表）
audit-state/chains.md           # 差一环的链（不计数）
audit-reports/审计报告.md        # 最终报告
```

档案格式示例：[examples/finding-example.md](examples/finding-example.md)

## 许可

MIT — 仅限授权测试。

## Community

<img width="300" alt="Code Audit Skill Discussion Group" src="https://github.com/user-attachments/assets/60bea1d5-be2e-4234-857e-9d4396733595" />
