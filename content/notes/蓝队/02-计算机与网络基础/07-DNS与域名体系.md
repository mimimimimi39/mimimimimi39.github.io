---
title: "DNS 与域名体系"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 7
showTableOfContents: true
---

> 难度：★★☆☆☆ ｜ 重要度：重要 ｜ 建议学时：6 小时 ｜ 前置：[计算机网络基础](04-计算机网络基础.md)

## 1. 为什么蓝队要学 DNS

- DNS 是几乎所有网络访问的起点，也是攻击者常用的隐蔽通道。
- DNS 日志能揭示恶意域名、DGA、隧道、C2、数据外传。
- 域名信誉、新注册域名、可疑 TLD 是威胁情报的重要维度。
- 钓鱼、域名劫持、子域接管都与 DNS 相关。

---

## 2. 域名结构

```text
www.example.com.
│   │       │
│   │       └── 顶级域 TLD（com）
│   └────────── 二级域（example）
└────────────── 主机名/子域（www）
```

- 根域：`.`
- 顶级域（TLD）：`.com`、`.org`、`.cn`、`.xyz` 等
- 国家代码顶级域（ccTLD）：`.cn`、`.us`、`.ru`
- 二级域：注册者购买
- 子域：由域名持有者管理

---

## 3. DNS 解析流程

### 3.1 递归解析

```text
应用 → 本地 DNS 缓存 → 递归解析器（如 8.8.8.8、企业 DNS）
  → 根服务器（.）
  → 顶级域服务器（.com）
  → 权威服务器（example.com）
  → 返回 A 记录
```

### 3.2 迭代 vs 递归

| 类型 | 说明 |
|---|---|
| 递归 | 客户端请求解析器「给我最终答案」 |
| 迭代 | 解析器逐级询问，自己完成查询 |

### 3.3 缓存与 TTL

- 每级都可缓存
- TTL 决定缓存时间
- TTL 过短可能用于快速切换（fast flux）
- TTL 过长不利于应急封禁

---

## 4. DNS 记录类型

| 记录 | 作用 | 示例 |
|---|---|---|
| A | 域名 → IPv4 | `example.com. 300 IN A 93.184.216.34` |
| AAAA | 域名 → IPv6 | `example.com. 300 IN AAAA 2606:...` |
| CNAME | 别名 | `www.example.com. IN CNAME example.com.` |
| MX | 邮件服务器 | `example.com. IN MX 10 mail.example.com.` |
| NS | 权威服务器 | `example.com. IN NS ns1.example.com.` |
| TXT | 文本记录 | SPF、DKIM、DMARC、验证 |
| SOA | 区域起始授权 | 主 NS、管理员邮箱、序列号 |
| PTR | IP → 域名（反查） | `34.216.184.93.in-addr.arpa.` |
| SRV | 服务定位 | `_ldap._tcp.example.com.` |
| CAA | 允许颁发证书的 CA | `example.com. IN CAA 0 issue "letsencrypt.org"` |

---

## 5. 常见 DNS 命令

```bash
dig example.com
dig example.com A
dig example.com MX
dig example.com TXT
dig +short example.com
dig @8.8.8.8 example.com
dig -x 93.184.216.34
nslookup example.com
host example.com
whois example.com
```

Windows：

```powershell
nslookup example.com
Resolve-DnsName example.com
ipconfig /displaydns
ipconfig /flushdns
```

---

## 6. DNS 安全机制

### 6.1 DNSSEC

用数字签名保证 DNS 响应未被篡改。

- 签名链从根到权威服务器
- 需要解析器支持
- 不提供机密性

### 6.2 DoH / DoT

- DoH（DNS over HTTPS）：DNS 查询走 HTTPS（443）
- DoT（DNS over TLS）：DNS 查询走 TLS（853）

优点：隐私、防窃听。
缺点：企业可见性下降，可能被恶意软件滥用绕过检测。

### 6.3 企业 DNS 安全

- 内部 DNS 解析器
- DNS 日志留存
- 恶意域名拦截
- DNSSEC 验证
- 限制外部 DNS（防止绕过）

---

## 7. DNS 攻击与滥用

