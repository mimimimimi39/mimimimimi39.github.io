---
title: "Linux 基础"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 2
showTableOfContents: true
---

> 难度：★★☆☆☆ ｜ 重要度：必学 ｜ 建议学时：20 小时 ｜ 前置：[计算机组成与操作系统基础](01-计算机组成与操作系统基础.md)

## 1. 为什么蓝队要学 Linux

- 大量服务器、Web 服务、数据库、容器运行在 Linux 上。
- 安全设备、SIEM、日志平台、靶场大多基于 Linux。
- Linux 入侵排查、日志分析、应急响应是蓝队核心技能。
- 云原生与容器安全离不开 Linux。

> 目标不是成为 Linux 运维专家，而是能**独立登录服务器、定位异常、分析日志、排查持久化**。

---

## 2. 学习目标

学完本篇你应该能够：

- 熟练使用 30+ 常用命令
- 理解文件权限与用户组
- 管理进程、服务、网络、计划任务
- 找到并分析关键日志
- 发现常见 Linux 入侵痕迹
- 编写简单 Bash 排查脚本

---

## 3. 文件系统与目录

### 3.1 目录结构（FHS）

| 目录 | 作用 |
|---|---|
| `/` | 根目录 |
| `/bin`、`/usr/bin` | 常用命令 |
| `/sbin`、`/usr/sbin` | 管理员命令 |
| `/etc` | 配置文件 |
| `/home` | 普通用户家目录 |
| `/root` | root 家目录 |
| `/var` | 可变数据，日志在 `/var/log` |
| `/tmp` | 临时文件，任何人可写 |
| `/dev` | 设备文件 |
| `/proc` | 进程与内核虚拟文件系统 |
| `/sys` | 内核与设备信息 |
| `/opt` | 第三方软件 |
| `/srv` | 服务数据 |
| `/lib`、`/usr/lib` | 库文件 |
| `/boot` | 内核与引导文件 |

### 3.2 常用文件命令

| 命令 | 作用 | 示例 |
|---|---|---|
| `pwd` | 当前目录 | `pwd` |
| `ls` | 列目录 | `ls -alh` |
| `cd` | 切换目录 | `cd /var/log` |
| `cat` | 查看文件 | `cat /etc/passwd` |
| `less` | 分页查看 | `less auth.log` |
| `head/tail` | 头/尾 | `tail -f auth.log` |
| `grep` | 搜索 | `grep -i "failed" auth.log` |
| `find` | 查找文件 | `find / -perm -4000 -type f 2>/dev/null` |
| `locate` | 快速查找（需 updatedb） | `locate sshd_config` |
| `which/whereis` | 命令路径 | `which python3` |
| `file` | 文件类型 | `file suspicious.bin` |
| `stat` | 文件元数据 | `stat /etc/passwd` |
| `du` | 目录大小 | `du -sh /var/log` |
| `df` | 磁盘空间 | `df -h` |
| `mount` | 挂载 | `mount` |
| `ln` | 链接 | `ln -s target link` |
| `cp/mv/rm` | 复制/移动/删除 | `cp -a src dst` |
| `chmod/chown` | 权限/属主 | `chmod 750 file` |
| `tar` | 打包压缩 | `tar -czf log.tar.gz /var/log` |
| `sha256sum` | 哈希 | `sha256sum sample.bin` |

### 3.3 权限模型

```text
-rwxr-xr--  1 root root  1234 Nov 9 10:00 script.sh
│└┬┘└┬┘└┬┘
│ │  │  └── 其他人：r--
│ │  └───── 组：r-x
│ └──────── 属主：rwx
└────────── 文件类型：- 普通文件，d 目录，l 链接
```

数字权限：

| 数字 | 权限 |
|---|---|
| 4 | r 读 |
| 2 | w 写 |
| 1 | x 执行 |
| 7 | rwx |
| 6 | rw- |
| 5 | r-x |
| 4 | r-- |
| 0 | --- |

常见：

```bash
chmod 755 script.sh      # rwxr-xr-x
chmod 644 config.conf    # rw-r--r--
chmod 600 id_rsa         # rw-------
chown user:group file
umask                    # 查看默认权限掩码
```

### 3.4 用户与组

