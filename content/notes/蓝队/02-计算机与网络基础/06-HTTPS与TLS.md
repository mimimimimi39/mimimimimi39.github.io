---
title: "HTTPS 与 TLS"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 6
showTableOfContents: true
---

> 难度：★★★☆☆ ｜ 重要度：重要 ｜ 建议学时：8 小时 ｜ 前置：[HTTP 协议详解](05-HTTP协议详解.md) ｜ 建议同步：[密码学基础](../04-安全基础知识/02-密码学基础.md)

## 1. 为什么蓝队要学 TLS

- 越来越多攻击流量走 HTTPS，传统明文检测失效。
- 需要理解证书、SNI、JA3/JA3S，才能做加密流量分析。
- 企业常部署 TLS 解密设备，涉及隐私、合规与性能。
- 证书异常、自签名、过期、域名不匹配都是检测线索。

---

## 2. HTTPS 是什么

HTTPS = HTTP + TLS。

TLS 提供：

- **机密性**：加密应用数据
- **完整性**：防篡改
- **认证**：通过证书验证服务器身份（通常是单向认证，也可双向）

> SSL 是 TLS 的前身，SSL 2.0/3.0 已废弃，口语中的「SSL 证书」实际是 TLS 证书。

---

## 3. TLS 握手

### 3.1 TLS 1.2（简化）

```text
客户端                                     服务器
  ClientHello（支持的版本、套件、随机数、SNI）
  ---------------------------------------->
                                            ServerHello（选定版本、套件、随机数）
                                             Certificate（服务器证书链）
                                             ServerKeyExchange（视套件）
                                             ServerHelloDone
  <----------------------------------------
  ClientKeyExchange
  ChangeCipherSpec
  Finished
  ---------------------------------------->
                                            ChangeCipherSpec
                                            Finished
  <----------------------------------------
  应用数据（加密）
```

特点：

- 通常 2-RTT 建立连接
- 支持 RSA 密钥交换（已不推荐）与 ECDHE（前向保密）

### 3.2 TLS 1.3（简化）

```text
客户端                                     服务器
  ClientHello + KeyShare
  ---------------------------------------->
                                            ServerHello + KeyShare
                                            {EncryptedExtensions}
                                            {Certificate}
                                            {CertificateVerify}
                                            {Finished}
  <----------------------------------------
  {Finished}
  ---------------------------------------->
  应用数据（加密）
```

特点：

- 1-RTT（会话恢复可 0-RTT）
- 只保留安全套件，强制前向保密
- 握手大部分加密，传统设备更难检测
- 0-RTT 存在重放风险

### 3.3 前向保密（PFS）

即使服务器私钥泄露，历史会话也无法解密。ECDHE 提供 PFS。

---

## 4. 证书与 PKI

### 4.1 X.509 证书关键字段

| 字段 | 说明 |
|---|---|
| Subject | 证书主体（域名/组织） |
| Issuer | 颁发者（CA） |
| Subject Alternative Name (SAN) | 可用域名列表 |
| Validity | 生效/过期时间 |
| Public Key | 公钥 |
| Signature | CA 签名 |
| Serial Number | 序列号 |
| Key Usage / EKU | 用途限制 |
| CRL Distribution Points | 吊销列表 |
| OCSP | 在线状态查询 |

### 4.2 证书链

```text
根 CA（受信任，预置在系统/浏览器）
  └─ 中间 CA
       └─ 服务器证书（example.com）
```

验证内容：

- 签名链是否可信
- 域名是否匹配（SAN）
- 是否在有效期内
- 是否被吊销（CRL/OCSP）
- 用途是否正确

### 4.3 证书类型

| 类型 | 验证内容 | 用途 |
|---|---|---|
| DV | 域名所有权 | 普通网站 |
| OV | 组织信息 | 企业网站 |
| EV | 严格组织验证 | 高信任场景 |
| 自签名 | 无第三方 CA | 内部测试、攻击者常用 |
| 通配符 | `*.example.com` | 多子域 |
| 多域名 | SAN 多个域名 | 多站点 |

### 4.4 常见证书问题

- 自签名证书
- 过期证书
- 域名不匹配
- 证书链不完整
- 使用弱签名算法（SHA-1、MD5）
- 私钥泄露
- 证书与流量行为不一致（如 SNI 与证书域名不符）

---

## 5. SNI、ALPN 与 ECH

### 5.1 SNI

TLS ClientHello 中的 `server_name` 扩展，告诉服务器要访问哪个域名。

- 明文传输（TLS 1.3 中仍通常明文）
- 可用于流量分类、阻断、检测
- 与 DNS 查询、证书域名可交叉验证

### 5.2 ALPN

应用层协议协商，如 `h2`、`http/1.1`、`h3`。

### 5.3 ECH（Encrypted Client Hello）

加密 ClientHello，隐藏 SNI。部署尚不广泛，但会削弱基于 SNI 的检测。

---

## 6. 常见 TLS 攻击与风险

