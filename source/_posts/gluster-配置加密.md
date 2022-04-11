---
title: gluster-配置加密
date: 2022-04-06 17:29:45
tags:
  - glusterfs
categories: 
  - storage
  - 分布式存储
---
# 配置glusterfs加密

<!-- more -->
- 配置文件
  - /etc/ssl/glusterfs.pem
  - /etc/ssl/glusterfs.key
  - /etc/ssl/glusterfs.ca
  - /var/lib/glusterd/secure-access
  
#### 为卷启用加密
- 启用加密
  ```
  # gluster volume set testvol server.ssl on
  # gluster volume set testvol client.ssl on
  # gluster volume set testvol auth.allow ip
  # gluster volume set testvol auth.ssl-allow ip
  ```