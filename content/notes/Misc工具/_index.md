---
title: "Misc 工具"
description: "文件头速查、命令行基础、不可打印字符和 Misc 题型思路。"
tags: ["Misc", "工具", "CTF"]
showTableOfContents: true
---

Misc 题杂，但杂项有杂项的打法：先确认文件真实类型，再看结构，再按题型套路排查。

## 一、文件头速查

| 类型 | 文件头 | 文件尾/备注 |
| --- | --- | --- |
| PNG | `89 50 4E 47 0D 0A 1A 0A` | `49 45 4E 44 AE 42 60 82` |
| JPEG | `FF D8 FF` | `FF D9` |
| GIF | `47 49 46 38 39 61` / `47 49 46 38 37 61` | `00 3B` |
| BMP | `42 4D` | |
| TIFF | `49 49 2A 00` / `4D 4D 00 2A` | |
| WEBP | `52 49 46 46` + `WEBP` | |
| ZIP | `50 4B 03 04` | |
| RAR | `52 61 72 21` | `C4 3D 7B 00 40 07 00` |
| 7z | `37 7A BC AF 27 1C` | |
| GZIP | `1F 8B 08` | |
| BZ2 | `42 5A 68` | |
| XZ | `FD 37 7A 58 5A 00` | |
| PDF | `25 50 44 46` | |
| MP3 | `49 44 33` / `FF FB` | |
| WAV | `52 49 46 46` + `WAVE` | |
| MIDI | `4D 54 68 64` | |
| AVI | `52 49 46 46` + `AVI` | |
| MOV | `66 74 79 70` | |
| PYC | `03 F3 0D 0A`（版本相关） | |
| Office 2003 | `D0 CF 11 E0` | |
| Office 2007+ | ZIP 结构 | docx/xlsx/pptx |

```bash
file target
xxd target | head
binwalk target
foremost -i target -o out
```

## 二、命令行

### PowerShell / Bash 对照

| 操作 | PowerShell | Bash |
| --- | --- | --- |
| 当前目录 | `Get-Location` / `pwd` | `pwd` |
| 列目录 | `Get-ChildItem` / `ls` | `ls` |
| 读文件 | `Get-Content` / `cat` | `cat` |
| 搜文本 | `Select-String` / `sls` | `grep` / `rg` |
| 找文件 | `Get-ChildItem -Recurse -Filter` | `find` |
| 哈希 | `Get-FileHash` | `sha256sum` |
| 复制 | `Copy-Item` | `cp` |
| 移动 | `Move-Item` | `mv` |
| 删除 | `Remove-Item` | `rm` |
| 解压 | `Expand-Archive` | `unzip` / `tar` |
| 进程 | `Get-Process` | `ps` |

```powershell
Get-ChildItem -Recurse -File | Select-String "flag"
Get-FileHash .\a.zip -Algorithm SHA256
```

```bash
rg -a "flag" .
find . -type f -name "*.zip"
sha256sum a.zip
```

## 三、不可打印字符

密码/文本里出现看不见的字符时：

1. 用十六进制视图看原始字节
2. 对照 ASCII 表（`08` 退格、`09` Tab、`0A` 换行、`0D` 回车、`7F` DEL）
3. 注意退格会「删掉」前一个字符，复制出来的结果和实际显示可能不同

遇到「看起来一样但比较不相等」的字符串，先怀疑零宽字符、同形字或不可见控制符。

## 四、Misc 题型思路

### 图片

- 看属性、EXIF、文件头尾
- `binwalk` / `foremost` 分离附加数据
- StegSolve 看通道、位平面、Image Combiner
- LSB、盲水印、像素 RGB 值、图片拼接、马赛克还原
- PNG 宽高/IDAT/PLTE，JPEG 的 steghide/outguess/F5，GIF 分帧与帧间隔
- 二维码损坏、反色、拼接、透视

### 音频

- Audacity 看波形和频谱
- 摩斯、DTMF、SSTV
- MP3Stego、SilentEye、DeepSound、steghide、OpenPuff
- WAV LSB、MIDI、OGG

### 压缩包

- 伪加密、CRC 碰撞、暴力破解、字典
- 分卷合并、套娃、压缩包炸弹
- 不可见密码、明文攻击（bkcrack）
- ZIP/RAR/7z 结构，010 模板修参数

### 文档

- PDF：附加文件、隐藏文字、图层、增量更新、弱密码、交叉引用
- Office 2007+：解压看 XML、隐藏文字、批注、宏
- Office 2003：OLE 结构、olevba/oledump
- TXT：空白字符、零宽字符、SNOW、spammimic、Cloakify
- HTML：注释、源码、隐藏元素

### 视频与其他

- 分帧、帧间差异、音轨分离
- 视频/音频中的频谱和波形
- OSINT：识图、地图、GitHub 提交、哈希反查
- 隐写工具很多，先看题目提示和文件结构，不要盲目跑工具

## 五、常见坑

- 只看后缀，不看真实文件类型
- 不确认文件头就改扩展名
- 图片能打开就以为没有附加数据
- 忽略 EXIF、注释、文件尾
- 压缩包不区分 ZipCrypto/AES
- 音频只看波形，不看频谱
- 遇到不可见字符直接复制，丢字符
- 工具跑一遍没结果就放弃，不换思路

> 参考：Lunatic BLOG《[持续更新] CTF-Misc Guide》<https://goodlunatic.github.io/posts/1ad9200/>
