---
title: "Sysmon 详解"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 3
showTableOfContents: true
---

> 难度：★★★★☆ ｜ 重要度：必学 ｜ 建议学时：10 小时 ｜ 前置：[Windows 日志分析](02-Windows日志分析.md)、[Windows 安全机制](01-Windows安全机制.md)

## 1. Sysmon 是什么

Sysmon（System Monitor）是 Microsoft Sysinternals 提供的 Windows 系统监控工具，作为服务和驱动运行，把进程、网络、文件、注册表、DNS 等事件写入 Windows 事件日志。

优势：

- 比默认审核更细（命令行、哈希、父进程、网络、DNS）
- 事件结构化，便于 SIEM 分析
- 可通过配置精细控制
- 社区规则丰富

> Sysmon 不是 EDR，但它是 Windows 主机检测的重要数据源。

---

## 2. 安装与配置

### 2.1 下载

https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

### 2.2 安装

```powershell
Sysmon64.exe -accepteula -i sysmonconfig.xml
```

### 2.3 更新配置

```powershell
Sysmon64.exe -c sysmonconfig.xml
```

### 2.4 查看状态

```powershell
Get-Service Sysmon64
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10
```

### 2.5 卸载

```powershell
Sysmon64.exe -u
```

> 生产环境通过 GPO/SCCM/Intune 部署，配置纳入版本管理。

---

## 3. 事件 ID 全表

| ID | 事件 | 关键价值 |
|---|---|---|
| 1 | 进程创建 | 命令行、父进程、哈希、签名 |
| 2 | 文件创建时间修改 | 时间戳篡改 |
| 3 | 网络连接 | 外连、横向、C2 |
| 4 | Sysmon 服务状态 | 服务被停止/修改 |
| 5 | 进程终止 | 进程生命周期 |
| 6 | 驱动加载 | 恶意驱动、BYOVD |
| 7 | 镜像加载 | DLL 侧加载、注入 |
| 8 | CreateRemoteThread | 进程注入 |
| 9 | RawAccessRead | 磁盘原始读取 |
| 10 | ProcessAccess | LSASS 访问、注入 |
| 11 | FileCreate | 文件落地 |
| 12 | 注册表对象创建/删除 | 持久化 |
| 13 | 注册表值设置 | Run/Services 修改 |
| 14 | 注册表键/值重命名 | 规避检测 |
| 15 | FileCreateStreamHash | ADS 备用数据流 |
| 16 | Sysmon 配置变更 | 配置被篡改 |
| 17 | 命名管道创建 | 进程间通信、C2 |
| 18 | 命名管道连接 | 横向、C2 |
| 19 | WMI 事件过滤器 | WMI 持久化 |
| 20 | WMI 事件消费者 | WMI 持久化 |
| 21 | WMI 消费者到过滤器 | WMI 持久化 |
| 22 | DNS 查询 | 恶意域名、隧道、DGA |
| 23 | 文件删除（归档） | 删除痕迹 |
| 24 | 剪贴板变更 | 数据窃取 |
| 25 | 进程篡改 | 进程替换/挖空 |
| 26 | 文件删除检测 | 删除行为 |
| 27 | 文件阻止执行 | 防护 |
| 28 | 文件阻止粉碎 | 防护 |
| 29 | 文件可执行检测 | 创建可执行文件 |

> 不同 Sysmon 版本支持的事件略有差异，以官方文档为准。

---

## 4. 核心事件详解

### 4.1 事件 1：进程创建

关键字段：

- `Image`：可执行文件路径
- `CommandLine`：命令行
- `ParentImage`：父进程路径
- `ParentCommandLine`：父进程命令行
- `User`：用户
- `Hashes`：哈希
- `Signed`/`Signature`：签名
- `ProcessId`/`ParentProcessId`
- `CurrentDirectory`
- `IntegrityLevel`

检测思路：

- Office/浏览器 → PowerShell/cmd/mshta
- Web 进程 → cmd/powershell
- 路径在 Temp/Public/ProgramData
- 无签名/签名异常
- 命令行含编码/下载/隐藏
- 异常父进程

### 4.2 事件 3：网络连接

关键字段：

- `Image`：发起进程
- `SourceIp`/`SourcePort`
- `DestinationIp`/`DestinationHostname`/`DestinationPort`
- `Protocol`
- `Initiated`
- `User`

检测思路：

- 非浏览器进程外连
- 非常见端口
- 固定心跳
- 内网横向
- 恶意 IP/域名

### 4.3 事件 7：镜像加载

- DLL 侧加载
- 未签名 DLL
- 从 Temp/用户目录加载
- 已知恶意 DLL 名称

### 4.4 事件 8：CreateRemoteThread

- 进程注入
- 源/目标进程
- 目标进程为 lsass、explorer 等

### 4.5 事件 10：ProcessAccess

- LSASS 访问（`TargetImage=lsass.exe`）
- `GrantedAccess` 权限
- 源进程是否安全软件/系统进程

> Mimikatz 等工具会打开 LSASS 并请求高权限。

### 4.6 事件 11：FileCreate

- Webshell 落地
- 工具下载
- 临时目录可执行文件
- 文件名/扩展名异常

### 4.7 事件 12/13/14：注册表

