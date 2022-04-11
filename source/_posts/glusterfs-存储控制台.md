---
title: glusterfs-存储控制台与监控
date: 2022-04-05 19:37:07
tags:
  - glusterfs
categories: 
  - storage
  - 分布式存储
---
# 安装红帽Gluster 存储控制台
<!-- more -->
- 需要开启的repo
  `jb-eap-7-for-rhel-7-server-rpms`
  `rhsc-3-for-rhel-7-server-rpms`
  `rhs-nagios-3-for-rhel-7-server-rpms`
- 安装必要软件包并执行配置脚本
  ```
  # yum install rhsc
  # rhsc-setup
  ```
---
# Nagios监控