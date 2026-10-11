---
name: audit-rce
description: 白盒审计分类任务：命令注入与其他 RCE 专项（Runtime/ProcessBuilder、SpEL/OGNL/MVEL、模板注入 SSTI、动态类加载、JSP webshell 链）。当用户要求审计命令注入、表达式注入、模板注入或在新会话中以本模板开始审计时使用。
user-invocable: true
---

# 白盒审计分类任务：命令注入 / 表达式注入 / 其他 RCE

本会话只审命令执行与各类 RCE（不含上传与反序列化，那两类有专属模板），其他类别不展开（发现线索写入 `audit-findings/handoff.md` 一行带过）。

## 起点

优先读取 `audit-map.md` 中的命令/表达式执行 sink 清单；若不存在，先定位：

1. **命令执行**：`Runtime.getRuntime().exec` / `ProcessBuilder` / `new Process...`（重点：系统监控、ping/网络工具、压缩/转换、备份功能把用户参数拼进命令）
2. **表达式引擎**：SpEL `parseExpression`、OGNL（Struts2 场景）、MVEL、EL `evaluate`、JEXL
3. **模板引擎 SSTI**：FreeMarker、Velocity、Thymeleaf、Beetl 中模板内容或模板名用户可控
4. **脚本引擎**：ScriptEngine（Nashorn/Groovy）`eval`
5. **动态类加载**：Class.forName/loadClass 参数可控、URLClassLoader、defineClass、反射调用链
6. **JSP 部署链**：结合 audit-upload 结论的落地点判断可写 → 可解析的 webshell 路径（仅引用，不重复分析）

## 审计要点

- **exec 圆括号变形**：exec(String) 单参数会按空格切分，拼接用户输入即命令注入；注意重定向/管道在 Java exec 里如何被处理，利用方式按真实行为写
- **表达式注入的输入面**：参数名/值进 OGNL、排序字段进 SpEL、消息模板进 FreeMarker
- **可控性回溯**：每个 sink 回溯到 source，只报用户可控的；来自配置文件但管理员可写入的，单独标"需前提"
- **鉴权检查**：逐个入口确认未授权可达性

## 排除项

需要登录且确认无未授权路径的入口、纯本机 CLI 工具参数不构成 Web RCE 的、弱口令类。

## 输出要求

结论写入 `audit-findings/rce.md`，每个发现按此格式：

```
### [已确认|高度可疑|需动态验证] <一句话标题>
- 前端触发点：<完整 URL> + <参数名> + <请求方法>
- 调用链：source 文件:行号 → ... → exec/eval 文件:行号
- 利用方式：具体 payload（命令拼接变形 / 表达式 payload / SpEL 对象构造）
- 危害评级：高危（未授权 RCE）/中危（需前提或链式）/低危
- 证据：关键代码原文
```

- 文件末尾附"已排查面"清单，明确结论："发现 N 个 / 未发现可确认漏洞"
- 上下文接近 80% 时把未完成部分写入"未完成"一节，提示开新会话继续

最后一轮对话给出简短总结：每个发现一句话 + 排查面概述。

## 通杀判定（新：每条发现必答）

对每条「已确认」发现额外回答五问，答案写进发现条目的「通杀」字段：
1. 未授权？（触发全程不需要任何账号口令；内部人滥用不算）
2. 默认部署成立？（无需管理员特殊配置；默认注释/禁用的能力须能被攻击者可写的配置激活才算）
3. 不依赖可轮换秘密？（管理员改一个口令/密钥就失效的入口不算；但以 SQL 注入等代码缺陷为入口的链不在此列）
4. 不依赖网络位置/本地前提？（无需同网段投毒、无需先落地本地文件写；公网明文端点是厂商部署事实，不受此限）
5. 不依赖员工账号？

判定：五问全过 = **通杀**（独立计数）。缺任何一问 = 在文件末尾「链路资产」节登记：链路 + 缺哪问 + 可与哪条通杀根组合（组合链不计数，仅作根的危害放大证据）。

## 状态落盘（新：接 audit-state 协议）

- 本类结论照旧写 audit-findings/<类别>.md；另在 audit-state/log.md 追加一行完成记录（含发现数）
- 若 audit-state/state.json 存在：把本类对应 coverage 格标 done/traced，新线索写入 next_round
- 涉及的发现编号 F-XXX（若主控已分配）保持一致；未分配编号时用位置 hash 临时标识
