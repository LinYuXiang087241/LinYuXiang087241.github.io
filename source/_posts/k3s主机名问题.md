---
title: k3s主机名问题
date: 2022-03-28 17:52:29
tags: 
  - k3s
categories: 
  - installation
---

# 相同主机名注册失败

- 解决办法
  - 1. 注册时添加 --with-node-id
  - 2. 注册时添加 `K3S_NODE_NAME=`或 `--node-name` 