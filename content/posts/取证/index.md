---
title: "取证：从镜像到证据链，一份入门路线"
date: 2026-10-30
draft: false
description: "磁盘、内存、系统痕迹三条主线，附工具清单、Volatility 常用命令、做题流程和常见坑。"
tags: ["CTF", "取证", "Misc", "电子取证"]
categories: ["学习笔记"]
showTableOfContents: true
---

取证题的核心不是「找一个工具跑一下」，而是**在大量正常数据里找到那一条异常痕迹**。它比隐写稳定，比流量更杂，既考工具熟练度，也考对操作系统和文件系统的理解。

这篇是我自己的取证入门路线，按磁盘、内存、系统痕迹三条线展开。

## 先说结论

- 取证的主线：**文件系统 → 磁盘镜像 → 内存镜像 → 系统痕迹**
- 必备工具：Sleuth Kit、Autopsy、Volatility、Registry Explorer、strings、binwalk
- 最重要的习惯：只读挂载、先算哈希、先做 triage
- 时间预算：两周左右打基础，之后靠真实镜像练
- 不要一上来就 Volatility 全插件跑一遍，先想清楚要找什么

## 一、知识地图

### 1. 文件与文件系统

- 文件头、扩展名、真实类型
- MBR/GPT 分区表，分区偏移
- NTFS：MFT、属性、ADS、时间戳、USN Journal
- ext4：inode、block、journal
- 删除文件、回收站、未分配空间、slack space
- 文件雕刻（carving）与文件修复

### 2. 磁盘镜像

- 镜像格式：raw/dd、E01、VMDK、VHD
- 只读挂载与写保护
- 分区定位、挂载偏移
- 文件列表、按 inode 提取、时间线
- 哈希校验与证据链

### 3. 内存镜像

- 进程列表、父子关系、命令行
- 网络连接、监听端口
- 注入代码、隐藏进程
- 从内存导出文件、提取密码哈希
- 内存中的字符串与配置

### 4. Windows 系统痕迹

- 注册表：SAM、SYSTEM、SOFTWARE、NTUSER.DAT
- Prefetch、Amcache、Shimcache
- 事件日志：登录、进程创建、服务安装
- $MFT、USN Journal、LNK、Jump List
- 浏览器历史、下载记录、Cookie
- 回收站 `$I` / `$R`

### 5. 日志与网络

- Web 访问日志、错误日志
- SSH 登录日志、sudo 记录
- 流量包与主机痕迹互相印证
- 容器、云环境日志（进阶）

## 二、分阶段路线

### 阶段 0：文件与基础工具（2~3 天）

先练「拿到一个未知文件，怎么判断它是什么」：

```bash
file target
binwalk target
exiftool target
strings -n 6 target
xxd target | head
foremost -i target -o out
```

**掌握标准**：拿到任意文件，能判断真实类型、有没有附加数据、能不能雕刻出内容。

### 阶段 1：磁盘镜像（3~5 天）

- 用 `mmls` 看分区，用 `fls` 列文件
- 用 `icat` 按 inode 提取文件
- 只读挂载：

```bash
mmls disk.dd
sudo mount -o ro,loop,offset=$((2048*512)) disk.dd /mnt
```

- 熟悉 Autopsy 的界面和报告导出
- 练习删除文件恢复、回收站、未分配空间

**掌握标准**：能独立挂载镜像，找到被删除的文件，并说明它原本在哪个目录。

### 阶段 2：内存镜像（3~5 天）

Volatility 3 常用命令：

```bash
vol -f memory.raw windows.info
vol -f memory.raw windows.pslist
vol -f memory.raw windows.pstree
vol -f memory.raw windows.cmdline
vol -f memory.raw windows.netscan
vol -f memory.raw windows.filescan
vol -f memory.raw windows.dumpfiles --virtaddr 0x...
vol -f memory.raw windows.hashdump
```

