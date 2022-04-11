---
title: glusterfs-异地存储配置
date: 2022-04-04 22:04:07
tags: 
  - glusterfs
categories: 
  - storage
  - 分布式存储
---

# 使用非特权账户配置异地复制
<!-- more -->
- 异地节点配置
  - 前提，已经配置有gluster volume copyvol
  - 1. 配置账户
    ```
    # grouadd syncgrp
    # useradd -G syncgrp -m syncuser
    # echo redhat | passwd --stdin syncuser
    ```
  - 2. 配置 mountbroker
    ```
	# mkdir /var/mountbroker-root
	# chmod 711 /var/mountbroker-root
	# gluster system:: execute mountbroker opt mountbroker-root /var/mountbroker-root
	# gluster system:: execute mountbroker user syncuser <copyvol>
	# gluster system:: execute mountbroker opt geo-replication-log-group syncgrp
	# gluster system:: execute mountbroker opt rpc-auth-allow-insecure on
	# gluster system:: execute mountbroker info <<< 查看完整配置
	```
	mountbroker配置文件`/ect/glusterfs/glusterd.vol`
	配置完成后，重启服务
	```
	# systemctl restart glusterd
	```
- 本地节点配置
  - 前提，已经配置有需要备份的gluster volume testvol
  - 配置异地节点 syncuser ssh 免密登录
    ```
	# ssh-keygen
    # ssh-copy-id syncuser@<remote-server-ip>	
	```
  - 本地节点创建用于异地守护进程的ssh密钥对
    ```
	# gluster system:: execute gsec_create
	```
  - 创建并推送密钥
    ```
	# gluster volume geo-replication testvol syncuser@<remote-server-ip>::copyvol create push-pem	
	```
  - 异地节点将被推动的密钥复制到正确位置
    ```
	# /usr/libexec/glusterfs/set_geo_rep_pem_keys.sh syncuser testvol copyvol
	```
  - 本地节点配置异地复制连接
    ```
	# gluster volume geo-replication testvol syncuser@<remote-server-ip>::copyvol config use_meta_volume true
	```
  - 启动复制并验证
    ```
	# gluster volume geo-replication testvol syncuser@<remote-server-ip>::copyvol start
	# gluster volume geo-replication testvol syncuser@<remote-server-ip>::copyvol status
    ```	
---
# 管理异地复制
### 调优配置
- 1. 查看异地复制所有可用项与当前配置
  ```
  # gluster volume geo-replication testvol syncuser@<remote-server-ip>::copyvol config
  ```
- 2. 配置主卷上删除的文件不会同时在从卷上删除
  ```
  # gluster volume geo-replication testvol syncuser@<remote-server-ip>::copyvol config ignore-deletes true
  ```
- 3. 主卷以什么频率检查更改日志要同步到从卷的更改，默认15S以下更改为5S
  ```
  # gluster volume geo-replication testvol syncuser@<remote-server-ip>::copyvol config changelog.rollover-time 5
  ```
- 4. 配置检查点，检查特定时间点之前的所有更改是否同步，可以配置now和
  ```
  # gluster volume geo-replication testvol syncuser@<remote-server-ip>::copyvol config checkpoint now
  # gluster volume geo-replication testvol syncuser@<remote-server-ip>::copyvol status detail
  删除检查点
  # gluster volume geo-replication testvol syncuser@<remote-server-ip>::copyvol config checkpoint '!checkpoint'
  ```
### 添加节点以及break
- 添加节点
  - 1. 添加的节点配置ssh 密钥对
    ```
	# ssh-copy-id syncuser@<remote-server-ip>
	# gluster volume geo-replication testvol syncuser@<remote-server-ip>::copyvol create push-pem force
	```
  - 2. 停止并重新启动异地复制
- 提升从卷-主卷故障，从卷可以作为客户端新卷
  - 1. 提升从卷前，从卷节点进行以下配置
    ```
	# gluster volume set copyvol geo-replication.indexing on
	# gluster volume set copyvol changelog on
	```
	2. 主卷可用后
	  - 2.1 创建从卷至主卷异地复制，但不要启动
	  - 2.2 新异地复制配置 `special-sync-mode` 为`recover`
	  - 2.3 停止所有从卷的i/o访问，并创建 `checkpoint` now
	  - 2.4 所有数据同步后，停止新的异地复制协议，并重置之前从卷的配置
	    ```
		# gluster volume reset copyvol geo-replication.indexing force
		# gluster volume reset copyvol changelog
		```
	  - 2.5 应用重新指向主卷
- 日志目录
  - 主节点上 `/var/log/glusterfs/geo-replication/`，每个正在异地复制的卷有子目录
  - 主从节点 `/var/log/glusterfs/geo-replication-slaves`
  
 
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  