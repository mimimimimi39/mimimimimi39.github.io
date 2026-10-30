---
title: "流量分析"
date: 2026-10-30
draft: false
description: "Wireshark 只是工具，真正要学的是协议、过滤和还原思路。附常用过滤、tshark 命令和常见坑。"
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
- 学习顺序：先补网络基础，再练协议还原，最后做自动化

## 一、学习内容分类

### 1. 网络与协议基础

- 封装关系、TCP 握手与重传
- 常见端口与协议特征
- HTTP 请求/响应、DNS 查询、TLS 握手

重点：能看懂一次请求在各层的形态，知道加密发生在哪一层。

### 2. 抓包与过滤

- pcap / pcapng、snaplen、抓包截断
- capture filter 与 display filter
- Follow Stream、Statistics、Export Objects
- 大文件先用 tshark 过滤

重点：拿到陌生 pcap 能快速判断协议分布和可疑会话。

### 3. 协议还原

- TCP 重组、乱序、重传
- HTTP：Header、Body、chunked、gzip
- DNS：TXT 记录、隧道特征
- FTP/SMTP：明文凭据
- SMB：文件传输
- TLS：SNI、证书、key log
- 提取账号、Cookie、Token

重点：能独立还原完整会话，而不只是看懂单个包。

### 4. 文件与数据提取

- 导出 HTTP/SMB 对象
- 处理编码、压缩、分片
- 从上传、下载、Webshell 流量中提取内容

重点：能把流量里的文件完整还原出来并继续分析。

### 5. 自动化分析

- tshark 按字段批量筛选
- Zeek 生成连接日志
- Python 自定义解析

重点：能用命令行批量筛选，不靠手点。

### 6. 跨方向结合

- DNS/ICMP 隧道与数据外传
- 内存镜像里的网络连接
- 流量中的隐写与加密文件

重点：能把流量和取证、隐写题联系起来。

## 二、常用过滤与命令

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

## 三、做题流程

1. `Statistics → Protocol Hierarchy` 看协议分布
2. `Conversations` 看流量最大的会话
3. 按协议过滤，先找 HTTP、DNS、FTP 这类明文
4. Follow Stream 还原会话
5. Export Objects 导出文件
6. 对可疑数据做解码：URL、Base64、gzip、Hex
7. 建立时间线，把多个会话串起来

## 四、常见坑

- 把 capture filter 当 display filter 用
- 只看单个包，不看完整流
- 忽略 TCP 重组，导致内容缺一段
- HTTP 是 chunked 或 gzip，直接看原始字节像乱码
- TLS 流量硬啃，没先找 key log 或证书
- pcap 太大，全量加载卡死
- 导出文件后不确认文件类型
- 忘记 URL 解码和 Base64 解码
- 只盯一个流，漏掉旁边的 DNS 或 ICMP

## 结语

流量分析的核心不是工具，而是「先缩小范围，再还原内容」。协议熟了、过滤会了、文件能导出来，大部分题就能做。

把上面的内容过一遍，然后去刷题。真正的手感来自反复还原完整会话，而不是看更多教程。

> 本文只讨论 CTF 比赛和授权环境中的流量分析，请勿用于未授权的网络监听或数据提取。
