---
title: "取证"
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
- 学习顺序：先文件系统，再磁盘/内存，最后做时间线综合
- 不要一上来就 Volatility 全插件跑一遍，先想清楚要找什么

## 一、学习内容分类

### 1. 文件与文件系统

- 文件头、真实类型、扩展名
- MBR/GPT、分区偏移
- NTFS：MFT、ADS、USN Journal
- ext4：inode、block、journal
- 删除文件、回收站、未分配空间、slack space

重点：知道文件在磁盘上的位置，以及删除后还能从哪里找回来。

### 2. 磁盘镜像与恢复

```bash
mmls disk.dd
sudo mount -o ro,loop,offset=$((2048*512)) disk.dd /mnt
```

- raw/E01/VMDK 镜像
- 只读挂载、写保护
- `mmls` / `fls` / `icat` / Autopsy
- 文件雕刻、删除恢复、时间线

重点：能独立挂载镜像、找到被删除的文件并说明来源。

### 3. 内存取证

```bash
vol -f memory.raw windows.pslist
vol -f memory.raw windows.cmdline
vol -f memory.raw windows.netscan
vol -f memory.raw windows.filescan
vol -f memory.raw windows.dumpfiles --virtaddr 0x...
```

- 进程树、命令行、网络连接
- 注入与隐藏进程
- 导出内存中的文件
- 提取密码哈希与配置

重点：能从内存镜像还原出可疑行为和落地文件。

### 4. Windows 系统痕迹

- 注册表、Prefetch、Amcache
- 事件日志、服务安装
- LNK、Jump List、最近打开文件
- 浏览器历史与下载记录

重点：能用至少三种痕迹互相印证一个行为。

### 5. 日志与时间线

- Web/SSH/sudo 日志
- 容器与云环境日志
- 时区统一
- Timeline Explorer / plaso
- 磁盘、内存、流量关联

重点：能写出包含时间、主机、用户、行为的完整叙述。

### 6. 证据链与报告

- 哈希校验、只读操作
- 记录工具与步骤
- 区分事实与推断

重点：结论可复现，证据来源清楚。

## 二、工具清单

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

## 三、做题流程

1. 确认镜像类型和大小，先算哈希
2. 磁盘镜像先 `mmls` 看分区，再只读挂载
3. 看用户目录、桌面、下载、回收站
4. 找最近修改的文件、可疑程序和脚本
5. 内存镜像先看进程树、命令行、网络连接
6. 根据题目问题选择深挖方向，不要全插件乱跑
7. 把发现按时间排序，互相印证
8. 提取 flag 后写清楚证据来源

## 四、常见坑

- 直接读写挂载，破坏证据
- 用错分区偏移，挂载后看到的是空目录
- E01 镜像没先 `ewfmount`
- Volatility 2 没选对 profile，结果全是空
- 只看进程列表，不看命令行和网络连接
- 忽略时区，时间线全错
- 看到删除文件就以为要恢复，忘了回收站和卷影副本
- 忘记检查 NTFS 的 ADS（备用数据流）
- 大镜像直接全盘扫描，等到天荒地老

## 结语

取证的本质是「用证据回答问题」。工具只是帮你把数据摆到桌面上，真正值钱的是你知道该找什么、以及多个证据之间能不能对得上。

把磁盘、内存、系统痕迹三条线各练几题，再去打真实镜像，成长会很快。

> 本文只讨论 CTF 比赛和授权环境中的取证技术。分析他人设备前请获得明确授权，并遵守相关法律法规。
