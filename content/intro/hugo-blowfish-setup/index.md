---
title: "用 Hugo 和 Blowfish 搭建这个博客"
date: 2026-10-15
draft: false
description: "记录本站的搭建过程与配置要点。"
summary: "用 Hugo、Blowfish 和 GitHub Pages 搭建本站的过程与目录说明。"
categories: ["介绍"]
aliases: ["/posts/hugo-blowfish-setup/"]
showTableOfContents: true
---

这个站点由 [Hugo](https://gohugo.io/) 生成，主题是 [Blowfish](https://blowfish.page/)，部署在 GitHub Pages 上。

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
| `content/posts/` | 博客文章 |
| `content/intro/` | 站点介绍（本文所在） |
| `content/about/` | 关于页 |
| `assets/img/` | 需要 Hugo 处理的图片（头像等） |
| `static/` | 原样拷贝的静态文件（favicon 等） |
| `themes/blowfish/` | Blowfish 主题（submodule） |
| `config/_default/` | 站点与主题配置 |

## 部署

推送到 `main` 后，GitHub Actions 会自动构建并发布到 Pages。
