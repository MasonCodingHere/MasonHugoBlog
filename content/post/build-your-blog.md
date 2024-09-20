---
title: "Hugo + GitHub Pages搭建个人博客"
description: 
categories:
  - "Hugo"
tags:
  - "Hugo"
date: 2024-09-20T15:18:02+08:00
image: https://www.tomasbeuzen.com/post/making-a-website-with-hugo/featured_hu40cbd56aa319431e2f94c340d268efa8_55522_720x0_resize_lanczos_3.png
slug: "build-your-blog"
math: 
license: 
hidden: false
draft: false
---

# 前言

本文待完善。

# 整体思路

使用`Hugo`作为博客框架，在本地编写`Markdown`文件，构成博客站点源码库。将源码库上传至GitHub仓库`MasonHugoBlog`，并设置GitHub Action，由GitHub Action运行`hugo`命令生成博客HTML文件并上传至`MasonCodingHere.github.io`，最终由GitHub Pages完成部署。

所以需要两个GitHub仓库。

1. `MasonHugoBlog`：用于存储站点源码。
2. `MasonCodingHere.github.io`：用于存储生成的HTML文件。

# Hugo安装与配置

参考[Hugo官方文档](https://gohugo.io/categories/installation/)，根据操作系统选择安装方式。本文以macOS为例。

## 前提

Hugo依赖[Git](https://git-scm.com/)和[Go](https://go.dev/)，所以要先安装这两个。

> 不展开讲了。

## 安装hugo

本文采用Homebrew的安装方式

```bash
brew install hugo
```

安装结束，执行以下命令，看到版本号说明安装成功。

```bash
hugo version
```

## 新建站点

在你希望的位置执行下面的命令，新建一个Hugo站点。

```bash
hugo new site MasonHugoBlog
```

这样会出现一个名为`MasonHugoBlog`的文件夹，这就是博客源码。

## 选择主题

在[官方主题库](https://themes.gohugo.io/)选择一个主题，当然也可以去GitHub找官方主题库里没有的主题。本文选择[Stack主题](https://stack.jimmycai.com/)。

本文选择Git Submodule的方式安装。

```bash
cd MasonHugoblog
git init
git submodule add https://github.com/CaiJimmy/hugo-theme-stack/ themes/hugo-theme-stack
```

这样`theme`文件夹下就会出现该主题。我想让我的博客跟Stack主题提供的[Demo站点](https://demo.stack.jimmycai.com/)一样，所以做了一下事情。

1. 把`MasonHugoblog/themes/Hugo-theme-stack/exampleSite/content/`下的内容复制到`MasonHugoblog/content/`。
2. 删除`MasonHugoblog/hugo.toml`，把`MasonHugoblog/themes/Hugo-theme-stack/Hugo.yaml`复制到`MasonHugoblog/`。

这样执行以下命令，在本地预览，发现已经跟Demo一致。

```bash
hugo server -D
```

## 配置博客

各种个性化配置，包括该博客名称、换头像、改链接、去除自己不需要的多语言支持等等，在`hugo.yaml`中配置。

删除默认博文、删除默认分类和标签、修改博文默认模版等等。

> 不展开说了。

# GitHub Actions

## Push to GitHub

接下来就是把整个文件夹推到GitHub，在GitHub新建一个仓库MasonHugoBlog。把本地的`MasonHugoBlog`文件夹推到这个仓库。不细说了。

之后，因为我们以后写文章、更新博客，都是要推这个文件夹，所以写一个脚本更方便，在`MasonHugoBlog`下新建一个名为`deploy.sh`的脚本文件，把以下内容复制粘贴过去。这样以后写完md文件，直接执行`./deploy.sh`就完成部署了。

```shell
#!/bin/bash

# Check if the directory /public exists
if [ -d "./public" ]; then
  # If it exists, delete the directory
  rm -rf ./public
  echo "./public directory has been deleted."
else
  # If it doesn't exist, print a message
  echo "./public directory does not exist."
fi

# Check if a commit message was provided as an argument
if [ -z "$1" ]; then
    # If no commit message is provided, use the current date and time
    COMMIT_MSG="Update at $(date)"
else
    # Use the provided commit message
    COMMIT_MSG="$1"
fi

# Add all changes
git add .

# Commit changes with the provided message
git commit -m "$COMMIT_MSG"

# Push changes to the 'main' branch
git push origin main

# Print a success message
echo "Changes have been pushed to GitHub successfully!"
```

> 因为我们想这个库只保存源码，所以生成的静态HTML文件我们不要，所以在上传之前先检测有无`/public`文件夹，有的话删掉再上传。

## Deploy Key

在本地用sshkeygen生成一对key

公钥给MasonCodingHere.github.io的Deploy key

私钥给MasonHugoBlog的Actions的secrets。

MasonHugoBlog的Workflow permissions改为Read and write permissions

## GitHub Actions

给MasonHugoBlog添加Actions

```yml
name: Deploy Hugo to GitHub Pages

on:
  push:
    branches:
      - main  # Trigger deployment when changes are pushed to the main branch

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout the repository
        uses: actions/checkout@v2
        with:
          submodules: true

      - name: Set up Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.134.2'
          extended: true
      
      - name: Build the website
        run: hugo

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          deploy_key: ${{ secrets.DEPLOY_KEY }}
          publish_dir: ./public  # This is the directory where Hugo generates static files
          external_repository: MasonCodingHere/MasonCodingHere.github.io
          publish_branch: main
```

# GitHub图床

每个repo有1G的大小限制，如果把图片也传到博客仓库，很快就会满。

在GitHub新建一个图床库`PicsBed_1`

生成一个Token，选public_repo

安装PicGo，设置Token。

为PicsBed_1添加Actions如下

```yml
name: Update README with Directory Structure

on:
  push:
    branches:
      - main

jobs:
  update-readme:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Generate directory structure
        run: |
          echo "# Directory Structure" > structure.txt
          echo "\`\`\`" >> structure.txt
          tree -I ".git|node_modules|structure.txt" >> structure.txt
          echo "\`\`\`" >> structure.txt

      - name: Update README
        run: |
          cat structure.txt > README.md

      - name: Commit changes
        run: |
          git config --local user.name "GitHub Action"
          git config --local user.email "action@github.com"
          git add README.md
          git commit -m "Update directory structure in README"
          git push
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```



# References

> 1. [Hugo官方文档](https://gohugo.io/documentation/)
> 1. [Stack主题官方网站](https://stack.jimmycai.com/)
> 1. [Stack主题官方GitHub仓库](https://github.com/CaiJimmy/hugo-theme-stack)
> 1. [图片上传工具PicGo官网](https://molunerfinn.com/PicGo/)
> 1. [图片上传工具PicGo官方GitHub仓库](https://github.com/Molunerfinn/PicGo)
