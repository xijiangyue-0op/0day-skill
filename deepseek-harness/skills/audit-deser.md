---
name: audit-deser
description: 白盒审计分类任务：反序列化专项（Java 原生/ fastjson / Shiro / XStream / Hessian / JNDI 等）。当用户要求审计反序列化漏洞或在新会话中以本模板开始审计时使用。
user-invocable: true
---

# 白盒审计分类任务：反序列化

本会话只审反序列化，其他类别不展开（发现线索写入 `audit-findings/handoff.md` 一行带过）。

## 起点

优先读取 `audit-map.md` 中的反序列化 sink 清单与依赖库版本；若不存在，先定位：

1. **原生反序列化**：`ObjectInputStream.readObject` / `readUnshared`（含 Redis 缓存值、RMI/JMX、自定义协议中的调用）
2. **JSON 库**：fastjson `JSON.parseObject` / `parseArray`（按版本判断 autoType 与绕过）、Jackson `enableDefaultTyping` / `@JsonTypeInfo`、GSON `@class` 场景
3. **XML 反序列化**：XStream（版本对应 CVE）、XMLDecoder
4. **框架特征**：Shiro rememberMe（检查 AES key：默认 key/硬编码 key → 无需密钥也可构造）， 及 CustomObjectInputStream 是否重写 resolveClass
5. **远程组件**：Hessian / Burlap / Dubbo / RMI / JNDI 调用入口
6. ** gadget 库可用性**：从 pom.xml / WEB-INF/lib 确认 commons-collections、commons-beanutils、spring-core、groovy 等版本，匹配对应 gadget 链（CC 链按 3.x/4.x 判定，注明可用链名）

## 审计要点

- 每个 sink 回溯到 source：cookie（Shiro 场景重点）、请求参数、上传文件、数据库回读、RPC 入口
- 鉴权检查：Shiro rememberMe 天然未授权可达；其余入口确认 Filter 覆盖
- 版本对应：fastjson 1.2.x 各版本 autoType 状态与已知绕过、XStream CVE 版本线，报告中注明版本依据

## 排除项

无法回溯到外部可控 source 的内部反序列化（纯进程内缓存等）、需登录且确认无未授权路径的入口、弱口令类。

## 输出要求

结论写入 `audit-findings/deser.md`，每个发现按此格式：

```
### [已确认|高度可疑|需动态验证] <一句话标题>
- 前端触发点：<完整 URL 或 Cookie 字段> + <可控字段>
- 调用链：source 文件:行号 → ... → readObject/parseObject 文件:行号
- gadget 链：<库-版本> + <具体链名>，或 fastjson autoType 依据
- 利用方式：构造要点（序列化 payload 的生成方式、key 前提、探测回显/延时方案）
- 危害评级：高危（有可用 gadget 且未授权可达）/中危/低危
- 证据：关键代码原文 + 依赖版本清单
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
