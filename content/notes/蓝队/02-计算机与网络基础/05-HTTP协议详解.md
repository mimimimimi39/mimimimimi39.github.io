---
title: "HTTP 协议详解"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 5
showTableOfContents: true
---

> 难度：★★★☆☆ ｜ 重要度：必学 ｜ 建议学时：10 小时 ｜ 前置：[计算机网络基础](04-计算机网络基础.md)

## 1. 为什么蓝队要学 HTTP

- 绝大多数 Web 攻击、Webshell、C2、数据外传都发生在 HTTP/HTTPS 上。
- Web 日志、WAF 日志、代理日志、流量分析都围绕 HTTP 字段展开。
- 理解正常 HTTP 行为，才能识别异常请求。

---

## 2. URL 结构

```text
https://user:pass@www.example.com:443/path/to/file?id=1&name=test#section
└─┬─┘   └───┬───┘ └──────┬──────┘└┬┘└──────┬──────┘└────┬─────┘└──┬──┘
 协议      用户信息       主机       端口      路径          查询     片段
```

| 部分 | 说明 | 安全关注 |
|---|---|---|
| 协议 | http/https/ftp 等 | 明文 vs 加密 |
| 用户信息 | 用户名密码 | 可能泄露凭据 |
| 主机 | 域名或 IP | 域名信誉、DGA |
| 端口 | 默认 80/443 | 非标准端口 |
| 路径 | 资源路径 | 目录穿越、上传、Webshell |
| 查询 | `key=value` | 注入、XSS、SSRF |
| 片段 | `#` 后内容 | 不发送给服务器（DOM XSS） |

---

## 3. HTTP 报文

### 3.1 请求报文

```http
GET /index.php?id=1 HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: text/html,application/xhtml+xml
Accept-Language: zh-CN,zh;q=0.9
Cookie: PHPSESSID=abc123; theme=dark
Referer: https://www.example.com/list
X-Forwarded-For: 203.0.113.10
Connection: keep-alive

```

请求行结构：`方法 路径 协议版本`

### 3.2 响应报文

```http
HTTP/1.1 200 OK
Date: Mon, 09 Nov 2026 10:00:00 GMT
Server: nginx/1.24.0
Content-Type: text/html; charset=UTF-8
Content-Length: 1234
Set-Cookie: PHPSESSID=abc123; HttpOnly; Secure; SameSite=Lax
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000; includeSubDomains

<html>...</html>
```

---

## 4. 请求方法

| 方法 | 作用 | 安全关注 |
|---|---|---|
| GET | 获取资源 | 参数在 URL，易留日志；可能被用于注入/XSS |
| POST | 提交数据 | 登录、上传、命令执行；Webshell 常用 |
| PUT | 上传/替换 | 配置错误可上传文件 |
| DELETE | 删除资源 | 未授权删除 |
| HEAD | 只取响应头 | 探测 |
| OPTIONS | 查询支持方法 | 信息泄露、CORS 预检 |
| TRACE | 回显请求 | XST 攻击（通常应禁用） |
| CONNECT | 建立隧道 | 代理滥用、隧道 |

> 蓝队注意：只允许必要的方法；`PUT`、`DELETE`、`TRACE` 通常应禁用。

---

## 5. 状态码

| 范围 | 含义 | 常见 |
|---|---|---|
| 1xx | 信息 | 100 Continue、101 Switching Protocols |
| 2xx | 成功 | 200 OK、201 Created、204 No Content |
| 3xx | 重定向 | 301 永久、302 临时、304 未修改、307/308 |
| 4xx | 客户端错误 | 400、401、403、404、405、408、429 |
| 5xx | 服务端错误 | 500、502、503、504 |

蓝队关注：

- 大量 401/403 → 目录爆破、权限探测
- 大量 404 → 扫描、路径猜测
- 200 但响应异常 → Webshell、上传成功
- 500 → 漏洞利用、注入错误
- 502/504 → 后端异常、代理问题

---

## 6. 常见请求头

