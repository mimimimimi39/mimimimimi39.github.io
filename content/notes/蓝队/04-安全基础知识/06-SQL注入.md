---
title: "SQL 注入"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 6
showTableOfContents: true
---

> 难度：★★★☆☆ ｜ 重要度：必学 ｜ 建议学时：8 小时 ｜ 前置：[数据库基础](../02-计算机与网络基础/08-数据库基础.md)、[HTTP 协议详解](../02-计算机与网络基础/05-HTTP协议详解.md)

## 1. 原理

应用把用户输入直接拼接到 SQL 语句中，导致输入改变原语句语义。

```php
// 错误示例
$sql = "SELECT * FROM users WHERE username='" . $_GET['u'] . "' AND password='" . $_GET['p'] . "'";
```

输入 `u=admin' -- ` 后：

```sql
SELECT * FROM users WHERE username='admin' -- ' AND password='...'
```

`--` 注释掉后面的密码校验，绕过登录。

> 本篇仅用于理解与防御，禁止对未授权系统测试。

---

## 2. 注入类型

| 类型 | 特点 | 检测难度 |
|---|---|---|
| 联合查询（UNION） | 利用 UNION 返回额外数据 | 低 |
| 布尔盲注 | 根据真假页面差异推断 | 中 |
| 时间盲注 | 根据响应时间推断 | 中 |
| 报错注入 | 利用数据库报错回显 | 低 |
| 堆叠查询 | 多条语句执行 | 取决于驱动 |
| 二次注入 | 恶意数据先存储后触发 | 高 |
| 带外（OOB） | 通过 DNS/HTTP 外带数据 | 中 |

---

## 3. 常见注入点

- URL 查询参数：`?id=1`
- POST 表单
- JSON 字段
- Cookie
- HTTP 头（User-Agent、Referer、X-Forwarded-For）
- 排序/分页参数（`order by`、`limit`）
- 搜索框
- 登录/注册
- API 参数

> 任何进入 SQL 的用户可控数据都可能成为注入点。

---

## 4. 日志与流量特征

### 4.1 Web 日志特征

- 参数含单引号 `'`、双引号 `"`、分号 `;`
- 关键字：`UNION`、`SELECT`、`SLEEP`、`BENCHMARK`、`WAITFOR`、`INFORMATION_SCHEMA`
- 注释：`--`、`#`、`/* */`
- 编码：`%27`、`%20`、`%23`、双重编码
- 布尔条件：`AND 1=1`、`AND 1=2`、`OR '1'='1'`
- 时间函数：`SLEEP(5)`、`pg_sleep(5)`、`WAITFOR DELAY`
- 大量相似请求（盲注逐字符）
- 异常长的参数
- 状态码 500、数据库错误
- 响应大小差异（布尔盲注）

### 4.2 WAF 日志

- 规则命中：SQLi 规则、异常关键字
- 动作：拦截/放行/观察
- 误报：正常业务含引号、搜索词

### 4.3 主机与数据库日志

- Web 进程异常子进程（命令执行）
- 数据库错误日志
- 慢查询日志
- 异常 SQL（大量 SELECT、INFORMATION_SCHEMA）
- 数据库账号异常登录

### 4.4 流量特征

- URI 中大量编码字符
- POST body 中含 SQL 关键字
- 响应时间异常（时间盲注）
- 同一参数高频变化（盲注）
- 外带 DNS/HTTP 请求

---

## 5. 检测思路

1. **Web 日志基线**：正常参数长度、字符集、关键字。
2. **规则匹配**：SQL 关键字 + 特殊字符组合。
3. **频率分析**：同一 IP 对同一 URI 高频请求。
4. **响应差异**：状态码、响应大小、响应时间。
5. **WAF 联动**：观察模式先记录，再拦截。
6. **主机关联**：Web 进程是否拉起 shell、是否访问数据库异常。
7. **情报**：源 IP 是否扫描器/恶意 IP。

### 示例 SPL

```spl
index=web
| search uri="*UNION*" OR uri="*SELECT*" OR uri="*SLEEP*" OR uri="*%27*"
| stats count by src_ip, uri, status
| where count > 5
| sort -count
```

```spl
index=waf action=blocked
| stats count by src_ip, rule
| where count > 20
```

> 字段名按实际环境调整。注意 URL 编码与大小写。

---

## 6. 防御

### 6.1 根本措施

- **参数化查询/预编译语句**（首选）
- ORM 正确使用（避免字符串拼接）
- 输入校验（类型、长度、白名单）
- 最小权限数据库账号
- 错误信息不返回给用户
- 关闭不必要的数据库功能
- WAF 作为纵深防御，不是唯一措施

### 6.2 代码示例

```python
# 错误：字符串拼接
cursor.execute("SELECT * FROM users WHERE name='" + name + "'")

# 正确：参数化
cursor.execute("SELECT * FROM users WHERE name = %s", (name,))
```

```java
// 正确：PreparedStatement
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE name = ?");
ps.setString(1, name);
```

### 6.3 数据库加固

- 应用账号只授予必要权限
- 禁止应用账号访问系统表
- 禁止堆叠查询（视驱动）
- 数据库不暴露公网
- 审计与监控

---

## 7. 应急响应

1. 确认注入点与影响范围。
2. 检查是否被拖库（大批量查询、导出）。
3. 检查是否被写入 Webshell/命令执行。
4. 检查数据库账号与权限变更。
5. 保存日志与流量证据。
6. 修复代码、加 WAF 规则、重置凭据。
7. 评估数据泄露与合规上报。
8. 复盘并补充检测。

---

## 8. 实操

在 DVWA 或本地靶场：

1. 低难度下尝试联合查询，观察回显。
2. 中难度下观察过滤与绕过（仅学习）。
3. 查看 Web 日志与 WAF 日志，记录特征。
4. 用 Python 写一个简单的盲注检测脚本（仅本地）。
5. 用参数化查询修复一个示例应用。

> 所有测试必须在本地或书面授权的环境中进行。

---

## 9. 自测

1. SQL 注入的原理是什么？
2. 联合注入、布尔盲注、时间盲注的区别？
3. 参数化查询为什么能防注入？
4. WAF 能完全防止 SQL 注入吗？
5. 如何从 Web 日志发现 SQL 注入？
6. 注入成功后攻击者可能做什么？
7. 数据库最小权限如何配置？

---

## 10. 延伸阅读

- [OWASP：SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [OWASP SQL 注入防护备忘单](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [PortSwigger：SQL Injection](https://portswigger.net/web-security/sql-injection)
- [MITRE ATT&CK：Exploit Public-Facing Application](https://attack.mitre.org/techniques/T1190/)
- [DVWA](https://github.com/digininja/DVWA)
- [SQLMap](https://sqlmap.org/) —— 理解自动化注入的特征

---

## 11. 下一步

- [XSS 与 CSRF](07-XSS与CSRF.md)
- [Webshell 基础与检测](11-Webshell基础与检测.md)
- [Web 安全与 OWASP Top 10](05-Web安全与OWASP-Top10.md)
