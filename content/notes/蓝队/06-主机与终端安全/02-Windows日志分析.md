---
title: "Windows 日志分析"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 2
showTableOfContents: true
---

> 难度：★★★★☆ ｜ 重要度：必学 ｜ 建议学时：20 小时 ｜ 前置：[Windows 安全机制](01-Windows安全机制.md)、[日志分析与日志源](../05-蓝队核心能力/03-日志分析与日志源.md)

## 1. Windows 日志体系

| 日志 | 内容 | 位置 |
|---|---|---|
| Security | 登录、账号、权限、审核 | `%SystemRoot%\System32\winevt\Logs\Security.evtx` |
| System | 系统、服务、驱动 | `System.evtx` |
| Application | 应用事件 | `Application.evtx` |
| Setup | 安装 | `Setup.evtx` |
| ForwardedEvents | 转发 | `ForwardedEvents.evtx` |
| PowerShell | 脚本执行 | `Microsoft-Windows-PowerShell/Operational` |
| Sysmon | 进程/网络/文件/注册表 | `Microsoft-Windows-Sysmon/Operational` |
| TaskScheduler | 计划任务 | `Microsoft-Windows-TaskScheduler/Operational` |
| WMI-Activity | WMI | `Microsoft-Windows-WMI-Activity/Operational` |
| Defender | 杀软事件 | `Microsoft-Windows-Windows Defender/Operational` |

---

## 2. 审核策略

### 2.1 查看

```powershell
auditpol /get /category:*
auditpol /list /subcategory:*
```

### 2.2 推荐开启

```powershell
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Process Creation" /success:enable
auditpol /set /subcategory:"Security System Extension" /success:enable /failure:enable
auditpol /set /subcategory:"Other Object Access Events" /success:enable /failure:enable
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable
auditpol /set /subcategory:"Sensitive Privilege Use" /success:enable /failure:enable
auditpol /set /subcategory:"Audit Policy Change" /success:enable
auditpol /set /subcategory:"Security State Change" /success:enable
```

> 中文系统子类别名称不同，建议用英文系统或 `auditpol /list` 查看本地化名称。对应关系：4697 服务安装 → `Security System Extension`；4698-4702 计划任务 → `Other Object Access Events`。

开启命令行记录：

```text
计算机配置 → 管理模板 → 系统 → 审核进程创建
→ 启用「审核进程创建」并勾选「包含命令行」
```

---

## 3. 关键事件 ID

### 3.1 登录与认证

| ID | 含义 | 关键字段 |
|---|---|---|
| 4624 | 登录成功 | 登录类型、账号、源 IP、工作站 |
| 4625 | 登录失败 | 失败原因、账号、源 IP |
| 4634 | 注销 | 账号、登录类型 |
| 4647 | 用户发起注销 | 账号 |
| 4648 | 显式凭据登录 | 目标账号、目标服务器 |
| 4672 | 授予特殊权限 | 账号、特权列表 |
| 4776 | NTLM 凭据验证 | 账号、工作站 |
| 4768 | 请求 TGT | 账号、加密类型 |
| 4769 | 请求服务票据 | 服务名、加密类型 |
| 4771 | Kerberos 预认证失败 | 账号、失败码 |
| 4740 | 账户锁定 | 账号、调用方 |
| 4778/4779 | 会话重连/断开 | 账号、会话 |

### 3.2 登录类型

| 类型 | 含义 | 场景 |
|---|---|---|
| 2 | 交互式 | 本地键盘登录 |
| 3 | 网络 | 共享/远程访问 |
| 4 | 批处理 | 计划任务 |
| 5 | 服务 | 服务启动 |
| 7 | 解锁 | 屏幕解锁 |
| 8 | 网络明文 | 明文凭据 |
| 9 | 新凭据 | `runas /netonly` |
| 10 | 远程桌面 | RDP |
| 11 | 缓存凭据 | 断网登录 |

### 3.3 账号与组

| ID | 含义 |
|---|---|
| 4720 | 创建用户 |
| 4722 | 启用用户 |
| 4723 | 用户修改密码 |
| 4724 | 重置密码 |
| 4725 | 禁用用户 |
| 4726 | 删除用户 |
| 4728 | 添加到全局组 |
| 4732 | 添加到本地组 |
| 4756 | 添加到通用组 |
| 4738 | 用户属性变更 |
| 4740 | 账户锁定 |
| 4767 | 账户解锁 |

