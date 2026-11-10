---
title: "漏洞管理与 CVE 体系"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 4
showTableOfContents: true
---

> 难度：★★★☆☆ ｜ 重要度：必学 ｜ 建议学时：8 小时 ｜ 前置：[网络安全基本概念](01-网络安全基本概念.md)

## 1. 漏洞管理是什么

漏洞管理不是「扫一次漏洞」，而是一个持续闭环：

```text
资产发现 → 漏洞识别 → 风险评估 → 优先级排序 → 修复/缓解
→ 验证 → 度量 → 持续改进
```

蓝队需要理解漏洞，因为：

- 告警常与漏洞利用相关（如 Log4j、ProxyLogon）。
- 应急时要判断攻击者利用了哪个入口。
- 加固与补丁是防御的基础。
- 面试常问 CVE、CVSS、漏洞优先级。

---

## 2. 核心概念

| 概念 | 含义 |
|---|---|
| 漏洞（Vulnerability） | 系统、应用、配置中的弱点 |
| 利用（Exploit） | 利用漏洞的代码/方法 |
| PoC | 概念验证，证明漏洞可利用 |
| 0day | 厂商未发布补丁的漏洞 |
| Nday | 已有补丁但未及时修复的漏洞 |
| 攻击面 | 可被攻击的入口 |
| 暴露面 | 实际可从外部/内网访问的范围 |
| 误报 | 扫描器认为有漏洞但实际不存在 |
| 漏报 | 实际存在但扫描器未发现 |

---

## 3. 漏洞编号与数据库

### 3.1 CVE

- Common Vulnerabilities and Exposures
- 由 MITRE 维护，CVE Program
- 格式：`CVE-YYYY-NNNN`（至少 4 位数字，可变长）
- 提供唯一标识，不提供评分与修复建议
- 官网：https://www.cve.org/

### 3.2 NVD

- National Vulnerability Database，NIST 维护
- 基于 CVE，增加 CVSS、CWE、参考链接
- 官网：https://nvd.nist.gov/

### 3.3 中国漏洞库

| 名称 | 说明 |
|---|---|
| CNVD | 国家信息安全漏洞共享平台 |
| CNNVD | 国家信息安全漏洞库 |
| 厂商安全公告 | 微软、Oracle、Apache 等 |

- CNVD：https://www.cnvd.org.cn/
- CNNVD：https://www.cnnvd.org.cn/

### 3.4 其他

- GitHub Advisory Database：开源依赖漏洞
- OSV：开源漏洞数据库
- Exploit-DB：公开 Exploit
- Metasploit：利用模块
- 厂商公告与补丁日（Patch Tuesday）

---

## 4. CWE

Common Weakness Enumeration，软件弱点分类。

- 描述「哪类缺陷」
- 与 CVE 的关系：CVE 是具体漏洞，CWE 是漏洞类型
- 官网：https://cwe.mitre.org/

常见 CWE：

| CWE | 名称 |
|---|---|
| CWE-79 | XSS |
| CWE-89 | SQL 注入 |
| CWE-78 | OS 命令注入 |
| CWE-22 | 路径穿越 |
| CWE-287 | 认证不当 |
| CWE-306 | 关键功能缺少认证 |
| CWE-502 | 不可信数据反序列化 |
| CWE-798 | 硬编码凭据 |
| CWE-918 | SSRF |

---

## 5. CVSS

Common Vulnerability Scoring System，漏洞严重性评分。

### 5.1 CVSS v3.1 指标

**基础指标（Base）**：

| 指标 | 含义 | 取值 |
|---|---|---|
| Attack Vector (AV) | 攻击路径 | Network/Adjacent/Local/Physical |
| Attack Complexity (AC) | 攻击复杂度 | Low/High |
| Privileges Required (PR) | 所需权限 | None/Low/High |
| User Interaction (UI) | 是否需要用户交互 | None/Required |
| Scope (S) | 影响范围 | Unchanged/Changed |
| Confidentiality (C) | 机密性影响 | None/Low/High |
| Integrity (I) | 完整性影响 | None/Low/High |
| Availability (A) | 可用性影响 | None/Low/High |

**评分范围**：

| 分数 | 等级 |
|---|---|
| 0.0 | None |
| 0.1-3.9 | Low |
| 4.0-6.9 | Medium |
| 7.0-8.9 | High |
| 9.0-10.0 | Critical |

> CVSS v4.0 已于 2023 年发布，指标体系更细（攻击要求、用户交互、后续系统影响等）。企业环境可能仍以 v3.1 为主，需注意版本差异。

### 5.2 CVSS 的局限

- 只描述「技术严重性」，不描述「你的资产是否暴露」
- 不包含威胁情报（是否已被利用）
- 不包含业务影响
- 评分可能滞后或争议
- 不能单独作为修复优先级

---

## 6. EPSS 与 KEV

### 6.1 EPSS

Exploit Prediction Scoring System，由 FIRST 维护。

- 预测未来 30 天内漏洞被利用的概率（0-1）
- 用于辅助优先级排序
- 官网：https://www.first.org/epss/

### 6.2 CISA KEV

Known Exploited Vulnerabilities Catalog。

- 已知被在野利用的漏洞列表
- 美国 CISA 维护
- 官网：https://www.cisa.gov/known-exploited-vulnerabilities-catalog
- 列入 KEV 通常意味着应优先修复

