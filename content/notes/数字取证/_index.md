---
title: "数字取证"
description: "内存取证、磁盘镜像、加密容器、系统痕迹与日志取证的知识整理。"
tags: ["数字取证", "Volatility", "CTF"]
showTableOfContents: true
---

取证的核心是**用证据回答问题**：拿到镜像先做 triage，再围绕题目问题深挖，最后用多个痕迹互相印证。

## 一、内存取证

### 文件格式

- `.vmem`、`.raw`、`.mem`、`.dump`、`.img` 都可能是内存镜像
- `.dmp` 可能是蓝屏转储或进程转储，`lsass.DMP` 常见于凭据提取
- 不确定版本时先 `imageinfo` / `windows.info`，不要直接上插件

### Volatility 2

```bash
vol.py -f mem.dump imageinfo
vol.py -f mem.raw --profile=Win7SP1x64 pslist
vol.py -f mem.raw --profile=Win7SP1x64 pstree
vol.py -f mem.raw --profile=Win7SP1x64 cmdline
vol.py -f mem.raw --profile=Win7SP1x64 netscan
vol.py -f mem.raw --profile=Win7SP1x64 filescan
vol.py -f mem.raw --profile=Win7SP1x64 dumpfiles -Q 0x... --dump-dir=./
vol.py -f mem.raw --profile=Win7SP1x64 hashdump
vol.py -f mem.raw --profile=Win7SP1x64 lsadump
```

其他常用：`malfind` 找注入、`svcscan` 看服务、`hivelist`/`printkey` 看注册表、`screenshot` 看屏幕、`clipboard` 看剪贴板、`iehistory` 看浏览器历史、`mftparser` 生成时间线。

### Volatility 3

```bash
vol -f mem.raw windows.info
vol -f mem.raw windows.pslist
vol -f mem.raw windows.pstree
vol -f mem.raw windows.cmdline
vol -f mem.raw windows.netscan
vol -f mem.raw windows.filescan
vol -f mem.raw windows.dumpfiles --virtaddr 0x...
vol -f mem.raw windows.hashdump
```

Vol3 不需要手动选 profile，但可能需要 symbols；没有网络时可以手动指定 symbol 目录。

### Linux / macOS

- Linux：`linux_banner`、`linux_pslist`、`linux_psaux`、`linux_bash`、`linux_lsof`、`linux_netstat`、`linux_find_file`、`linux_recover_filesystem`
- macOS：Vol2/Vol3 都有对应插件，先确认版本和符号
- 识别不出系统时：先看镜像头部字符串、`strings`、版本信息，再决定工具

### 一些经验

- 一定要看桌面、下载、文档目录里的文件
- 进程列表要结合命令行、父进程、网络连接一起看
- `memdump` 把进程内存 dump 出来后可以再 `strings`
- Vol2 提取失败时换 Vol3 或 R-Studio 再试
- `lsadump`/`hashdump` 拿到哈希后先判断类型，再爆破或传递

## 二、磁盘镜像

### 常见格式

| 格式 | 处理方式 |
| --- | --- |
| raw / dd / img | 直接 `mmls`、`mount` 或 Autopsy |
| E01 | `ewfinfo` 看元数据，`ewfmount` 或 `ewfexport` 转 dd |
| AD1 | FTK Imager 挂载 |
| VMDK | DiskGenius / Imdisk / 7-Zip / FTK |
| VHD | Windows 直接挂载或 DiskGenius |
| VDI / QCOW2 | 对应虚拟机工具或转换后挂载 |

### 挂载与查看

```bash
mmls disk.dd
sudo mount -o ro,loop,offset=$((2048*512)) disk.dd /mnt
fls -r -p disk.dd
icat disk.dd <inode>
```

- 只读挂载，避免破坏证据
- 分区偏移算错会看到空目录
- 多段 E01 先看 `ewfinfo` 元数据，可能直接藏线索
- RAID 0 可用 UFS Explorer 自动重组
- BTRFS/FAT 等特殊文件系统优先用 DiskGenius、WinHex 或专用工具

### 加密容器

- **VeraCrypt**：不同密钥/密钥文件可能挂载出不同隐藏卷；不知道密码可尝试 Passware Kit
- **TrueCrypt / EncFS / GPG / PGP / Cipher**：先找密钥文件、口令、内存镜像中的密钥
- **BitLocker**：有内存镜像时可用 Passware 解密；E01 多段时密钥可能藏在某个小分段里
- **DPAPI**：从注册表和用户目录提取 master key，再解密浏览器/邮件凭据

## 三、系统痕迹

- 注册表：SAM、SYSTEM、SOFTWARE、NTUSER.DAT
- Prefetch、Amcache、Shimcache：程序是否运行过
- 事件日志：登录、进程创建、服务安装、计划任务
- `$MFT`、USN Journal：文件创建/修改/删除
- LNK、Jump List、最近打开文件
- 浏览器历史、下载记录、Cookie、保存的密码
- 回收站 `$I` / `$R`
- NTFS ADS 备用数据流

常见问题：

- 系统名、主机名、用户名
- PowerShell 历史记录
- 用户登录密码、浏览器保存密码
- 定时任务内容
- Docker 容器中的关键文件
- 环境变量、系统产品名、软件版本

## 四、日志、服务器与邮件

- EVTX 用 Event Viewer、EvtxECmd 或 Timeline Explorer 解析
- Web 日志看访问频率、上传、命令执行、异常 UA
- 服务器取证看 `/var/www`、配置、计划任务、SSH 记录、Docker 数据卷
- MySQL 5.7 的 `qp.xb` 可用对应恢复工具还原
- FoxMail 邮件取证看账号配置、邮件正文、附件
- 把所有时间统一时区，再做时间线

## 五、做题流程

1. 确认镜像类型，计算哈希
2. 磁盘镜像先看分区，再只读挂载
3. 看用户目录、桌面、下载、回收站
4. 内存镜像先看进程树、命令行、网络
5. 根据题目问题选择工具，不要全插件乱跑
6. 用至少两种痕迹互相印证
7. 记录证据来源和工具步骤
8. 提取 flag，写清楚结论

## 六、常见坑

- 直接读写挂载，破坏证据
- Vol2 profile 选错，结果全是空
- E01 没先 `ewfmount`
- 忽略时区，时间线全错
- 只看进程列表，不看命令行和网络
- 忘记检查 ADS、回收站、卷影副本
- 大镜像全盘扫描等到天荒地老
- 拿到哈希不判断类型就直接爆破

> 参考：Lunatic BLOG《Misc-数字取证》<https://goodlunatic.github.io/posts/761da51/>
