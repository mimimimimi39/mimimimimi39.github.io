---
title: "Windows 安全机制"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 1
showTableOfContents: true
---

> 难度：★★★☆☆ ｜ 重要度：必学 ｜ 建议学时：12 小时 ｜ 前置：[Windows 基础](../02-计算机与网络基础/03-Windows基础.md)、[网络安全基本概念](../04-安全基础知识/01-网络安全基本概念.md)

## 1. 为什么蓝队要懂 Windows 安全机制

- 判断一个行为是否异常，必须知道「正常应该怎样」。
- 攻击者的提权、持久化、凭据窃取都针对特定机制。
- 日志字段（SID、登录类型、令牌、完整性级别）需要机制知识才能解释。

---

## 2. 账号与 SID

### 2.1 账号类型

| 类型 | 说明 |
|---|---|
| 本地账号 | 存储在 SAM，仅本机有效 |
| 域账号 | 存储在 AD，域内通用 |
| 服务账号 | 服务运行身份 |
| 计算机账号 | 域内计算机身份 |
| 托管服务账号（gMSA） | 自动管理密码 |

### 2.2 SID

安全标识符，唯一标识安全主体。

```text
S-1-5-21-<域>-<RID>
```

常见 SID：

| SID | 含义 |
|---|---|
| S-1-5-18 | SYSTEM |
| S-1-5-19 | LOCAL SERVICE |
| S-1-5-20 | NETWORK SERVICE |
| S-1-5-32-544 | Administrators |
| S-1-5-32-545 | Users |
| S-1-5-32-546 | Guests |
| S-1-5-32-551 | Backup Operators |
| S-1-5-11 | Authenticated Users |
| S-1-5-7 | Anonymous |

> 日志中出现异常 SID、未知 SID、RID 500（内置 Administrator）要关注。

---

## 3. 访问令牌与完整性级别

### 3.1 令牌（Token）

用户登录后获得访问令牌，包含：

- 用户 SID
- 组 SID
- 权限
- 完整性级别
- 会话 ID

进程以令牌身份运行。

### 3.2 完整性级别

| 级别 | 说明 |
|---|---|
| System | 系统 |
| High | 管理员提权后 |
| Medium | 普通用户 |
| Low | 浏览器沙箱/受限 |
| Untrusted | 最低 |

UAC 提权本质是从 Medium 令牌切换到 High 令牌。

### 3.3 攻击视角

- 令牌窃取（Token Impersonation）
- 令牌伪造
- 提权到 SYSTEM
- 利用服务/计划任务以高权限运行

**检测**：异常令牌、`whoami /priv` 特权、`SeDebugPrivilege`、`SeImpersonatePrivilege` 滥用。

---

## 4. UAC

### 4.1 机制

- 管理员登录后默认使用 Medium 令牌
- 执行特权操作时提示同意/凭据
- 提权后获得 High 令牌

### 4.2 常见绕过

- `fodhelper.exe`、`computerdefaults.exe`
- `sdclt.exe`
- COM 接口劫持
- 注册表劫持
- 计划任务

**检测**：

- Sysmon 1 中这些系统程序异常启动
- 父进程异常
- 注册表修改（HKCU\Software\Classes）
- 命令行含 `cmd.exe`/`powershell.exe`

---

## 5. ACL 与权限

### 5.1 访问控制列表

- DACL：谁可以访问
- SACL：审计谁访问
- ACE：单条规则（允许/拒绝）

### 5.2 常见权限

| 权限 | 风险 |
|---|---|
| Full Control | 最高 |
| Modify | 修改 |
| Write | 写入 |
| Read | 读取 |
| Execute | 执行 |
| Take Ownership | 取得所有权 |
| SeBackupPrivilege | 备份，可读任意文件 |
| SeRestorePrivilege | 还原 |
| SeDebugPrivilege | 调试任意进程 |
| SeImpersonatePrivilege | 模拟其他用户 |

### 5.3 弱权限风险

- 服务目录可写 → DLL 劫持/替换
- 计划任务脚本可写 → 提权
- 注册表键可写 → 持久化/劫持
- 共享目录权限过宽 → 横向

**检测**：文件/目录 ACL 变更、异常写操作、服务路径权限。

---

## 6. 审核策略与日志

### 6.1 高级审核策略

| 类别 | 关键子类 |
|---|---|
| 账户登录 | 凭据验证、Kerberos 身份验证服务、Kerberos 服务票证操作 |
| 登录/注销 | 登录、注销、账户锁定、特殊登录 |
| 账户管理 | 用户/计算机/组账户管理 |
| 详细跟踪 | 进程创建、进程终止、DPAPI、RPC |
| 对象访问 | 文件系统、注册表、句柄、SAM |
| 策略更改 | 审核策略、授权策略、MPSSVC |
| 特权使用 | 敏感权限、非敏感权限 |
| 系统 | 安全状态更改、安全系统扩展、系统完整性、IPsec |

> 日志清除（1102/104）是事件，不属于可单独开启的审核子类别，建议单独监控。

### 6.2 必须开启的审核

- 登录成功/失败
- 账户管理
- 进程创建（含命令行）
- 服务/计划任务
- 特权使用
- 日志清除
- PowerShell 脚本块

### 6.3 日志

- Security
- System
- Application
- PowerShell
- Sysmon
- Windows Defender
- 任务计划
- WMI

---

## 7. 域与 Active Directory

### 7.1 核心组件

