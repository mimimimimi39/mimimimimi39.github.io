---
title: "流量分析"
description: "从协议分层到 Wireshark/tshark、特殊流量、Webshell 与认证协议的流量分析知识整理。"
tags: ["流量分析", "Wireshark", "CTF"]
showTableOfContents: true
---

流量分析的核心是固定下手顺序：**先统计、再分类、再追踪、再还原，最后才考虑解密**。工具只是帮你把数据摆出来，真正决定效率的是知不知道先看哪里。

学习路线见 [文章 / 流量分析](/posts/流量分析/)。

## 一、先分清几个词

- **证书**：TLS/SSL 证书，证明「这个服务是谁」，里面带公钥和 CA 签名
- **凭据**：能证明身份的东西，包括用户名密码、Cookie、Token、JWT、Kerberos 票据、API Key
- **Cookie**：服务器发给浏览器的一小段状态，浏览器之后自动带回
- **Session / Token**：Session 是服务器记住的登录状态，Token 是证明这段状态的值
- **公钥 / 私钥**：公钥可公开，私钥自己保管；TLS 里用它们完成身份验证和密钥协商
- **握手**：通信前先商量规则，TCP 握手建连接，TLS 握手建加密

## 二、协议分层与价值

| 层 | 常见协议 | CTF 里通常看什么 |
| --- | --- | --- |
| 链路层 | Ethernet、ARP、802.11 | 谁和谁通信、网关、无线握手 |
| 网络层 | IPv4/IPv6、ICMP、IPsec | 地址、分片、TTL、隧道、Echo 数据 |
| 传输层 | TCP、UDP、QUIC | 连接、重传、流重组、端口 |
| 应用层 | HTTP、DNS、SMB、FTP、SMTP、Kerberos | 请求内容、文件、凭据、认证 |

优先级口诀：**先应用层明文，再认证和文件，再加密突破口，最后啃底层噪声。**

| 协议 | 价值 | 第一步 |
| --- | --- | --- |
| HTTP / HTTP2 / WebSocket | 很高 | 请求行、参数、Cookie、POST body、导出对象 |
| DNS | 很高 | 查询名、TXT、子域名、隧道 |
| SMB / FTP | 很高 | 文件名、共享目录、账号密码、导出文件 |
| SMTP / IMAP | 很高 | 邮件正文、附件、验证码 |
| Kerberos / NTLMSSP | 高 | 票据、challenge/response、keytab/hash |
| TLS / QUIC | 本体难，突破后高 | key log、私钥、证书、SNI |
| ICMP | 中 | Echo 数据区、隧道 |
| ARP / DHCP / mDNS | 环境信息 | 主机关系、网关、设备发现 |

## 三、做题流程

1. `Statistics → Protocol Hierarchy` 看协议占比
2. `Conversations / Endpoints` 看谁在通信、谁流量最大
3. 先筛高价值协议：HTTP、DNS、SMB、FTP、SMTP、WebSocket
4. Follow Stream 看完整会话
5. Export Objects 导出传输文件
6. 对可疑数据做 URL/Base64/gzip/Hex 解码
7. 有加密先找 key log、私钥、口令、内存线索
8. 没有密钥就做元数据分析：域名、证书、时间、长度、方向
9. 建立时间线，把多个会话串起来

## 四、Wireshark 与 tshark

显示过滤器（display filter）是打开 pcap 后用的，捕获过滤器（capture filter）是抓包前用的，两者语法不同。

```text
http
http.request.method == "POST"
http.response.code >= 400
http.host contains "example.com"
dns.qry.name contains "flag"
tcp.stream eq 3
tcp.len > 0
ip.addr == 192.168.1.10
tcp.flags.syn == 1 and tcp.flags.ack == 0
frame contains "flag"
tcp contains 66:6c:61:67
tls.handshake.type == 1
```

