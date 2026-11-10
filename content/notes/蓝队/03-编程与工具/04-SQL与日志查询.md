---
title: "SQL 与日志查询"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 4
showTableOfContents: true
---

> 难度：★★★☆☆ ｜ 重要度：必学 ｜ 建议学时：15 小时 ｜ 前置：[数据库基础](../02-计算机与网络基础/08-数据库基础.md)

## 1. 为什么蓝队要学查询语言

蓝队每天都要从海量日志中「找针」：

- 哪个 IP 在爆破？
- 哪个账号在异常时间登录？
- 哪台主机访问了恶意域名？
- 哪个进程拉起了 PowerShell？

SQL 是结构化查询的基础；SIEM 的 SPL、KQL、Lucene 都是查询语言。掌握一种，迁移很快。

---

## 2. SQL 快速复习

### 2.1 基本结构

```sql
SELECT columns
FROM table
WHERE condition
GROUP BY columns
HAVING condition
ORDER BY columns
LIMIT n;
```

### 2.2 安全分析常用查询

```sql
-- 登录失败 Top IP
SELECT src_ip, COUNT(*) AS failures
FROM auth_logs
WHERE event = 'failed_login'
  AND ts >= '2026-11-01'
GROUP BY src_ip
HAVING COUNT(*) > 20
ORDER BY failures DESC;

-- 成功登录后紧跟失败（可疑，MySQL 8+/PostgreSQL/SQL Server 支持 LAG）
SELECT user, src_ip, ts, event
FROM (
  SELECT user, src_ip, ts, event,
         LAG(event) OVER (PARTITION BY user ORDER BY ts) AS prev_event
  FROM auth_logs
  WHERE ts >= '2026-11-01'
) t
WHERE prev_event = 'login_success' AND event = 'failed_login'
ORDER BY user, ts;

-- 非工作时段登录
SELECT user, src_ip, ts
FROM auth_logs
WHERE event = 'login_success'
  AND (EXTRACT(HOUR FROM ts) < 6 OR EXTRACT(HOUR FROM ts) >= 22);

-- 进程与网络关联
SELECT p.host, p.user, p.process_name, p.command_line, n.remote_ip, n.remote_port
FROM process_events p
JOIN network_events n ON p.pid = n.pid AND p.host = n.host
WHERE n.remote_ip NOT LIKE '10.%'
  AND n.remote_ip NOT LIKE '192.168.%';
```

> 字段名因数据源而异，重点是查询思路：过滤 → 聚合 → 关联 → 排序。

---

## 3. Splunk SPL 基础

### 3.1 结构

```spl
index=security sourcetype=WinEventLog:Security EventCode=4625
| stats count by src_ip
| sort -count
| head 20
```

### 3.2 常用命令

| 命令 | 作用 |
|---|---|
| `search` | 过滤 |
| `stats` | 聚合统计 |
| `timechart` | 时间序列 |
| `eval` | 计算字段 |
| `rex` | 正则提取 |
| `table` | 选择字段 |
| `sort` | 排序 |
| `dedup` | 去重 |
| `head/tail` | 前/后 N 条 |
| `where` | 过滤计算结果 |
| `join` | 关联 |
| `lookup` | 查表/情报 |
| `transaction` | 会话聚合 |
| `tstats` | 加速数据模型查询 |

### 3.3 示例

```spl
index=windows EventCode=4625
| stats count by src_ip, user
| where count > 10
| sort -count
```

```spl
index=windows EventCode=4688
| search CommandLine="*powershell*" OR CommandLine="*cmd.exe*"
| table _time host user ParentProcessName NewProcessName CommandLine
```

```spl
index=proxy
| eval bytes_out_mb = bytes_out/1024/1024
| where bytes_out_mb > 100
| table _time src_ip dest_host bytes_out_mb
| sort -bytes_out_mb
```

```spl
index=dns
| eval qlen = len(query)
| where qlen > 100
| stats count by src_ip, query
| sort -count
```

> 字段名以实际 sourcetype 为准。先用 `| fieldsummary` 或事件详情确认字段。

---

## 4. Elastic / Kibana 查询

### 4.1 KQL 示例

```text
event.code: 4625 and winlog.event_data.LogonType: 3
```

```text
process.name: "powershell.exe" and process.command_line: *-enc*
```

```text
dns.question.name: * and not dns.question.name: *.microsoft.com
```

### 4.2 Lucene 示例

```text
event.code:4625 AND source.ip:[10.0.0.0 TO 10.255.255.255]
```

> Kibana 的 KQL 与 Lucene 语法不同，注意当前查询语言。

### 4.3 常用聚合

- Terms：按字段分组
- Date Histogram：时间趋势
- Filter：条件过滤
- Metrics：计数、去重、百分位

