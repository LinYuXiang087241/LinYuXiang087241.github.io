---
title: glusterfs-管理员全记录（四）
date: 2022-04-23 18:19:55
tags:
  - gluster
categories:
  - storage
  - 分布式存储
---

<h2>glusterfs 学习记录 <font color=yellow>四</font></h2>

![22-412-g1](/images/22412/g1.png)
<html><pre><code>前提：基于红帽商业版glusterfs 3.5
本文主要内容：client端管理</code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

## 原生客户端升级
- 不同版本的repo
  - rhel6
    - rhel-6-server-rpms 
	- rhel-6-server-rhs-client-1-rpms
  - rhel7
    - rhel-7-server-rpms
	- rh-gluster-3-client-for-rhel-7-server-rpms
  - rhel8
    - rh-gluster-3-client-for-rhel-8-x86_64-rpms
- 升级前提请 umount  gluster 挂载点
- 升级 glusterfs 客户端
  ```
  # yum update glusterfs glusterfs-fuse
  ```
---
  
## 原生客户端glusterfs 常用挂载选项
- 