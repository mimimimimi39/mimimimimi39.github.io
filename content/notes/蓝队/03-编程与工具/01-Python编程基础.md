---
title: "Python 编程基础"
date: 2026-11-10
draft: false
categories: ["蓝队"]
weight: 1
showTableOfContents: true
---

> 难度：★★★☆☆ ｜ 重要度：必学 ｜ 建议学时：30 小时 ｜ 前置：[计算机组成与操作系统基础](../02-计算机与网络基础/01-计算机组成与操作系统基础.md)

## 1. 为什么蓝队要学 Python

蓝队日常有大量重复工作：

- 解析日志、提取 IP/域名/URL
- 批量查询威胁情报
- 统计登录失败、告警 TopN
- 处理 CSV/JSON/PCAP/EVTX
- 调用 SIEM/EDR/防火墙 API
- 自动化封禁、工单、报表

Python 语法简单、生态丰富，是安全行业最常用的胶水语言。

> 目标不是成为开发工程师，而是能写 100-300 行脚本解决实际问题。

---

## 2. 环境准备

### 2.1 安装

- 官网：https://www.python.org/downloads/
- 建议 Python 3.10+
- Windows 安装时勾选 `Add Python to PATH`

### 2.2 虚拟环境

```bash
python -m venv venv
# Windows
venv\Scripts\activate
# Linux/macOS
source venv/bin/activate

pip install requests
pip freeze > requirements.txt
pip install -r requirements.txt
```

### 2.3 编辑器

- VS Code + Python 扩展
- PyCharm Community
- Jupyter Notebook（数据分析）

---

## 3. 基础语法

### 3.1 变量与类型

```python
name = "blue"
count = 10
ratio = 0.75
is_active = True
nothing = None

print(type(name))   # <class 'str'>
print(type(count))  # <class 'int'>
```

### 3.2 字符串

```python
s = "Hello, Blue Team"
print(s.lower())
print(s.upper())
print(s.replace("Blue", "Red"))
print(s.split(", "))
print(s.strip())
print(s.startswith("Hello"))
print(f"length = {len(s)}")
```

### 3.3 列表、元组、字典、集合

```python
ips = ["10.0.0.1", "10.0.0.2", "10.0.0.1"]
ips.append("10.0.0.3")
print(len(ips))
print(set(ips))              # 去重
print(sorted(set(ips)))

point = (1, 2)               # 不可变

log = {"ip": "10.0.0.1", "status": 200, "uri": "/login"}
print(log["ip"])
print(log.get("ua", "unknown"))
log["ua"] = "curl"
for key, value in log.items():
    print(key, value)
```

### 3.4 条件与循环

```python
status = 403

if status == 200:
    print("ok")
elif status in (401, 403):
    print("auth issue")
else:
    print("other")

for ip in ips:
    print(ip)

for i in range(3):
    print(i)

while count > 0:
    count -= 1
```

### 3.5 函数

```python
def is_private_ip(ip: str) -> bool:
    parts = ip.split(".")
    if len(parts) != 4:
        return False
    first = int(parts[0])
    second = int(parts[1])
    return (
        first == 10
        or (first == 172 and 16 <= second <= 31)
        or (first == 192 and second == 168)
    )

print(is_private_ip("192.168.1.10"))
```

---

## 4. 文件与异常

### 4.1 读文件

```python
with open("access.log", "r", encoding="utf-8", errors="ignore") as f:
    for line in f:
        line = line.strip()
        if not line:
            continue
        print(line)
```

### 4.2 写文件

```python
with open("result.txt", "w", encoding="utf-8") as f:
    f.write("hello\n")
```

### 4.3 异常处理

```python
try:
    with open("missing.log", "r", encoding="utf-8") as f:
        data = f.read()
except FileNotFoundError:
    print("file not found")
except PermissionError:
    print("permission denied")
except Exception as e:
    print(f"unexpected error: {e}")
finally:
    print("done")
```

> 不要写裸 `except:`，会吞掉所有异常，难以排查。

