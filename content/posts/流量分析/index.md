---
title: "流量分析：从抓包到还原，一份入门路线"
date: 2026-10-30
draft: false
description: "Wireshark 只是工具，真正要学的是协议、过滤和还原思路。附分阶段路线、常用过滤、tshark 命令和常见坑。"
tags: ["CTF", "流量分析", "Wireshark", "Misc"]
categories: ["学习笔记"]
showTableOfContents: true
---

流量分析题看起来很吓人：一个几十兆的 pcap，几万个包，堆在 Wireshark 里密密麻麻。但做多了会发现，大部分题并不难，只是对协议不熟、不知道该看哪里。

这篇是我自己的流量分析入门路线，重点不是背过滤语法，而是建立「先看什么、再怎么缩小范围」的直觉。

## 先说结论

- 核心三件事：**认协议、会过滤、能还原**
- 必学协议：TCP、HTTP、DNS、TLS，其次 FTP、SMTP、SMB、ICMP
- 必会操作：Follow Stream、Export Objects、Statistics、display filter
- 工具够用就行：Wireshark + tshark + CyberChef
- 时间预算：一到两周过一遍，之后靠题补

## 一、知识地图

### 1. 网络基础

不需要学到能写协议栈，但下面这些必须清楚：

- 以太网帧、IP、TCP/UDP 的封装关系
- 三次握手、四次挥手，以及 seq/ack 在还原中的作用
- 常见端口：80/443、53、21、22、25、445、3306、6379
- HTTP 请求/响应结构、状态码、Header、Cookie
- DNS 查询与响应，A/AAAA/CNAME/TXT 记录
- TLS 握手流程，以及为什么加密流量不能直接看内容

### 2. 抓包文件

- `pcap` 和 `pcapng` 的区别，pcapng 可以带注释和多接口
- 抓包过滤（capture filter）和显示过滤（display filter）不是一回事
- 抓包时的 snaplen 可能导致包被截断
- 大文件不要全量加载，先用 tshark 过滤再导入

### 3. 协议分析

按出现频率排：

1. HTTP / HTTPS
2. DNS
3. TCP / UDP
4. FTP、SMTP、POP3
5. SMB
6. ICMP
7. WebSocket、MQTT 等

### 4. 还原与取证

- Follow TCP/HTTP Stream：把一次会话拼起来
- Export Objects：把 HTTP/SMB 传输的文件导出来
- 重组分片、处理乱序和重传
- 从流量里提取账号、密码、Token、Cookie
- TLS 有 key log 时可以解密
- DNS/ICMP 隧道、HTTP 外传数据

### 5. 自动化

- tshark 按字段提取，适合批量筛选
- Zeek 生成连接日志、HTTP 日志、DNS 日志
- Python + scapy 处理特殊协议或自定义解析

## 二、分阶段路线

### 阶段 0：网络基础（2~3 天）

- 理解封装、握手、端口、HTTP 报文
- 用浏览器开发者工具看真实请求
- 用 `curl -v` 观察请求和响应

**掌握标准**：能手动描述一次 HTTP 请求从 DNS 到 TCP 再到响应的过程。

### 阶段 1：Wireshark 基本操作（2 天）

必须熟练：

- 选择网卡、开始/停止抓包
- display filter 输入与补全
- Follow Stream
- Statistics → Protocol Hierarchy / Conversations / Endpoints
- File → Export Objects
- 标记、着色、查找（Ctrl+F）

**掌握标准**：拿到一个陌生 pcap，5 分钟内能说出里面主要有哪些协议、有没有明文 HTTP。

### 阶段 2：常见协议（3~5 天）

- HTTP：方法、URI、Header、Body、chunked、gzip
- DNS：查询名、响应码、TXT 记录、隧道特征
- TLS：ClientHello/ServerHello、SNI、证书
- FTP/SMTP：明文账号密码
- SMB：文件传输、NTLM 认证

**掌握标准**：看到协议名就知道该看哪些字段，能独立还原一次 HTTP 会话。

