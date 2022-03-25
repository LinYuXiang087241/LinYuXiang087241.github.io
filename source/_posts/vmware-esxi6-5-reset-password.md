---
title: vmware-esxi6.5-reset-password
date: 2022-03-19 16:23:44
tags: 
  - vmware
    - esxi
categories:
  - usage
---

# 重置vmware esxi6.5 root 密码
- 准备：
    - RHEL 7 启动U盘或ISO
	- 服务器配置从U盘或ISO启动

-  详细步骤：
    1. 从U盘启动后，选择`Troubleshooting`
	![3-19-1-1](/images/3-19/3-19-1-1.jpg)
	2. 选择`Rescue a Red Hat Enterprise Linux system`
	![3-19-1-2](/images/3-19/3-19-1-2.jpg)
	3. 选择`1`回车
	![3-19-1-3](/images/3-19/3-19-1-3.jpg)
	4. 挂载`sda5`设备后，将root密码移除
	```
	# mkdir /mnt/sda5
	# mount /dev/sda5 /mnt/sda5
	# cp /mnt/sda5/state.tgz /tmp
	# cd /tmp
	# tar zxvf state.tgz
	# tar zxvf local.tgz
	# vi /tmp/shadow    <<<将root后加密数据移除
	```
	删除`$` 至`:18934`之前
	![3-19-1-4](/images/3-19/3-19-1-4.png)
	删除后如图：
	![3-19-1-5](/images/3-19/3-19-1-5.png)
	5. 将修改后的`shadow`文件打包放回
	```
	# rm /tmp/state.tgz /tmp/local.tgz
	# tar czvf local.tgz etc/
	# tar czvf state.tgz local.tgz
	# cp state.tgz /mnt/sda5/
	```
	6. 退出救援模式，配置从硬盘启动。启动后，登录名为`root`，密码为空。
	
**<u><font color=red>仅用作经验记录</font></u>**