Volatility 2 还要先 `imageinfo` 选 profile，再跑 `pslist`、`netscan`、`cmdscan`、`filescan`、`dumpfiles`。

**掌握标准**：能从内存镜像里找出可疑进程、它的命令行、网络连接和落地文件。

### 阶段 3：系统痕迹（3~5 天）

- 用 Registry Explorer / RegRipper 看注册表
- 用 EvtxECmd 解析事件日志
- 看 Prefetch 判断程序是否运行过
- 看 LNK、Jump List、最近打开文件
- 用 `$MFT` 和 USN Journal 还原文件操作

**掌握标准**：给定一个「某人做过什么」的问题，能从至少三个不同痕迹互相印证。

### 阶段 4：时间线与综合（2~3 天）

- 把文件时间、日志时间、内存时间统一到同一时区
- 用 Timeline Explorer 或 plaso 生成时间线
- 把磁盘、内存、流量三份证据串起来

**掌握标准**：能写出一段包含时间、主机、用户、行为的完整叙述，而不是只交一个 flag。

## 三、工具清单

| 用途 | 工具 |
| --- | --- |
| 镜像与挂载 | FTK Imager、`ewfmount`、`mmls`、`mount` |
| 磁盘分析 | Autopsy、Sleuth Kit（`fls`、`icat`、`blkls`） |
| 文件恢复 | TestDisk、PhotoRec、foremost、scalpel |
| 内存分析 | Volatility 2/3、MemProcFS |
| 注册表 | Registry Explorer、RegRipper |
| 事件日志 | EvtxECmd、Event Viewer |
| 时间线 | Timeline Explorer、plaso |
| 基础排查 | `file`、`strings`、`exiftool`、`binwalk`、`xxd` |

## 四、做题流程

1. 确认镜像类型和大小，先算哈希
2. 磁盘镜像先 `mmls` 看分区，再只读挂载
3. 看用户目录、桌面、下载、回收站
4. 找最近修改的文件、可疑程序和脚本
5. 内存镜像先看进程树、命令行、网络连接
6. 根据题目问题选择深挖方向，不要全插件乱跑
7. 把发现按时间排序，互相印证
8. 提取 flag 后写清楚证据来源

## 五、常见坑

- 直接读写挂载，破坏证据
- 用错分区偏移，挂载后看到的是空目录
- E01 镜像没先 `ewfmount`
- Volatility 2 没选对 profile，结果全是空
- 只看进程列表，不看命令行和网络连接
- 忽略时区，时间线全错
- 看到删除文件就以为要恢复，忘了回收站和卷影副本
- 忘记检查 NTFS 的 ADS（备用数据流）
- 大镜像直接全盘扫描，等到天荒地老

## 六、练哪里

- **CTFHub**：取证技能树，适合按知识点刷
- **BUUCTF**：Misc 取证题量大
- **攻防世界**：Misc 新手区，经典题多
- **MemLabs**：内存取证专项练习
- **CyberDefenders**：偏真实事件的蓝队练习
- **picoCTF**：英文 forensics 入门题

练习方法：

- 同一道题至少用两种工具验证结论
- 每道题记录「问题 → 证据 → 工具 → 结论」
- 不要只记命令，要理解每条命令在找什么

## 七、什么时候该止损

- 30 分钟没有方向，就看提示或跳题
- 不要为了一个插件参数折腾一晚上
- 不要把所有工具都跑一遍再开始思考
- 取证是长期积累，别指望刷十道题就精通

## 结语

取证的本质是「用证据回答问题」。工具只是帮你把数据摆到桌面上，真正值钱的是你知道该找什么、以及多个证据之间能不能对得上。

把磁盘、内存、系统痕迹三条线各练几题，再去打真实镜像，成长会很快。

> 本文只讨论 CTF 比赛和授权环境中的取证技术。分析他人设备前请获得明确授权，并遵守相关法律法规。