### 3.4 进程与服务

| ID | 含义 | 说明 |
|---|---|---|
| 4688 | 进程创建 | 需审核策略 + 命令行 |
| 4689 | 进程退出 | 状态码 |
| 4697 | 服务安装（Security） | 需审核系统扩展 |
| 7045 | 服务安装（System） | 新服务 |
| 4698 | 计划任务创建 | 任务 XML |
| 4699 | 计划任务删除 | — |
| 4700/4701 | 计划任务启用/禁用 | — |
| 4702 | 计划任务修改 | — |
| 4688 | 进程创建 | 父进程、命令行、用户 |

### 3.5 日志与审核

| ID | 含义 |
|---|---|
| 1102 | 安全日志被清除 |
| 104 | System 日志被清除 |
| 4719 | 审核策略变更 |
| 4902 | 每用户审核策略表已创建 |
| 4904/4905 | 注册/注销安全事件源 |
| 4906 | CrashOnAuditFail 值变更 |
| 4907 | 对象审核设置变更 |

### 3.6 PowerShell

| ID | 含义 |
|---|---|
| 400 | 引擎启动 |
| 403 | 引擎停止 |
| 4103 | 模块日志 |
| 4104 | 脚本块日志 |
| 800 | 管道执行（旧） |
| 600 | 提供程序启动 |

### 3.7 其他

| ID | 含义 |
|---|---|
| 5140 | 网络共享访问 |
| 5145 | 详细文件共享访问 |
| 5156 | Windows 筛选平台允许连接 |
| 5157 | 筛选平台阻止连接 |
| 4656/4658 | 对象句柄请求/关闭 |
| 4663 | 对象访问尝试 |
| 4670 | 对象权限变更 |
| 4696 | 分配主令牌 |
| 4703 | 令牌权限调整 |
| 4611 | 受信任登录进程注册 |

---

## 4. 日志分析工作流

```text
1. 明确问题（登录？进程？服务？任务？）
2. 选择日志与事件 ID
3. 确定时间范围
4. 过滤账号/主机/IP
5. 查看原始事件（Message/XML）
6. 关联其他事件与数据源
7. 形成时间线与结论
```

### 4.1 查看事件

```powershell
Get-WinEvent -LogName Security -MaxEvents 50
Get-WinEvent -FilterHashtable @{LogName="Security"; Id=4624} -MaxEvents 20
Get-WinEvent -FilterHashtable @{LogName="Security"; Id=4625; StartTime=(Get-Date).AddDays(-1)}
```

### 4.2 解析 XML 字段

```powershell
$e = Get-WinEvent -FilterHashtable @{LogName="Security"; Id=4625} -MaxEvents 1
$xml = [xml]$e.ToXml()
$xml.Event.EventData.Data | Select Name, "#text"
```

### 4.3 常用查询

```powershell
# 最近登录
Get-WinEvent -FilterHashtable @{LogName="Security"; Id=4624} -MaxEvents 20 |
  Select TimeCreated, Id, Message

# 登录失败 Top
Get-WinEvent -FilterHashtable @{LogName="Security"; Id=4625} -MaxEvents 1000 |
  ForEach-Object {
    $x = [xml]$_.ToXml()
    [pscustomobject]@{
      User = ($x.Event.EventData.Data | Where-Object Name -eq "TargetUserName")."#text"
      IP = ($x.Event.EventData.Data | Where-Object Name -eq "IpAddress")."#text"
      Time = $_.TimeCreated
    }
  } | Group-Object IP | Sort-Object Count -Descending | Select -First 10

# 服务安装
Get-WinEvent -FilterHashtable @{LogName="System"; Id=7045} -MaxEvents 50

# 计划任务
Get-WinEvent -FilterHashtable @{LogName="Security"; Id=4698} -MaxEvents 50

# 日志清除
Get-WinEvent -FilterHashtable @{LogName="Security"; Id=1102} -MaxEvents 10
```

---

## 5. 典型场景分析

### 5.1 登录异常

关注：

