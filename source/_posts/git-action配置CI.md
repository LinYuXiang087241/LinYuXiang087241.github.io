---
layout: posts
title: git_action配置CI
date: 2022-03-25 23:49:57
tags: 
  - github
  - CI
categories: 
  - configuration
---

# 通过配置`git_action`实现`hexo`博客自动发布
- 1 初始化博客，创建并切换 `gh-pages branch` 以及推送
  ```
# git init
# git remote show
# git remote add origin git@github.com:LinYuXiang087241/LinYuXiang087241.github.io.git
# git push  --set-upstream origin gh-pages
  ```
- 2. 创建对应 `github actions` 实现 `hexo` 自动编译发布至 `master branch`
     其中github token <font color=red>ACCESS_TOKEN</font>将会在下一步创建，
	 其他的信息填对应的即可
<pre><code># cat .github/workflows/main.yml
name: WITH CI
# 只监听 gh-pages 分支的改动
on:
  push:
    branches: [ gh-pages ]
jobs:
  Blog_CI:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2  #软件市场的名称
        with: # 参数
          submodules: false
      # 这里用的是 Node.js 13.x，14.x 生成 Hexo 静态页面会有问题
      - name: Set up Node.js
        uses: actions/setup-node@v1
        with:
          node-version: '16.x'
      # 配置Hexo环境 
      - name: Setup Hexo
        env:
          ACTION_DEPLOY_KEY: ${{ secrets.<font color=red>ACCESS_TOKEN</font> }}
        run: |
          npm install hexo-cli -g
          npm install
      # 生成静态文件
      - name: Build
        run: |
          hexo clean 
          hexo g
      # 部署到 GitHub Pages
      - name: Upload GitHub Repository
        env: 
          # Github 仓库
          GITHUB_REPO: <font color=blue>github.com/LinYuXiang087241/LinYuXiang087241.github.io</font>
         # 将编译后的博客文件推送到指定仓库
        run: |
          cd ./public && git init && git add .
          git config user.name <font color=blue>"LinYuXiang087241"       #username改为你github的用户名</font>
          git config user.email <font color=blue>"1524204934@qq.com"     #username改为你github的注册邮箱</font>
          git add .
          git commit -m "GitHub Actions Auto Builder at $(date +'%Y-%m-%d %H:%M:%S')"
          git push --force --quiet "https://${{ secrets.<font color=red>ACCESS_TOKEN</font> }}@$GITHUB_REPO" master:master</code></pre>
- 3. 创建github `access token`并创建hexo repo 的`secret`
  - 3.1 创建`access token`: `Settings ` > `Developer settings` > `Personal access tokens` > `Generate new token` 并复制好token
        ![one](/images/3-26/1.png)  
  - 3.2 创建对应repo secret: `Settings` > `Secrets` > `Actions` > `New repository secret` secret的名称改为`ACCESS_TOKEN`，并粘贴token
        ![TWO](/images/3-26/2.png)  
- 4. 每次推送至 gh-pages 分支自动发布
**<u><font color=red>仅用作经验记录</font></u>**