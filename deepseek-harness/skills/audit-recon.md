---
name: audit-recon
description: 白盒审计第一步（audit-state 协议版）：侦察技术栈/入口点/防护覆盖/框架睡眠能力，初始化 audit-state/（recon.md + state.json 覆盖度矩阵）。当用户要求"先摸清代码库结构""建立审计地图"或开始一个新的整体审计时使用。本会话不做任何漏洞分析。
user-invocable: true
---

# 白盒审计第 1 步：侦察与状态初始化

本会话**只做情报收集与状态初始化，不下任何漏洞结论**。产出 `audit-state/`（recon.md + state.json），供后续分类审计会话与子代理作为起点。

## 侦察清单（按序执行）

1. **技术栈指纹**：读依赖清单确定语言/框架/精确版本（pom.xml / composer.json / requirements.txt / package.json / web.config / go.mod）
2. **已知风险组件标注**：列出带已知利用链的组件（fastjson/Shiro/Log4j/Struts2/ThinkPHP/Spring 等，写明版本与漏洞类型）——只标注，不验证
3. **入口点地图**：全部路由（注解/配置/约定式）、冷门入口（定时任务/WebSocket/WebService/上传接口/调试端点 druid、actuator、swagger）
4. **全局防护盘点（最重要）**：Filter/Interceptor/中间件覆盖哪些路径、排除哪些（excludePathPatterns、静态资源、注解遗漏）——产出**匿名可达面**结论
5. **模块划分**：按目录/功能边界给出模块清单（供矩阵切格子）
6. **配置敏感项**：数据库连接、反序列化白名单、上传目录、模板引擎、硬编码密钥
7. **框架睡眠能力盘点（反漏报）**：目标实际调用的框架能力中"配置/数据驱动即可激活危险行为"的项（.NET Csla DataPortal/WCF 配置节/machineKey、Java JNDI/RMI 配置、反序列化 filter、上传/编译组件配置开关），每项注明配置入口位置与写权限——这是"配置注入→框架激活"组合链的素材清单

## 产出

### 1. `audit-state/recon.md`（固定小节）

技术栈 / 已知风险组件 / 入口点表（路由→处理类:方法→鉴权状态）/ 防护覆盖表 / **匿名可达面** / 模块清单 / 配置敏感项 / 框架睡眠能力。所有结论带 `文件:行号`。

### 2. `audit-state/state.json`（初始化，schema 见 audit.md）

按侦察结果填充：target、stack、modules（每模块含 entries 与 auth）、coverage（**模块 × 六类全排开**，全部 pending）、rules_learned（匿名可达面结论必须最先写入，如"AuthFilter 排除 /api/* → 该前缀全部匿名可达"）、depth=1、findings=[]。另建空文件 pending.md、log.md、chains.md。

## 硬规则

- 不报告漏洞，不评价代码好坏，只给事实情报
- 目标项目只读，禁止修改任何文件（audit-state/ 除外）
- 最后一轮对话输出 ≤15 行摘要：技术栈一行、匿名可达面结论、模块清单（每模块一行：名称+入口数+鉴权状态）、风险组件列表——提醒用户"下一步跑 /audit 或 /audit-swarm 开始分类审计"；输出前把会话 todo 列表全部标 completed（残留 pending 会被误读为"任务没跑完"）