```bash
tshark -r capture.pcap -c 20
tshark -r capture.pcap -Y 'http.request'
tshark -r capture.pcap -Y 'http.request' -T fields -e http.host -e http.request.uri
tshark -r capture.pcap --export-objects http,out
tshark -r capture.pcap -q -z io,phs
```

常用操作：

- Follow TCP / HTTP / TLS / WebSocket Stream
- File → Export Objects → HTTP / SMB
- Statistics → Protocol Hierarchy / Conversations / Endpoints
- 右键 → Apply as Filter / Decode As
- 大文件先用 tshark 过滤，再导入 Wireshark

## 五、特殊流量

### USB HID

键盘流量把按键码转成字符，鼠标流量看按键和坐标，数位板看压力/坐标，Xbox 手柄看按键序列。

```bash
tshark -r usb.pcapng -T fields -e usb.capdata > usb.txt
```

注意高版本 Wireshark 可能默认解析 USB-HID，需要先禁用 USB-HID 协议再提取原始 `usb.capdata`。键盘还要处理 Shift、Delete、Caps Lock。

### 无线与蓝牙

- 无线：先找 SSID/BSSID，有握手和密码时用 `aircrack-ng` 爆破，再用 `airdecap-ng` 解密
- 蓝牙：OBEX 负责文件传输，ATT 是属性协议，SMP 负责配对；重点看文件传输和配对过程

### SQL 注入流量

看 HTTP 请求参数中的 `union select`、`sleep`、`information_schema`、布尔条件；结合响应长度和状态码判断注入结果。

### Webshell 流量

- **菜刀**：请求体常带 base64/URL 编码的 PHP 代码
- **哥斯拉**：`pass` 参数、AES/XOR/base64 多种模式，先看请求头和密钥
- **冰蝎**：AES + base64，密钥常来自 `pass` 或内置默认值
- **蚁剑**：参数编码、`chr` 拼接、base64 等，注意请求体和响应体
- 通用方法：找固定参数名、异常 UA、重复 POST、响应里带命令输出

### Cobalt Strike

- 先看 Beacon 流量：心跳间隔、URI、Cookie、 metadata
- 可能用 RSA/AES 加密，找到密钥对或 Beacon 配置才能还原
- 关注 `checksum8`、malleable C2 配置、DNS/HTTP 混合通道

## 六、认证与工控

- **NTLM**：看 `NTLMSSP` 的 Type1/2/3、challenge/response；有 NT hash 或口令时可进一步利用
- **Kerberos**：看 AS-REQ/TGS-REQ/AP-REQ、SPN、票据；有 keytab/口令/hash 时可解密
- **AD 域**：把 DNS、Kerberos、LDAP、SMB、DCE/RPC 串起来看，不要只盯一个协议
- **Modbus / S7comm / DNP3 / IEC 60870**：重点看功能码、寄存器地址、写入操作和异常值
- **RTP / 邮件 / ADS-B / OICQ**：先确认协议，再找文件、正文、坐标或账号线索

## 七、加密流量

- TLS 解密通常需要 `SSLKEYLOGFILE`、RSA 私钥或 PSK
- 没有密钥时看 SNI、证书、ALPN、JA3、连接时长、数据方向
- QUIC/HTTP3 把 TLS 握在 UDP 上，思路和 TLS 类似
- IPsec/ESP、WireGuard、802.11 都需要对应的密钥材料才能解密

## 八、常见坑

- 把捕获过滤器当显示过滤器用
- 只看单个包，不看完整流
- 忘记 TCP 重组，正文只看到一半
- HTTP chunked/gzip 直接看原始字节像乱码
- 死磕 TLS 密文，不先找 key log
- 忽略 DNS、ICMP、端口号里的隐蔽信道
- pcap 太大全量加载卡死
- 只看一个流，漏掉旁边的认证和文件传输

> 参考：Lunatic BLOG《Misc-流量分析》<https://goodlunatic.github.io/posts/5422d65/>
