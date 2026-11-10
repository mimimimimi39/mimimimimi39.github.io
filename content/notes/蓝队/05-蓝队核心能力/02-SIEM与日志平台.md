---
title: "SIEM 与日志平台"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 2
showTableOfContents: true
---

> 难度：★★★★☆ ｜ 重要度：必学 ｜ 建议学时：20 小时 ｜ 前置：[日志分析与日志源](03-日志分析与日志源.md)、[SQL 与日志查询](../03-编程与工具/04-SQL与日志查询.md)

## 1. SIEM 是什么

SIEM（Security Information and Event Management）集中采集、存储、分析安全日志，产生告警并支持调查。

核心能力：

- 日志采集与解析
- 归一化与富化
- 关联分析
- 告警与工单
- 搜索与仪表盘
- 报表与合规
- 案件管理（部分产品）

> SIEM 不是「装上就有效」。效果取决于**数据质量、规则质量、运营流程**。

---

## 2. 架构

```text
数据源
  ├─ 终端/服务器（Agent、Syslog、API）
  ├─ 网络设备（Syslog、NetFlow）
  ├─ 安全设备（Syslog、API）
  ├─ 应用/数据库（文件、API）
  └─ 云（API、日志服务）
        │
        ▼
采集层（Agent/Forwarder/Receiver）
        │
        ▼
解析与归一化（Parser、Schema）
        │
        ▼
富化（资产、身份、GeoIP、威胁情报）
        │
        ▼
存储与索引（热/温/冷）
        │
        ▼
关联规则 → 告警 → 工单/SOAR
        │
        ▼
搜索、仪表盘、报表、案件
```

---

## 3. 主流产品

| 产品 | 类型 | 特点 |
|---|---|---|
| Splunk | 商业 | SPL 强大，生态好，成本较高 |
| Elastic Security | 开源/商业 | ELK + 安全规则，灵活 |
| Microsoft Sentinel | 云原生 | KQL、Azure 集成 |
| IBM QRadar | 商业 | 关联能力强 |
| Micro Focus ArcSight | 商业 | 大型企业 |
| Wazuh | 开源 | HIDS + SIEM，轻量 |
| Security Onion | 开源 | NSM + SIEM |
| 奇安信/深信服/绿盟/安恒 | 国内商业 | 本地化、合规 |

---

## 4. 日志接入

### 4.1 接入方式

| 方式 | 适用 |
|---|---|
| Agent/Forwarder | 服务器、终端 |
| Syslog | 网络设备、Linux、安全设备 |
| API | 云、SaaS、应用 |
| 文件监控 | 应用日志 |
| 流量镜像 | NIDS/NDR |
| 数据库 | 业务日志 |

### 4.2 接入前要确认

- 日志格式与字段
- 时间戳与时区
- 数据量（EPS）
- 保留期
- 敏感信息（脱敏）
- 采集权限
- 网络与带宽
- 断点续传
- 日志轮转

### 4.3 常见问题

- 时间不同步 → 时间线错乱
- 字段缺失 → 规则失效
- 编码问题 → 中文乱码
- 日志重复 → 统计偏差
- 采集断流 → 盲区
- 数据量过大 → 成本失控

---

## 5. 解析与归一化

### 5.1 Parser

把原始日志解析为结构化字段。

- 正则
- Grok
- JSON 解析
- CSV
- KV
- 自定义解码

**示例（Grok）**：

```text
%{IP:src_ip} - - \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{URIPATHPARAM:uri} HTTP/%{NUMBER:http_version}" %{INT:status} %{INT:bytes}
```

### 5.2 归一化

统一字段名，便于跨数据源查询。

常见标准：

- ECS（Elastic Common Schema）
- OSSEM（Open Source Security Events Metadata）
- CIM（Splunk Common Information Model）
- 自定义 schema

示例字段：

```text
timestamp, host, user, src_ip, dst_ip, src_port, dst_port,
process_name, command_line, parent_process, file_path,
event_type, action, result, severity, raw
```

> 没有统一字段，跨源关联会非常痛苦。

---

## 6. 富化

- 资产信息：负责人、业务、重要性、OS、补丁
- 身份信息：部门、角色、特权
- GeoIP：国家、城市、ASN
- 威胁情报：IOC 匹配、信誉
- 历史行为：基线、异常
- 漏洞信息：资产漏洞

---

## 7. 关联规则

### 7.1 规则类型

