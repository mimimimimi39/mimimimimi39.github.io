---
title: "mimi(qbot)"
date: 2026-10-30
description: "从协议端选型到多模态、联网、定时任务与 QQ 空间，一次完整的 QQ Bot 搭建复盘。"
tags: ["qqbot", "python", "napcat", "deepseekv4.1", "image2.5"]
categories: ["项目"]
---

# mimi

我的专属q群aibot，大致人设为一个120岁的精灵女仆（容貌和心智相当于12岁人类小孩），身高142cm，雪白色长发，天蓝色眼睛，穿水色连衣裙，性格可爱温柔。唯一的主人是「みみみみみ」
代理qq：3642694923
LLM：deepseekV4.1 flash
image：image2.5
- 自行作画展示
  ![自画](\mimi1.jpg) ![自画](\mimi2.jpg) ![自画](\mimi3.jpg)
- 部分动态效果及日常对话展示
  ![日常](\richang.jpg) ![动态](\dongtai.jpg)


- 真的是非常可爱呢（感叹）

## 一、现有功能

1. 私聊对话（独立人格、独立记忆）
2. 群聊里只响应 @ 她的消息，并回复到群
3. 图片识别（理解对方发的图）
4. 出图（image2.5，能在对话中按需作画）
5. 引用消息理解
6. 睡眠状态机（主人专属开关）
7. 群聊与私聊消息互通
8. 通过对话修改配置
9. 联网搜索
10. 定时主动搭话、定时发 QQ 空间说说（可带图，需审批）
11. 每日 token 消耗统计
12. 配置文件可读性改造（JSONC 注释 + 分条人设）

## 二、最终架构

```
QQ 小号 ←→ NapCat（QQ NT，OneBot 11）
              │  WS 3001（事件推送）
              │  HTTP 3000（API 调用）
              ▼
        Python 桥接 bot.py
              ├── DeepSeek API（对话 / 工具调用）
              ├── image2.5 API（出图）
              └── QQ 空间 API（send_qzone_msg）
```

- 协议端：NapCat.Shell + 本地 QQ NT 客户端
- 桥接：纯 Python 标准库 + websocket-client
- 配置：config.json（JSONC，支持注释与数组人设）
- 状态：state.json（睡眠/定时任务）、usage.json（token 统计）

## 三、协议端选型：Lagrange 的失败与 NapCat 的胜利

**第一版选了 Lagrange.Core**，折腾半天后卡在登录：

```
System.Exception: Signer server returned a NotFound
Unhandled exception. System.Exception: All login failed!
```

原因：Lagrange 依赖签名服务器，官方公开服务已停，没有签名服务就登不上。

**结论**：换 NapCat。它基于官方 QQ NT 客户端注入，扫码即登录，不需要签名服务。

踩过的两个小坑：

- NapCat 需要管理员权限注入，`launcher.bat` 会弹 UAC
- 首次运行报 `Cannot find module .../loadNapCat.js`，手动把 `loadNapCat.js` 复制到 QQ 版本目录的 `resources/app/` 下解决

## 四、核心实现

### 1. OneBot 双通道

NapCat 侧开启：

- `Http` 服务（3000）：桥接主动调 API 发消息
- `WebSocketServers`（3001）：事件推送

**为什么最后用 WS 而不是 HTTP 上报？**

NapCat 的 `httpClients` 上报通道在启动时连接失败一次后就不再重试，导致 QQ 收到了消息但桥接完全收不到。改成 WS 长连接（带自动重连）后彻底稳定。

### 2. 消息处理管线

```
收到事件
  ├─ 判断私聊 / 群聊
  ├─ 群聊：只有 @ 自己才继续
  ├─ 白名单 / 主人身份判定
  ├─ 睡眠状态：主人正常，其他人回 zZZ...
  ├─ 渲染文本（@ 转昵称、清理 CQ 码、带时间戳）
  ├─ 下载图片 → base64（视觉输入）+ 本地存档
  ├─ 引用消息：get_msg 拉原文和原图
  ├─ 命令识别（/config /usage /sleep /wake）
  ├─ 触发词识别（发说说）
  └─ 交给 LLM（带工具调用循环）
```

每条消息都会在开头注入：

```
（发送者QQ：2235228080，身份：主人｜现在时间：2026-10-30 周五 13:38）
```

让模型准确识别身份和时间，不再「下午说早安」。

### 3. 出图管线