---

## 7. 漏洞优先级

不要只看 CVSS。建议综合：

```text
优先级 = 资产重要性 × 暴露程度 × 漏洞严重性 × 利用可能性 × 业务影响
```

| 因素 | 说明 |
|---|---|
| 资产重要性 | 核心业务、数据敏感度 |
| 暴露程度 | 公网、内网、隔离网 |
| 漏洞严重性 | CVSS |
| 利用可能性 | EPSS、KEV、PoC 是否公开 |
| 业务影响 | 停机、合规、声誉 |
| 补偿控制 | WAF、网络隔离、EDR |

### 7.1 修复策略

| 情况 | 建议 |
|---|---|
| 公网 + 已被利用 + 核心资产 | 立即修复/临时下线 |
| 公网 + PoC 公开 | 尽快修复，临时缓解 |
| 内网 + 高权限 | 计划修复 |
| 低风险 + 隔离 | 排期修复 |
| 无法修复 | 补偿控制 + 监控 |

---

## 8. 漏洞扫描

### 8.1 工具

| 工具 | 类型 |
|---|---|
| Nessus | 商业漏洞扫描 |
| Qualys | 云扫描 |
| OpenVAS/Greenbone | 开源 |
| Nuclei | 模板化漏洞扫描 |
| Nmap | 端口与服务识别 |
| Trivy | 容器/镜像/依赖扫描 |
| Dependency-Check | 依赖漏洞 |
| OSV-Scanner | 开源依赖 |
| Semgrep | 代码静态分析 |
| SonarQube | 代码质量与安全 |

### 8.2 扫描注意事项

- 必须获得授权
- 选择合适时间，避免影响业务
- 控制扫描频率与并发
- 处理误报
- 扫描器不能替代人工验证
- 认证扫描可发现更多问题
- 扫描结果要进资产/漏洞管理流程

---

## 9. 补丁管理

```text
资产清单 → 补丁评估 → 测试 → 灰度发布 → 全量部署
→ 验证 → 例外管理 → 度量
```

关键点：

- 知道你有什么资产（CMDB）
- 知道哪些组件有漏洞（SBOM）
- 评估补丁兼容性
- 有回滚方案
- 无法及时打补丁时有补偿控制
- 记录例外与风险接受

---

## 10. 漏洞生命周期与披露

```text
发现 → 报告 → 验证 → 分配 CVE → 厂商修复 → 披露 → 利用 → 修复
```

- 负责任披露：给厂商合理修复时间
- 漏洞赏金（Bug Bounty）
- 众测
- 0day 交易与风险
- 漏洞利用武器化速度越来越快

---

## 11. 蓝队与漏洞管理

| 蓝队动作 | 说明 |
|---|---|
| 资产梳理 | 知道资产、版本、暴露面 |
| 漏洞情报 | 关注 KEV、厂商公告、在野利用 |
| 检测规则 | 针对高危漏洞的利用特征 |
| 应急响应 | 判断入口、影响范围、是否被利用 |
| 加固 | 补丁、配置、最小权限 |
| 验证 | 修复后复测、监控复发 |
| 度量 | 修复率、平均修复时间、暴露时长 |

### 11.1 漏洞利用的检测线索

- Web 日志中的异常路径/参数（如 Log4j 的 `${jndi:...}`）
- 异常进程/子进程（如 Web 服务拉起 `cmd.exe`）
- 异常外联（漏洞利用后下载载荷）
- 文件落地（Webshell、工具）
- 账号/权限变更
- 日志清除

---

## 12. 实操

1. 在 NVD 查询一个 CVE，阅读 CVSS 向量。
2. 在 CISA KEV 搜索一个漏洞，查看要求修复时间。
3. 用 EPSS 查询同一漏洞的利用概率。
4. 用 OpenVAS 扫描实验靶机（仅授权）。
5. 用 Trivy 扫描一个容器镜像。
6. 为一个漏洞写「检测思路 + 缓解措施」卡片。
7. 模拟一次漏洞优先级排序。

---

## 13. 自测

1. CVE、CWE、CVSS 的区别？
2. CVSS v3.1 的基础指标有哪些？
3. 为什么 CVSS 10 分不一定最优先修复？
4. EPSS 与 KEV 分别解决什么问题？
5. 漏洞管理的完整闭环是什么？
6. 无法及时打补丁时怎么办？
7. 如何检测一个漏洞是否被利用？

---

## 14. 延伸阅读

- [CVE 官网](https://www.cve.org/)
- [NVD](https://nvd.nist.gov/)
- [CWE](https://cwe.mitre.org/)
- [FIRST CVSS](https://www.first.org/cvss/)
- [FIRST EPSS](https://www.first.org/epss/)
- [CISA KEV](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
- [CNVD](https://www.cnvd.org.cn/)
- [CNNVD](https://www.cnnvd.org.cn/)
- [Exploit-DB](https://www.exploit-db.com/)
- [OSV](https://osv.dev/)
- [GitHub Advisory Database](https://github.com/advisories)

---

## 15. 下一步

- [Web 安全与 OWASP Top 10](05-Web安全与OWASP-Top10.md)
- [常见入侵手法与检测](../08-威胁框架与攻防对抗/04-常见入侵手法与检测.md)
- [安全基线加固](../06-主机与终端安全/08-安全基线加固.md)
