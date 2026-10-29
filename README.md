# みみみみみ 的博客

基于 [Hugo](https://gohugo.io/) 和 [Blowfish](https://blowfish.page/) 主题，通过 GitHub Actions 自动部署到 GitHub Pages。

## 本地开发

需要 Hugo Extended（主题支持 0.164.0 – 0.167.0，CI 使用 0.167.0）以及 git。

```bash
git clone --recursive git@github.com:mimimimimi39/mimimimimi39.github.io.git
cd mimimimimi39.github.io
hugo server -D
```

如果克隆时忘了 `--recursive`：

```bash
git submodule update --init --recursive
```

## 写文章

```bash
hugo new content posts/my-new-post/index.md
hugo server -D
```

文章放在 `content/posts/`，`draft: true` 的文章只在 `hugo server -D` 下可见。

## 配置

- 站点：`config/_default/hugo.toml`
- 主题参数：`config/_default/params.toml`
- 语言与作者信息：`config/_default/languages.zh-cn.toml`
- 导航菜单：`config/_default/menus.zh-cn.toml`

## 部署

推送到 `main` 后，`.github/workflows/deploy.yml` 会构建并发布到 GitHub Pages。

仓库 Settings → Pages → Build and deployment → Source 需要选择 **GitHub Actions**。

## 更新主题

```bash
git submodule update --remote themes/blowfish
```

## 目录

| 路径 | 说明 |
| --- | --- |
| `content/` | 文章与页面 |
| `assets/img/` | 需要 Hugo 处理的图片（头像等） |
| `static/` | 原样拷贝的静态文件（favicon 等） |
| `themes/blowfish/` | Blowfish 主题（git submodule） |
| `config/_default/` | 站点配置 |
