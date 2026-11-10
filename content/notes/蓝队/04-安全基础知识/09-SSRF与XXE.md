---
title: "SSRF 与 XXE"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 9
showTableOfContents: true
---

> 难度：★★★☆☆ ｜ 重要度：重要 ｜ 建议学时：6 小时 ｜ 前置：[HTTP 协议详解](../02-计算机与网络基础/05-HTTP协议详解.md)、[Web 安全与 OWASP Top 10](05-Web安全与OWASP-Top10.md)

## 1. SSRF（服务端请求伪造）

### 1.1 原理

应用根据用户输入请求 URL，攻击者诱导服务器访问内网、本机或云元数据。

```text
用户 → 应用（有内网权限） → 内网服务/云元数据
```

危害：

- 探测内网服务与端口
- 访问未授权接口
- 读取云元数据获取临时凭据
- 访问本机管理接口
- 结合文件协议读取文件
- 作为跳板攻击内网

### 1.2 类型

| 类型 | 说明 |
|---|---|
| 基本 SSRF | 响应直接返回给攻击者 |
| 盲 SSRF | 无响应，只能通过时间/外带判断 |
| 半盲 SSRF | 只能看到部分信息（状态码、错误） |

### 1.3 常见目标

- `127.0.0.1`、`localhost`、`0.0.0.0`
- 内网网段：`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`
- 云元数据：
  - AWS：`169.254.169.254`
  - 阿里云：`100.100.100.200`
  - 腾讯云：`metadata.tencentyun.com`
  - GCP：`metadata.google.internal`
- 内部管理后台、Redis、Elasticsearch、数据库
- 容器/宿主机服务

### 1.4 常见协议

- `http://`、`https://`
- `file://`（读本地文件）
- `gopher://`（构造任意 TCP 数据）
- `dict://`
- `ftp://`

### 1.5 绕过思路（防御视角）

- 十进制/十六进制/八进制 IP
- IPv6 映射
- DNS 解析到内网
- URL 解析歧义
- 重定向到内网
- 大小写/编码
- 域名黑名单绕过

> 防御要基于「解析后的真实 IP」和「网络层限制」，而不是简单字符串黑名单。

### 1.6 检测线索

- 应用服务器主动外连内网地址
- 访问云元数据 IP/域名
- 非业务端口的连接
- 异常协议（file/gopher/dict）
- 服务器发起 DNS 查询到可疑域名
- 响应大小/时间异常
- Web 日志参数含 URL/IP
- 内网扫描特征

### 1.7 防御

- URL 白名单（协议、域名、端口）
- 解析后校验 IP，禁止内网/回环/链路本地
- 禁止重定向或校验重定向目标
- 禁用不必要协议
- 网络层隔离（应用服务器不能访问敏感内网）
- 云元数据加固（IMDSv2、最小权限）
- 统一出口代理
- 响应内容不直接返回

---

## 2. XXE（XML 外部实体注入）

### 2.1 原理

XML 解析器允许引用外部实体，攻击者构造 XML 读取文件、发起 SSRF、执行 DoS。

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<foo>&xxe;</foo>
```

> 仅用于理解，禁止对未授权系统测试。

### 2.2 危害

- 读取本地文件
- SSRF（HTTP 外部实体）
- 内网探测
- 拒绝服务（实体扩展，如 Billion Laughs）
- 结合反序列化/命令执行

### 2.3 常见入口

- SOAP 接口
- REST API 接收 XML
- SAML 断言
- SVG 上传
- DOCX/XLSX 等 Office 文件
- 配置文件导入
- RSS/XML 解析

### 2.4 检测线索

- 请求体含 `<!DOCTYPE`、`<!ENTITY`、`SYSTEM`、`PUBLIC`
- XML 请求异常大/嵌套深
- 应用服务器外连内部/外部地址
- 读取敏感文件（`/etc/passwd`、`win.ini`）
- 解析错误/异常堆栈
- 响应时间异常（实体扩展）

### 2.5 防御

- 禁用 DTD 与外部实体
- 使用安全解析器配置
- 限制 XML 大小与嵌套深度
- 使用 JSON 等更简单格式（如可行）
- WAF 拦截（补充）
- 最小权限运行

**Java 示例（概念）**：

```java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
dbf.setXIncludeAware(false);
dbf.setExpandEntityReferences(false);
```

> 不同解析器/版本的配置项不同，需查阅官方安全指南。Python 的 `defusedxml` 是更安全的选择。

---

## 3. SSRF 与 XXE 的关系

- XXE 可以通过外部实体发起 SSRF。
- SSRF 可以通过 `file://` 读取文件。
- 两者都可能成为「从 Web 进入内网」的跳板。
- 检测时都要关注**服务端主动外连**与**异常协议/文件访问**。

