---
title: "Bash 脚本基础"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 3
showTableOfContents: true
---

> 难度：★★☆☆☆ ｜ 重要度：必学 ｜ 建议学时：12 小时 ｜ 前置：[Linux 基础](../02-计算机与网络基础/02-Linux基础.md)

## 1. 为什么蓝队要学 Bash

- Linux 服务器排查第一现场，Bash 最直接。
- 日志统计、批量处理、应急采集都离不开 Shell。
- 很多安全工具本身就是 Shell 脚本。
- 理解 Bash 也能看懂攻击者的命令与持久化脚本。

---

## 2. 第一个脚本

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "Hello, Blue Team"
```

```bash
chmod +x hello.sh
./hello.sh
```

`set -euo pipefail`：

- `-e`：命令失败立即退出
- `-u`：使用未定义变量报错
- `-o pipefail`：管道中任一命令失败即失败

---

## 3. 变量与引号

```bash
name="blue"
count=10
echo "$name"
echo "${name}_team"

files=$(ls /var/log)
echo "$files"

# 只读
readonly PI=3.14

# 环境变量
export LOG_DIR="/var/log"
```

### 引号规则

| 写法 | 行为 |
|---|---|
| `"$var"` | 变量展开，推荐 |
| `'$var'` | 原样输出，不展开 |
| `$var` | 展开，但空格/通配符可能出问题 |
| `` `cmd` `` | 命令替换（旧） |
| `$(cmd)` | 命令替换（推荐） |

> 变量赋值等号两边不能有空格：`a=1` 正确，`a = 1` 错误。

---

## 4. 条件判断

### 4.1 if

```bash
if [ -f /etc/passwd ]; then
    echo "exists"
elif [ -d /etc ]; then
    echo "directory"
else
    echo "missing"
fi
```

### 4.2 常用测试

| 测试 | 含义 |
|---|---|
| `-f file` | 普通文件 |
| `-d dir` | 目录 |
| `-e path` | 存在 |
| `-r/-w/-x` | 可读/写/执行 |
| `-z str` | 空字符串 |
| `-n str` | 非空 |
| `a -eq b` | 数字相等 |
| `a -ne b` | 数字不等 |
| `a -gt b` | 大于 |
| `a -lt b` | 小于 |
| `str1 = str2` | 字符串相等 |
| `str1 != str2` | 字符串不等 |

### 4.3 逻辑

```bash
if [ "$user" = "root" ] && [ -f /etc/shadow ]; then
    echo "root and shadow exists"
fi

if [ "$port" -eq 22 ] || [ "$port" -eq 2222 ]; then
    echo "ssh port"
fi
```

### 4.4 case

```bash
case "$1" in
    start) echo "start" ;;
    stop)  echo "stop" ;;
    *)     echo "usage: $0 {start|stop}" ;;
esac
```

---

## 5. 循环

```bash
for ip in 10.0.0.1 10.0.0.2 10.0.0.3; do
    echo "$ip"
done

for i in $(seq 1 5); do
    echo "$i"
done

while read -r line; do
    echo "$line"
done < access.log

until [ "$count" -ge 3 ]; do
    count=$((count + 1))
done
```

> 逐行读文件用 `while read -r line`，不要用 `for line in $(cat file)`（会按空格拆分）。

---

## 6. 函数与参数

```bash
usage() {
    echo "usage: $0 <logfile> [top_n]"
}

top_ips() {
    local file="$1"
    local n="${2:-10}"
    awk '{print $1}' "$file" | sort | uniq -c | sort -nr | head -n "$n"
}

