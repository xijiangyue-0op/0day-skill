# findings/F-XXX.md 档案示例

> 本示例展示 verifier 立档的 finding 格式。真实档案见各项目 audit-state/findings/。

# F-003 [rce] 未授权命令注入：load_without_code 的 readpipe + 匿名 mkdir 满足 -f 前置

- 编号：F-003
- 类别：rce（CWE-78 命令注入）
- verdict：通杀（六问全过）

## 位置
- source：`adm/libs/Ext/MgrApp/Login.pm:138`（`$without=$q->xsscgi('without')`）
- 前置满足原语：`adm/libs/Ext/CGI.pm:249-261`（`mkdir "/tmp/$sid",0700`，sid 请求全控）
- sink：`adm/libs/Ext/MgrApp/Login.pm:143`（`readpipe("/bin/cat $tmpfile")` 未转义入 shell）

## 通杀六问

| 问 | 判定 | 依据 |
|---|---|---|
| ① 未授权 | ✅ | `Login.pm:1` 包名匹配 `MgrApp.pm:1603-1606` 的 `permit` 免鉴权白名单；不带 `sid` 参数绕过 `:304` 会话分支 |
| ② 默认部署成立 | ✅ | 无需任何配置开关，随包出厂 |
| ③ 不依赖可轮换秘密 | ✅ | 无口令/密钥/token |
| ④ 无需员工账号 | ✅ | 匿名直达 |
| ⑤ 路由注册 | ✅ | `adm/cgi/index.cgi:171-172` 入口分发 |
| ⑥ 可利用 | ✅ | `-f` 前置由同请求匿名 `mkdir` 满足；`DESTROY` 只清最终 `$CGIDATA{sid}` ⇒ 补诱饵 sid 即持久化 |

## 前端触发点

- 触发请求：`POST /api/adm/cgi/index.cgi`，`without=rsid=x;id #/../../../etc/passwd`（不带 sid）
- 前端位置：SPA 路由 `/#/home/verification?rsid=<载荷>`（`adm/dist/js/chunk-6a93aad4.6af86d9a.js:1@25113`）——验证页把 `location.href` 问号后整串作为 without 发送

## 污点链

```
请求参数 without（Login.pm:138）
  → :139 $rsid=(.+) 捕获
  → :141 $tmpfile="/tmp/wmailman/".$rsid  拼接
  → :142 -f 前置（由 CGI.pm:249-261 匿名 mkdir /tmp/<含元字符目录> 满足）
  → :143 readpipe("/bin/cat $tmpfile")  ← sink（shell 断句 ; / 注释 #）
  → :147-155 stdout 经 JSON 回显
```

净化缺口：`readpipe` 入 shell 无转义；`xsscgi` 过滤在该 multipart 分支（`CGI.pm:448-455`）零覆盖。

## 利用条件与危害

- 利用条件：目标 HTTP 可达；同请求先发一个 `filename=` 部件触发 `mkdir`（目录名含 `;`/空格/`#`），再发诱饵部件持久化目录。
- 危害：**未授权命令执行**（Web 进程身份），回显通道可探测。
- 组合链引用：F-006（认证绕过）与本档不同入口不同判据，互不覆盖；C-129（Ext::Lang require）以 F-008 为根，与本档独立。

## 涉及文件（供报告引用）

- `adm/libs/Ext/MgrApp/Login.pm`
- `adm/libs/Ext/CGI.pm`
- `adm/cgi/index.cgi`