| 文件 | 作用 |
|---|---|
| `/etc/passwd` | 用户信息（UID、GID、家目录、Shell） |
| `/etc/shadow` | 密码哈希（仅 root 可读） |
| `/etc/group` | 组信息 |
| `/etc/sudoers` | sudo 权限 |
| `/etc/sudoers.d/` | sudo 附加配置 |

常用命令：

```bash
whoami
id
who
w
last
lastb
useradd / usermod / userdel
groupadd / groupmod / groupdel
passwd
su - user
sudo -l
```

**排查重点**：

- UID 为 0 的非 root 账号
- 新增的陌生账号
- 特权组（sudo、wheel、docker）
- `authorized_keys` 中的陌生公钥
- sudoers 中的 NOPASSWD 规则

```bash
awk -F: '$3==0 {print $1}' /etc/passwd
grep -v '^#' /etc/sudoers | grep -v '^$'
find / -name authorized_keys 2>/dev/null
```

---

## 4. 进程与网络

### 4.1 进程管理

| 命令 | 作用 |
|---|---|
| `ps aux` | 查看所有进程 |
| `ps -ef --forest` | 进程树 |
| `top` / `htop` | 实时资源 |
| `pstree -p` | 进程树 |
| `pgrep -a name` | 按名称查 PID |
| `kill -9 PID` | 强制结束 |
| `lsof -p PID` | 进程打开的文件 |
| `lsof -i :80` | 端口占用 |
| `strace -p PID` | 跟踪系统调用 |
| `cat /proc/PID/cmdline` | 完整命令行 |
| `ls -l /proc/PID/exe` | 可执行文件路径 |
| `ls -l /proc/PID/cwd` | 工作目录 |

**异常进程特征**：

- 进程名伪装成 `sshd`、`kworker`、`systemd` 等
- 可执行文件位于 `/tmp`、`/dev/shm`、`/var/tmp`
- 命令行含 Base64、`curl | bash`、`wget`、`nc`
- 父进程异常（如 Web 服务拉起 shell）
- 高 CPU（挖矿）
- 删除后仍运行的进程（`/proc/PID/exe` 显示 `(deleted)`）

```bash
ls -l /proc/*/exe 2>/dev/null | grep deleted
ps aux --sort=-%cpu | head
```

### 4.2 网络排查

| 命令 | 作用 |
|---|---|
| `ip addr` | IP 地址 |
| `ip route` | 路由 |
| `ss -tulnp` | 监听端口与进程 |
| `ss -antp` | TCP 连接 |
| `netstat -antp` | 老式连接查看 |
| `curl -v URL` | HTTP 调试 |
| `dig domain` | DNS 查询 |
| `tcpdump -i any -w cap.pcap` | 抓包 |
| `iptables -L -n` / `nft list ruleset` | 防火墙 |
| `traceroute` | 路由追踪 |

**排查重点**：

- 陌生监听端口（后门、Webshell 服务）
- 异常外联 IP/域名
- 大量 SYN 连接（扫描/爆破）
- DNS 长域名（隧道）
- 非标准端口上的常见服务

```bash
ss -tulnp
ss -antp state established
lsof -i -P -n | grep -v "127.0.0.1"
```

---

## 5. 服务与计划任务

### 5.1 systemd

```bash
systemctl status sshd
systemctl start/stop/restart sshd
systemctl enable/disable sshd
systemctl list-units --type=service --state=running
systemctl list-unit-files --state=enabled
journalctl -u sshd -n 50
```

自定义服务目录：

- `/etc/systemd/system/`
- `/usr/lib/systemd/system/`
- `/lib/systemd/system/`

**排查重点**：

- 新建或修改的服务文件
- 服务执行的命令含 `/tmp`、`/dev/shm`、`bash -c`、`curl`
- 服务以 root 运行
- 服务文件时间异常

```bash
systemctl list-unit-files --state=enabled
find /etc/systemd/system /lib/systemd/system -name "*.service" -mtime -7
```

### 5.2 计划任务

| 位置 | 说明 |
|---|---|
| `/etc/crontab` | 系统级 |
| `/etc/cron.d/` | 系统级片段 |
| `/etc/cron.hourly/daily/weekly/monthly/` | 周期任务 |
| `/var/spool/cron/` | 用户 crontab |
| `/var/spool/cron/crontabs/` | Debian/Ubuntu 用户 crontab |
| `crontab -l` | 当前用户任务 |
| `systemctl list-timers` | systemd 定时器 |

