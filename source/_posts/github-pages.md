---
title: GitHub Pages 免费托管静态网站
date: 2026-04-21 12:00:00
categories:
  - 技术
tags:
  - GitHub
  - 静态网站
  - 托管
---

GitHub Pages 是 GitHub 提供的免费静态网站托管服务，非常适合个人博客、项目主页等。

## 主要特点

- **免费托管** - 完全免费，无需付费
- **自定义域名** - 支持绑定自己的域名
- **SSL 支持** - 自动启用 HTTPS
- **版本控制** - 利用 Git 进行版本管理

## 使用方法

### 方式一：用户/组织站点

1. 创建名为 `username.github.io` 的仓库
2. 将静态文件推送到仓库的 `main` 分支
3. 访问 `https://username.github.io`

### 方式二：项目站点

1. 在项目仓库中创建 `gh-pages` 分支
2. 将静态文件推送到该分支
3. 访问 `https://username.github.io/project-name`

## 自动化部署

结合 GitHub Actions，可以实现推送到仓库后自动部署：

```yaml
name: Deploy
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run build
      - uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

GitHub Pages 是静态网站托管的绝佳选择！
