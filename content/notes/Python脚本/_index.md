---
title: "Python 脚本"
description: "Python 基础语法和 CTF Misc 常用脚本模板。"
tags: ["Python", "脚本", "CTF"]
showTableOfContents: true
---

CTF 里写脚本不是为了炫技，而是为了把重复操作变成一行命令。先把变量、bytes、文件读写弄明白，再记几个高频模板。

## 一、Python 基础

### 变量与类型

```python
name = "miku"
age = 18
score = 99.5
ok = True
```

CTF 里最常用：`str`、`bytes`、`list`、`dict`、`int`。

### 输出与输入

```python
print("hello")
print(f"flag{{{name}}}")
value = input("input: ")
```

### 判断与循环

```python
if x > 0:
    print("positive")
elif x == 0:
    print("zero")
else:
    print("negative")

for i in range(10):
    print(i)

while n > 0:
    n -= 1
```

### 字符串

```python
s = "flag{test}"
s.upper()
s.lower()
s.replace("flag", "ctf")
s.split("_")
s.strip()
"flag" in s
```

### 列表与字典

```python
nums = [1, 2, 3]
nums.append(4)
nums[0]

d = {"a": 1, "b": 2}
d["a"]
d.get("c", 0)
```

### 函数

```python
def xor(data, key):
    return bytes(b ^ key for b in data)
```

### 文件读写

```python
with open("a.txt", "r", encoding="utf-8") as f:
    text = f.read()

with open("a.bin", "rb") as f:
    data = f.read()

with open("out.bin", "wb") as f:
    f.write(data)
```

### bytes 与 str

```python
s = "flag"
b = s.encode()        # str -> bytes
t = b.decode()        # bytes -> str

bytes.fromhex("666c6167")
b.hex()
```

### 异常与模块

```python
try:
    data = bytes.fromhex(s)
except ValueError:
    print("不是合法 hex")

import base64
from urllib.parse import unquote
```

## 二、Misc 常用脚本

### Base64 / URL / Hex

```python
import base64
from urllib.parse import unquote

base64.b64decode("ZmxhZ3t0ZXN0fQ==").decode()
base64.b64encode(b"flag{test}").decode()
unquote("%66%6c%61%67")
bytes.fromhex("666c6167")
```

### 单字节 XOR 爆破

```python
data = open("cipher.bin", "rb").read()

for key in range(256):
    out = bytes(b ^ key for b in data)
    if b"flag" in out or b"ctf" in out:
        print(key, out)
```

### 循环 XOR

```python
def xor(data, key):
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))
```

### 正则提取

```python
import re

text = open("a.txt", encoding="utf-8", errors="ignore").read()
re.findall(r"flag\{[^}]+\}", text)
re.findall(r"[A-Za-z0-9+/]{16,}={0,2}", text)
```

### 遍历文件

```python
import os

for root, dirs, files in os.walk("."):
    for name in files:
        path = os.path.join(root, name)
        with open(path, "rb") as f:
            data = f.read()
        if b"flag" in data:
            print(path)
```

### 图片 LSB

```python
from PIL import Image

im = Image.open("a.png").convert("RGB")
bits = []
for r, g, b in im.getdata():
    bits.append(r & 1)
    bits.append(g & 1)
    bits.append(b & 1)

data = bytes(int("".join(map(str, bits[i:i+8])), 2) for i in range(0, len(bits), 8))
print(data)
```

### PNG 宽高与 CRC

PNG 的 IHDR 宽高被改时，图片可能显示异常。读取 IHDR，修改宽高后用 `zlib.crc32` 重算 CRC。

```python
import struct, zlib

data = bytearray(open("a.png", "rb").read())
width, height = struct.unpack(">II", data[16:24])
data[16:24] = struct.pack(">II", 800, 600)
crc = zlib.crc32(data[12:29]) & 0xffffffff
data[29:33] = struct.pack(">I", crc)
open("fixed.png", "wb").write(data)
```

（偏移仅适用于标准 PNG，实战先确认块结构。）

### ZIP 与 Office

```python
import zipfile

with zipfile.ZipFile("a.zip") as z:
    print(z.namelist())
    z.extractall("out")
```

Office 文件本质是 ZIP，先解压看 `word/`、`xl/`、`ppt/`。

### PCAP

```python
import subprocess

subprocess.run(["tshark", "-r", "a.pcap", "-Y", "http.request"])
```

### HTTP 请求

```python
import requests

r = requests.get("https://example.com", timeout=10)
print(r.status_code, r.text)
```

### AES

```python
from Crypto.Cipher import AES

cipher = AES.new(key, AES.MODE_ECB)
plain = cipher.decrypt(data)
```

## 三、万能模板

### 读取二进制并找 flag

```python
data = open("target", "rb").read()
print(data[:32])
print(data.find(b"flag"))
```

### 提取可见字符串

```python
import re

data = open("target", "rb").read()
print(re.findall(rb"[\x20-\x7e]{4,}", data))
```

### 批量 Base64 解码

```python
import base64

lines = open("a.txt").read().splitlines()
for line in lines:
    try:
        print(base64.b64decode(line).decode("utf-8", "ignore"))
    except Exception:
        pass
```

## 四、常见坑

- `bytes` 和 `str` 混用
- 解码后直接 `print`，没有 `.decode()`
- 文件读写没加 `b`，遇到二进制坏掉
- 编码参数没写 `encoding="utf-8"`
- 异常全吞掉，出错不知道在哪
- 不确认文件真实类型就按后缀处理

> 整理自本地笔记《Python 基础知识点》《CTF Misc 脚本专项》。
