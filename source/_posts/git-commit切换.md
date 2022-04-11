---
title: git_commit切换
date: 2022-04-11 20:55:24
tags:
  - github
  - change commit
categories: 
  - configuration
---

# git commit 切换
### 问提回忆
<html><pre><code>在commit提交后，发现当前工作目录不小心存在一个大小为1G的image镜像包。
导致后续push都由于大小原因无法成功。通过reset重置commite后，发现近期修改消失。
通过reflog找回最新的内容，并重新成功push的过程 </code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

- 1. 查看当前所在commit，并恢复到对应commit
  ```
  # git reflog show
  1e29262 HEAD@{0}: reset: moving to HEAD
  1e29262 HEAD@{1}: reset: moving to HEAD
  1e29262 HEAD@{2}: commit: 411-2     <<<
  8d7c706 HEAD@{3}: commit: 411-1
  5e73698 HEAD@{4}: commit: 411
  1374dac HEAD@{5}: commit: new
  61139a1 HEAD@{6}: commit: latest

  # git reset --hard 1e29262 
  ```
- 2. 通过reflog恢复到之前的commit
  ```
  # git reflog show
  1e29262 HEAD@{0}: reset: moving to 1e29262  <<<切换回该commit
  1e29262 HEAD@{1}: reset: moving to HEAD
  1e29262 HEAD@{2}: reset: moving to HEAD
  1e29262 HEAD@{3}: commit: 411-2
  8d7c706 HEAD@{4}: commit: 411-1
  5e73698 HEAD@{5}: commit: 411
  ```