模型按约定输出：

```
好呀～想看看mimi画的小猫吗喵 (๑>ᴗ<๑)
[DRAW]a cute chibi kitten, pastel colors, kawaii style[/DRAW]
```

桥接解析后：

1. 先把她说的那句话发出去
2. `subprocess` 调 `imagegen.py` 生成图片
3. 通过 OneBot `image` 段（`file:///` 本地路径）发图

超时保护：单次 180 秒，超时 `taskkill /T /F` 杀掉进程树，默认再重试一次。

### 4. 说说（QQ 空间）

NapCat 只提供了两个接口：

- `send_qzone_msg`：`{content, images[]}`
- `delete_qzone_msg`：`{tid}`

于是做了「草稿 → 预览 → 自然语言修改 → 发布」的流程：

- 「发条说说，写今天下雨」→ 出草稿推送到私聊
- 「配图换成雨中的小猫」→ 重新生成文字和配图
- 「发吧」→ 发布；「算了」→ 取消

**多图**：消息带图 + 「再配一张你自己画的」→ 原图 + 她画的图一起发。

### 5. 联网

用 DeepSeek 的 function calling：

- `web_search`：Bing RSS 接口（比 HTML 稳定）
- `fetch_url`：抓网页正文

多轮工具循环最多 8 轮，最后两轮不再提供工具，强制输出答案。中途兜底：空回复时再请求一次纯文本。

### 6. 定时任务

`scheduler_loop` 每 20 秒检查一次：

- 08:00 自动起床 + 早安
- 00:00 自动入睡 + 晚安
- 每 100 分钟主动闲聊（睡着时跳过）
- 23:30 自动发一条说说（无需审批）

状态存 `state.json`，重启不会重复触发。

### 7. Token 统计

每次 API 响应里的 `usage` 都按天累加到 `usage.json`，主人可用 `/usage` 或直接问「今天用了多少 token」。

### 8. 配置也能对话改

```
/config get reply_delay
/config set reply_delay [1,2]
/config set qzone_enabled false
```

支持点路径（`llm.model`），改完立即热重载并写回文件。

## 五、踩坑清单

| 现象 | 根因 | 解决 |
|---|---|---|
| Lagrange 登录失败 | 签名服务器 404 | 换 NapCat |
| 桥接日志全是乱码 | GBK 控制台打印 emoji 抛异常 | `sys.stdout.reconfigure(utf-8)` + 日志写入 try/except |
| 出图 SSL EOF | 走了系统代理 | 子进程环境变量 `no_proxy=*` |
| WS 一直 10061 | 只启动了桥接，NapCat 没开 | 一键脚本先健康检查再启动 |
| 收不到消息但 QQ 有记录 | NapCat HTTP 上报通道失效 | 改 WebSocket |
| 搜索返回「玄」字解释 | HTML 解析抓到词典卡片 | 换 Bing RSS + 相关性过滤 |
| 回复「这次没想出来」 | 工具轮次耗尽 / max_tokens 被思考吃光 | 轮次提到 8、最后强制输出、兜底请求 |
| 回复里出现 `<||DSML||>` | 禁用工具时模型吐出内部标记 | 后几轮直接不传 tools + 正则清理 |
| 全部回复「脑袋卡住了」 | `max_tokens: 0` 被原样传给 API（400） | 0 视为「不传」 |
| 配置改完读不了 | 加了 JSONC 注释，`json.load` 不认 | 自写 `strip_jsonc` |
| bat 双击乱码不执行 | UTF-8 中文被 cmd 按 GBK 解析 | 脚本转 GBK 编码 |
| 手动编辑后配置失效 | 数组元素漏逗号 | 补逗号 + 日志暴露解析错误 |

## 六、部署形态

```
C:\AAA\AiQQ\
├── NapCat\                # 协议端
├── bridge\
│   ├── bot.py             # 桥接主程序
│   ├── config.json        # JSONC 配置
│   ├── state.json         # 睡眠/定时状态
│   ├── usage.json         # token 统计
│   ├── images\            # 出图 + 收到的图
│   └── bot.log
├── restart_napcat.ps1     # 提权重启协议端
└── start_all.bat          # 一键启动（UAC → 健康检查 → 桥接）
```

## 七、后续计划

1. 让她越来越像真人，目前在发动态或是画图时还是会要求指令输出
2. 接入符合人设的语言模型
3. 接入视频模型