---

## 4. 蓝队检测示例

### 4.1 SSRF

```spl
index=web
| search uri="*url=*" OR uri="*image=*" OR uri="*fetch=*" OR uri="*callback=*"
| regex uri="(127\.0\.0\.1|169\.254\.169\.254|10\.|172\.(1[6-9]|2[0-9]|3[01])\.|192\.168\.)"
| table _time src_ip uri status
```

### 4.2 XXE

```spl
index=web
| search body="*<!DOCTYPE*" OR body="*<!ENTITY*"
| table _time src_ip uri status
```

> 字段名与日志内容取决于是否记录请求体。生产环境要注意隐私与日志量。

### 4.3 主机侧

- 应用进程发起异常网络连接（Sysmon 3）
- 应用进程读取敏感文件（EDR/auditd）
- 应用进程拉起 shell（命令执行）
- 云元数据访问日志

---

## 5. 应急响应

1. 确认漏洞入口与请求。
2. 检查服务器外连记录（内网、元数据、外部）。
3. 检查是否读取敏感文件/获取凭据。
4. 检查是否进一步命令执行/横向。
5. 保存日志与流量。
6. 修复解析配置/URL 校验，增加网络隔离。
7. 轮换可能泄露的凭据。
8. 复盘并补充检测。

---

## 6. 实操

在本地靶场：

1. 搭建一个存在 SSRF 的示例应用，观察内网请求日志。
2. 用 Wireshark 观察服务器发起的连接。
3. 搭建 XML 解析示例，测试禁用 DTD 前后的行为。
4. 用 `defusedxml` 替换不安全的 XML 解析。
5. 写一条检测规则，识别 XML 中的 `DOCTYPE/ENTITY`。
6. 在云实验环境（如有）了解 IMDSv2。

> 仅限本地或授权环境。

---

## 7. 自测

1. SSRF 的原理与危害？
2. 云元数据为什么是 SSRF 的重点目标？
3. 为什么字符串黑名单不能可靠防御 SSRF？
4. XXE 的原理？
5. 如何禁用 XXE？
6. SSRF 与 XXE 如何组合？
7. 如何从主机与流量发现 SSRF？

---

## 8. 延伸阅读

- [OWASP：SSRF](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
- [OWASP SSRF 防护备忘单](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
- [OWASP：XXE](https://owasp.org/www-community/vulnerabilities/XML_External_Entity_%28XXE%29_Processing)
- [OWASP XXE 防护备忘单](https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html)
- [PortSwigger：SSRF](https://portswigger.net/web-security/ssrf)
- [PortSwigger：XXE](https://portswigger.net/web-security/xxe)
- [defusedxml](https://github.com/tiran/defusedxml)
- [MITRE ATT&CK：Exploit Public-Facing Application](https://attack.mitre.org/techniques/T1190/)

---

## 9. 下一步

- [反序列化漏洞](10-反序列化漏洞.md)
- [Webshell 基础与检测](11-Webshell基础与检测.md)
- [云安全与容器基础](12-云安全与容器基础.md)