---

## 5. Microsoft Sentinel KQL 基础

```kusto
SecurityEvent
| where TimeGenerated > ago(24h)
| where EventID == 4625
| summarize FailedLogins = count() by IpAddress, Account
| where FailedLogins > 20
| order by FailedLogins desc
```

```kusto
DeviceProcessEvents
| where Timestamp > ago(7d)
| where FileName =~ "powershell.exe"
| where ProcessCommandLine contains "-enc"
| project Timestamp, DeviceName, AccountName, ProcessCommandLine
```

> Sentinel 使用 Kusto 查询语言（KQL），与 Kibana KQL 同名但语法不同。

---

## 6. 查询思路：从告警到结论

### 6.1 五步法

```text
1. 明确问题：我要验证什么假设？
2. 选择数据源：哪张表/哪个索引有相关数据？
3. 过滤：时间、资产、账号、事件类型。
4. 聚合/关联：统计、分组、连接。
5. 解释：结果是否支持假设？还需要什么数据？
```

### 6.2 示例：疑似 SSH 爆破

```spl
index=linux sourcetype=linux_secure "Failed password"
| stats count by src_ip, user
| where count > 50
| sort -count
```

然后关联：

```spl
index=linux sourcetype=linux_secure "Accepted password"
| search src_ip=<可疑IP>
| table _time host user src_ip
```

判断：

- 是否成功登录？
- 登录后执行了什么命令？
- 是否有横向移动？
- 是否有持久化？

### 6.3 示例：异常 PowerShell

```spl
index=windows EventCode=4688
| search CommandLine="*powershell*"
| rex field=CommandLine "(?i)(?<flag>-enc|-encodedcommand|-nop|-w hidden)"
| where isnotnull(flag)
| stats count by host, user, CommandLine
```

---

## 7. 查询优化

1. **先缩小时间范围**：不要一上来查 30 天全量。
2. **先用索引/分区**：`index=`、`sourcetype=`。
3. **过滤前置**：尽早过滤，减少后续计算。
4. **避免 `*` 开头通配**：性能差。
5. **限制返回条数**：`head`、`limit`。
6. **使用字段提取**：避免重复正则。
7. **使用加速**：数据模型、摘要索引、物化视图。
8. **了解保留期**：热/温/冷数据。

---

## 8. 常见错误

| 错误 | 后果 | 正确做法 |
|---|---|---|
| 时间范围过大 | 查询超时 | 先小范围验证 |
| 字段名写错 | 查不到数据 | 先看字段列表 |
| 只查单一数据源 | 结论片面 | 多源关联 |
| 忽略时区 | 时间线错乱 | 统一 UTC/本地时间 |
| 把 `count` 当唯一用户数 | 统计错误 | 用 `dc()`/`COUNT(DISTINCT)` |
| 不看原始日志 | 误判 | 抽样查看原文 |
| 不用引号/转义 | 语法错误 | 按平台规则 |

---

## 9. 动手练习

1. 用 SQL 统计登录失败 Top10 IP。
2. 用 SQL 找出非工作时段登录。
3. 用 Splunk Free 导入一份 CSV 日志，写 3 条 SPL 查询。
4. 用 Kibana 导入样本日志，按 IP 做 Terms 聚合。
5. 写一条查询：找出访问恶意域名的内网主机。
6. 写一条查询：找出父进程为 `winword.exe` 的 PowerShell。
7. 对比 SQL、SPL、KQL 的异同，整理成速查表。

---

## 10. 自测

1. SQL 中 WHERE 与 HAVING 的区别？
2. `COUNT(*)` 与 `COUNT(DISTINCT field)` 的区别？
3. SPL 中 `stats`、`eval`、`rex` 分别做什么？
4. 如何关联进程与网络日志？
5. 查询为什么要先缩小时间范围？
6. KQL 与 Splunk SPL 的字段过滤写法有何不同？
7. 如何验证「爆破后是否成功登录」？

---

## 11. 延伸阅读

- [Splunk 搜索参考](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference)
- [Splunk 免费版](https://www.splunk.com/en_us/download/splunk-enterprise.html)
- [Elasticsearch Query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)
- [Kibana KQL](https://www.elastic.co/guide/en/kibana/current/kuery-query.html)
- [Microsoft Sentinel KQL](https://learn.microsoft.com/en-us/azure/sentinel/kusto-overview)
- [SQLBolt](https://sqlbolt.com/)
- [SQLZoo](https://sqlzoo.net/)

---

## 12. 下一步

- [正则表达式](05-正则表达式.md)
- [SIEM 与日志平台](../05-蓝队核心能力/02-SIEM与日志平台.md)
- [告警研判与误报分析](../05-蓝队核心能力/04-告警研判与误报分析.md)