| 类型 | 示例 |
|---|---|
| 单事件 | 登录失败 |
| 阈值 | 5 分钟内失败 20 次 |
| 序列 | 登录成功 → 创建用户 → 加入管理员组 |
| 时序 | 非工作时间登录 |
| 聚合 | 同一 IP 访问多个主机 |
| 情报匹配 | 外联恶意 IP |
| 基线偏离 | 流量突增 |

### 7.2 规则设计

一条好规则应包含：

```text
名称：SSH 暴力破解成功
描述：同一源 IP 在 10 分钟内失败超过 20 次后成功登录
数据源：Linux auth.log / SSH
逻辑：失败计数 > 20 且出现 Accepted
严重级别：高
ATT&CK：T1110 Brute Force
误报场景：运维批量登录、监控系统
处置建议：确认账号、检查来源、禁用/改密
测试：正常登录、失败登录、爆破后成功
```

### 7.3 规则调优

- 白名单（扫描器、监控、备份）
- 阈值调整
- 时间窗口
- 字段准确性
- 资产重要性加权
- 关联更多上下文
- 降低重复告警（去重/聚合）

---

## 8. 检测工程

### 8.1 规则生命周期

```text
需求 → 设计 → 实现 → 测试 → 上线 → 监控 → 调优 → 下线
```

### 8.2 测试

- 单元测试：构造日志验证命中
- 回归测试：修改后不影响其他规则
- 误报测试：正常业务不触发
- 覆盖测试：ATT&CK 映射

### 8.3 规则管理

- 版本控制（Git）
- 规则说明与作者
- 变更记录
- 依赖字段
- 数据源健康检查
- 定期评审

---

## 9. 查询示例

### 9.1 Splunk SPL

```spl
index=windows EventCode=4625
| stats count by src_ip, user
| where count > 20
| sort -count
```

```spl
index=proxy
| stats sum(bytes_out) as out_bytes by src_ip
| eval out_mb=out_bytes/1024/1024
| where out_mb > 500
| sort -out_mb
```

### 9.2 Elastic KQL

```text
event.code:4625 and source.ip:*
```

### 9.3 Sentinel KQL

```kusto
SecurityEvent
| where EventID == 4625
| summarize count() by IpAddress, Account
| where count_ > 20
```

---

## 10. 仪表盘与报表

常见仪表盘：

- 告警总览（按级别/类型/资产）
- 登录与账号
- 终端威胁
- 网络流量
- 漏洞与补丁
- 合规（等保、日志留存）
- 运营指标（MTTD/MTTR/误报率）

> 仪表盘要服务于决策，不要堆图表。

---

## 11. 成本与性能

- 日志量越大，存储与计算成本越高
- 分层存储（热/温/冷）
- 只采集有价值字段
- 采样（谨慎，可能漏证据）
- 保留期与合规平衡
- 规则复杂度影响性能
- 避免全量正则解析

---

## 12. 实操

1. 用 Wazuh 或 Splunk Free 接入 Windows/Linux 日志。
2. 写 3 条规则：SSH 爆破、异常 PowerShell、恶意 IP 外联。
3. 构造测试日志验证规则。
4. 为规则添加 ATT&CK 映射。
5. 统计一周告警的误报率。
6. 设计一个「登录异常」仪表盘。
7. 做一次规则调优并记录前后对比。

---

## 13. 自测

1. SIEM 的核心能力有哪些？
2. 日志接入前要确认什么？
3. 解析与归一化的区别？
4. 什么是一条好规则？
5. 如何降低误报？
6. 规则如何做测试与版本管理？
7. SIEM 成本如何优化？
8. 为什么 SIEM 上线不等于安全？

---

## 14. 延伸阅读

- [Splunk 文档](https://docs.splunk.com/)
- [Elastic Security 文档](https://www.elastic.co/guide/en/security/current/index.html)
- [Microsoft Sentinel 文档](https://learn.microsoft.com/en-us/azure/sentinel/)
- [Wazuh 文档](https://documentation.wazuh.com/)
- [Sigma 规则](https://github.com/SigmaHQ/sigma)
- [MITRE ATT&CK](https://attack.mitre.org/)
- [Detection Engineering 资源](https://github.com/FrankHassanabad/awesome-detection-engineering)

---

## 15. 下一步

- [日志分析与日志源](03-日志分析与日志源.md)
- [告警研判与误报分析](04-告警研判与误报分析.md)
- [威胁狩猎](08-威胁狩猎.md)