| 组件 | 作用 |
|---|---|
| DC | 域控制器，认证与目录 |
| AD DS | 目录服务 |
| DNS | 域内定位 |
| Kerberos | 认证协议 |
| LDAP | 目录查询 |
| SMB | 文件共享 |
| GPO | 组策略 |
| OU | 组织单位 |
| Trust | 域信任 |

### 7.2 Kerberos 认证

```text
AS-REQ/AS-REP：获取 TGT
TGS-REQ/TGS-REP：获取服务票据
AP-REQ：访问服务
```

事件：

- 4768 TGT
- 4769 服务票据
- 4771 预认证失败
- 4776 NTLM

### 7.3 常见攻击

| 攻击 | 说明 | 检测 |
|---|---|---|
| 密码喷洒 | 少量密码尝试多账号 | 4625 分散 |
| Kerberoasting | 请求服务票据离线破解 | 4769 RC4、异常 SPN |
| AS-REP Roasting | 无预认证账号 | 4768 无预认证 |
| DCSync | 模拟 DC 复制 | 4662、DRSUAPI |
| Golden Ticket | 伪造 TGT | 4768/4769 异常 |
| Silver Ticket | 伪造服务票据 | 服务日志异常 |
| Pass-the-Hash | 使用 NTLM 哈希 | 4624 类型 3 |
| 委派滥用 | 利用委派 | 4769、服务行为 |
| Zerologon | Netlogon 漏洞 | 异常 Netlogon |
| PrintNightmare | 打印服务漏洞 | 驱动加载 |

### 7.4 域加固

- 分层管理员模型（Tier 0/1/2）
- 域管只用于 DC
- LAPS 管理本地管理员密码
- 保护 LSASS（RunAsPPL）
- Credential Guard
- 限制 NTLM
- 监控 Kerberos 异常
- 定期审计特权组
- 禁用不必要委派
- 打补丁

---

## 8. 服务、驱动与签名

### 8.1 服务

- 运行在特定账号
- LocalSystem 权限最高
- 服务可执行文件路径与权限是关键
- 服务失败恢复可执行程序

### 8.2 驱动

- 内核态
- 驱动签名强制
- 易受攻击驱动可被滥用（BYOVD）

### 8.3 检测

- 7045 服务安装
- 4697（Security）
- Sysmon 6 驱动加载
- 签名验证
- 异常路径/无签名

---

## 9. 缓解机制

| 机制 | 作用 |
|---|---|
| DEP | 数据执行保护 |
| ASLR | 地址空间随机化 |
| CFG | 控制流保护 |
| SEHOP | 异常处理保护 |
| Credential Guard | 保护凭据 |
| LSA Protection | 保护 LSASS |
| Device Guard/WDAC | 应用控制 |
| AppLocker | 应用白名单 |
| AMSI | 脚本/内存扫描接口 |
| ETW | 事件跟踪 |
| Windows Defender | 反恶意软件 |

> 攻击者会尝试绕过或关闭这些机制，蓝队要监控相关注册表/服务/进程变更。

---

## 10. 攻击面地图

| 面 | 例子 | 检测数据源 |
|---|---|---|
| 账号 | 弱口令、Kerberoasting | 4624/4625/4768/4769 |
| 进程 | 注入、无文件 | Sysmon 1/8/10 |
| 服务 | 持久化、提权 | 7045/4697 |
| 计划任务 | 持久化 | 4698/4702 |
| 注册表 | 持久化、劫持 | Sysmon 12/13/14 |
| 网络 | 横向、C2 | Sysmon 3、防火墙 |
| 文件 | 落地、替换 | Sysmon 11、FIM |
| 域 | 票据、委派 | DC 日志、EDR |
| 浏览器 | 钓鱼、扩展 | 代理、EDR |
| 远程管理 | RDP/WinRM/WMI | 登录、网络 |

---

## 11. 实操

1. 用 `whoami /all` 查看当前令牌与特权。
2. 用 `Get-LocalGroupMember -Group "Administrators"` 查看管理员。
3. 创建一个本地用户并加入管理员组，观察 4720/4732。
4. 开启审核策略，制造登录失败，观察 4625。
5. 用 Process Explorer 查看一个进程的令牌与完整性级别。
6. 用 `auditpol /get /category:*` 查看审核策略。

---

## 12. 自测

1. SID、令牌、完整性级别的关系？
2. UAC 提权的本质？
3. DACL 与 SACL 的区别？
4. 哪些特权最危险？
5. 必须开启哪些审核策略？
6. Kerberos 认证流程与关键事件？
7. Kerberoasting 与 AS-REP Roasting 的区别？
8. 常见 Windows 缓解机制有哪些？

---

## 13. 延伸阅读

- [Microsoft：Windows 安全文档](https://learn.microsoft.com/en-us/windows/security/)
- [Microsoft：审核策略](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/advanced-security-audit-policy-settings)
- [Microsoft：AD DS](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/active-directory-domain-services)
- [MITRE ATT&CK：Windows](https://attack.mitre.org/matrices/enterprise/windows/)
- [ADSecurity](https://adsecurity.org/)
- [Microsoft：Credential Guard](https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/)

---

## 14. 下一步

- [Windows 日志分析](02-Windows日志分析.md)
- [Sysmon 详解](03-Sysmon详解.md)
- [横向移动与检测](../08-威胁框架与攻防对抗/06-横向移动与检测.md)
