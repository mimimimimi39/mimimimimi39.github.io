---
title: "Wireshark 与抓包分析"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 1
showTableOfContents: true
---

> 难度：★★★★☆ ｜ 重要度：必学 ｜ 建议学时：20 小时 ｜ 前置：[计算机网络基础](../02-计算机与网络基础/04-计算机网络基础.md)、[HTTP 协议详解](../02-计算机与网络基础/05-HTTP协议详解.md)

## 1. 为什么蓝队要学 Wireshark

- 流量是主机日志之外最重要的证据源。
- 可以还原攻击过程、提取文件、定位 C2。
- 主机日志可能被清除，流量镜像/PCAP 可能保留真相。
- 面试高频：过滤器、三次握手、异常流量判断。

---

## 2. 抓包基础

### 2.1 抓包位置

| 位置 | 能看到什么 |
|---|---|
| 主机网卡 | 本机流量 |
| 交换机 SPAN | 指定端口/ VLAN 流量 |
| 网络 TAP | 全流量，无丢包 |
| 防火墙/代理 | 经过设备的流量 |
| 云 VPC Flow | 元数据，非全包 |
| 端点 EDR | 进程关联的网络 |

### 2.2 混杂模式

网卡默认只接收发给自己的帧；混杂模式接收网段内所有帧（受交换机限制）。

### 2.3 抓包过滤（BPF）

在抓包阶段减少数据量：

```text
host 10.0.0.1
net 10.0.0.0/24
port 80
tcp port 443
udp port 53
not port 22
src host 10.0.0.1 and dst port 80
```

### 2.4 显示过滤

在已抓取的数据中筛选，语法更丰富：

```text
ip.addr == 10.0.0.1
ip.src == 10.0.0.1 && ip.dst == 8.8.8.8
tcp.port == 445
tcp.flags.syn == 1 && tcp.flags.ack == 0
http.request.method == "POST"
http.response.code >= 400
dns.qry.name contains "example"
tls.handshake.extensions_server_name == "example.com"
frame contains "password"
http.user_agent contains "curl"
```

> 抓包过滤（BPF）与显示过滤语法不同，不要混用。

---

## 3. 常用显示过滤器

| 目的 | 过滤器 |
|---|---|
| 某 IP 所有流量 | `ip.addr == 10.0.0.1` |
| 某会话 | `ip.addr == 10.0.0.1 && tcp.port == 445` |
| SYN 包 | `tcp.flags.syn == 1 && tcp.flags.ack == 0` |
| RST | `tcp.flags.reset == 1` |
| HTTP GET | `http.request.method == "GET"` |
| HTTP POST | `http.request.method == "POST"` |
| HTTP 错误 | `http.response.code >= 400` |
| DNS 查询 | `dns.flags.response == 0` |
| DNS 失败 | `dns.flags.rcode != 0` |
| TLS SNI | `tls.handshake.extensions_server_name` |
| TLS 证书（≤1.2） | `tls.handshake.type == 11`（TLS 1.3 证书在加密握手中，需密钥日志/解密） |
| 明文凭据 | `http.authorization` |
| 文件传输 | `http.content_type contains "octet-stream"` |
| SMB | `smb or smb2` |
| RDP | `tcp.port == 3389` |
| ICMP | `icmp` |
| ARP | `arp` |

---

## 4. 分析操作

### 4.1 跟踪流

右键 → Follow → TCP/UDP/HTTP Stream。

用途：

- 看完整请求响应
- 还原 Webshell 交互
- 查看下载内容
- 分析 C2 协议

### 4.2 导出对象

File → Export Objects → HTTP/SMB/TFTP。

用途：

- 提取下载文件
- 提取 Webshell 上传内容
- 提取恶意样本

### 4.3 统计

- Statistics → Capture File Properties
- Protocol Hierarchy：协议分布
- Conversations：会话
- Endpoints：端点
- IO Graph：流量趋势
- HTTP → Requests：请求列表

### 4.4 时间显示

View → Time Display Format：

- 绝对时间
- 相对时间（从第一个包）
- 会话时间

> 与主机日志对齐时，注意时区与时间基准。

---

## 5. 常见协议分析

### 5.1 TCP

- 三次握手：SYN → SYN/ACK → ACK
- 四次挥手：FIN/ACK → ACK → FIN/ACK → ACK
- 重传、乱序、RST
- 窗口大小
- 半开连接

**异常**：

- 大量 SYN 无 ACK → 扫描/攻击
- 大量 RST → 拒绝/防火墙
- 重传多 → 网络质量/丢包
- 异常窗口 → 扫描/攻击

### 5.2 HTTP

- 请求行、头、体
- 响应码
- User-Agent
- Cookie
- 上传/下载
- 代理头

**异常**：

- 扫描器 UA
- 异常 URI
- 大量 404
- Webshell POST
- 大文件上传/下载

### 5.3 DNS

- 查询/响应
- 记录类型
- 响应码
- TTL
- 长域名/高熵

**异常**：

