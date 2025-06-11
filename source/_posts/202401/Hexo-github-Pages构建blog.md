---
title: Hexo + Github Pages搭建blog
date: 2024-01-17 22:00:19
category:
- experiment
tags: 
- Github Pages
---
介绍Hexo + Github Pages（Actions部署）搭建静态blog的过程

> [Hexo官网](https://hexo.io/zh-cn/docs/)

> [fluid官网](https://hexo.fluid-dev.com/docs/start/#%E4%B8%BB%E9%A2%98%E7%AE%80%E4%BB%8B)
### 一、环境准备
```shell
1. node - 20.11.0
2. Git
```

### 二、Hexo安装步骤

#### 1.安装hexo，这里选择全局安装
```shell
npm install -g hexo-cli
```

#### 2.初始化--有node版本限制，报错注意看提示
```shell
hexo init <folder>
cd <folder>
npm install
```

#### 3.修改配置 _config.yml文件
主要将url修改为Github Pages的地址
```yaml
url: https://chenzhenhong.Github.io/blog
```

#### 4.安装fluid主题（optional）
```shell
npm install --save hexo-theme-fluid
touch _config.fluid.yml
```
将[fluid配置文件](https://Github.com/fluid-dev/hexo-theme-fluid/blob/master/_config.yml)的内容拷贝到_config.fluid.yml

修改_config.yml的theme
```yaml
theme: fluid
```
首次使用创建 [关于页]
```shell
hexo new page about
```
创建成功后修改/source/about/index.md，添加 layout 属性。
内容如下
```yaml
---
title: 标题
layout: about
---

这里写关于页的正文，支持 Markdown, HTML
```

#### 5.运行预览
```shell
hexo server
```

#### 6.Github Actions配置
- 开启Github Pages，source改成Github Actions

- 目录下.Github -> workflows -> 添加pages.yml，内容如下
```yaml
name: Pages

on:
  push:
    branches:
      - master

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          submodules: recursive
      - name: Use Node.js 20.11.0
        uses: actions/setup-node@v4
        with:
          node-version: '20.11.0'
      - name: Cache NPM dependencies
        uses: actions/cache@v3
        with:
          path: node_modules
          key: ${{ runner.OS }}-npm-cache
          restore-keys: |
            ${{ runner.OS }}-npm-cache
      - name: Install Dependencies
        run: npm install
      - name: Build
        run: npm run build
      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
  deploy:
    needs: build
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```
