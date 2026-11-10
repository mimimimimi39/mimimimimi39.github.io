---
title: "EDR 与 HIDS"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 5
showTableOfContents: true
---

> 难度：★★★★☆ ｜ 重要度：必学 ｜ 建议学时：12 小时 ｜ 前置：[Windows 安全机制](01-Windows安全机制.md)、[Linux 日志与审计](04-Linux日志与审计.md)

## 1. EDR 是什么

EDR（Endpoint Detection and Response）是终端检测与响应平台，持续采集终端行为，检测威胁并支持响应。

与杀毒软件（AV）的区别：

| 维度 | AV | EDR |
|---|---|---|
| 检测 | 特征为主 | 行为、IOA、ML、情报 |
| 数据 | 文件扫描 | 进程、网络、文件、注册表、内存 |
| 响应 | 隔离/删除 | 隔离、终止、隔离文件、远程调查 |
| 重点 | 已知恶意 | 未知/无文件/横向 |

---

## 2. 架构

```text
终端 Agent（用户态/内核驱动）
  → 本地检测与缓存
  → 云端/管理平台
  → 控制台/API
  → SIEM/SOAR
```

组件：

- Agent：采集、检测、响应
- 内核驱动/回调：进程、文件、网络、注册表
- 用户态 Hook：API 监控
- ETW/AMSI：Windows 事件与脚本扫描
- 云端：情报、沙箱、ML、关联
- 控制台：告警、调查、响应

---

## 3. 检测技术

| 技术 | 说明 |
|---|---|
| 签名 | 哈希、YARA、特征 |
| 行为 | 进程树、命令行、文件/注册表 |
| IOA | 攻击行为指标 |
| 内存扫描 | 注入、无文件 |
| ML | 异常模型 |
| 情报 | IP/域名/哈希 |
| 沙箱 | 动态行为 |
| 关联 | 多事件/多主机 |

### 3.1 Windows 数据源

- 进程创建（含命令行）
- 网络连接
- 文件创建/修改
- 注册表
- 模块加载
- 进程访问/注入
- 脚本（AMSI/PowerShell）
- ETW
- 登录

### 3.2 Linux 数据源

- 进程执行
- 文件完整性
- 网络连接
- 登录
- 内核模块
- 容器
- auditd

---

## 4. 响应能力

- 隔离主机（网络隔离）
- 终止进程
- 隔离/删除文件
- 禁止运行
- 远程 shell/调查
- 采集文件/内存
- 回滚（部分产品）
- 下发扫描
- 策略调整

> 响应动作要谨慎，避免影响业务与破坏证据。

---

## 5. 告警分析

### 5.1 看什么

- 告警类型与严重级别
- 主机、用户、时间
- 进程树与命令行
- 文件哈希/签名/路径
- 网络连接
- 持久化
- 关联其他主机

### 5.2 常见告警

| 告警 | 可能含义 | 下一步 |
|---|---|---|
| 可疑进程 | 木马/工具 | 看命令行、父进程、网络 |
| LSASS 访问 | 凭据窃取 | 看源进程、权限 |
| 进程注入 | 无文件/规避 | 看源/目标、内存 |
| 服务创建 | 持久化 | 看路径、账号 |
| 计划任务 | 持久化 | 看命令、触发器 |
| 异常外连 | C2 | 看进程、域名、频率 |
| 勒索行为 | 加密 | 立即隔离 |
| 挖矿 | 资源滥用 | 看进程、矿池 |
| EDR 篡改 | 攻击者关闭防护 | 高优先级 |

---

## 6. 绕过概念（防御视角）

攻击者可能：

- **Unhooking**：恢复被 Hook 的 API
- **Direct Syscalls**：绕过用户态 Hook
- **BYOVD**：利用易受攻击驱动关闭 EDR
- **LOLBAS**：滥用系统二进制
- **进程注入/挖空**
- **AMSI 绕过**
- **ETW 绕过**
- **驱动/服务篡改**
- **安全模式**
- **卸载/停止服务**

蓝队应对：

- 监控 EDR 服务/驱动/配置变更
- 监控 BYOVD 驱动加载
- 监控 AMSI/ETW 相关注册表
- 行为检测而非仅 API Hook
- 多层防御（EDR + Sysmon + 网络）
- 定期测试绕过

---

## 7. HIDS

HIDS（Host-based Intrusion Detection System）基于主机采集与检测。

| 工具 | 特点 |
|---|---|
| Wazuh | 开源，HIDS + SIEM，规则丰富 |
| OSSEC | 老牌 HIDS |
| AIDE | 文件完整性 |
| Tripwire | 商业 FIM |
| osquery | SQL 查询系统状态 |
| Velociraptor | 终端采集与狩猎 |
| Falco | 容器运行时安全 |
| Auditd | Linux 审计 |

### 7.1 文件完整性监控（FIM）

监控：

- 关键系统文件
- Web 目录
- 配置文件
- 二进制
- 权限/属主

检测：

- 未授权修改
- Webshell 落地
- 配置篡改
- Rootkit

---

## 8. 部署与调优

- 覆盖所有关键终端/服务器
- 策略按资产分组
- 排除已知业务噪音
- 与 SIEM 集成
- 告警分级
- 定期测试检测能力
- 监控 Agent 在线率
- 监控 Agent 被卸载/停止
- 版本与规则更新

---

## 9. 局限

- 依赖 Agent 正常运行
- 内核对抗（BYOVD/漏洞）
- 加密/混淆
- 高噪音导致疲劳
- 资源占用
- 云/容器覆盖不足
- 不能替代网络与身份检测

---

## 10. 实操

1. 在虚拟机安装 Wazuh Agent，接入日志。
2. 用 Sysmon/EDR 观察进程注入测试（本地）。
3. 尝试停止 Agent，观察告警。
4. 配置 FIM 监控 Web 目录。
5. 分析一条 EDR 告警并写处置建议。
6. 用 osquery 查询进程/网络/用户。
7. 用 Velociraptor 采集终端信息。

> 仅在自己的实验环境操作。

---

## 11. 自测

1. EDR 与 AV 的区别？
2. EDR 的检测技术有哪些？
3. EDR 的响应能力有哪些？
4. 常见 EDR 绕过思路？
5. 如何检测 EDR 被篡改？
6. HIDS 与 EDR 的关系？
7. FIM 能发现什么？
8. EDR 有哪些局限？

---

## 12. 延伸阅读

- [Wazuh 文档](https://documentation.wazuh.com/)
- [OSSEC](https://www.ossec.net/)
- [osquery](https://osquery.io/)
- [Velociraptor](https://docs.velociraptor.app/)
- [Falco](https://falco.org/)
- [AIDE](https://aide.github.io/)
- [MITRE ATT&CK：Defense Evasion](https://attack.mitre.org/tactics/TA0005/)
- [Microsoft Defender 文档](https://learn.microsoft.com/en-us/defender-endpoint/)

---

## 13. 下一步

- [恶意软件基础](06-恶意软件基础.md)
- [样本分析与病毒查杀](07-样本分析与病毒查杀.md)
- [安全基线加固](08-安全基线加固.md)
