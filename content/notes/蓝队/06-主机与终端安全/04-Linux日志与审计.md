---
title: "Linux 日志与审计"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 4
showTableOfContents: true
---

> 难度：★★★★☆ ｜ 重要度：必学 ｜ 建议学时：15 小时 ｜ 前置：[Linux 基础](../02-计算机与网络基础/02-Linux基础.md)、[日志分析与日志源](../05-蓝队核心能力/03-日志分析与日志源.md)

## 1. Linux 日志体系

| 日志 | 内容 | 常见系统 |
|---|---|---|
| `/var/log/auth.log` | 认证、SSH、sudo | Debian/Ubuntu |
| `/var/log/secure` | 认证、SSH、sudo | RHEL/CentOS |
| `/var/log/syslog` | 系统日志 | Debian/Ubuntu |
| `/var/log/messages` | 系统日志 | RHEL/CentOS |
| `/var/log/kern.log` | 内核 | Debian/Ubuntu |
| `/var/log/cron` | 计划任务 | RHEL/CentOS；Debian/Ubuntu 看 syslog 或 journald |
| `/var/log/audit/audit.log` | auditd | 多数 |
| `/var/log/wtmp` | 登录成功 | 多数 |
| `/var/log/btmp` | 登录失败 | 多数 |
| `/var/log/lastlog` | 最后登录 | 多数 |
| `/var/log/faillog` | 失败记录 | 多数 |
| `/var/log/nginx/` | Web | Nginx |
| `/var/log/apache2/` | Web | Debian/Ubuntu |
| `/var/log/httpd/` | Web | RHEL/CentOS |
| `journalctl` | systemd 日志 | systemd 系统 |

---

## 2. rsyslog 与 journald

### 2.1 rsyslog

配置文件：`/etc/rsyslog.conf`、`/etc/rsyslog.d/*.conf`

```text
auth,authpriv.*    /var/log/auth.log
*.*;auth,authpriv.none  -/var/log/syslog
cron.*             /var/log/cron
```

转发到 SIEM：

```text
*.* @@siem.example.com:514   # TCP
*.* @siem.example.com:514    # UDP
```

> 生产环境建议 TCP + TLS，避免日志丢失与窃听。

### 2.2 journald

```bash
journalctl -xe
journalctl -u ssh --since "2026-11-09 00:00"   # Debian/Ubuntu
journalctl -u sshd --since "2026-11-09 00:00"  # RHEL/CentOS
journalctl -p err -b
journalctl --disk-usage
```

配置：`/etc/systemd/journald.conf`

- `Storage=persistent`
- `SystemMaxUse=`
- `ForwardToSyslog=yes`

---

## 3. auditd

### 3.1 作用

Linux 内核审计系统，记录系统调用、文件访问、权限变更、命令执行等。

### 3.2 常用命令

```bash
auditctl -l
auditctl -s
ausearch -k <key>
ausearch -m EXECVE
aureport --auth
aureport --failed
aureport --summary
```

### 3.3 规则示例

```bash
# 监控敏感文件
auditctl -w /etc/passwd -p wa -k passwd_change
auditctl -w /etc/shadow -p wa -k shadow_change
auditctl -w /etc/sudoers -p wa -k sudoers_change

# 监控命令执行
auditctl -a always,exit -F arch=b64 -S execve -k exec

# 监控权限提升
auditctl -a always,exit -F arch=b64 -S setuid -S setgid -k privilege

# 监控网络连接
auditctl -a always,exit -F arch=b64 -S connect -k network
```

持久化：`/etc/audit/rules.d/audit.rules`

> 全量 execve 日志量很大，生产环境要评估性能与存储。

### 3.4 日志字段

| 字段 | 含义 |
|---|---|
| `type` | 事件类型（SYSCALL、EXECVE、PATH） |
| `auid` | 审计用户 ID |
| `uid`/`gid` | 实际用户/组 |
| `euid`/`egid` | 有效用户/组 |
| `pid`/`ppid` | 进程 ID |
| `exe` | 可执行文件 |
| `comm` | 命令名 |
| `key` | 规则标签 |
| `success` | 是否成功 |

---

## 4. 用户与登录审计

```bash
who
w
last -a | head -20
lastb | head -20
lastlog
faillog
```

### 4.1 SSH 日志

```bash
grep "Failed password" /var/log/auth.log
grep "Accepted" /var/log/auth.log
grep "Invalid user" /var/log/auth.log
grep "session opened" /var/log/auth.log
```

统计爆破源 IP：

```bash
grep "Failed password" /var/log/auth.log \
  | awk '{for(i=1;i<=NF;i++) if($i=="from") print $(i+1)}' \
  | sort | uniq -c | sort -nr | head
```

### 4.2 sudo

```bash
grep "sudo:" /var/log/auth.log
```

关注：

- 非管理员使用 sudo
- `sudo su`、`sudo -i`
- 异常命令
- 失败尝试

### 4.3 账号变更

```bash
grep -E "useradd|usermod|userdel|groupadd|passwd" /var/log/auth.log
```

