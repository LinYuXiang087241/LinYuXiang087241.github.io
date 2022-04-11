---
title: github_配置https免密码
date: 2022-04-11 21:20:32
tags:
  - hexo
  - github
  - https免密
categories: 
  - configuration
---
# github_配置https免密码
### 创建token参考 git-action配置CI 的博客
<html><pre><code>前提：之前搭建hexo，配置了如何使用ssh免密推送。
本文主要讲述如何配置https免密push到github。</code></pre></html>
<!-- more -->

- 前提：
  - 创建的权限较大的token，可以配置永不过期token
- 在环境中执行以下命令：
  ```
  git remote set-url origin https://<github username>:<github_token>@github.com/<githuab username>/<github repository>.git
  ```
- 为保证网络的稳定，配置 git proxy
  ```
  # git config --global http.proxy http://192.168.31.115:1081
  # git config --global https.proxy https://192.168.31.115:1081
  # git config --list | grep proxy
  ```