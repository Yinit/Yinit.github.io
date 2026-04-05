# 个人博客

## 环境配置

### 1. 安装依赖

```bash
# 安装 Hugo extended 版（必须用 extended，支持 SCSS）
HUGO_VERSION="0.145.0"
wget "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb" -O /tmp/hugo.deb
sudo dpkg -i /tmp/hugo.deb

# 验证安装
hugo version
```

### 2. 拉取仓库并初始化主题

PaperMod 主题以 git submodule 方式管理，clone 后需要单独初始化：

```bash
git clone git@github.com:thekingking/thekingking.github.io.git
cd thekingking.github.io

# 初始化并拉取 submodule（PaperMod 主题）
git submodule update --init --recursive
```

---

## 常用命令

### 本地预览

```bash
# 启动本地服务器（包含草稿文章）
hugo server -D

# 指定端口 + 允许局域网/远程访问
hugo server -D --bind 0.0.0.0 --port 1313

# 关闭快速渲染（修改后全量重建，排查样式问题时用）
hugo server -D --disableFastRender
```

访问 http://localhost:1313 预览。

### 新建文章

```bash
# 在 content/posts/ 下新建文章（会自动套用 archetypes/default.md 模板）
hugo new posts/文章名称.md
```

文章头部 Front Matter 说明：

```yaml
---
title: "文章标题"
date: 2026-04-05
lastMod: 2026-04-05
draft: true          # true=草稿（本地 -D 可见，不会发布）
author: ["tkk"]
categories: []
tags: []
description: ""      # SEO 描述
summary: ""          # 首页摘要
weight:              # 填 1 可置顶
comments: false
mermaid: true        # 是否启用 mermaid 图表
---
```

### 构建静态文件

```bash
# 生成静态文件到 public/ 目录
hugo

# 构建时清理旧文件（推荐）
hugo --cleanDestinationDir
```

---

## 发布到 GitHub Pages

```bash
# 提交内容更新
git add .
git commit -sm "新增文章: xxx"
git push origin main
```

> 博客通过 GitHub Actions 自动构建并部署，push 到 main 分支后自动生效。

---

## 主题管理（PaperMod）

```bash
# 更新 PaperMod 到最新版
git submodule update --remote --merge themes/PaperMod

# 查看当前 submodule 状态
git submodule status
```

---

## 目录结构说明

```
├── config.yaml          # 站点全局配置（主题、菜单、参数等）
├── go.mod               # Hugo Module 依赖（PaperMod 版本锁定）
├── archetypes/
│   └── default.md       # hugo new 新建文章时的 Front Matter 模板
├── assets/
│   ├── css/extended/    # 自定义 CSS 样式（会覆盖主题默认样式）
│   └── js/              # 自定义 JS（如 pangu.js 中英文间距）
├── content/
│   └── posts/           # 博客文章（Markdown 文件）
├── layouts/
│   ├── partials/        # 自定义模板片段（评论、版权、TOC 等）
│   └── shortcodes/      # 自定义短代码（notice、quote、github 卡片等）
├── static/
│   ├── favicon/         # 网站图标
│   ├── fonts/           # 字体文件
│   └── images/          # 文章图片资源
├── themes/PaperMod/     # PaperMod 主题（git submodule）
└── public/              # hugo 构建输出（不需要手动修改）
```

---

## 参考文档

- [Hugo 官方文档](https://gohugo.io/documentation/)
- [PaperMod 主题文档](https://github.com/adityatelange/hugo-PaperMod/wiki)
- [PaperMod FAQ](https://github.com/adityatelange/hugo-PaperMod/wiki/FAQs)