---

## 5. 持久化排查

| 位置 | 检查 |
|---|---|
| cron | `crontab -l`、`/etc/crontab`、`/etc/cron.*`、`/var/spool/cron` |
| systemd | `systemctl list-unit-files --state=enabled`、`/etc/systemd/system` |
| rc.local | `/etc/rc.local` |
| profile | `/etc/profile`、`~/.bashrc`、`~/.profile` |
| SSH | `~/.ssh/authorized_keys` |
| 服务 | `init.d`、`systemd` |
| 内核模块 | `lsmod`、`/etc/modules` |
| 动态链接 | `/etc/ld.so.preload` |
| 定时器 | `systemctl list-timers` |

```bash
crontab -l
cat /etc/crontab
ls -la /etc/cron.* /var/spool/cron*
systemctl list-unit-files --state=enabled
find / -name authorized_keys 2>/dev/null
cat /etc/ld.so.preload 2>/dev/null
lsmod
```

---

## 6. 提权与后门排查

```bash
# UID 0 账号
awk -F: '$3==0 {print $1}' /etc/passwd

# 空密码账号
awk -F: '($2=="") {print $1}' /etc/shadow

# SUID/SGID
find / -perm -4000 -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null

# 可写目录中的可执行文件
find /tmp /var/tmp /dev/shm -type f -executable 2>/dev/null

# capabilities
getcap -r / 2>/dev/null

# sudo 配置
grep -v '^#' /etc/sudoers
ls -la /etc/sudoers.d/
```

---

## 7. Web 日志分析

### 7.1 Nginx 日志格式

```text
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
```

### 7.2 常用统计

```bash
# Top IP
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head

# 404 Top
awk '$9 == 404 {print $7}' access.log | sort | uniq -c | sort -nr | head

# POST 请求
grep '"POST ' access.log | head

# 可疑 UA
grep -Ei "sqlmap|nikto|nmap|masscan|curl|python" access.log

# 敏感路径
grep -Ei "\.php|\.jsp|\.aspx|admin|upload|shell" access.log
```

---

## 8. 入侵排查清单

1. 账号：UID 0、空密码、sudo、SSH key
2. 进程：异常进程、路径、父进程、CPU
3. 网络：监听端口、外连、DNS
4. 持久化：cron、systemd、rc.local、profile、ld.so.preload
5. 文件：/tmp、/dev/shm、Web 目录、SUID、最近修改
6. 日志：auth、audit、journald、Web
7. 历史：bash_history、last、lastb
8. 内核：模块、rootkit
9. 容器：Docker/K8s
10. 样本：二进制、脚本、Webshell

---

## 9. 日志采集与保护

- rsyslog/journald 转发到集中平台
- 使用 TLS
- 限制日志目录权限
- 日志轮转（logrotate）
- 监控日志服务状态
- 防篡改（远程副本、WORM）
- 时间同步（NTP/chrony）
- 保留期符合合规

```bash
systemctl status rsyslog
systemctl status systemd-journald
timedatectl
```

---

## 10. 实操

1. 在 Ubuntu 上查看 auth.log 的 SSH 登录。
2. 模拟 SSH 爆破，统计源 IP。
3. 配置 auditd 监控 `/etc/passwd`，修改后查询。
4. 配置 rsyslog 转发到另一台虚拟机。
5. 检查 cron、systemd、authorized_keys。
6. 查找 SUID 文件与 UID 0 账号。
7. 分析 Nginx 日志中的扫描与 Webshell 访问。

---

## 11. 常见误区

- 只看 auth.log，不看 auditd/journald
- 认为删了 auth.log 就没痕迹（还有 journald/远程日志）
- 忽略 Web 日志
- 忽略 cron 与 systemd
- 不检查 authorized_keys
- 不看 SUID/capabilities
- 日志未集中，攻击者一删就没
- 时区不统一

---

## 12. 自测

1. auth.log 与 secure 的区别？
2. journald 与 rsyslog 的关系？
3. auditd 能记录什么？
4. 如何统计 SSH 爆破源 IP？
5. 常见 Linux 持久化位置？
6. 如何发现 UID 0 后门账号？
7. 如何检查 SUID 提权？
8. 日志如何防篡改？

---

## 13. 延伸阅读

- [Linux audit 文档](https://github.com/linux-audit/audit-documentation)
- [Red Hat：审计指南](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/)
- [Ubuntu：日志文档](https://ubuntu.com/server/docs)
- [rsyslog 文档](https://www.rsyslog.com/doc/)
- [systemd journal 文档](https://www.freedesktop.org/software/systemd/man/journalctl.html)
- [MITRE ATT&CK：Linux](https://attack.mitre.org/matrices/enterprise/linux/)
- [GTFOBins](https://gtfobins.github.io/)
- [Linux 应急响应工具](https://github.com/meirwah/awesome-incident-response)

---

## 14. 下一步

- [EDR 与 HIDS](05-EDR与HIDS.md)
- [恶意软件基础](06-恶意软件基础.md)
- [安全基线加固](08-安全基线加固.md)