### 阶段 3：文件与数据还原（3 天）

- 导出 HTTP 对象，确认文件类型
- 处理 gzip、chunked、URL 编码、Base64
- 从图片、压缩包、文档里继续挖
- 关注上传点、下载点、 Webshell 流量

**掌握标准**：能从流量里完整还原一个被上传或下载的文件，并继续分析。

### 阶段 4：自动化与脚本（2 天）

```bash
tshark -r capture.pcap -Y 'http.request' -T fields -e http.host -e http.request.uri
tshark -r capture.pcap -Y 'dns' -T fields -e dns.qry.name
tshark -r capture.pcap --export-objects http,exported
```

**掌握标准**：能用 tshark 一行命令筛出想要的字段，而不是靠手点。

### 阶段 5：和取证、隐写结合（持续）

- DNS 外传、ICMP 隧道
- 内存镜像里的网络连接
- 流量中的隐写图片、加密压缩包
- 日志与流量互相印证

## 三、常用过滤与命令

### display filter

```text
http
http.request.method == "POST"
http.response.code >= 400
http contains "flag"
http.host contains "example.com"
dns.qry.name contains "flag"
tcp.stream eq 3
ip.addr == 192.168.1.10
tcp.port == 8080
tls.handshake.type == 1
frame contains "password"
```

### tshark

```bash
# 看前 20 个包
tshark -r capture.pcap -c 20

# 只看 HTTP 请求
tshark -r capture.pcap -Y 'http.request'

# 提取 URL
tshark -r capture.pcap -Y 'http.request' -T fields -e http.host -e http.request.uri

# 导出 HTTP 对象
tshark -r capture.pcap --export-objects http,out

# 统计协议分布
tshark -r capture.pcap -q -z io,phs
```

## 四、做题流程

1. `Statistics → Protocol Hierarchy` 看协议分布
2. `Conversations` 看流量最大的会话
3. 按协议过滤，先找 HTTP、DNS、FTP 这类明文
4. Follow Stream 还原会话
5. Export Objects 导出文件
6. 对可疑数据做解码：URL、Base64、gzip、Hex
7. 建立时间线，把多个会话串起来
8. 卡住 30 分钟就看提示或跳题

## 五、常见坑

- 把 capture filter 当 display filter 用
- 只看单个包，不看完整流
- 忽略 TCP 重组，导致内容缺一段
- HTTP 是 chunked 或 gzip，直接看原始字节像乱码
- TLS 流量硬啃，没先找 key log 或证书
- pcap 太大，全量加载卡死
- 导出文件后不确认文件类型
- 忘记 URL 解码和 Base64 解码
- 只盯一个流，漏掉旁边的 DNS 或 ICMP

## 六、练哪里

- **CTFHub**：流量分析技能树，适合按知识点刷
- **BUUCTF**：Misc 流量题量大
- **攻防世界**：Misc 新手区
- **picoCTF**：英文 forensics 题，适合练查资料
- **Malware-Traffic-Analysis.net**：真实恶意流量练习
- **Wireshark 官方 sample captures**：熟悉协议

练习方法：

- 每类协议先做 5 题，做完关掉 writeup 重做
- 每道题记录「协议 → 过滤 → 还原方式」
- 不要只收藏过滤器，要能解释为什么这么过滤

## 七、什么时候该止损

- 一道题 30 分钟没有方向，就看提示或跳题
- 不要为了一个包逐字节手翻几万条记录
- 不要死磕加密流量，先确认有没有密钥
- 流量分析是取证的一部分，别把它学成 Wireshark 快捷键背诵

## 结语

流量分析的核心不是工具，而是「先缩小范围，再还原内容」。协议熟了、过滤会了、文件能导出来，大部分题就能做。

把这份路线走一遍，然后去刷题。真正的手感来自反复还原完整会话，而不是看更多教程。

> 本文只讨论 CTF 比赛和授权环境中的流量分析，请勿用于未授权的网络监听或数据提取。
