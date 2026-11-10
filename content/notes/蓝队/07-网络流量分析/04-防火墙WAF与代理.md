---
title: "防火墙、WAF 与代理"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 4
showTableOfContents: true
---

> 难度：★★★☆☆ ｜ 重要度：必学 ｜ 建议学时：8 小时 ｜ 前置：[计算机网络基础](../02-计算机与网络基础/04-计算机网络基础.md)、[HTTP 协议详解](../02-计算机与网络基础/05-HTTP协议详解.md)

## 1. 防火墙

### 1.1 类型

| 类型 | 说明 |
|---|---|
| 包过滤 | 五元组 |
| 状态检测 | 跟踪连接状态 |
| 应用层防火墙 | 识别应用 |
| NGFW | 应用识别、IPS、URL 过滤、威胁情报 |
| 云安全组 | 虚拟防火墙 |
| 主机防火墙 | iptables/nftables/Windows Firewall |

### 1.2 策略原则

- 默认拒绝
- 最小开放
- 按业务/区域分段
- 管理面隔离
- 定期审计
- 变更审批
- 日志记录

### 1.3 NAT

- SNAT：源地址转换
- DNAT：目的地址转换/端口映射
- 日志溯源要结合 NAT 表

### 1.4 日志

- 允许/拒绝
- 五元组
- 时间
- 规则 ID
- NAT 前后地址
- 应用/URL（NGFW）

---

## 2. WAF

### 2.1 部署方式

| 方式 | 说明 |
|---|---|
| 反向代理 | 流量经过 WAF |
| 透明代理 | 串接（网桥/策略路由），必须在流量路径上 |
| 插件/模块 | 集成到 Web 服务器 |
| 云 WAF | DNS 接入 |
| 容器侧 | Ingress/Service Mesh |

### 2.2 检测技术

- 签名规则（OWASP CRS）
- 语义分析
- 异常/行为
- 机器学习
- 虚拟补丁
- 人机验证/速率限制

### 2.3 常见防护

- SQL 注入
- XSS
- 命令注入
- 文件上传
- 路径穿越
- SSRF
- CC 攻击
- Bot 管理
- API 防护

### 2.4 产品

- ModSecurity + OWASP CRS
- 长亭雷池（SafeLine）
- 阿里云/腾讯云/Cloudflare WAF
- F5、Imperva、Akamai
- Nginx + Lua（自研）

### 2.5 日志

- 请求时间、源 IP、URI、方法、UA
- 规则 ID、动作（放行/拦截/观察）
- 攻击类型
- 响应码
- 请求体（注意隐私）

### 2.6 调优

- 观察模式先上线
- 白名单（业务、IP、URL）
- 规则例外
- 阈值调整
- 误报分析
- 定期评审

### 2.7 绕过概念（防御视角）

- 编码/混淆
- 分块传输
- 参数污染
- 协议解析差异
- 大小写/注释
- 白名单滥用
- 源 IP 伪造（XFF）

> WAF 是纵深防御的一层，不能替代安全开发与补丁。

---

## 3. 代理

### 3.1 正向代理

客户端配置代理访问外部。

- 显式代理
- 透明代理
- PAC
- 认证

### 3.2 反向代理

服务端代理（Nginx、HAProxy、WAF）。

- 负载均衡
- TLS 终结
- 缓存
- 安全头
- 访问控制

### 3.3 日志

- 源 IP、用户、URL、方法、状态码
- UA、Referer
- 上传/下载大小
- 响应时间
- 分类/信誉

### 3.4 TLS 检查

- 代理解密（企业根证书）
- 隐私与合规
- 证书固定导致失败
- 性能

### 3.5 匿名化与绕过

- VPN
- Tor
- 公共代理
- DoH/DoT
- 域前置
- CDN

检测：

- 代理/VPN 情报
- 异常 TLS/SNI
- 流量特征
- 登录地点变化
- 阻断策略

---

## 4. 封禁与处置流程

```text
告警/情报 → 验证 → 确定范围（IP/域名/URL/账号）
→ 评估影响 → 封禁（防火墙/WAF/DNS/代理/EDR）
→ 监控 → 记录 → 到期复审 → 解封/延长
```

### 4.1 注意事项

- 避免误封 CDN/云服务
- 共享 IP 要谨慎
- 保留封禁依据
- 设置有效期
- 业务确认
- 记录工单

### 4.2 封禁层次

| 层次 | 优点 | 缺点 |
|---|---|---|
| 防火墙 IP | 快速 | 共享 IP 误伤 |
| DNS 域名 | 覆盖多协议 | 需 DNS 控制 |
| 代理 URL | 精准 | 仅代理流量 |
| WAF | 应用层 | 仅 Web |
| EDR | 终端 | 仅终端 |
| 账号禁用 | 阻断身份 | 影响用户 |

---

## 5. 蓝队检测思路

- 防火墙拒绝突增 → 扫描/攻击
- WAF 拦截突增 → Web 攻击
- 代理异常上传 → 外传
- 非工作时段访问 → 异常
- 新域名/低信誉 → 可疑
- 非常见端口 → 隧道
- 代理/VPN 登录 → 账号风险
- 规则命中分布 → 攻击趋势

---

## 6. 实操

1. 配置 iptables/nftables 默认拒绝并放行 22/80/443。
2. 用 ModSecurity + CRS 保护本地 DVWA。
3. 触发 SQLi/XSS，观察 WAF 日志。
4. 配置 Nginx 反向代理与安全头。
5. 配置代理日志并分析 Top URL。
6. 模拟一次 IP 封禁与解封流程。
7. 分析 WAF 误报并加白名单。

---

## 7. 自测

1. 防火墙策略原则？
2. 状态检测与包过滤的区别？
3. WAF 的部署方式？
4. WAF 能防所有 Web 攻击吗？
5. 如何降低 WAF 误报？
6. 正向代理与反向代理的区别？
7. 封禁 IP 有哪些风险？
8. 如何检测代理/VPN 滥用？

---

## 8. 延伸阅读

- [ModSecurity](https://modsecurity.org/)
- [OWASP CRS](https://coreruleset.org/)
- [长亭雷池 WAF](https://waf-ce.chaitin.cn/)
- [Nginx 文档](https://nginx.org/en/docs/)
- [HAProxy 文档](https://docs.haproxy.org/)
- [Cloudflare WAF](https://www.cloudflare.com/learning/ddos/glossary/web-application-firewall-waf/)
- [MITRE ATT&CK：Web Service](https://attack.mitre.org/techniques/T1102/)
- [iptables 文档](https://netfilter.org/documentation/)

---

## 9. 下一步

- [MITRE ATT&CK 框架](../08-威胁框架与攻防对抗/01-MITRE-ATTACK框架.md)
- [网络杀伤链](../08-威胁框架与攻防对抗/02-网络杀伤链.md)
- [红蓝对抗基础](../08-威胁框架与攻防对抗/03-红蓝对抗基础.md)
