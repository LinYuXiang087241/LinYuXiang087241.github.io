---
title: memory-failed-write-dqdqs-DIMM-A2
date: 2022-03-23 11:18:13
tags: 
  - x86_64
  - memory
categories: 
  - usage
---

# 服务器开机时报出 memeory failed错误，如下图
![3-23-a](/images/3-23/3-23-a.jpg)

- 原因：
  - 如果是第一次打开的服务器，可能是由于对应的内存松动引起，重新插拔内存即可修复。<font color=blue>[参考此DELL官方回复](https://www.dell.com/community/PowerEdge-Hardware-General/Failed-Write-DqDqs-DIMM-B2/m-p/4532092)</font>
  - 如果是旧服务器，可能由于内存功率过大引起。恢复办法有两种：
    - 换更低功率的内存
	- 少插几根内存
	
**<u><font color=red>仅用作经验记录</font></u>**	