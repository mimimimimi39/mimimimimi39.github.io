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

## 四、实用解题脚本

以下脚本来自本地 `C:\AAA\解题脚本`，按用途整理，复制到本地后可直接运行。

### 1. 01 文本转二维码图

适用：文件里是一串 ASCII `0/1`，总位数是完全平方数，想还原成黑白方阵图。

```python
import math
import os
import struct
import sys
import zlib


def read_bits(path):
    with open(path, "rb") as f:
        data = f.read()
    bits = []
    for b in data:
        if b == 0x30:
            bits.append(0)
        elif b == 0x31:
            bits.append(1)
    return bits


def write_png(path, width, height, pixels):
    raw = bytearray()
    for y in range(height):
        raw.append(0)
        raw.extend(pixels[y * width:(y + 1) * width])
    compressed = zlib.compress(bytes(raw), 9)

    def chunk(typ, data):
        body = typ + data
        return (struct.pack(">I", len(data)) + body +
                struct.pack(">I", zlib.crc32(body) & 0xffffffff))

    sig = b"\x89PNG\r\n\x1a\n"
    ihdr = struct.pack(">IIBBBBB", width, height, 8, 0, 0, 0, 0)
    with open(path, "wb") as f:
        f.write(sig)
        f.write(chunk(b"IHDR", ihdr))
        f.write(chunk(b"IDAT", compressed))
        f.write(chunk(b"IEND", b""))


def main():
    path = sys.argv[1] if len(sys.argv) > 1 else input("请输入文件路径: ").strip().strip('"')
    if not os.path.exists(path):
        print("文件不存在")
        return

    bits = read_bits(path)
    total = len(bits)
    print(f"提取到的 0/1 字节总数: {total}")

    n = math.isqrt(total)
    if n * n != total:
        print(f"总 01 字节数 {total} 不能被开方，无法生成方阵图片")
        return

    def make_image(black_is_one, suffix):
        px = bytearray()
        for v in bits:
            is_black = (v == 1) if black_is_one else (v == 0)
            px.append(0 if is_black else 255)
        out = f"qr_{suffix}_{n}x{n}.png"
        write_png(out, n, n, px)
        print(f"已生成: {out}")

    make_image(True, "0white1black")
    make_image(False, "0black1white")


if __name__ == "__main__":
    main()
```

```bash
python 01_to_qr.py bits.txt
```

会生成两种反色版本，哪个看起来像二维码/图案就用哪个。

### 2. Base64 转图片

适用：文本或接口响应里带 `data:image/png;base64,...` 前缀的 Base64。

```python
import base64


def base64_to_img(b64_str, out_path):
    b64_str = "".join(b64_str.split())
    if b64_str.lower().startswith("data:") and "," in b64_str:
        b64_str = b64_str.split(",", 1)[1]
    img_data = base64.b64decode(b64_str)
    with open(out_path, "wb") as f:
        f.write(img_data)
    print(f"图片已保存到 {out_path}")


if __name__ == "__main__":
    src = input("输入含 Base64 的 txt 路径: ").strip().strip('"')
    with open(src, "r", encoding="utf-8") as f:
        base64_to_img(f.read(), "output.png")
```

注意：输出固定为 `output.png`，实际格式看文件头，必要时改成 `.jpg` / `.gif`。

### 3. ASCII 与三进制互转

适用：题目只给 `0/1/2`，按字符的 ASCII 码转成三进制。

```python
def to_base3(n):
    if n == 0:
        return "0"
    out = ""
    while n:
        out = str(n % 3) + out
        n //= 3
    return out


def ascii_to_base3(text):
    return [to_base3(ord(ch)) for ch in text]


def base3_to_ascii(groups):
    return "".join(chr(int(g, 3)) for g in groups)


if __name__ == "__main__":
    text = input("请输入文本：")
    result = ascii_to_base3(text)
    print("转换结果：", result)
```

每个字符单独转，所以输出是列表；反解时需要按分隔符或固定宽度重新分组。

### 4. 文本倒序

适用：整段文本或文件被反转，先还原再继续分析。

```python
import os
import sys


def detect_encoding(path):
    with open(path, "rb") as f:
        raw = f.read(4)
    if raw.startswith(b"\xff\xfe"):
        return "utf-16-le"
    if raw.startswith(b"\xfe\xff"):
        return "utf-16-be"
    if raw.startswith(b"\xef\xbb\xbf"):
        return "utf-8-sig"
    return "utf-8"


def main():
    if len(sys.argv) < 2:
        print("用法: python reverse_txt.py 输入文件 输出文件")
        return

    src = sys.argv[1]
    dst = sys.argv[2] if len(sys.argv) > 2 else "reversed_" + os.path.basename(src)
    encoding = detect_encoding(src)

    with open(src, "r", encoding=encoding) as f:
        content = f.read()

    with open(dst, "w", encoding=encoding) as f:
        f.write(content[::-1])

    print(f"完成，已生成 {dst} (编码: {encoding})")


if __name__ == "__main__":
    main()
```

这个脚本按整个文件倒序，换行位置也会一起反转；如果只想倒序每一行，需要先 `splitlines()` 再逐行反转。

## 五、常见坑


- `bytes` 和 `str` 混用
- 解码后直接 `print`，没有 `.decode()`
- 文件读写没加 `b`，遇到二进制坏掉
- 编码参数没写 `encoding="utf-8"`
- 异常全吞掉，出错不知道在哪
- 不确认文件真实类型就按后缀处理

> 整理自本地笔记《Python 基础知识点》《CTF Misc 脚本专项》；实用脚本来自本地 `C:\AAA\解题脚本`。
