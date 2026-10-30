---
title: "Web 入门"
date: 2026-10-30
draft: false
description: "Web 方向不是背 payload，而是理解请求怎么流动。附常见漏洞速查和常用工具。"
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
- 学习顺序：先 HTTP，再本地靶场，最后刷漏洞类型
- 只在授权靶场练习，别碰真实站点

## 一、学习内容分类

### 1. HTTP 与浏览器

- 请求/响应结构、方法、状态码、Header
- Cookie、Session、Token
- DevTools 的 Network 与 Application
- 用 `curl -v` 复现请求

重点：能说清楚一次登录从表单提交到服务端返回的全过程。

### 2. 前端与数据传输

- 表单、隐藏字段、JavaScript 请求
- 前端校验不能当安全边界
- URL 编码、HTML 实体、Base64

重点：知道数据在浏览器、请求、响应之间是怎么变的。

### 3. 后端与数据库

- 路由、参数接收、模板渲染
- 文件操作、命令执行
- SQL 基础、联合查询、报错、盲注
- 序列化、身份认证、会话管理

重点：理解输入最终被哪一段后端逻辑处理。

### 4. 漏洞类型

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

重点：每类至少掌握原理、入口和最小验证方式，而不是只背 payload。

### 5. 工具与代理

- Burp Suite：Proxy、Repeater、Intruder
- `curl`、`ffuf`、`sqlmap`
- Python `requests`

重点：能抓包、改包、重放，并解释每个工具的参数。

### 6. CTF 实战方法

- 看源码、注释、备份文件、robots.txt
- 抓登录/搜索/上传请求，观察参数和响应差异
- 有源码先审计，没源码先扫目录
- 题目名和描述经常是提示

重点：能独立完成基础难度的 CTF Web 题，并写清楚利用链。

## 二、常用命令

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

## 三、做题流程

1. 读题，记录题目名、描述、提示
2. 看页面源码、注释、JS 文件
3. 扫目录和备份文件
4. 抓登录/搜索/上传请求，观察参数
5. 判断漏洞类型，做最小验证
6. 读源码（如果有）确认过滤逻辑
7. 构造利用链，拿 flag
8. 写 writeup，记录卡点

## 四、常见坑

- 只背 payload，不理解为什么能打
- 忽略 URL 编码和二次编码
- 改了参数但没带 Cookie
- 看到 403 就放弃，没试路径绕过
- 盲注没耐心，直接上 sqlmap 又不会调
- 文件上传只改后缀，没看内容检测
- 有源码不看，硬猜
- 在真实站点测试，触碰法律红线
- 把前端校验当安全措施

## 结语

Web 入门的关键是理解请求怎么走、数据在哪一层被处理。工具会过时，payload 会失效，但数据流思维不会。

把本地靶场刷熟，再去打 CTF 题，最后再碰代码审计。一步一步来，比收藏一百个 payload 有用得多。

> 本文只讨论 CTF 比赛和授权靶场中的 Web 安全技术。未经授权对真实站点进行测试属于违法行为。
