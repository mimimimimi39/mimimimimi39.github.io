---
title: "XSS 与 CSRF"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 7
showTableOfContents: true
---

> 难度：★★★☆☆ ｜ 重要度：必学 ｜ 建议学时：8 小时 ｜ 前置：[HTTP 协议详解](../02-计算机与网络基础/05-HTTP协议详解.md)、[Web 安全与 OWASP Top 10](05-Web安全与OWASP-Top10.md)

## 1. XSS（跨站脚本）

### 1.1 原理

应用把不可信输入作为 HTML/JS 输出到页面，浏览器将其当作代码执行。

```html
<!-- 正常 -->
<div>欢迎，<?php echo $name; ?></div>

<!-- 输入 <script>alert(1)</script> 后 -->
<div>欢迎，<script>alert(1)</script></div>
```

> 这是经典教学示例，仅用于理解。实际测试请在本地靶场进行。

### 1.2 类型

| 类型 | 说明 | 数据流 |
|---|---|---|
| 反射型（Reflected） | 恶意输入立即回显 | 请求 → 响应 |
| 存储型（Stored） | 恶意内容存入数据库 | 请求 → 存储 → 其他用户访问 |
| DOM 型（DOM-based） | 前端 JS 不安全地操作 DOM | 客户端 |

### 1.3 危害

- 窃取 Cookie/Session（若未 HttpOnly）
- 窃取页面数据、Token
- 钓鱼、篡改页面
- 键盘记录
- 蠕虫传播
- 绕过 CSRF 防护

### 1.4 检测

**Web 日志**：

- 参数含 `<script>`、`onerror`、`onload`、`javascript:`
- 编码：`%3Cscript%3E`、HTML 实体、Unicode
- 短时间大量不同 payload
- 存储型：正常页面请求后出现异常脚本内容
- 响应中回显未编码输入

**WAF**：

- XSS 规则命中
- 编码绕过、大小写混淆

**浏览器/终端**：

- CSP 报告
- 异常弹窗、跳转
- 浏览器扩展告警

### 1.5 防御

| 措施 | 说明 |
|---|---|
| 输出编码 | 按上下文（HTML/属性/JS/URL/CSS）编码 |
| 输入校验 | 类型、长度、白名单，但不能替代输出编码 |
| CSP | 限制脚本来源，禁止内联脚本 |
| HttpOnly | 降低 Cookie 被 JS 读取风险 |
| 框架安全 | 使用自动转义的模板引擎 |
| DOM 安全 | 避免 `innerHTML`、`eval`、`document.write` |
| 富文本过滤 | 使用成熟 sanitizer（如 DOMPurify） |

**上下文示例**：

```text
HTML 文本：<div>用户输入</div>        → HTML 实体编码
HTML 属性：<div title="用户输入">     → 属性编码
JS 字符串：var x = "用户输入";        → JS 编码
URL 参数：?q=用户输入                 → URL 编码
CSS：style="color: 用户输入"          → 严格白名单
```

> 不同上下文需要不同编码，不能只用一种「通用过滤」。

---

## 2. CSRF（跨站请求伪造）

### 2.1 原理

攻击者诱导已登录用户在不知情的情况下向目标站点发送请求，利用浏览器自动携带 Cookie 的特性。

```text
用户登录 bank.com，获得 Cookie
用户访问 evil.com
evil.com 页面自动请求 bank.com/transfer?to=attacker&amount=1000
浏览器自动带上 bank.com 的 Cookie
银行认为是用户本人操作
```

### 2.2 条件

经典三条件：

1. 存在可被利用的相关操作（状态变更）。
2. 站点依赖 Cookie 会话认证。
3. 请求参数可预测/可构造，且没有不可预测的 Token。

补充前提：用户已登录目标站点，浏览器会自动携带 Cookie 发起跨站请求。

### 2.3 检测

- 请求的 `Referer`/`Origin` 异常或为空
- 跨站请求命中敏感接口
- 短时间内敏感操作
- 用户投诉「未操作却发生变更」
- CSRF Token 缺失/重复使用

