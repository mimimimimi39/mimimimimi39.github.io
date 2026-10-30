---
title: "编码密码"
description: "进制、Base、古典密码、特殊编码、现代加密与压缩包攻击的整理。"
tags: ["编码", "密码学", "Misc", "CTF"]
showTableOfContents: true
---

编码和密码题的关键是**先认出来，再选工具**。编码没有密钥，只有规则；古典密码靠语言特征；现代加密没有密钥基本只能爆破或找泄露。

## 一、进制与 Base 家族

- 二进制：只有 `0/1`，每 8 位一组转 ASCII
- 八进制：只有 `0-7`，先转十六进制或十进制
- 十六进制：`0-9A-F`，`xxd -r -p` 或 CyberChef
- 三进制：只有 `0/1/2`，`int(s, 3)`
- 四进制：只有 `0-3`
- Base16：就是十六进制
- Base32：`A-Z2-7` + `=`，长度通常 8 的倍数
- Base58：去掉 `0OIl+/`，常见于比特币
- Base62：纯字母数字，变种顺序不固定
- Base64：`A-Za-z0-9+/`，URL-safe 用 `-_`
- Base85/Ascii85、Base91、Base100（emoji）

Base64 细节：

- 标准字符集 `+/`，URL-safe 换成 `-_`
- 填充 `=` 可能被故意删掉，补到 4 的倍数
- 解码结果是字节，可能是图片、压缩包，不一定是文本

Base64 隐写：

- 常见思路是把信息藏在 Base64 字符的低位、填充位或每组的最后一个字符
- 先正常解码看能否得到文本，再比较不同位置的字符差异
- 工具：CyberChef、脚本逐位提取

## 二、Hash 与爆破

| Hash | 长度 | Hashcat 模式 |
| --- | --- | --- |
| MD5 | 32 位 hex | `-m 0` |
| SHA1 | 40 位 hex | `-m 100` |
| SHA256 | 64 位 hex | `-m 1400` |
| SHA512 | 128 位 hex | `-m 1700` |
| NTLM / NT Hash | 32 位 hex | `-m 1000` |
| bcrypt | `$2y$` 开头 | `-m 3200` |

判断思路：

- Windows 内存取证、SAM、lsass、hashdump、NTDS.dit → 优先 NTLM
- 文件校验、数据库 password、普通字符串 → 可能是 MD5
- 先 `hashid` / `hash-identifier`，再选模式

```bash
hashcat -m 1000 hash.txt rockyou.txt
hashcat -m 0 hash.txt -a 3 ?l?l?l?d?d
john --wordlist=rockyou.txt hash.txt
```

## 三、古典密码

| 密码 | 特征 | 处理 |
| --- | --- | --- |
| 凯撒 / ROT | 字母整体位移 | 爆破 26 种，ROT13 是位移 13 |
| ROT47 | 可打印 ASCII 位移 47 | CyberChef |
| Atbash | 字母表倒序 | `a↔z`、`b↔y` |
| 维吉尼亚 | 多表替换，需要 key | 词频/已知明文，CyberChef |
| Gronsfeld | 数字密钥的维吉尼亚 | 爆破 key |
| 仿射 | `ax+b mod 26` | 枚举 a、b |
| 希尔 | 矩阵乘法 | 需要密钥矩阵或已知明文 |
| 栅栏 | 按栏数重新排列 | 枚举栏数 |
| 培根 | A/B 两组，5 位一字母 | 在线工具/脚本 |
| Polybius / ADFGX | 坐标替换 | 查表 |
| Rabbit / 云影 | 特定算法/数字串 | 在线解码、脚本 |
| Quipqiup | 单表替换 | 词频与词形 |

古典密码先做频率分析，再看题目有没有给 key、字母表或提示。

## 四、特殊编码