---

## 5. 常用标准库

### 5.1 JSON

```python
import json

data = {"ip": "10.0.0.1", "count": 3}
text = json.dumps(data, ensure_ascii=False)
obj = json.loads(text)
```

### 5.2 CSV

```python
import csv

with open("alerts.csv", newline="", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row["src_ip"], row["rule"])
```

### 5.3 时间

```python
from datetime import datetime, timezone, timedelta

now = datetime.now(timezone.utc)
print(now.isoformat())
print(now - timedelta(hours=1))
print(now.strftime("%Y-%m-%d %H:%M:%S"))
```

### 5.4 哈希与编码

```python
import hashlib
import base64
import binascii

print(hashlib.sha256(b"hello").hexdigest())
print(hashlib.md5(b"hello").hexdigest())  # 仅用于校验，不用于安全
print(base64.b64encode(b"hello").decode())
print(base64.b64decode("aGVsbG8=").decode())
print(binascii.hexlify(b"hello").decode())
```

### 5.5 正则

```python
import re

text = "src=10.0.0.1 dst=8.8.8.8"
ips = re.findall(r"\b(?:\d{1,3}\.){3}\d{1,3}\b", text)
print(ips)
```

> 正则详见 [正则表达式](05-正则表达式.md)。

### 5.6 命令行参数

```python
import argparse

parser = argparse.ArgumentParser(description="Analyze access log")
parser.add_argument("logfile")
parser.add_argument("--top", type=int, default=10)
args = parser.parse_args()

print(args.logfile, args.top)
```

---

## 6. 第三方库

### 6.1 requests

```python
import requests

resp = requests.get(
    "https://example.com",
    timeout=10,
    headers={"User-Agent": "blue-team-lab"}
)
print(resp.status_code)
print(resp.headers.get("Server"))
print(resp.text[:200])
```

> 生产脚本要设置超时、异常处理、重试与限速。

### 6.2 其他常用

| 库 | 用途 |
|---|---|
| `requests` | HTTP 请求 |
| `beautifulsoup4` | HTML 解析 |
| `lxml` | XML/HTML |
| `pandas` | 数据分析 |
| `pyyaml` | YAML |
| `python-dateutil` | 时间解析 |
| `scapy` | 构造/解析数据包 |
| `pyshark` | 调用 tshark |
| `python-evtx` | 解析 Windows EVTX |
| `yara-python` | YARA 规则 |
| `paramiko` | SSH |
| `ldap3` | LDAP |
| `elasticsearch` | Elasticsearch 客户端 |
| `splunk-sdk` | Splunk API |

> 不要盲目安装来路不明的包，注意供应链投毒。使用官方 PyPI 并核对包名。

---

## 7. 蓝队实战脚本

### 7.1 统计访问日志 Top IP

```python
from collections import Counter

def top_ips(path: str, n: int = 10):
    counter = Counter()
    with open(path, "r", encoding="utf-8", errors="ignore") as f:
        for line in f:
            parts = line.split()
            if len(parts) >= 1:
                counter[parts[0]] += 1
    for ip, count in counter.most_common(n):
        print(f"{count:8d}  {ip}")

if __name__ == "__main__":
    top_ips("access.log")
```

### 7.2 提取 IOC

```python
import re
import json

IP_RE = re.compile(r"\b(?:\d{1,3}\.){3}\d{1,3}\b")
DOMAIN_RE = re.compile(r"\b(?:[a-zA-Z0-9-]+\.)+[a-zA-Z]{2,}\b")
URL_RE = re.compile(r"https?://[^\s\"'<>]+")
HASH_RE = re.compile(r"\b[a-fA-F0-9]{32,64}\b")

def extract(text: str) -> dict:
    return {
        "ip": sorted(set(IP_RE.findall(text))),
        "domain": sorted(set(DOMAIN_RE.findall(text))),
        "url": sorted(set(URL_RE.findall(text))),
        "hash": sorted(set(HASH_RE.findall(text))),
    }

if __name__ == "__main__":
    sample = "visit http://evil.example.com/a from 10.0.0.1 hash d41d8cd98f00b204e9800998ecf8427e"
    print(json.dumps(extract(sample), ensure_ascii=False, indent=2))
```