### 2.4 防御

| 措施 | 说明 |
|---|---|
| CSRF Token | 随机、每会话/每请求、服务端校验 |
| SameSite Cookie | `Lax`/`Strict` 限制跨站发送 |
| Origin/Referer 校验 | 校验请求来源 |
| 二次确认 | 敏感操作要求重新认证 |
| 自定义头 | API 要求 `X-Requested-With` 等（有一定效果） |
| 避免 GET 改数据 | 敏感操作使用 POST/PUT/DELETE |

> XSS 可以绕过 CSRF Token（同源执行），所以 XSS 与 CSRF 要一起防。

---

## 3. XSS 与 CSRF 对比

| 维度 | XSS | CSRF |
|---|---|---|
| 攻击目标 | 用户浏览器/会话 | 用户身份/服务端请求 |
| 是否需要用户交互 | 反射/存储需访问，DOM 需触发 | 需用户访问恶意页面 |
| 核心问题 | 不可信输入当代码执行 | 请求缺少来源校验 |
| 主要防御 | 输出编码、CSP | Token、SameSite、Origin 校验 |
| 日志线索 | 恶意脚本参数、CSP 报告 | 异常 Referer/Origin、敏感操作 |

---

## 4. 蓝队检测规则示例

### 4.1 XSS 参数特征

```spl
index=web
| search uri="*%3Cscript*" OR uri="*<script*" OR uri="*onerror*" OR uri="*javascript:*"
| stats count by src_ip, uri, status
| where count > 3
```

### 4.2 CSRF 异常来源

```spl
index=web action=transfer
| where isnull(referer) OR NOT match(referer, "https://bank\.example\.com/")
| table _time src_ip uri referer user_agent
```

> 实际规则需结合业务接口、Token 校验与白名单，避免误报。

---

## 5. 应急响应

1. 确认受影响页面/接口与影响范围。
2. 检查是否有恶意内容存储（存储型）。
3. 检查会话是否被劫持、是否有异常操作。
4. 清理恶意内容、修复输出编码。
5. 重置受影响用户会话/密码。
6. 加 WAF 临时规则。
7. 复盘并补充 CSP/安全头。

---

## 6. 实操

在 DVWA 或 Juice Shop：

1. 反射型 XSS：在搜索框输入测试脚本，观察回显。
2. 存储型 XSS：在留言板输入测试脚本，刷新观察。
3. DOM 型 XSS：观察前端 JS 如何处理 URL 参数。
4. 查看 Web 日志中的编码形式。
5. 配置 CSP 并观察报告。
6. 在本地示例中实现 CSRF Token 并测试。

> 仅限本地或授权环境。

---

## 7. 自测

1. 反射型、存储型、DOM 型 XSS 的区别？
2. 输出编码为什么必须按上下文？
3. CSP 如何降低 XSS 风险？
4. HttpOnly 能完全防 XSS 吗？
5. CSRF 的原理与三个必要条件？
6. SameSite Cookie 的作用？
7. XSS 与 CSRF 如何组合利用？
8. 如何从日志检测 XSS 与 CSRF？

---

## 8. 延伸阅读

- [OWASP：XSS](https://owasp.org/www-community/attacks/xss/)
- [OWASP：CSRF](https://owasp.org/www-community/attacks/csrf)
- [OWASP XSS 防护备忘单](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [OWASP CSRF 防护备忘单](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [PortSwigger：XSS](https://portswigger.net/web-security/cross-site-scripting)
- [PortSwigger：CSRF](https://portswigger.net/web-security/csrf)
- [MDN：CSP](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CSP)
- [DOMPurify](https://github.com/cure53/DOMPurify)

---

## 9. 下一步

- [文件上传与命令注入](08-文件上传与命令注入.md)
- [SSRF 与 XXE](09-SSRF与XXE.md)
- [Webshell 基础与检测](11-Webshell基础与检测.md)
