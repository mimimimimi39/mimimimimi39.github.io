---
title: "Web 入门：从 HTTP 到第一个 flag"
date: 2026-10-30
draft: false
description: "Web 方向不是背 payload，而是理解请求怎么流动。附知识地图、分阶段路线、常见漏洞速查和练习平台。"
tags: ["CTF", "Web", "Web安全", "学习路线"]
categories: ["学习笔记"]
showTableOfContents: true
---

Web 入门最容易被带偏：收藏一堆 payload，看到输入框就乱试，最后什么也没学会。真正需要建立的是**数据流思维**——用户输入从浏览器出发，经过 HTTP、后端、数据库，再回到页面，每一层都可能出问题。

这篇是我的 Web 入门路线，目标是能独立完成基础 CTF Web 题，并为后面的代码审计和漏洞利用打底。

## 先说结论

- 核心：**HTTP + 浏览器 + 代理 + 常见漏洞 + 本地靶场**
- 必学漏洞：SQL 注入、XSS、文件上传、文件包含、SSRF、命令执行
- 必会工具：DevTools、Burp Suite、curl、ffuf
- 时间预算：两到三周打基础，之后靠靶场和题目积累
- 只在授权靶场练习，别碰真实站点

## 一、知识地图

### 1. HTTP 与浏览器

- 请求行、请求头、请求体
- 响应行、响应头、响应体
- 常见方法：GET、POST、PUT、DELETE
- 状态码：200、301、302、400、403、404、500
- Cookie、Session、Token
- 同源策略、CORS
- URL 编码、HTML 实体、Base64

### 2. 前端基础

- HTML 表单、输入框、隐藏字段
- JavaScript 基本语法、事件、`fetch`
- 浏览器 DevTools：Elements、Network、Console、Application
- 前端校验只是体验，不能当安全边界

### 3. 后端与数据库

- 路由、参数接收、模板渲染
- 文件系统操作、命令执行
- 数据库：SELECT、UNION、报错、盲注
- 序列化与反序列化
- 身份认证与会话管理

### 4. 常见漏洞

| 漏洞 | 典型入口 | 关键点 |
| --- | --- | --- |
| SQL 注入 | 登录、搜索、ID 参数 | 拼接 SQL、报错、盲注 |
| XSS | 评论、搜索、URL 参数 | 输出未转义、存储型/反射型 |
| 文件上传 | 头像、附件 | 后缀、MIME、内容检测、解析漏洞 |
| 文件包含 | `?page=`、模板 | 本地/远程包含、伪协议 |
| SSRF | 图片代理、URL 抓取 | 内网地址、协议、重定向 |
| 命令执行 | `ping`、系统工具 | 拼接命令、过滤绕过 |
| 反序列化 | Cookie、参数 | 魔术方法、POP 链 |
| JWT | 登录态 | 弱密钥、算法混淆、未校验 |
| SSTI | 模板渲染 | 模板语法、沙箱绕过 |
| 越权 | 用户 ID、接口 | 水平/垂直越权 |

### 5. 工具

- 浏览器 DevTools：看请求、改参数、看源码
- Burp Suite Community：抓包、改包、重放、Intruder
- `curl`：快速复现请求
- `ffuf` / `dirsearch`：目录和参数爆破
- `sqlmap`：SQL 注入辅助，但要理解原理
- Python `requests`：写脚本

## 二、分阶段路线

### 阶段 0：HTTP 与浏览器（2~3 天）

- 用 DevTools 观察登录请求
- 手动构造 GET/POST 请求
- 理解 Cookie 和 Session
- 用 `curl -v` 看完整请求响应

**掌握标准**：能说清楚一次登录从表单提交到服务端返回的全过程。

### 阶段 1：本地靶场（5~7 天）

先搭环境，不要直接上 CTF 题：

- DVWA：经典漏洞分类
- Juice Shop：现代 Web 漏洞
- upload-labs：文件上传专项
- sqli-labs：SQL 注入专项

用 Burp 代理浏览器，习惯抓包和改包。