| 攻击 | 原理 | 检测线索 |
|---|---|---|
| DNS 隧道 | 将数据编码到域名/查询中 | 长域名、高熵、频繁 TXT/NULL、固定间隔 |
| DGA | 算法生成大量随机域名 | 随机字符串、大量 NXDOMAIN、短 TTL |
| Fast Flux | 快速更换解析 IP | 同一域名多 IP、TTL 短、IP 分散 |
| 域名劫持 | 篡改解析记录 | 解析结果与权威记录不一致 |
| 缓存投毒 | 污染递归解析器缓存 | 异常解析、DNSSEC 失败 |
| DNS 重绑定 | 解析结果在公网/内网间切换 | 短 TTL、解析变化、SSRF 关联 |
| 子域接管 | 子域指向已释放的云资源 | CNAME 指向不存在的服务 |
| 恶意域名 | 钓鱼、C2、下载站 | 新注册、低信誉、可疑 TLD |
| DNS 放大攻击 | 利用 DNS 响应放大流量 | 大量查询、异常源 IP |
| 内部 DNS 探测 | 枚举内网主机/服务 | 大量 PTR/SRV 查询 |

### 7.1 DNS 隧道特征

- 域名长度异常（超过 50-100 字符）
- 子域熵高（随机字符）
- 查询类型异常（TXT、NULL、CNAME、MX）
- 查询频率固定
- 同一域名大量不同子域
- 响应数据量大
- 与已知 DNS 隧道工具特征匹配（如 iodine、dnscat2、DNSExfiltrator）

### 7.2 DGA 特征

- 随机字母数字组合
- 长度固定或变化
- 大量 NXDOMAIN
- 失败后尝试下一个域名
- 与恶意软件家族关联

---

## 8. 威胁情报中的域名

关注：

- 域名注册时间（新注册高风险）
- 注册商、注册人（隐私保护常见）
- TLD（免费/廉价 TLD 高风险）
- 解析 IP 与 ASN
- 被动 DNS（历史解析）
- 证书透明日志（CT）
- 域名信誉（VT、OTX、微步等）
- 是否与已知恶意样本关联

工具：

- VirusTotal：https://www.virustotal.com/
- AlienVault OTX：https://otx.alienvault.com/
- URLhaus：https://urlhaus.abuse.ch/
- ThreatFox：https://threatfox.abuse.ch/
- SecurityTrails、Censys、Shodan（部分收费）
- crt.sh（证书透明）：https://crt.sh/

---

## 9. 蓝队 DNS 检测思路

1. **数量基线**：每台主机/用户的 DNS 查询量。
2. **域名特征**：长度、熵、字符集、TLD。
3. **查询类型**：TXT/NULL 异常增多。
4. **响应码**：大量 NXDOMAIN。
5. **时间模式**：固定间隔、非工作时段。
6. **目的地**：外部 DNS、DoH 端点、少见解析器。
7. **关联**：DNS + 代理 + 防火墙 + 终端进程。
8. **情报匹配**：恶意域名、IOC、被动 DNS。

### 示例查询（Splunk SPL 思路）

```spl
index=dns
| stats count dc(query) as unique_queries by src_ip
| where unique_queries > 1000
```

```spl
index=dns query_type=TXT
| eval len=len(query)
| where len > 100
| table _time src_ip query query_type
```

> 具体字段名取决于日志接入方式，需按实际 schema 调整。

---

## 10. 实操

1. 用 `dig` 查询 A、MX、TXT、NS 记录。
2. 用 `dig +trace example.com` 观察迭代解析。
3. 用 `dig -x` 反查 IP。
4. 用 Wireshark 过滤 `dns`，观察查询与响应。
5. 在实验环境搭建 dnsmasq 或 Bind，查看查询日志。
6. 用 `tcpdump port 53` 抓取 DNS 流量。
7. 分析一份 DNS 日志，找出查询量最大的主机与最长域名。

---

## 11. 自测

1. 递归解析与迭代解析的区别？
2. A、CNAME、MX、TXT、PTR 记录分别做什么？
3. DNS 隧道有哪些特征？
4. DGA 是什么？如何检测？
5. DoH/DoT 对蓝队有什么影响？
6. 如何判断一个域名是否可疑？
7. DNSSEC 能解决什么问题？不能解决什么？

---

## 12. 延伸阅读

- [Cloudflare：What is DNS?](https://www.cloudflare.com/learning/dns/what-is-dns/)
- [RFC 1034/1035](https://www.rfc-editor.org/rfc/rfc1034)
- [DNSViz](https://dnsviz.net/)
- [crt.sh](https://crt.sh/)
- [VirusTotal](https://www.virustotal.com/)
- [URLhaus](https://urlhaus.abuse.ch/)
- [Zeek DNS 日志文档](https://docs.zeek.org/en/master/scripts/base/protocols/dns/main.zeek.html)
- [MITRE ATT&CK：DNS 隧道](https://attack.mitre.org/techniques/T1071/004/)

---

## 13. 下一步

- [数据库基础](08-数据库基础.md)
- [网络流量异常检测](../07-网络流量分析/02-网络流量异常检测.md)
- [威胁情报](../05-蓝队核心能力/07-威胁情报.md)
