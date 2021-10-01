categories:
- 杂
tags:
- wp
title: 用github action 让hexo体验++
date: 2021-10-02
---

# 用github action 让hexo体验++

## 起因

因为电脑机械硬盘不知道啥原因，有时候磁盘上某个文件的数据就坏掉了

然后 node_modules 或者某篇文章就GG了，虽然现在把那个机械换掉了

但是每次都要`hexo deploy`好麻烦啊

于是打算搞个 github action 自动生成静态文件，我只要关注 source 还有 config 就好了

## 设计

我们需要把 配置，文章，hexo生成的静态文件隔离开来，显然直接把它们放在不同的分支就好了

新建三个分支：master(存放文章)，config(存放配置，敏感配置放到secret中)，website(存放生成的网站)

github action 的逻辑：

```
安装 hexo 环境

克隆文章分支，克隆配置分支，克隆主题

配置文件字符串替换(如果有敏感数据的话)

生成静态文件

submit
```

## 实现

```yml
on:
  workflow_dispatch:
  push:
    branches:
      - master
      - config

# 自定义环境变量


jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          ref: config
      - name: Checkout
        uses: actions/checkout@v2
        with:
          ref: master
          path: source
      - name: Checkout
        uses: actions/checkout@v2
        with:
          ref: website
          path: public
      - uses: actions/setup-node@v2
        with:
          node-version: '14'
          cache: 'npm'
      - run: npm install


      # 生成并部署
      - name: Deploy
        run: |
          npx hexo generate
          cd public 
          git config --global user.name 'ccreater222'
          git config --global user.email 'ccreater222@users.noreply.github.com'
          git add *
          git commit -am "Automated generation"
          git push origin website
```