**掌握标准**：能在 DVWA 里独立完成 low/medium 难度的常见漏洞。

### 阶段 2：OWASP Top 10（7~10 天）

按漏洞类型逐个过：

- 注入：SQL、命令、模板
- 认证失效：弱口令、会话固定、JWT
- XSS：反射、存储、DOM
- 访问控制：越权、IDOR
- 安全配置错误：目录列表、备份文件、默认口令
- SSRF、文件上传、文件包含
- 反序列化、组件漏洞

推荐 PortSwigger Web Security Academy，讲解和实验都很系统。

**掌握标准**：看到一种输入点，能列出至少三种可能的测试方向。

### 阶段 3：CTF 实战（持续）

CTF Web 题有自己的套路：

- 先看源码、注释、备份文件、robots.txt
- 再看请求头、Cookie、隐藏参数
- 关注报错信息、状态码、响应长度
- 有源码先审计，没源码先扫目录
- 题目名和描述经常是提示

**掌握标准**：能独立完成基础难度的 CTF Web 题，并写清楚利用链。

### 阶段 4：代码审计与进阶（长期）

- PHP/Java/Python 常见危险函数
- 反序列化链、模板注入
- 中间件、框架漏洞
- 内网渗透基础（授权环境）

## 三、常用命令

```bash
# 看响应头
curl -I https://target/

# 带 Cookie 请求
curl -b "session=xxx" https://target/admin

# 发送 JSON
curl -X POST https://target/api -H "Content-Type: application/json" -d '{"id":1}'

# 目录扫描
ffuf -u https://target/FUZZ -w wordlist.txt

# 参数爆破
ffuf -u "https://target/?id=FUZZ" -w numbers.txt
```

Burp 里最常用的功能：

- Proxy：抓包和改包
- Repeater：反复修改请求
- Intruder：爆破参数
- Decoder：编解码
- Comparer：比较响应差异

## 四、做题流程

1. 读题，记录题目名、描述、提示
2. 看页面源码、注释、JS 文件
3. 扫目录和备份文件
4. 抓登录/搜索/上传请求，观察参数
5. 判断漏洞类型，做最小验证
6. 读源码（如果有）确认过滤逻辑
7. 构造利用链，拿 flag
8. 写 writeup，记录卡点

## 五、常见坑

- 只背 payload，不理解为什么能打
- 忽略 URL 编码和二次编码
- 改了参数但没带 Cookie
- 看到 403 就放弃，没试路径绕过
- 盲注没耐心，直接上 sqlmap 又不会调
- 文件上传只改后缀，没看内容检测
- 有源码不看，硬猜
- 在真实站点测试，触碰法律红线
- 把前端校验当安全措施

## 六、练哪里

- **PortSwigger Web Security Academy**：最系统的免费 Web 安全实验
- **DVWA**：本地经典靶场
- **OWASP Juice Shop**：现代 Web 漏洞
- **upload-labs / sqli-labs**：专项练习
- **CTFHub**：Web 技能树
- **BUUCTF / 攻防世界**：CTF Web 题
- **TryHackMe / Hack The Box**：授权靶场，适合进阶

练习方法：

- 每类漏洞至少做 5 个实验
- 做完关掉 writeup 重做一遍
- 记录「输入点 → 测试 → 响应 → 结论」

## 七、什么时候该止损

- 一道题 30 分钟没有方向，就看提示或跳题
- 不要在没有授权的情况下测试任何真实站点
- 不要用 sqlmap 代替理解
- Web 方向很大，先把基础漏洞打牢，再碰框架和中间件

## 结语

Web 入门的关键是理解请求怎么走、数据在哪一层被处理。工具会过时，payload 会失效，但数据流思维不会。

把本地靶场刷熟，再去打 CTF 题，最后再碰代码审计。一步一步来，比收藏一百个 payload 有用得多。

> 本文只讨论 CTF 比赛和授权靶场中的 Web 安全技术。未经授权对真实站点进行测试属于违法行为。
