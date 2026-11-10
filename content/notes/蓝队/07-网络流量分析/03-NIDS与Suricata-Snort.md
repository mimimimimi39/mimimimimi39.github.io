---
title: "NIDS 与 Suricata/Snort"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 3
showTableOfContents: true
---

> 难度：★★★★☆ ｜ 重要度：重要 ｜ 建议学时：10 小时 ｜ 前置：[网络流量异常检测](02-网络流量异常检测.md)、[Wireshark 与抓包分析](01-Wireshark与抓包分析.md)

## 1. IDS/IPS/NIDS

| 类型 | 说明 | 部署 |
|---|---|---|
| HIDS | 主机入侵检测 | 终端 Agent |
| NIDS | 网络入侵检测（旁路） | SPAN/TAP |
| NIPS | 网络入侵防御（串联） | Inline |
| NDR | 网络检测与响应 | 旁路 + 分析平台 |

- IDS：只告警，不阻断
- IPS：可阻断，但可能影响业务
- NIDS：网络层 IDS

> 旁路部署不影响业务，适合检测；串联部署能阻断，但要谨慎防误杀。

---

## 2. Snort/Suricata 对比

| 维度 | Snort | Suricata |
|---|---|---|
| 规则 | 成熟 | 兼容 Snort 规则 |
| 性能 | 好 | 多线程，性能强 |
| 输出 | 文本/统一格式 | EVE JSON |
| 功能 | IDS/IPS | IDS/IPS/NSM |
| 社区 | 大 | 活跃 |

Zeek 不是规则型 IDS，而是网络元数据/NSM 平台，常与 Suricata 配合。

---

## 3. 规则结构

```text
action protocol src_ip src_port -> dst_ip dst_port (options)
```

示例：

```text
alert tcp any any -> any 80 (msg:"Possible SQL Injection"; content:"UNION SELECT"; nocase; http_uri; sid:1000001; rev:1;)
```

### 3.1 动作

| 动作 | 含义 |
|---|---|
| alert | 告警 |
| log | 记录 |
| pass | 放行 |
| drop | 丢弃 |
| reject | 拒绝并重置 |

> 只有 IPS/Inline 模式下 drop/reject 才生效。

### 3.2 协议与方向

- `tcp`、`udp`、`icmp`、`ip`、`http`、`dns`、`tls` 等（Suricata 支持应用层协议）
- `->` 单向
- `<>` 双向

### 3.3 常用选项

| 选项 | 作用 |
|---|---|
| `msg` | 告警消息 |
| `sid` | 规则 ID |
| `rev` | 版本 |
| `content` | 内容匹配 |
| `nocase` | 忽略大小写 |
| `depth`/`offset` | 匹配范围 |
| `distance`/`within` | 相对匹配 |
| `flow` | 流方向/状态 |
| `flags` | TCP 标志 |
| `http_uri` | 匹配 URI |
| `http_method` | 匹配方法 |
| `http_header` | 匹配头 |
| `http_user_agent` | 匹配 UA |
| `urilen` | URI 长度 |
| `threshold`/`detection_filter` | 阈值 |
| `classtype` | 分类 |
| `reference` | 参考 |
| `metadata` | 元数据 |
| `priority` | 优先级 |

---

## 4. 规则示例

### 4.1 扫描

```text
alert tcp any any -> $HOME_NET any (msg:"Possible SYN Scan"; flags:S; threshold:type both, track by_src, count 20, seconds 10; sid:1000100; rev:1;)
```

### 4.2 SQL 注入

```text
alert http any any -> $HOME_NET any (msg:"SQL Injection Attempt"; content:"UNION"; nocase; http_uri; content:"SELECT"; nocase; http_uri; distance:0; sid:1000101; rev:1;)
```

### 4.3 Webshell 访问

```text
alert http $EXTERNAL_NET any -> $HOME_NET any (msg:"Suspicious POST to PHP"; http.method; content:"POST"; http.uri; content:".php"; nocase; sid:1000102; rev:1;)
```

> 该规则误报较高，需结合请求体/响应/频率/目录调优。

### 4.4 可疑 User-Agent

```text
alert http any any -> any any (msg:"Scanner User-Agent"; http.user_agent; content:"sqlmap"; nocase; sid:1000103; rev:1;)
```

### 4.5 DNS 长域名

```text
alert dns any any -> any any (msg:"Possible DNS Tunneling"; dns.query; content:".example.com"; nocase; sid:1000104; rev:1;)
```

> 更可靠的方式是用 Zeek/脚本做长度与熵分析。

---

## 5. 规则管理

- 规则集：ET Open、ET Pro、Snort Community 等
- 更新工具：Suricata-Update、PulledPork 等
- 版本管理（Git）
- 启用/禁用
- 优先级/分类
- 误报调优
- 定期更新
- 测试环境验证
- 与 SIEM 集成

```bash
suricata-update
suricata -T -c /etc/suricata/suricata.yaml -S rules.rules
```

---

## 6. 部署

### 6.1 位置

- 互联网出口
- 数据中心入口
- 核心交换机 SPAN
- 云 VPC 镜像
- 内网关键区

### 6.2 性能

- 多线程/CPU 亲和
- 规则数量
- 流量大小
- 硬件加速
- 丢包监控
- 缓冲区

### 6.3 高可用

- 多实例
- 负载均衡
- 故障切换
- 规则同步

---

## 7. 告警分析

- 告警类型、源/目的、时间
- 原始包/PCAP
- 规则内容与误报可能
- 资产重要性
- 关联主机日志
- 情报
- 是否需要升级

> 不要只信告警标题，要看触发规则与原始流量。

---

## 8. 局限与规避

- 加密流量（HTTPS 内容不可见）
- 分片/编码/混淆
- 慢速攻击
- 规则覆盖不足
- 误报/漏报
- 性能瓶颈
- 旁路无法阻断
- 隧道/隐蔽通道

应对：

- TLS 解密/元数据
- 行为分析
- Zeek/NDR
- 多源关联
- 威胁狩猎

---

## 9. 实操

1. 安装 Suricata，配置 SPAN 或 `-i` 抓包。
2. 加载 ET Open 规则。
3. 用本地靶场触发一条规则。
4. 编写自定义规则并测试。
5. 查看 EVE JSON 日志。
6. 与 Wazuh/SIEM 集成。
7. 分析告警误报并调优。
8. 监控丢包与性能。

---

## 10. 自测

1. IDS 与 IPS 的区别？
2. NIDS 部署位置？
3. Snort/Suricata 规则结构？
4. 常用规则选项有哪些？
5. 如何降低误报？
6. 加密流量下 NIDS 的局限？
7. 如何测试规则？
8. Zeek 与 Suricata 的区别？

---

## 11. 延伸阅读

- [Suricata 文档](https://docs.suricata.io/)
- [Snort 文档](https://www.snort.org/documents)
- [Emerging Threats Open](https://rules.emergingthreats.net/)
- [Zeek 文档](https://docs.zeek.org/)
- [Suricata-Update](https://github.com/OISF/suricata-update)
- [MITRE ATT&CK](https://attack.mitre.org/)
- [Security Onion](https://docs.securityonion.net/)

---

## 12. 下一步

- [防火墙、WAF 与代理](04-防火墙WAF与代理.md)
- [威胁狩猎](../05-蓝队核心能力/08-威胁狩猎.md)
- [综合实战项目](../09-实战与靶场/03-综合实战项目.md)
