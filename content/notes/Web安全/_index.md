---
title: "Web 安全"
description: "HTTP 基础、AWD 攻防赛生存流程和 PHP 危险函数速查。"
tags: ["Web", "AWD", "CTF"]
showTableOfContents: true
---

Web 方向先理解数据流，再谈 payload。AWD 则是另一套能力：在有限时间里备份、加固、找后门、批量拿分。

## 一、HTTP 基础

- 请求行：`GET /index HTTP/1.1`
- 响应行：状态码 200、301、302、403、404、500
- 常见请求头：`Host`、`Cookie`、`Authorization`、`User-Agent`、`Content-Type`、`Referer`
- 请求体：GET 参数在 URL，POST 数据在 body
- `Content-Type` 决定 body 怎么解析：表单、JSON、multipart
- HTTP 是无状态的，服务器靠 Cookie/Token/Session 识别用户
- HTTPS 是 TLS 加密的 HTTP，HTTP 是明文

学习 Web 前至少要能看懂：谁在请求、请求了什么、带了什么身份、服务端返回了什么。

## 二、AWD 规则与准备

AWD（Attack With Defence）一般每队维护多台服务器，服务器上存在多个漏洞和 flag。攻击别人得分，自己被拿走 flag 扣分，服务宕机还会扣分。

开局要做的事：

1. 确认 SSH 账号、端口、密码
2. 改掉默认密码，避免被别人先改
3. 备份源码和数据库
4. 找预留后门
5. 扫端口、看服务
6. 修漏洞、加监控
7. 写批量利用脚本

```bash
ssh ctf@192.168.1.10
ssh -p 2222 ctf@192.168.1.10
sftp -i id_rsa ctf@192.168.1.10
```

密码修改：

```bash
passwd
```

数据库密码修改（进入 MySQL 后）：

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password';
FLUSH PRIVILEGES;
```

## 三、备份

```bash
cd /var/www && tar -czvf /tmp/html.tgz html
mysqldump -uroot -proot --databases DB1 DB2 > /tmp/db.sql
mysqldump -uroot -proot --all-databases --skip-lock-tables > /tmp/db.sql
scp -P 22 user@host:/tmp/db.sql ./
```

注意 `tar` 默认可能漏掉隐藏文件，必要时用 `tar -czvf /tmp/html.tgz .` 或明确包含。

## 四、查找后门

- 最近修改的文件、异常权限文件、隐藏文件
- 一句话木马、`eval`、`assert`、`system`、`base64_decode`
- 不死马、内存马、软链接、计划任务、SSH 公钥
- Web 目录外的可写目录、上传目录
- 数据库中的恶意存储内容

```bash
find /var/www -type f -mtime -1
grep -RIn --include='*.php' -E 'eval|assert|system|exec|shell_exec|base64_decode' /var/www
```

## 五、PHP 常见危险函数

```text
eval
assert
system
exec
shell_exec
passthru
popen
proc_open
base64_decode
gzinflate
gzuncompress
preg_replace /e
create_function
call_user_func
call_user_func_array
```

看到这些函数不等于有漏洞，但值得重点审计参数从哪里来。

## 六、防御思路

- 备份：源码、数据库、配置
- 改密码：SSH、数据库、Web 后台
- 修漏洞：上传、包含、SQL、命令执行、反序列化
- 删后门：webshell、隐藏账号、计划任务、SSH key
- 监控：文件变化、进程、登录日志、异常请求
- WAF/限制：能上就上，至少限制上传目录执行
- 日志：保留访问日志，方便定位被打的点
- 批量脚本：拿到一个后门后快速批量利用

## 七、常见坑

- 开局不改密码，被其他队直接接管
- 只备份网站根目录，漏掉数据库和软链接目录
- 只找 `eval`，漏掉加密/拼接/回调后门
- 修了漏洞但没删后门
- 服务被自己改崩，导致 check 失败扣分
- 批量脚本没做异常处理，打到一半全挂

> 参考：先知社区《浅谈AWD攻防赛的生存攻略》<https://xz.aliyun.com/news/10856>
