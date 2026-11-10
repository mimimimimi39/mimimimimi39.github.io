---
title: "MITRE ATT&CK 框架"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 1
showTableOfContents: true
---

> 难度：★★★☆☆ ｜ 重要度：必学 ｜ 建议学时：12 小时 ｜ 前置：[常见攻击类型总览](../04-安全基础知识/03-常见攻击类型总览.md) ｜ 相关：[网络杀伤链](02-网络杀伤链.md)
>
> 学习顺序：阶段 3 先读战术/技术概览，建立地图；阶段 6 再回来做检测映射与覆盖评估。

## 1. ATT&CK 是什么

ATT&CK（Adversarial Tactics, Techniques, and Common Knowledge）是 MITRE 维护的攻击行为知识库，基于真实攻击观察，描述攻击者「做了什么」。

官网：https://attack.mitre.org/

组成：

- **Tactics（战术）**：攻击者的目标（为什么）
- **Techniques（技术）**：达成目标的方法（怎么做）
- **Sub-techniques（子技术）**：更细的方法
- **Procedures（过程）**：具体攻击者的实现
- **Groups/Software/Campaigns**：攻击组织、工具、活动
- **Mitigations**：缓解措施
- **Detections**：检测思路（部分）

---

## 2. 矩阵

| 矩阵 | 范围 |
|---|---|
| Enterprise | Windows、Linux、macOS、云、网络、容器 |
| Mobile | Android、iOS |
| ICS | 工业控制系统 |

企业矩阵是蓝队最常用的。

---

## 3. 企业战术（14 个）

| ID | 战术 | 目标 | 技术示例 |
|---|---|---|---|
| TA0043 | Reconnaissance | 侦察 | 主动扫描、钓鱼信息收集 |
| TA0042 | Resource Development | 资源准备 | 获取基础设施、开发工具 |
| TA0001 | Initial Access | 初始访问 | 钓鱼、漏洞利用、有效账号 |
| TA0002 | Execution | 执行 | 命令行、脚本、计划任务 |
| TA0003 | Persistence | 持久化 | 服务、计划任务、启动项 |
| TA0004 | Privilege Escalation | 权限提升 | 漏洞、令牌、UAC 绕过 |
| TA0005 | Defense Evasion | 防御规避 | 混淆、禁用安全工具、注入 |
| TA0006 | Credential Access | 凭据访问 | LSASS、Kerberoasting、键盘记录 |
| TA0007 | Discovery | 发现 | 主机/网络/账号发现 |
| TA0008 | Lateral Movement | 横向移动 | RDP、SMB、WMI、WinRM |
| TA0009 | Collection | 收集 | 文件、邮件、屏幕截图 |
| TA0011 | Command and Control | 命令与控制 | HTTP、DNS、WebSocket |
| TA0010 | Exfiltration | 数据外传 | C2 通道、云存储、物理介质 |
| TA0040 | Impact | 影响 | 勒索、破坏、DoS |

> 战术顺序不代表攻击一定按顺序发生，攻击者可能跳过或循环。

---

## 4. 常见技术

| 技术 | ID | 说明 | 检测数据源 |
|---|---|---|---|
| Phishing | T1566 | 钓鱼 | 邮件网关、代理、终端 |
| Exploit Public-Facing Application | T1190 | 利用公网应用 | WAF、Web 日志、EDR |
| Command and Scripting Interpreter | T1059 | 命令/脚本 | 4688、Sysmon 1、PowerShell |
| Scheduled Task/Job | T1053 | 计划任务 | 4698、Sysmon 1 |
| Create or Modify System Process | T1543 | 服务/Systemd/Launch Agent 等；驱动加载另看 Sysmon 6 | 7045/4697、Sysmon 13/1；驱动看 Sysmon 6 |
| Boot or Logon Autostart Execution | T1547 | 启动项 | Sysmon 13、Autoruns |
| OS Credential Dumping | T1003 | 凭据窃取 | Sysmon 10、EDR |
| Kerberoasting | T1558.003 | 服务票据 | 4769 |
| Pass the Hash | T1550.002 | 哈希传递 | 4624 类型 3 |
| Remote Services | T1021 | 远程服务 | 4624、网络 |
| Process Injection | T1055 | 进程注入 | Sysmon 8/10、EDR |
| Web Shell | T1505.003 | Webshell | Web 日志、文件、EDR |
| Clear Windows Event Logs | T1070.001 | 清除日志 | 1102 |
| Application Layer Protocol | T1071 | C2 协议 | 代理、流量 |
| DNS Tunneling | T1071.004 | DNS 隧道 | DNS 日志 |
| Exfiltration Over C2 Channel | T1041 | 经 C2 外传 | 流量 |
| Data Encrypted for Impact | T1486 | 勒索加密 | 文件、EDR |