| 头 | 作用 | 安全关注 |
|---|---|---|
| `Host` | 目标主机 | 虚拟主机、Host 头注入 |
| `User-Agent` | 客户端标识 | 可伪造；异常 UA 是检测点 |
| `Accept` | 可接受类型 | 内容协商 |
| `Accept-Language` | 语言 | 可伪造 |
| `Referer` | 来源页面 | CSRF 检测、隐私泄露 |
| `Cookie` | 会话凭据 | 窃取、固定、注入 |
| `Authorization` | 认证信息 | Basic/JWT/Bearer |
| `Content-Type` | 请求体类型 | 上传绕过、JSON/XML |
| `Content-Length` | 请求体长度 | 请求走私 |
| `Transfer-Encoding` | 分块传输 | 请求走私 |
| `X-Forwarded-For` | 真实客户端 IP | 可伪造、日志污染 |
| `X-Real-IP` | 真实 IP | 代理场景 |
| `Origin` | 请求来源 | CORS、CSRF |
| `Connection` | 连接管理 | keep-alive |

---

## 7. 常见响应头与安全头

| 头 | 作用 | 安全意义 |
|---|---|---|
| `Server` | 服务端软件 | 信息泄露，建议隐藏 |
| `X-Powered-By` | 技术栈 | 信息泄露，建议关闭 |
| `Set-Cookie` | 设置 Cookie | HttpOnly/Secure/SameSite |
| `Content-Type` | 响应类型 | 防 MIME 嗅探 |
| `Content-Disposition` | 下载文件名 | 防文件名注入 |
| `Strict-Transport-Security` | HSTS | 强制 HTTPS |
| `Content-Security-Policy` | CSP | 防 XSS |
| `X-Frame-Options` | 防点击劫持 | `DENY`/`SAMEORIGIN` |
| `X-Content-Type-Options` | 防 MIME 嗅探 | `nosniff` |
| `Referrer-Policy` | 控制 Referer | 隐私 |
| `Permissions-Policy` | 浏览器功能控制 | 降低攻击面 |
| `Access-Control-Allow-Origin` | CORS | 配置不当可被跨域读取 |

---

## 8. Cookie、Session 与 Token

### 8.1 Cookie 属性

| 属性 | 作用 |
|---|---|
| `Domain` | 生效域名 |
| `Path` | 生效路径 |
| `Expires/Max-Age` | 过期时间 |
| `HttpOnly` | JS 不可读，降低 XSS 窃取风险 |
| `Secure` | 仅 HTTPS 发送 |
| `SameSite` | 跨站发送策略（Strict/Lax/None） |

### 8.2 Session 与 Token

- Session：服务端存储会话状态，客户端持有 Session ID。
- JWT：客户端持有签名令牌，服务端验证签名。
- OAuth2/OIDC：授权框架。

**安全关注**：

- Session ID 可预测/固定
- JWT 算法混淆（none、HS/RS 混淆）
- Token 泄露、过期时间过长
- Cookie 未设置 HttpOnly/Secure/SameSite

---

## 9. HTTP 版本与连接

| 版本 | 特点 |
|---|---|
| HTTP/1.0 | 短连接，每次请求新建 TCP |
| HTTP/1.1 | 持久连接、管线化、Host 头 |
| HTTP/2 | 多路复用、二进制分帧、头部压缩 |
| HTTP/3 | 基于 QUIC/UDP，减少握手延迟 |

蓝队注意：

- HTTP/2 的伪头部 `:method`、`:path`、`:authority`
- HTTP/3 走 UDP 443，传统设备可能看不到
- 加密流量需要结合 SNI、证书、JA3/JA3S、端点日志

---

## 10. 代理与真实 IP

```http
X-Forwarded-For: client, proxy1, proxy2
X-Real-IP: client
Forwarded: for=192.0.2.60;proto=http;by=203.0.113.43
```

**风险**：

- 客户端可伪造 `X-Forwarded-For`，导致日志污染、IP 封禁绕过、越权。
- 应用应只信任可信代理设置的头部。

**蓝队**：

- 明确真实 IP 取值链。
- WAF/负载均衡日志保留原始 IP 与代理 IP。
- 不要仅凭 XFF 做封禁或溯源结论。

---

## 11. 访问日志格式

### 11.1 Common Log Format（CLF）

```text
127.0.0.1 - frank [10/Oct/2000:13:55:36 -0700] "GET /apache_pb.gif HTTP/1.0" 200 2326
```

字段：客户端 IP、用户、时间、请求行、状态码、响应大小。

### 11.2 Combined Log Format

```text
127.0.0.1 - frank [10/Oct/2000:13:55:36 -0700] "GET /apache_pb.gif HTTP/1.0" 200 2326 "http://www.example.com/start.html" "Mozilla/4.08 [en] (Win98; I ;Nav)"
```

增加 Referer 与 User-Agent。

