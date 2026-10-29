# 博客撰写指南

这份指南说明如何在本仓库里写文章、放图片和其他静态资源，以及如何发布。

## 目录

- [快速开始](#快速开始)
- [文章目录与 URL](#文章目录与-url)
- [Front Matter 速查](#front-matter-速查)
- [图片与静态资源](#图片与静态资源)
- [封面图与 Hero 图](#封面图与-hero-图)
- [常用简码](#常用简码)
- [数学公式](#数学公式)
- [草稿、日期与发布](#草稿日期与发布)
- [常见问题](#常见问题)
- [发布前检查清单](#发布前检查清单)

## 快速开始

```bash
# 新建文章（推荐用 page bundle 形式）
hugo new content posts/my-new-post/index.md

# 本地预览（-D 显示草稿）
hugo server -D

# 只构建一次，检查是否有报错
hugo --minify
```

编辑 `content/posts/my-new-post/index.md`，把 `draft: true` 改成 `false`，推到 `main` 就会自动部署到 <https://mimimimimi39.github.io/>。

## 文章目录与 URL

推荐使用「文件夹 + `index.md`」的 page bundle，这样图片可以和文章放在一起：

```text
content/
└── posts/
    └── my-new-post/
        ├── index.md
        ├── feature.jpg        # 封面 / 缩略图（可选）
        └── images/
            ├── screenshot-1.png
            └── screenshot-2.png
```

- 默认 URL 由目录名决定：`content/posts/my-new-post/index.md` → `/posts/my-new-post/`
- 目录名建议用英文小写和连字符，不要有空格或中文
- 想换 URL 可以在 front matter 里加 `slug: "custom-url"`
- 单文件形式 `content/posts/foo.md` 也可以用，但没法把图片作为「页面资源」处理，不太推荐
- 文章列表页是 `content/posts/_index.md`，一般不用动

## Front Matter 速查

```yaml
---
title: "文章标题"
date: 2026-10-29
lastmod: 2026-10-30          # 可选，配合 showDateUpdated
draft: false
description: "给搜索引擎和分享卡片看的描述"
summary: "列表页显示的摘要，可选，不写会自动截取"
tags: ["CTF", "流量分析"]
categories: ["学习笔记"]
series: ["Hugo 建站"]        # 可选，系列文章
series_order: 1
slug: "custom-url"           # 可选，自定义 URL
showTableOfContents: true    # 可选，覆盖站点默认值
showHero: true
heroStyle: "basic"           # basic / big / background / thumbAndBackground
featureimage: "feature.jpg"  # 可选，见下文
featureimagecaption: "图片说明"
imagePosition: "50% 40%"     # 可选，封面图裁剪焦点
---
```

常用可覆盖的开关：`showDate`、`showAuthor`、`showBreadcrumbs`、`showReadingTime`、`showWordCount`、`showTaxonomies`、`showSummary`、`showPagination`、`sharingLinks`。

## 图片与静态资源

### 放哪里

| 位置 | 适合放什么 | 引用方式 | 是否经过 Hugo 处理 |
| --- | --- | --- | --- |
| 文章包内，如 `content/posts/my-new-post/images/a.png` | 文章正文配图、截图 | `![说明](images/a.png)` 或 `figure` 简码 | `figure` 会缩放优化 |
| `assets/`，如 `assets/img/foo.png` | 全站共享、需要在模板里处理的图（头像等） | `{{< figure src="img/foo.png" >}}` 或模板 `resources.Get` | 会 |
| `static/`，如 `static/files/demo.zip` | favicon、下载文件、robots.txt、不需要处理的资源 | `/files/demo.zip`、`/img/xxx.png` | 不会，原样拷贝 |

简单判断：

- **文章自己的图** → 放进文章文件夹（page bundle）
- **全站共用、要压缩/裁剪的图** → `assets/`
- **下载文件、图标、原样发布的文件** → `static/`

### 插入正文图片

Markdown 方式（按原图输出，不经过缩放）：

```md
![Wireshark 过滤结果](images/wireshark.png "可选的标题")
```

Blowfish 的 `figure` 简码（推荐，页面资源会被优化并支持点击放大）：

```md
{{< figure src="images/wireshark.png" alt="Wireshark 过滤结果" caption="在 Wireshark 中过滤 HTTP 请求" >}}
```

外链图片会原样引用，不会被处理：

```md
{{< figure src="https://example.com/pic.jpg" alt="外链图片" >}}
```

`nozoom=true` 可以关闭点击放大，`class` 可以附加样式类。

### 图片建议

- 格式：截图用 `.png`，照片用 `.jpg` / `.webp`；矢量图用 `.svg`
- 单张尽量控制在 500 KB 以内，长图先裁剪或压缩
- 文件名用英文小写、连字符，例如 `traffic-filter.png`
- 路径区分大小写：本地 Windows 可能不报错，但 Linux 上的 Actions 会 404
- 不要提交 `public/`，它已经在 `.gitignore` 里，由 CI 生成

### 头像与其他全站图片

- 头像：`assets/img/avatar.png`，在 `config/_default/languages.zh-cn.toml` 的 `params.author.image` 中引用
- favicon：`static/` 下的 `favicon.ico`、`apple-touch-icon.png` 等
- 下载附件：`static/files/xxx.zip`，文章里写 `[下载](/files/xxx.zip)`

## 封面图与 Hero 图

在文章文件夹里放一个以 `feature`、`cover`、`thumbnail` 或 `background` 开头的图片，主题会自动识别：

```text
content/posts/my-new-post/
├── index.md
└── feature.jpg
```

- 推荐直接命名 `feature.jpg` / `feature.png`
- 也可以在 front matter 里用 `featureimage: "feature.jpg"` 或 `featureimage: "img/foo.png"`（后者从 `assets/` 取）
- 封面图会用于文章列表缩略图和文章顶部 hero 图
- 想让社交分享卡片也带图，可以在 front matter 里加 `images: ["feature.jpg"]`
- `heroStyle` 可选 `basic`、`big`、`background`、`thumbAndBackground`
- `heroStyle: "big"` 时 `featureimagecaption` 才会显示
- 用 `imagePosition` 调整裁剪焦点，例如 `"50% 30%"`

不想显示 hero 图就在 front matter 写 `showHero: false`。

## 常用简码

提示框：

```md
{{< alert >}}
**注意：**这是一条提示。
{{< /alert >}}
```

```md
{{< alert icon="fire" cardColor="#e63946" iconColor="#1d3557" textColor="#f1faee" >}}
自定义颜色的警告。
{{< /alert >}}
```

引导段落：

```md
{{< lead >}}
这是一段开场白。
{{< /lead >}}
```

标签页：

````md
{{< tabs group="lang" default="Python" >}}
  {{< tab label="Python" >}}
  ```python
  print("hello")
  ```
  {{< /tab >}}
  {{< tab label="Bash" >}}
  ```bash
  echo hello
  ```
  {{< /tab >}}
{{< /tabs >}}
````

图片画廊（图片放在文章包里，例如 `gallery/01.jpg`）：

```md
{{< gallery >}}
  <img src="gallery/01.jpg" class="grid-w33" />
  <img src="gallery/02.jpg" class="grid-w33" />
  <img src="gallery/03.jpg" class="grid-w33" />
{{< /gallery >}}
```

轮播：

```md
{{< carousel images="gallery/*" aspectRatio="16-9" interval="3000" >}}
```

Mermaid 流程图：

```md
{{< mermaid >}}
graph LR;
  A[抓包] --> B[过滤]
  B --> C[导出对象]
{{< /mermaid >}}
```

更多简码（`video`、`youtubeLite`、`github`、`chart`、`timeline`、`steps`、`swatches` 等）参考主题文档：<https://blowfish.page/zh-cn/docs/shortcodes/>

## 数学公式

在文章里加一次 `katex` 简码，页面上的公式就会自动渲染：

```md
{{< katex >}}

行内公式：\(\varphi = \dfrac{1+\sqrt5}{2}\)

块级公式：

$$
E = mc^2
$$
```

注意仓库的 `config/_default/markup.toml` 已开启 passthrough，`\( \)` 和 `$$ $$` 不会被 Markdown 吃掉。

## 草稿、日期与发布

- `draft: true` 的文章只在 `hugo server -D` 下可见；GitHub Actions 构建时 `buildDrafts = false`，不会发布
- 日期在未来的文章默认不发布（`buildFuture = false`），写 `date: 2026-12-01` 的话要等到那天
- 发布流程：改完 → `hugo server -D` 本地预览 → 提交 → `git push origin main`
- 推送后到 Actions 页看 `Deploy to GitHub Pages`，成功后刷新线上站点
- 首次部署可能有几十秒 CDN 缓存延迟

## 常见问题

**图片 404**

- 路径大小写是否和文件名一致（Linux 区分大小写）
- 图片是否和 `index.md` 在同一个文章文件夹里
- 用 `figure` 时路径相对于文章目录，例如 `images/a.png`，不要写成 `content/posts/xxx/images/a.png`

**文章没出现**

- `draft` 是否为 `false`
- 日期是否在未来
- 文件是否放在 `content/posts/` 下，且文件名是 `index.md`（page bundle）

**本地构建提示主题版本不兼容**

主题声明的支持范围是 Hugo Extended 0.164.0 – 0.167.0。本机若是更新版本会出现警告，CI 固定使用 0.167.0，一般不影响；建议本地也装 0.167.0。

**克隆后主题目录是空的**

```bash
git submodule update --init --recursive
```

**更新主题**

```bash
git submodule update --remote themes/blowfish
hugo --minify
```

**想改站点配置**

- 站点基础：`config/_default/hugo.toml`
- 主题参数：`config/_default/params.toml`
- 作者与站点标题：`config/_default/languages.zh-cn.toml`
- 顶部菜单：`config/_default/menus.zh-cn.toml`

## 发布前检查清单

- [ ] 本地 `hugo server -D` 预览正常
- [ ] `draft: false`，日期不是未来
- [ ] 标题、描述、标签、分类已填写
- [ ] 图片路径正确、文件不大、文件名是小写英文
- [ ] 封面图命名为 `feature.*`（如需要）
- [ ] 代码块标注了语言
- [ ] `hugo --minify` 无报错
- [ ] 推送到 `main` 后 Actions 显示成功