**排查重点**：

- 陌生定时任务
- 任务执行 `curl | bash`、Base64、`/dev/tcp`
- 隐藏的 `\r`、空格、注释伪装
- root 用户的 crontab

```bash
crontab -l
cat /etc/crontab
ls -la /etc/cron.* /var/spool/cron*
systemctl list-timers --all
```

---

## 6. 日志与审计

### 6.1 常见日志

| 路径 | 内容 |
|---|---|
| `/var/log/auth.log` | Debian/Ubuntu 认证日志 |
| `/var/log/secure` | RHEL/CentOS 认证日志 |
| `/var/log/syslog` | Debian/Ubuntu 系统日志 |
| `/var/log/messages` | RHEL/CentOS 系统日志 |
| `/var/log/kern.log` | 内核日志 |
| `/var/log/cron` | 计划任务日志 |
| `/var/log/audit/audit.log` | auditd 审计日志 |
| `/var/log/wtmp` | 登录成功记录（`last`） |
| `/var/log/btmp` | 登录失败记录（`lastb`） |
| `/var/log/lastlog` | 每个用户最后登录（`lastlog`） |
| `/var/log/faillog` | 失败记录 |
| `/var/log/nginx/access.log` | Nginx 访问日志 |
| `/var/log/apache2/access.log` | Apache 访问日志 |
| `/var/log/dpkg.log`、`/var/log/yum.log` | 软件安装 |

### 6.2 journalctl

```bash
journalctl -xe
journalctl -u sshd --since "2026-11-09 00:00"
journalctl -p err -b
journalctl --disk-usage
```

### 6.3 常见排查命令

```bash
# SSH 爆破
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr | head

# 成功登录
grep "Accepted" /var/log/auth.log

# 新增用户
grep -E "useradd|adduser" /var/log/auth.log

# sudo 使用
grep "sudo:" /var/log/auth.log

# 登录历史
last -a | head -20
lastb | head -20
lastlog

# 命令历史
cat ~/.bash_history
```

> 攻击者可能清理日志、修改历史。要结合 journald、auditd、SIEM、外部日志综合判断。

### 6.4 auditd 基础

```bash
auditctl -l
ausearch -k passwd_change
aureport --auth
aureport --failed
```

示例规则：

```bash
auditctl -w /etc/passwd -p wa -k passwd_change
auditctl -w /etc/shadow -p wa -k shadow_change
auditctl -w /etc/sudoers -p wa -k sudoers_change
auditctl -a always,exit -F arch=b64 -S execve -k exec
```

规则可写入 `/etc/audit/rules.d/audit.rules` 持久化。

---

## 7. SSH 与远程访问

```bash
ssh user@host
ssh -i key.pem user@host
scp file user@host:/path
sftp user@host
```

服务端配置：`/etc/ssh/sshd_config`

安全建议：

- 禁用 root 直接登录：`PermitRootLogin no`
- 使用密钥登录：`PasswordAuthentication no`
- 修改默认端口（降低噪音，不是安全边界）
- 限制来源 IP
- 开启登录日志与 fail2ban

**排查重点**：

- `~/.ssh/authorized_keys` 是否被添加
- `sshd_config` 是否被修改（如 AllowUsers、PermitRootLogin）
- `~/.ssh/config`、`known_hosts`
- 异常 SSH 进程或端口转发

```bash
find / -name authorized_keys -exec ls -l {} \; 2>/dev/null
grep -E "PermitRootLogin|PasswordAuthentication|AllowUsers" /etc/ssh/sshd_config
```

---

## 8. 软件包与文件查找

### 8.1 包管理

| 系统 | 命令 |
|---|---|
| Debian/Ubuntu | `apt update`、`apt install`、`dpkg -l` |
| RHEL/CentOS | `yum` / `dnf install`、`rpm -qa` |
| Arch | `pacman -S` |
| Alpine | `apk add` |

### 8.2 查找可疑文件

