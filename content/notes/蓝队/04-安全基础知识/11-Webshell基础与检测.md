---
title: "Webshell 基础与检测"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 11
showTableOfContents: true
---

> 难度：★★★☆☆ ｜ 重要度：必学 ｜ 建议学时：6 小时 ｜ 前置：[文件上传与命令注入](08-文件上传与命令注入.md)、[HTTP 协议详解](../02-计算机与网络基础/05-HTTP协议详解.md)

## 1. 什么是 Webshell

Webshell 是攻击者上传到 Web 服务器上的恶意脚本，用于远程执行命令、管理文件、提权、横向移动。

常见语言：

- PHP（`.php`、`.phtml`、`.php5`）
- JSP（`.jsp`、`.jspx`）
- ASP/ASPX（`.asp`、`.aspx`、`.ashx`）
- Python、Node.js、Ruby 等

---

## 2. 类型

| 类型 | 特点 | 检测难度 |
|---|---|---|
| 一句话木马 | 代码极短，依赖外部管理工具 | 中 |
| 小马 | 功能较少，上传/执行 | 中 |
| 大马 | 功能完整，文件管理、命令、数据库 | 低（特征多） |
| 加密/混淆马 | 编码、加密、混淆 | 高 |
| 内存马 | 不落地文件，驻留内存 | 高 |
| 合法工具滥用 | 利用管理脚本/插件 | 高 |

> 内存马见 [内存马与无文件攻击](../08-威胁框架与攻防对抗/07-内存马与无文件攻击.md)。

---

## 3. 常见管理工具（检测视角）

| 工具 | 特点 | 检测线索 |
|---|---|---|
| 中国菜刀（Chopper） | 早期工具，明文/简单编码 | 特定参数、UA、流量特征 |
| AntSword | 支持多种编码器 | 参数名、加密流量、UA |
| 冰蝎（Behinder） | 2.0 动态密钥协商；3.0+ 预共享密钥、加密流量 | 2.0 看握手；3.0+ 看内置老旧 UA 随机、Content-Type、固定请求结构、心跳 |
| 哥斯拉（Godzilla） | 加密流量、多种 payload | 流量加密、特征串、行为 |
| 其他 | 自定义 | 需结合行为与文件检测 |

> 工具名称用于威胁情报与检测对照，不提供使用方式。冰蝎 3.0 默认使用连接密码的 MD5 前 16 位作为 AES 密钥，UA 从内置列表随机选择，不能再按「固定 UA」简单匹配。

---

## 4. 流量特征

### 4.1 常见特征

- 对同一脚本高频 POST
- 参数名随机/无意义（如 `pass`、`cmd`、`z1`、`a`）
- 请求体为 Base64/加密数据
- 响应内容短小、固定格式
- User-Agent 异常（工具默认 UA、空 UA）
- Cookie 异常/固定
- 固定心跳间隔
- 访问路径位于上传目录/非业务目录
- 上传后立即访问
- 请求方法异常（PUT/自定义）
- Content-Type 与内容不符

### 4.2 行为特征

- Web 进程拉起 `cmd.exe`、`powershell.exe`、`/bin/sh`、`bash`
- Web 进程访问系统敏感文件
- Web 进程创建新文件/修改配置
- 外连 C2/下载工具
- 提权、持久化、横向

### 4.3 日志特征

- Web 访问日志中 `POST` 到 `.php/.jsp/.aspx`
- 响应状态 200、响应大小异常
- 请求体大小异常
- 同一 IP 高频访问同一脚本
- 非工作时段访问
- 访问路径不在正常路由中

---

## 5. 静态检测

### 5.1 文件层面

- Web 目录新增/修改脚本文件
- 文件时间与业务发布不一致
- 文件权限异常
- 文件内容含危险函数：
  - PHP：`eval`、`assert`、`system`、`exec`、`shell_exec`、`passthru`、`base64_decode`、`preg_replace`（/e）、`create_function`
  - JSP：`Runtime.getRuntime().exec`、`ProcessBuilder`、`ClassLoader`、`defineClass`
  - ASP：`eval`、`execute`、`CreateObject`
