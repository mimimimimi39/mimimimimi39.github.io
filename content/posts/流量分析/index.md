---
title: "流量分析"
date: 2026-10-30
draft: false
description: "流量分析的入门路线：该学什么、按什么顺序学，以及怎么把知识用进题目。"
tags: ["CTF", "流量分析", "Wireshark", "Misc"]
categories: ["学习笔记"]
showTableOfContents: true
---

流量分析题看起来很吓人：一个几十兆的 pcap，几万个包，堆在 Wireshark 里密密麻麻。但做多了会发现，大部分题并不难，只是对协议不熟、不知道该看哪里。

这篇只讲**学习路线**。过滤语法、命令、协议速查和特殊流量细节放在知识库里：

> 速查：[知识库 / 流量分析](/notes/流量分析/)

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

## 二、学习顺序

1. 先补网络分层、TCP/UDP、HTTP、DNS、TLS 这些最低限度的概念
2. 把 Wireshark 的统计、过滤、Follow Stream、Export Objects 练熟
3. 按协议刷题：HTTP → DNS → FTP/SMB → TLS
4. 再补特殊流量：USB、无线、蓝牙、Webshell、认证协议
5. 最后写脚本，把重复操作自动化

具体过滤器和命令见 [知识库 / 流量分析](/notes/流量分析/)。

## 结语

流量分析的核心不是工具，而是「先缩小范围，再还原内容」。协议熟了、过滤会了、文件能导出来，大部分题就能做。

把上面的路线走一遍，然后去刷题。真正的手感来自反复还原完整会话，而不是看更多教程。

> 本文只讨论 CTF 比赛和授权环境中的流量分析，请勿用于未授权的网络监听或数据提取。