| 攻击/风险 | 说明 | 检测/防御 |
|---|---|---|
| 中间人（MITM） | 劫持流量、伪造证书 | 证书校验、HSTS、证书固定 |
| 降级攻击 | 强制使用旧版本 | 禁用 TLS 1.0/1.1、SSL |
| 心脏出血（Heartbleed） | OpenSSL 漏洞读取内存 | 补丁、检测异常心跳 |
| 证书伪造 | 攻击者自签证书 | 终端信任链、证书告警 |
| 证书过期 | 业务中断 | 证书监控 |
| 弱套件 | RC4、3DES、CBC 模式 | 配置安全套件 |
| TLS 剥离 | 将 HTTPS 降为 HTTP | HSTS、预加载列表 |
| 0-RTT 重放 | TLS 1.3 早期数据重放 | 限制 0-RTT 使用 |
| 加密 C2 | 恶意流量走 TLS | JA3/JA3S、证书、行为分析 |

> 历史漏洞如 BEAST、POODLE、FREAK、Logjam 主要影响旧版本/旧套件，现代配置应禁用。

---

## 7. 加密流量分析

### 7.1 可见信息

即使无法解密，仍可观察：

- 源/目的 IP、端口
- SNI
- 证书（Subject、Issuer、SAN、有效期）
- TLS 版本、套件
- 握手长度、记录长度、时间间隔
- 上下行字节数、包大小分布
- JA3/JA3S 指纹（JARM 需主动探测，不能直接从 PCAP 提取）
- DNS 查询
- 连接频率与持续时间

### 7.2 JA3 / JA3S

JA3 将 ClientHello 中的版本、套件、扩展、椭圆曲线、曲线格式拼接后哈希，形成客户端指纹。

- 相同客户端（浏览器/工具）通常有稳定 JA3
- 恶意工具（Cobalt Strike、Metasploit）可能有已知 JA3
- 可被修改/混淆，需结合其他特征

JA3S 是服务器端指纹。

### 7.3 JARM

JARM 主动探测服务器 TLS 配置，生成指纹，用于识别 C2 服务器与恶意基础设施。

### 7.4 证书与域名情报

- 新注册域名
- 自签名证书
- 证书 SAN 与 SNI 不一致
- 证书颁发者异常
- 证书有效期极短
- 与已知恶意基础设施匹配

### 7.5 行为特征

- 固定心跳间隔（C2 beacon）
- 小包频繁上下行
- 非工作时段大流量
- 长连接、低频、低音量
- 目的 IP 集中或快速变化
- 使用非常见端口
- 上传远大于下载（外传）

---

## 8. 企业 TLS 解密

### 8.1 方式

- 反向代理终结 TLS（WAF/负载均衡）
- 正向代理 MITM（安装企业根证书）
- 终端代理/EDR 解密
- 网络设备解密（性能开销大）

### 8.2 风险与合规

- 隐私与员工知情
- 金融、医疗等敏感数据的合规要求
- 证书固定（pinning）会导致解密失败
- 解密设备自身漏洞
- 性能与单点故障

### 8.3 蓝队建议

- 明确解密范围与豁免名单
- 保护解密设备与私钥
- 监控解密失败与证书异常
- 记录访问日志并限制权限

---

## 9. 实操

### 9.1 查看证书

```bash
openssl s_client -connect example.com:443 -servername example.com
openssl x509 -in cert.pem -text -noout
curl -vI https://example.com
```

浏览器：点击地址栏锁图标 → 查看证书 → 证书路径、有效期、SAN。

### 9.2 抓包观察

用 Wireshark 过滤：

```text
tls.handshake.type == 1          # ClientHello
tls.handshake.type == 2          # ServerHello
tls.handshake.extensions_server_name
tls.record.content_type == 23    # Application Data
```

观察：

- SNI
- 支持的 TLS 版本与套件
- 证书域名与有效期
- 应用数据方向与大小

### 9.3 测试 TLS 配置

```bash
nmap --script ssl-enum-ciphers -p 443 example.com
testssl.sh https://example.com
```

> 仅对自有或授权目标测试。

---

## 10. 蓝队检测清单

- [ ] 证书过期/自签名/域名不匹配告警
- [ ] SNI 与 DNS/证书域名不一致
- [ ] 已知恶意 JA3/JA3S 指纹；JARM 需主动探测
- [ ] 固定间隔心跳
- [ ] 非工作时段大流量上传
- [ ] 非常见端口上的 TLS
- [ ] 新注册域名/低信誉域名
- [ ] 证书有效期异常短
- [ ] TLS 版本/套件异常
- [ ] 解密失败突增

---

## 11. 自测

1. HTTPS 提供哪三个安全属性？
2. TLS 1.2 与 1.3 握手的主要区别？
3. 什么是 SNI？为什么它对蓝队重要？
4. 证书链如何验证？
5. JA3 与 JA3S 分别是什么？
6. 加密流量下还能用哪些特征做检测？
7. 企业 TLS 解密有哪些风险？

---

## 12. 延伸阅读

- [Cloudflare：What is TLS?](https://www.cloudflare.com/learning/ssl/transport-layer-security-tls/)
- [RFC 8446 TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446)
- [Mozilla Server Side TLS 配置](https://wiki.mozilla.org/Security/Server_Side_TLS)
- [SSL Labs](https://www.ssllabs.com/ssltest/)
- [JA3](https://github.com/salesforce/ja3)
- [JARM](https://github.com/salesforce/jarm)
- [Wireshark TLS 分析](https://wiki.wireshark.org/TLS)

---

## 13. 下一步

- [DNS 与域名体系](07-DNS与域名体系.md)
- [网络流量异常检测](../07-网络流量分析/02-网络流量异常检测.md)
- [威胁情报](../05-蓝队核心能力/07-威胁情报.md)