- 高度混淆/压缩/加密
- 超长单行
- 隐藏文件、双扩展名、畸形文件名
- 图片马（图片中嵌入脚本）

### 5.2 工具

| 工具 | 用途 |
|---|---|
| D 盾 | Windows Web 目录查杀 |
| 河马 | Webshell 检测 |
| YARA | 自定义规则 |
| ClamAV | 通用杀毒 |
| Semgrep | 代码规则 |
| 文件完整性监控 | 发现变更 |

### 5.3 YARA 思路

```yara
rule Webshell_Generic_PHP
{
    meta:
        description = "Generic PHP webshell indicators"
    strings:
        $a = "eval(" ascii nocase
        $b = "base64_decode(" ascii nocase
        $c = "shell_exec(" ascii nocase
        $d = "system(" ascii nocase
    condition:
        2 of them
}
```

> 该规则仅用于示例，误报率较高，需结合白名单与人工确认。

---

## 6. 动态检测

- **进程**：Web 进程子进程、命令行、父进程、签名
- **网络**：Web 进程外连、监听端口、DNS
- **文件**：上传目录、临时目录、配置变更
- **内存**：内存马、注入代码
- **日志**：Web 访问、系统、EDR

### 6.1 进程树示例

```text
w3wp.exe (IIS)
  └─ cmd.exe /c whoami
       └─ whoami.exe
```

```text
nginx / php-fpm
  └─ sh -c curl http://...
```

这类父子关系通常高度可疑。

---

## 7. 应急响应

1. 隔离主机/下线站点（视业务）。
2. 保存 Web 日志、访问日志、上传目录、进程、网络连接、内存。
3. 定位 Webshell 文件，计算哈希，保存样本（加密压缩）。
4. 分析入口：上传漏洞、注入、反序列化、弱口令、供应链。
5. 检查持久化、账号、横向、数据外传。
6. 清除 Webshell，修复入口，重置凭据。
7. 恢复业务并持续监控。
8. 复盘，补充检测规则与基线。

---

## 8. 防御

- 上传校验（白名单、内容检测、重命名、存储隔离）
- 上传目录禁止执行
- Web 目录完整性监控
- 最小权限运行 Web 服务
- WAF + RASP + EDR 多层
- 及时打补丁
- 隐藏版本信息
- 定期扫描与人工审计
- 日志集中与告警

---

## 9. 实操

1. 在本地 DVWA 上传一个测试脚本，观察文件与日志。
2. 用 YARA 扫描 Web 目录。
3. 用 Sysmon 观察 Web 进程子进程。
4. 用 Wireshark 分析一次 Webshell 流量（本地）。
5. 写一条 SIEM 规则：Web 目录新增脚本文件。
6. 阅读一次真实 Webshell 应急报告。

> 仅限本地或授权环境，禁止上传到真实站点。

---

## 10. 自测

1. Webshell 的常见类型？
2. 一句话木马的特点？
3. Webshell 流量有哪些特征？
4. 如何从进程树发现 Webshell 执行？
5. 静态检测有哪些方法？
6. 上传目录为什么要禁止执行？
7. 发现 Webshell 后如何处置？

---

## 11. 延伸阅读

- [MITRE ATT&CK：Web Shell](https://attack.mitre.org/techniques/T1505/003/)
- [YARA 文档](https://virustotal.github.io/yara/)
- [D 盾](https://www.d99net.net/)
- [河马 Webshell 查杀](https://www.shellpub.com/)
- [ClamAV](https://www.clamav.net/)
- [SANS：Web Shell 检测](https://www.sans.org/blog/)
- [The DFIR Report](https://thedfirreport.com/)

---

## 12. 下一步

- [云安全与容器基础](12-云安全与容器基础.md)
- [内存马与无文件攻击](../08-威胁框架与攻防对抗/07-内存马与无文件攻击.md)
- [应急响应流程与实战](../05-蓝队核心能力/06-应急响应流程与实战.md)