### 11.3 JSON 日志

```json
{"time":"2026-11-09T10:00:00Z","src_ip":"203.0.113.10","method":"POST","uri":"/login","status":200,"ua":"curl/8.0","bytes":1234}
```

> 蓝队要能快速从日志中提取：时间、源 IP、方法、URI、状态码、UA、响应大小、Referer。

---

## 12. 常见 Web 攻击的 HTTP 特征

| 攻击 | 请求特征 | 响应特征 | 日志/流量线索 |
|---|---|---|---|
| SQL 注入 | 参数含 `'`、`UNION`、`SLEEP`、`OR 1=1` | 500、报错、响应时间变长 | URI 编码、WAF 告警、数据库错误日志 |
| XSS | 参数含 `<script>`、`onerror`、`javascript:` | 内容回显 | 反射参数、CSP 报告 |
| 命令注入 | 参数含 `;`、竖线、`&&`、`$(...)` | 命令输出 | 异常子进程、Web 日志 |
| 文件上传 | `multipart/form-data`、文件名 `.php/.jsp/.aspx` | 200、路径返回 | 上传目录新文件、Webshell 检测 |
| 目录穿越 | `../`、`%2e%2e%2f` | 文件内容 | 404/403、文件读取 |
| SSRF | 参数为 URL/IP、`169.254.169.254` | 内网响应 | 内网连接、云元数据访问 |
| Webshell | POST 到 `.php/.jsp`、参数乱码/Base64 | 200、短响应 | 访问日志、流量、文件监控 |
| 扫描 | 大量 404、多路径、多 UA | 404/403 | WAF、Nginx 日志、Suricata |
| 爆破 | 高频 POST `/login`、401/200 | 失败/成功 | 认证日志、WAF |
| C2 | 固定 URI、固定间隔、加密 | 小包 | 代理日志、JA3/JA3S、DNS |

---

## 13. 实操

```bash
# 查看请求与响应头
curl -I https://example.com

# 详细请求过程
curl -v https://example.com

# 发送 POST JSON
curl -X POST https://example.com/api/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test"}'

# 指定 UA 与 Referer
curl -A "Mozilla/5.0" -e "https://example.com" https://example.com

# 跟踪重定向
curl -L https://example.com

# 保存响应
curl -o page.html https://example.com
```

用浏览器 F12：

1. Network 面板查看请求头、响应头、Cookie、Timing。
2. 禁用缓存，观察 304。
3. 查看 Set-Cookie 属性。
4. 查看 CSP、HSTS 等安全头。

---

## 14. 蓝队检测思路

1. **基线**：正常业务的方法、URI、UA、状态码分布。
2. **异常方法**：PUT/DELETE/TRACE、异常 OPTIONS。
3. **异常 URI**：长路径、编码、特殊字符、敏感文件。
4. **异常 UA**：扫描器、curl、python-requests、空 UA。
5. **异常状态码**：大量 404/403/500。
6. **异常频率**：单 IP 高频、单 URI 高频。
7. **异常响应大小**：Webshell 短响应、数据外传大响应。
8. **异常时间**：非工作时段。
9. **关联**：Web 日志 + 主机日志 + 流量 + 情报。

---

## 15. 自测

1. HTTP 请求行由哪三部分组成？
2. GET 与 POST 的区别？安全上有什么不同？
3. 301、302、304、401、403、404、500 分别代表什么？
4. `HttpOnly`、`Secure`、`SameSite` 的作用？
5. `X-Forwarded-For` 为什么不能直接信任？
6. 如何从 Nginx 日志中找出大量 404 的源 IP？
7. Webshell 流量通常有什么特征？
8. HSTS 与 CSP 分别解决什么问题？

---

## 16. 延伸阅读

- [MDN HTTP 文档](https://developer.mozilla.org/zh-CN/docs/Web/HTTP)
- [RFC 9110 HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [Cloudflare：What is HTTP?](https://www.cloudflare.com/learning/ddos/glossary/hypertext-transfer-protocol-http/)
- [Nginx 日志文档](https://nginx.org/en/docs/http/ngx_http_log_module.html)

---

## 17. 下一步

- [HTTPS 与 TLS](06-HTTPS与TLS.md)
- [Web 安全与 OWASP Top 10](../04-安全基础知识/05-Web安全与OWASP-Top10.md)
- [Wireshark 与抓包分析](../07-网络流量分析/01-Wireshark与抓包分析.md)