| 名称 | 识别特征 | 备注 |
| --- | --- | --- |
| 摩斯电码 | `.`、`-`、分隔符 | 分隔符可能是空格、`/` 或换行 |
| 敲击码 | 数字对 1-5 | 与 Polybius 类似 |
| 博多码 | 5 位二进制 |  Baudot |
| 格雷码 | 相邻只变一位 | 先转二进制 |
| 曼彻斯特 | 01/10 表示位 | 需要同步 |
| 键盘密码 | 字母在键盘上的位置/QWE 位移 | 26 键、九宫格、坐标 |
| 手机键盘 | 2-9 数字与次数 | 老式 T9 |
| DNA 编码 | A/T/C/G 四种字符 | 每 2 位一组映射 |
| 盲文 | 6 点/8 点组合 | 查盲文表 |
| 旗语 | 手臂角度 | 查旗语表 |
| 中文电码 | 四位数字对应汉字 | 在线中文电码表 |
| 社会主义核心价值观 | 12 个词编码 | 在线工具 |
| 音乐符号 | 音符/简谱 | 按音名、频率、时长 |
| 零宽字符 | 肉眼不可见 | 提取 `U+200B/C/D/FEFF` |
| Whitespace | 空格/Tab/换行 | 空白符转二进制 |
| SNOW | 行尾空格藏数据 | `snow` 工具 |
| UUencode / XXencode | `begin 644` / 特定字符集 | `uudecode`、CyberChef |
| AAencode / JJencode | 颜文字/日文符号 | 浏览器控制台执行 |
| Quoted-Printable | `=E4=BD=A0` | 邮件常见 |
| Unicode 转义 | `\u0066` | CyberChef |
| 中文 ASCII | 中文转数字串 | 按 GBK/Unicode 码点 |
| 锟斤拷 | 编码转换错误 | 试 GBK/UTF-8 互转 |
| 当铺密码 | 汉字笔画/数字 | 查表 |
| 日历密码 | 日期/星期 | 查日历 |
| Tupper 自指公式 | 图像公式 | 在线渲染 |
| 异形文字 | 原神/瓦坎达/羊文等 | 在线对照表 |

拿到不认识的编码，先用 CyberChef 的 Magic，再查 dCode、quipqiup 或对应在线工具。

## 五、现代加密

- **RC4**：流密码，密钥短，常出现在简单题
- **DES**：56 位密钥，已不安全；注意 ECB/CBC 和填充
- **AES-ECB**：相同明文块得到相同密文块，不需要 IV
- **AES-CBC**：需要 IV，注意 PKCS#7 填充和 IV 泄露
- **国密 SM4**：分组加密，常见于国内题目
- **emoji-aes**：把 AES 密文/密钥映射成 emoji，本质还是 AES
- **RSA**：看 `n/e/c`、小公钥指数、共模、已知明文、低加密指数
- **非对称加密**：公钥加密、私钥解密；私钥签名、公钥验签

常见工具：CyberChef、PyCryptodome、openssl、RsaCtfTool。

## 六、压缩包与明文攻击

ZIP 加密分 ZipCrypto 和 AES-256，每种又分 Store/Deflate。只有 **ZipCrypto** 适合已知明文攻击。

常见攻击：

- 伪加密：只改了加密标志位，数据本身没加密
- CRC 碰撞：已知 CRC 和文件长度，构造内容
- 暴力破解：数字/小字典优先
- 已知明文攻击：用 `bkcrack`，需要至少 12 字节已知明文
- 分卷合并、压缩包套娃、压缩包炸弹
- 密码是不可打印字符：看 hex，别只看文本
- 时间戳隐写、NTFS 数据流隐写

```bash
bkcrack -C secret.zip -c target.txt -p plain.txt
```

## 七、常见坑

- 把编码当加密，看到 Base64 就以为要密钥
- 忽略 URL 二次编码
- Base64 换行、填充缺失
- 古典密码不做词频分析，直接硬猜
- AES 分不清 ECB/CBC，找不到 IV
- RSA 不分解 `n` 就硬爆
- 压缩包不确认是 ZipCrypto 还是 AES
- 不检查文件真实类型和时间戳

> 参考：Lunatic BLOG《[持续更新] CTF-Misc Guide》<https://goodlunatic.github.io/posts/1ad9200/>
