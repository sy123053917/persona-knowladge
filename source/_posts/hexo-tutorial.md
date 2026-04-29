---
title: Hexo 博客搭建教程
date: 2026-04-21 11:00:00
categories:
  - 教程
tags:
  - Hexo
  - 博客
  - GitHub Pages
---

# 使用 Hexo 搭建静态博客

Hexo 是一个快速、简洁且高效的博客框架，使用 Markdown 来撰写文章，自动生成静态网页。

## 安装 Hexo

首先确保已安装 Node.js，然后执行：

```bash
npm install -g hexo-cli
hexo init blog
cd blog
npm install
```

## 更换主题

Hexo 有丰富的主题可供选择，推荐使用 NexT 主题：

```bash
npm install hexo-theme-next
```

然后在 `_config.yml` 中设置：

```yaml
theme: next
```

## 部署到 GitHub Pages

安装部署插件：

```bash
npm install hexo-deployer-git
```

配置 `_config.yml`：

```yaml
deploy:
  type: git
  repo: https://github.com/username/repo.git
  branch: gh-pages
```

然后执行部署：

```bash
hexo clean
hexo generate
hexo deploy
```

## 常用命令

- `hexo new "文章标题"` - 创建新文章
- `hexo server` - 本地预览
- `hexo generate` - 生成静态文件
- `hexo deploy` - 部署到服务器

祝你博客之旅愉快！