- 4625 大量失败 → 爆破/密码喷洒
- 4625 后 4624 成功 → 爆破成功
- 4624 类型 3/10 异常来源 → 横向/RDP
- 4648 显式凭据 → 凭据滥用
- 4672 非管理员账号 → 提权
- 4768/4769 异常 → Kerberos 攻击
- 非工作时间/异地登录

**时间线示例**：

```text
10:00:00 4625 x 100 from 203.0.113.10 (admin)
10:05:00 4624 type 3 admin from 203.0.113.10
10:06:00 4688 cmd.exe /c whoami
10:07:00 4688 net user hacker P@ss /add
10:07:10 4732 hacker -> Administrators
10:08:00 7045 新服务
```

→ 高危：爆破成功 + 创建后门账号 + 提权 + 持久化。

### 5.2 进程异常

关注：

- 4688 命令行
- 父进程（Office → PowerShell）
- 路径（Temp/Public/ProgramData）
- 签名
- 哈希
- 网络连接

### 5.3 持久化

- 7045 服务
- 4698 计划任务
- 4720/4732 账号
- 注册表 Run（Sysmon 13）
- WMI 订阅（Sysmon 19/20/21）

### 5.4 横向移动

- 4624 类型 3/10
- 4648
- 4672
- 5140/5145
- 7045（PsExec）
- 4768/4769

### 5.5 日志清除

- 1102 Security
- 104 System
- 日志服务停止
- 日志文件删除
- 日志时间缺口

> 日志清除本身是高危信号，需结合其他主机/外部日志判断。

---

## 6. 分析工具

| 工具 | 用途 |
|---|---|
| Event Viewer | 手工查看 |
| PowerShell Get-WinEvent | 查询/导出 |
| EvtxECmd | EVTX 解析导出 CSV |
| Chainsaw | 快速狩猎/规则匹配 |
| Hayabusa | 事件日志威胁狩猎 |
| DeepBlueCLI | 命令行快速分析 |
| Sigma | 规则 |
| SIEM | 集中分析 |
| Velociraptor | 远程采集 |

---

## 7. 日志质量检查

- 审核策略是否开启
- 命令行是否记录
- PowerShell 日志是否开启
- Sysmon 是否部署
- 时间是否同步
- 日志是否集中
- 日志是否可被清除
- 保留期是否足够
- 是否有备份

---

## 8. 常见误区

- 只看 4624/4625，不看 4688/7045
- 不记录命令行
- 不开启 PowerShell 日志
- 不部署 Sysmon
- 不关联多事件
- 不关注父进程
- 不看原始 XML
- 时区错误
- 日志被清除却未发现

---

## 9. 实操

1. 开启审核策略，制造登录失败，查看 4625。
2. 创建用户并加入管理员组，查看 4720/4732。
3. 创建计划任务，查看 4698。
4. 安装一个测试服务，查看 7045。
5. 运行 PowerShell 脚本，查看 4104。
6. 用 Chainsaw/Hayabusa 分析一份 EVTX 样本。
7. 还原一次完整攻击时间线。

> 仅在自己的实验环境操作。

---

## 10. 自测

1. Windows 有哪些主要日志？
2. 4624 登录类型 2/3/10 的区别？
3. 4688 为什么要记录命令行？
4. 7045 与 4697 的区别？
5. 1102 意味着什么？
6. 如何发现爆破成功？
7. 如何发现计划任务持久化？
8. 如何关联登录、进程、服务事件？

---

## 11. 延伸阅读

- [Microsoft：审核事件](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/)
- [Microsoft：4624](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4624)
- [Microsoft：4688](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4688)
- [Ultimate Windows Security：事件 7045](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=7045)
- [Ultimate Windows Security：事件百科](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/)
- [Chainsaw](https://github.com/WithSecureLabs/chainsaw)
- [Hayabusa](https://github.com/Yamato-Security/hayabusa)
- [DeepBlueCLI](https://github.com/sans-blue-team/DeepBlueCLI)
- [Sigma](https://github.com/SigmaHQ/sigma)

---

## 12. 下一步

- [Sysmon 详解](03-Sysmon详解.md)
- [Linux 日志与审计](04-Linux日志与审计.md)
- [权限维持与检测](../08-威胁框架与攻防对抗/05-权限维持与检测.md)