- Run/RunOnce
- Services
- Winlogon
- IFEO
- COM
- 浏览器扩展

### 4.8 事件 15：ADS

- 文件备用数据流隐藏数据
- `Zone.Identifier` 是正常下载标记，需区分

### 4.9 事件 17/18：命名管道

- Cobalt Strike SMB Beacon
- PsExec
- 横向工具
- 异常管道名

### 4.10 事件 19/20/21：WMI

- WMI 持久化
- 事件过滤器/消费者
- 异常 WMI 订阅

### 4.11 事件 22：DNS

- 恶意域名
- DGA
- DNS 隧道
- 进程与域名关联

### 4.12 事件 25：进程篡改

- 进程挖空（Process Hollowing）
- 进程替换
- 镜像篡改

---

## 5. 配置要点

### 5.1 配置结构

```xml
<Sysmon schemaversion="4.90">
  <HashAlgorithms>SHA256</HashAlgorithms>
  <CheckRevocation>true</CheckRevocation>
  <EventFiltering>
    <ProcessCreate onmatch="exclude">
      <!-- 排除已知噪音 -->
    </ProcessCreate>
    <NetworkConnect onmatch="include">
      <!-- 包含重点 -->
    </NetworkConnect>
  </EventFiltering>
</Sysmon>
```

### 5.2 社区配置

- SwiftOnSecurity：https://github.com/SwiftOnSecurity/sysmon-config
- Olaf Hartong Modular：https://github.com/olafhartong/sysmon-modular
- Ion-Storm：https://github.com/ion-storm/sysmon-config

> 直接套用前要评估噪音与业务兼容性，按环境调优。

### 5.3 建议

- 开启命令行与哈希
- 开启网络与 DNS
- 开启进程访问（LSASS）
- 开启镜像加载（注意噪音）
- 开启 WMI/管道
- 排除已知高噪音但无害的进程
- 配置版本化管理
- 监控 Sysmon 服务/配置变更

---

## 6. 检测用例

### 6.1 Office 拉起 PowerShell

```text
EventID 1
ParentImage endswith \winword.exe / excel.exe / outlook.exe
Image endswith \powershell.exe / cmd.exe / mshta.exe
```

### 6.2 LSASS 访问

```text
EventID 10
TargetImage endswith \lsass.exe
SourceImage not in 安全软件白名单
```

### 6.3 服务持久化

```text
EventID 13
TargetObject contains \Services\<name>\ImagePath
```

### 6.4 计划任务

```text
EventID 1
Image endswith \schtasks.exe
CommandLine contains /create
```

或结合 4698。

### 6.5 可疑 DNS

```text
EventID 22
QueryName length > 100 或 entropy 高
Image 非浏览器
```

### 6.6 进程注入

```text
EventID 8 或 10
SourceImage 可疑
TargetImage lsass.exe / explorer.exe
GrantedAccess 含 PROCESS_VM_WRITE / CREATE_THREAD
```

---

## 7. 查询示例

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 20

Get-WinEvent -FilterHashtable @{
  LogName="Microsoft-Windows-Sysmon/Operational"; Id=1
} -MaxEvents 50
```

Splunk：

```spl
index=sysmon EventCode=1
| search ParentImage="*winword.exe" OR ParentImage="*excel.exe"
| table _time host User ParentImage Image CommandLine
```

---

## 8. 局限与注意事项

- Sysmon 需要正确配置，默认配置覆盖有限
- 高噪音事件（7、10、22）需调优
- 攻击者可尝试卸载/停止/篡改配置
- 日志可能被清除
- 不提供内存取证
- 不能替代 EDR 的响应能力
- 需集中采集与保留

---

## 9. 实操

1. 安装 Sysmon 并加载社区配置。
2. 创建进程，查看事件 1。
3. 访问网站，查看事件 3 与 22。
4. 创建文件，查看事件 11。
5. 修改 Run 注册表，查看事件 13。
6. 创建计划任务/服务，查看事件 1/13/7045。
7. 用 Splunk/Wazuh 接入 Sysmon 日志并写规则。
8. 尝试停止 Sysmon，观察事件 4。

---

## 10. 自测

1. Sysmon 与 Windows 安全日志的区别？
2. 事件 1、3、11、22 分别是什么？
3. 如何检测 LSASS 访问？
4. 如何检测进程注入？
5. 如何检测注册表持久化？
6. 命名管道事件有什么价值？
7. Sysmon 配置如何管理？
8. Sysmon 有哪些局限？

---

## 11. 延伸阅读

- [Sysmon 官方文档](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [SwiftOnSecurity 配置](https://github.com/SwiftOnSecurity/sysmon-config)
- [Olaf Hartong 配置](https://github.com/olafhartong/sysmon-modular)
- [Sysmon 事件参考](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon#events)
- [Sigma](https://github.com/SigmaHQ/sigma)
- [MITRE ATT&CK](https://attack.mitre.org/)
- [TrustedSec：Sysmon](https://www.trustedsec.com/)

---

## 12. 下一步

- [Linux 日志与审计](04-Linux日志与审计.md)
- [EDR 与 HIDS](05-EDR与HIDS.md)
- [权限维持与检测](../08-威胁框架与攻防对抗/05-权限维持与检测.md)