### 7.3 批量计算文件哈希

```python
import hashlib
from pathlib import Path

def sha256(path: Path) -> str:
    h = hashlib.sha256()
    with path.open("rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()

for p in Path(".").rglob("*"):
    if p.is_file():
        print(sha256(p), p)
```

### 7.4 检查端口开放

```python
import socket

def check(host: str, port: int, timeout: float = 2.0) -> bool:
    try:
        with socket.create_connection((host, port), timeout=timeout):
            return True
    except OSError:
        return False

for port in (22, 80, 443, 445, 3389):
    print(port, check("127.0.0.1", port))
```

> 仅对授权目标进行端口检查。

### 7.5 查询威胁情报 API

```python
import os
import requests

def vt_lookup(sha256: str) -> dict:
    api_key = os.environ["VT_API_KEY"]
    url = f"https://www.virustotal.com/api/v3/files/{sha256}"
    resp = requests.get(url, headers={"x-apikey": api_key}, timeout=20)
    resp.raise_for_status()
    return resp.json()

# 使用前先申请 API Key，并遵守平台条款与配额
```

> 不要把企业敏感样本/数据上传到公开平台。

---

## 8. 代码质量

- 变量名清晰：`failed_logins` 而不是 `a`。
- 函数单一职责。
- 处理异常与超时。
- 日志记录关键步骤。
- 不在代码中硬编码密钥。
- 使用配置文件/环境变量。
- 脚本加 `if __name__ == "__main__":`。
- 重要脚本写 README 与使用示例。
- 用 `ruff`、`black`、`mypy` 提升质量。

---

## 9. 学习路径

```text
语法基础 → 文件/JSON/CSV → 正则 → requests → 日志解析脚本
→ 威胁情报 API → 数据分析（pandas）→ 自动化平台对接
```

---

## 10. 动手练习

1. 统计 Nginx 日志中状态码分布。
2. 提取日志中的 IP、URL、域名并去重。
3. 计算目录下所有文件的 SHA256。
4. 读取 CSV 告警，按规则名统计数量。
5. 调用 VirusTotal API 查询一个公开样本哈希。
6. 写一个脚本，把 Windows 事件日志导出为 CSV 并统计 4625。
7. 用 argparse 把脚本改造成命令行工具。

---

## 11. 常见误区

- 认为会语法就会编程，不做项目。
- 不处理异常，脚本一遇到坏数据就崩。
- 不设超时，请求卡死。
- 硬编码 API Key 并上传 GitHub。
- 用 `eval` 处理不可信输入。
- 用 MD5 做安全用途。
- 不读官方文档，只抄代码。

---

## 12. 自测

1. 列表、元组、字典、集合的区别？
2. `with open` 的好处是什么？
3. 如何安全地处理用户输入的正则？
4. requests 请求为什么要设置 timeout？
5. 如何计算大文件的哈希而不占满内存？
6. 如何把脚本拆成可复用的函数？
7. 为什么不能把密钥写进代码？

---

## 13. 延伸阅读

- [Python 官方教程](https://docs.python.org/3/tutorial/)
- [Automate the Boring Stuff with Python](https://automatetheboringstuff.com/)
- [廖雪峰 Python 教程](https://www.liaoxuefeng.com/wiki/1016959663602400)
- [Real Python](https://realpython.com/)
- [Requests 文档](https://requests.readthedocs.io/)
- [Python 安全编码指南](https://owasp.org/www-project-python-security/)
- [VirusTotal API 文档](https://developers.virustotal.com/reference)

---

## 14. 下一步

- [PowerShell 脚本基础](02-PowerShell脚本基础.md)
- [正则表达式](05-正则表达式.md)
- [综合实战项目](../09-实战与靶场/03-综合实战项目.md)
