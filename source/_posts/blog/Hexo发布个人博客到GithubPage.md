---
title: Hexo发布个人博客到GithubPage
date: {{ date }}
categories:
    - Blog
---

## 摘要

2025年3月报名了开源操作系统训练营(Rust)，记录学习过程以及遇到的一些问题，开始通过Bolg的方式记录下来。以下内容介绍通过Hexo框架将博客发布到GithubPage上。

## 准备工具

- node
- git
- hexo
- github

## Github创建仓库

在github中创建仓库，仓库的名称必须为\<github username\>.github.io且为publish。

## 配置Github workflow实现自动部署

添加workflow配置文件：`.github/workflows/pages.yml`

```yml
name: Pages

on:
  push:
    branches:
      - main # default branch

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          # If your repository depends on submodule, please see: https://github.com/actions/checkout
          submodules: recursive
      - name: Use Node.js 20
        uses: actions/setup-node@v4
        with:
          # Examples: 20, 18.19, >=16.20.2, lts/Iron, lts/Hydrogen, *, latest, current, node
          # Ref: https://github.com/actions/setup-node#supported-version-syntax
          node-version: "20"
      - name: Cache NPM dependencies
        uses: actions/cache@v4
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

## 修改配置文件`_config.yml`

```yml
deploy:
  type: 'git'
  repo: git@github.com:monkey666-cr/\<github username\>.github.io.git
  branch: main
```