if [ $# -lt 1 ]; then
    usage
    exit 1
fi

top_ips "$1" "${2:-10}"
```

| 变量 | 含义 |
|---|---|
| `$0` | 脚本名 |
| `$1`...`$9` | 位置参数 |
| `$#` | 参数个数 |
| `$@` | 所有参数（推荐加引号） |
| `$*` | 所有参数（合并为一个字符串） |
| `$?` | 上一条命令退出码 |
| `$$` | 当前进程 PID |

---

## 7. 管道与重定向

```bash
command > file        # 覆盖 stdout
command >> file       # 追加 stdout
command 2> error.log  # stderr
command > out.log 2>&1 # 合并
command &> all.log    # Bash 写法
command < input.txt   # stdin
```

常用管道：

```bash
grep "Failed password" /var/log/auth.log | wc -l
ps aux | grep nginx | grep -v grep
cat access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head
```

---

## 8. 文本处理三剑客

### 8.1 grep

```bash
grep -i "failed" auth.log
grep -v "grep" ps.txt
grep -E "error|warn" app.log
grep -c "Failed password" auth.log
grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" log.txt | sort -u
grep -r "eval(" /var/www
```

### 8.2 sed

```bash
sed -n '1,10p' file
sed 's/old/new/g' file
sed -i 's/old/new/g' file   # 直接修改，慎用
sed -n '/error/p' app.log
```

### 8.3 awk

```bash
awk '{print $1}' access.log
awk '{print $1, $9}' access.log
awk '$9 == 404 {print $1}' access.log
awk '{sum += $10} END {print sum}' access.log
awk -F: '{print $1}' /etc/passwd
```

### 8.4 其他

| 命令 | 用途 |
|---|---|
| `cut` | 按列切割 |
| `sort` | 排序 |
| `uniq -c` | 去重计数 |
| `wc -l` | 行数 |
| `tr` | 字符转换 |
| `head/tail` | 头尾 |
| `xargs` | 批量传参 |
| `find` | 查找文件 |
| `tee` | 同时输出到屏幕与文件 |

---

## 9. 蓝队实战示例

### 9.1 SSH 爆破 Top IP

```bash
grep "Failed password" /var/log/auth.log \
  | awk '{for(i=1;i<=NF;i++) if($i=="from") print $(i+1)}' \
  | sort | uniq -c | sort -nr | head -20
```

### 9.2 成功登录记录

```bash
grep "Accepted" /var/log/auth.log | tail -20
last -a | head -20
```

### 9.3 Web 日志 Top IP 与 404

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head -20

awk '$9 == 404 {print $1}' access.log | sort | uniq -c | sort -nr | head -20
```

### 9.4 查找最近修改的 Webshell 嫌疑文件

```bash
find /var/www -type f -mtime -7 \
  \( -name "*.php" -o -name "*.jsp" -o -name "*.aspx" \) \
  -exec ls -lh {} \;
```

### 9.5 应急信息采集

```bash
#!/usr/bin/env bash
set -uo pipefail

OUT="/tmp/ir_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$OUT"

ps auxf > "$OUT/ps.txt" 2>&1
ss -tulnp > "$OUT/net.txt" 2>&1
last -a > "$OUT/last.txt" 2>&1
lastb > "$OUT/lastb.txt" 2>&1
cat /etc/passwd > "$OUT/passwd.txt" 2>&1
cat /etc/crontab > "$OUT/crontab.txt" 2>&1
crontab -l > "$OUT/user_crontab.txt" 2>&1
ls -la /etc/cron.* > "$OUT/cron_dirs.txt" 2>&1
systemctl list-units --type=service > "$OUT/services.txt" 2>&1
find / -perm -4000 -type f > "$OUT/suid.txt" 2>/dev/null
find /tmp /var/tmp /dev/shm -type f -executable > "$OUT/tmp_exec.txt" 2>/dev/null

tar -czf "$OUT.tar.gz" -C "$(dirname "$OUT")" "$(basename "$OUT")"
echo "collected: $OUT.tar.gz"
```

### 9.6 批量检查文件哈希

```bash
find /var/www -type f -print0 | xargs -0 sha256sum > /tmp/www_hashes.txt
```

---

## 10. 调试与风格

```bash
bash -n script.sh      # 语法检查
bash -x script.sh      # 调试执行
shellcheck script.sh   # 静态检查
```

建议：

- 变量加引号：`"$var"`
- 使用 `$(...)` 而不是反引号
- 使用 `[[ ]]`（Bash）而不是 `[ ]`（更安全）
- 错误处理与日志
- 脚本开头写用途、作者、日期
-  destructive 操作前确认

---

## 11. 动手练习

1. 统计 auth.log 中失败登录 Top10 IP。
2. 统计 Nginx 日志中状态码分布。
3. 找出 `/var/log` 下最近 3 天修改的文件。
4. 写一个脚本，接收目录参数，输出最大 10 个文件。
5. 用 `find + xargs` 批量计算哈希。
6. 写一个应急采集脚本并在虚拟机运行。
7. 用 shellcheck 检查并修复脚本问题。

---

## 12. 常见误区

- 变量不加引号导致空格/通配符问题。
- 用 `for` 遍历 `cat` 输出。
- 不检查命令退出码。
- `rm -rf "$dir/"` 变量为空导致误删。
- 在生产环境直接跑未测试脚本。
- 用 `sed -i` 修改关键配置不留备份。

---

## 13. 自测

1. `"$var"` 与 `'$var'` 的区别？
2. `$@` 与 `$*` 的区别？
3. 如何统计日志中出现次数最多的 IP？
4. `set -euo pipefail` 的作用？
5. `find` 如何按时间与权限查找文件？
6. `awk` 如何按列处理日志？
7. 如何安全地逐行读取文件？

---

## 14. 延伸阅读

- [GNU Bash 手册](https://www.gnu.org/software/bash/manual/)
- [Bash Guide](https://mywiki.wooledge.org/BashGuide)
- [ShellCheck](https://www.shellcheck.net/)
- [The Linux Command Line](https://linuxcommand.org/tlcl.php)
- [ExplainShell](https://explainshell.com/)
- [Awesome Bash](https://github.com/awesome-lists/awesome-bash)

---

## 15. 下一步

- [SQL 与日志查询](04-SQL与日志查询.md)
- [正则表达式](05-正则表达式.md)
- [Linux 日志与审计](../06-主机与终端安全/04-Linux日志与审计.md)