---

## 5. 如何用 ATT&CK

### 5.1 检测映射

把已有规则映射到技术：

```text
规则：Office 拉起 PowerShell
→ T1566 钓鱼 / T1204 用户执行 / T1059.001 PowerShell
```

### 5.2 覆盖评估

- 选择重点战术/技术
- 检查数据源是否具备
- 检查规则是否覆盖
- 标注「已覆盖/部分/未覆盖」
- 优先补高风险缺口

工具：ATT&CK Navigator（https://mitre-attack.github.io/attack-navigator/）。

### 5.3 威胁狩猎

- 选技术 → 设计假设 → 查数据 → 验证 → 产出规则

### 5.4 应急分析

- 把攻击行为映射到技术
- 找下一步可能行为
- 检查对应检测

### 5.5 红蓝对抗

- 红队按技术设计攻击
- 蓝队按技术验证检测
- 紫队评估覆盖

---

## 6. 示例：映射一次入侵

```text
入口：VPN 弱口令（T1078 Valid Accounts / T1110 Brute Force）
执行：PowerShell 下载（T1059.001 / T1105）
持久化：计划任务（T1053.005）
提权：令牌窃取（T1134）
凭据：LSASS（T1003.001）
横向：RDP（T1021.001）
C2：HTTPS（T1071.001）
外传：C2（T1041）
影响：勒索（T1486）
```

对应检测：

- VPN 登录失败/异常
- PowerShell 下载
- 4698 计划任务
- Sysmon 10 LSASS
- 4624 类型 10
- 代理/流量 C2
- 文件加密行为

---

## 7. 数据源

ATT&CK 定义了检测数据源（Data Sources），如：

- Process
- Command
- File
- Network Traffic
- Logon Session
- User Account
- Windows Registry
- Service
- Scheduled Job
- Module
- Driver
- Cloud Service
- Container

> 数据源是检测的基础。没有数据源，规则无从谈起。

---

## 8. 局限

- 不是攻击全集的完备列表
- 技术描述抽象，落地需结合环境
- 不提供具体检测规则
- 版本更新，需持续跟进
- 覆盖评估容易「纸面覆盖」
- 攻击者可能使用未记录技术

---

## 9. 实操

1. 打开 ATT&CK 网站，浏览 14 个战术。
2. 选 5 个技术，写出检测思路与数据源。
3. 用 Navigator 做一张覆盖图。
4. 把一次靶场攻击映射到技术。
5. 把现有 SIEM 规则映射到技术。
6. 找 3 个未覆盖技术，设计检测。
7. 关注一个 APT 组织的 TTP。

---

## 10. 自测

1. ATT&CK 的战术、技术、子技术、过程分别是什么？
2. 企业矩阵有哪些战术？
3. 如何用 ATT&CK 做检测覆盖？
4. 如何把攻击映射到 ATT&CK？
5. ATT&CK 与 Kill Chain 的区别？
6. 数据源为什么重要？
7. ATT&CK 有哪些局限？
8. Navigator 有什么用？

---

## 11. 延伸阅读

- [MITRE ATT&CK](https://attack.mitre.org/)
- [ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/)
- [MITRE D3FEND](https://d3fend.mitre.org/)
- [MITRE CAR](https://car.mitre.org/)
- [Center for Threat-Informed Defense](https://ctid.mitre.org/)
- [Sigma](https://github.com/SigmaHQ/sigma)
- [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team)

---

## 12. 下一步

- [网络杀伤链](02-网络杀伤链.md)
- [红蓝对抗基础](03-红蓝对抗基础.md)
- [常见入侵手法与检测](04-常见入侵手法与检测.md)