- DNS 隧道
- DGA
- 大量 NXDOMAIN
- TXT 异常

### 5.4 TLS

- ClientHello/ServerHello
- SNI
- 证书
- 套件
- 应用数据

**异常**：

- 自签名/异常证书
- SNI 与证书不一致
- 异常 JA3
- 固定心跳

### 5.5 SMB

- 协商、会话、树连接
- 文件读写
- 命名管道
- 横向

**异常**：

- 大量 445 连接
- PsExec 服务
- 异常文件读写
- 命名管道

---

## 6. TLS 解密

### 6.1 浏览器/客户端

设置 `SSLKEYLOGFILE` 环境变量，Wireshark 配置 TLS 密钥日志文件。

```text
Edit → Preferences → Protocols → TLS → (Pre)-Master-Secret log filename
```

仅适用于你有权解密的客户端流量。

### 6.2 服务器私钥

仅适用于 RSA 密钥交换（非前向保密），现代 TLS 多用 ECDHE，无法用私钥解密历史流量。

### 6.3 企业代理

通过代理解密，需安装企业根证书，注意合规。

---

## 7. 典型案例

### 7.1 端口扫描

特征：

- 大量 SYN 到多个端口
- 短时间
- 少量响应/RST
- 来源集中

过滤：

```text
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

### 7.2 暴力破解

特征：

- 多次登录请求
- 固定间隔
- 失败响应
- 少量成功

### 7.3 Webshell

特征：

- POST 到脚本
- 请求体编码/加密
- 响应短小
- 固定 UA
- 访问上传目录

### 7.4 C2 心跳

特征：

- 固定间隔
- 小包
- 加密
- 长连接/短连接
- 异常域名/证书

### 7.5 数据外传

特征：

- 大流量上传
- 非工作时段
- 异常目的地
- 压缩/加密
- 分块传输

### 7.6 DNS 隧道

特征：

- 长域名
- 高熵
- TXT/NULL
- 高频
- 固定解析器

### 7.7 ARP 欺骗

特征：

- 同一 IP 对应多个 MAC
- 大量 ARP 广播
- 网关 MAC 变化

---

## 8. tshark 命令

```bash
# 抓包
tshark -i eth0 -w capture.pcap

# 读取并过滤
tshark -r capture.pcap -Y "http.request"

# 提取字段
tshark -r capture.pcap -T fields -e ip.src -e ip.dst -e http.host -e http.request.uri

# 统计
tshark -r capture.pcap -q -z io,stat,60
tshark -r capture.pcap -q -z endpoints,ip
tshark -r capture.pcap -q -z conv,tcp

# 导出 HTTP 对象
tshark -r capture.pcap --export-objects http,exported
```

---

## 9. 分析流程

```text
1. 明确问题（扫描？C2？外传？）
2. 确定时间范围与关键 IP
3. 看协议分布与会话
4. 过滤可疑流量
5. 跟踪流/导出对象
6. 关联主机日志与情报
7. 形成时间线与结论
8. 保存证据
```

---

## 10. 实操

1. 抓取一次 HTTP 访问，展开各层。
2. 抓取 HTTPS 握手，查看 SNI 与证书。
3. 用 Follow Stream 还原一次登录。
4. 导出 HTTP 下载文件。
5. 分析 Malware Traffic Analysis 的公开 PCAP。
6. 识别扫描、爆破、C2、DNS 隧道。
7. 用 tshark 提取字段并统计。
8. 配置 SSLKEYLOGFILE 解密自己的流量。

---

## 11. 常见误区

- 抓包过滤与显示过滤混用
- 不看时间基准
- 只看单包不看流
- 忽略加密流量元数据
- 不导出对象
- 不关联主机日志
- 抓包位置不对（看不到目标流量）
- 抓包丢包（性能/缓冲区）

---

## 12. 自测

1. 抓包过滤与显示过滤的区别？
2. 如何过滤 SYN 包？
3. 如何跟踪一条 TCP 流？
4. 如何导出 HTTP 文件？
5. 如何识别端口扫描？
6. 如何识别 C2 心跳？
7. TLS 解密有哪些方式？
8. 如何分析 DNS 隧道？

---

## 13. 延伸阅读

- [Wireshark 用户指南](https://www.wireshark.org/docs/wsug_html_chunked/)
- [Wireshark 显示过滤器](https://wiki.wireshark.org/DisplayFilters)
- [Wireshark 样本抓包](https://wiki.wireshark.org/SampleCaptures)
- [Malware Traffic Analysis](https://www.malware-traffic-analysis.net/)
- [tshark 手册](https://www.wireshark.org/docs/man-pages/tshark.html)
- [Zeek 文档](https://docs.zeek.org/)
- [Arkime](https://arkime.com/)

---

## 14. 下一步

- [网络流量异常检测](02-网络流量异常检测.md)
- [NIDS 与 Suricata/Snort](03-NIDS与Suricata-Snort.md)
- [防火墙、WAF 与代理](04-防火墙WAF与代理.md)