```bash
# 最近 7 天修改的文件
find / -mtime -7 -type f 2>/dev/null | head -50

# SUID 文件
find / -perm -4000 -type f 2>/dev/null

# 可写目录中的可执行文件
find /tmp /var/tmp /dev/shm -type f -executable 2>/dev/null

# 大文件
find / -type f -size +100M 2>/dev/null

# 按内容搜索
grep -r "eval(" /var/www 2>/dev/null
```

---

## 9. Linux 入侵排查清单

1. **账号**：`/etc/passwd`、`/etc/shadow`、sudoers、SSH key
2. **进程**：异常进程、父进程、可执行文件路径、CPU/内存
3. **网络**：监听端口、外联连接、DNS 查询
4. **持久化**：systemd、cron、rc.local、profile、ld.so.preload、init.d
5. **文件**：/tmp、/dev/shm、Web 目录、最近修改文件、SUID
6. **日志**：auth.log、secure、audit.log、journald、Web 日志
7. **历史**：bash_history、登录记录
8. **内核**：内核模块、rootkit 迹象
9. **容器**：Docker/K8s 异常容器、挂载、逃逸
10. **样本**：可疑二进制、脚本、Webshell

```bash
# 一键收集（示例，按需调整；使用 mktemp 避免固定目录被抢占）
umask 077
IR_DIR=$(mktemp -d /tmp/ir.XXXXXX)
ps auxf > "$IR_DIR/ps.txt"
ss -tulnp > "$IR_DIR/net.txt"
last -a > "$IR_DIR/last.txt"
lastb > "$IR_DIR/lastb.txt"
cat /etc/passwd > "$IR_DIR/passwd.txt"
crontab -l > "$IR_DIR/crontab.txt" 2>&1
ls -la /etc/cron.* > "$IR_DIR/cron.txt" 2>&1
systemctl list-units --type=service > "$IR_DIR/services.txt"
find / -perm -4000 -type f > "$IR_DIR/suid.txt" 2>/dev/null
tar -czf "$IR_DIR.tar.gz" "$IR_DIR"
```

---

## 10. 动手练习

1. 在 Ubuntu 上创建用户、组，设置权限，测试 sudo。
2. 用 `ps`、`ss`、`lsof` 找到占用 80 端口的进程。
3. 配置 SSH 密钥登录，禁用密码登录。
4. 写一个 cron 任务，观察 `/var/log/cron` 或 journald。
5. 用 `tcpdump` 抓取一次 HTTP 请求并用 Wireshark 打开。
6. 在虚拟机模拟 SSH 爆破（仅本地），用 auth.log 统计源 IP。
7. 完成 OverTheWire Bandit 前 20 关。

---

## 11. 常见误区

- 只会 `cd/ls/cat`，不会看进程、网络、日志。
- 认为 `rm` 后文件一定消失。
- 认为隐藏文件就是恶意（先看内容和来源）。
- 忽略 `/tmp`、`/dev/shm` 等可写目录。
- 只看 `ps` 不看 `/proc` 与父进程。
- 不记录操作，破坏证据。

---

## 12. 自测

1. 如何找出 UID 为 0 的账号？
2. 如何查看一个进程的完整命令行、工作目录、可执行文件？
3. 如何统计 SSH 登录失败次数最多的 IP？
4. systemd 服务文件通常放在哪里？如何发现新增服务？
5. 常见 Linux 持久化位置有哪些？
6. `auditd` 能做什么？如何查询审计日志？

---

## 13. 延伸阅读

- [The Linux Command Line（免费书）](https://linuxcommand.org/tlcl.php)
- [Linux Journey](https://linuxjourney.com/)
- [OverTheWire Bandit](https://overthewire.org/wargames/bandit/)
- [Ubuntu 官方文档](https://ubuntu.com/server/docs)
- [Red Hat 系统管理文档](https://docs.redhat.com/)
- [Linux 审计文档](https://github.com/linux-audit/audit-documentation)
- [GTFOBins](https://gtfobins.github.io/) —— 理解攻击者如何滥用系统二进制
- [MITRE ATT&CK：Linux](https://attack.mitre.org/matrices/enterprise/linux/)

---

## 14. 下一步

- [Windows 基础](03-Windows基础.md)
- [Bash 脚本基础](../03-编程与工具/03-Bash脚本基础.md)
- [Linux 日志与审计](../06-主机与终端安全/04-Linux日志与审计.md)
