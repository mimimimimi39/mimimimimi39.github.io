---
title: "用 Hugo 和 Blowfish 搭建这个博客"
date: 2026-10-15
description: "记录本站的搭建过程与配置要点。"
tags: ["Hugo", "Blowfish", "博客"]
categories: ["教程"]
---

这个站点用 [Hugo](https://gohugo.io/) 生成，主题是 [Blowfish](https://blowfish.page/)，部署在 GitHub Pages 上。

## 本地预览

```bash
hugo server -D
```

打开 <http://localhost:1313> 就能看到效果。

## 写一篇文章

```bash
hugo new content posts/my-new-post/index.md
```

然后编辑 `content/posts/my-new-post/index.md`，把 `draft` 改成 `false` 即可。

## 目录结构

| 路径 | 作用 |
| --- | --- |
| `config/_default/` | 站点与主题配置 |
| `content/posts/` | 博客文章 |
| `assets/img/` | 头像等需要处理的图片 |
| `static/` | 原样拷贝的静态文件 |
| `themes/blowfish/` | 主题（git submodule） |

## 更新主题

```bash
git submodule update --remote themes/blowfish
```

提交后 GitHub Actions 会自动重新构建并部署。
