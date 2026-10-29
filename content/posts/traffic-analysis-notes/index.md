---
title: "CTF 流量分析入门笔记"
date: 2026-10-25
description: "Wireshark 常用过滤与流量题的基本思路。"
tags: ["CTF", "流量分析", "取证"]
categories: ["学习笔记"]
---

流量分析题通常给一个 `.pcap` / `.pcapng`，要你从里面找出 flag、被外传的文件或者攻击者的操作痕迹。

## 常用过滤

```text
http.request.method == "POST"
http contains "flag"
tcp.stream eq 3
ip.addr == 192.168.1.10
dns.qry.name contains "example"
```

## 导出对象

Wireshark 菜单：`File -> Export Objects -> HTTP`，先把所有传输的文件拉出来，再用 `file` 命令确认类型。

## 命令行一把梭

```bash
tshark -r capture.pcap -Y 'http.request' -T fields -e http.host -e http.request.uri
```

## 小技巧

- 先看 `Statistics -> Protocol Hierarchy`，判断题目类型
- 关注异常端口和非标准协议
- TLS 流量先去找密钥日志或证书
- 图片、压缩包可能藏在 HTTP body 里

> 做流量题最重要的是先建立时间线，再逐步缩小范围。

多练几题就有手感了喵～
