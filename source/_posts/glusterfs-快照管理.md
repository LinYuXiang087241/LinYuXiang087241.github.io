---
title: glusterfs-快照管理
date: 2022-04-05 17:24:22
tags:
  - glusterfs
categories: 
  - storage
  - 分布式存储
---

# glusterfs 快照管理
<!-- more -->
- 创建快照
  ```
  # gluster snapcreate create <snapname> <volume> no-timestamp
  ```
  快照上限 256 个
- 挂载快照，快照创建时默认为stop状态
  激活快照
  ```
  # gluster snapshot activate <snapname>
  # gluster volume set <volname> features.uss enable
  ```
  客户端挂载快照
  ```
   server:/snaps/snapname/volume
  ```
  删除快照，删除某个卷的搜友快照
  ```
  # gluster snapshot delete volume <volname> 
  ```
  删除所有卷的所有快照
  ```
  # gluster snapshot delete all
  ```
- 从快照恢复
  - 1. 挂载快照复制
  - 2. 通过快照克隆到新卷
    ```
	将快照克隆到新卷，仅存储与快照不同的数据，原始快照不受影响
	# gluster snapshot clone <newvol> <snapname>
	将快照提升为原始卷
	# gluster snapshot restore <snapname> 
	通过自愈出发完整修复
	# gluster volume heal <volname> full
	```
  - 3. 查看快照信息
    - 3.1 显示快照列表
	  ```
	  # gluster snapshot list
	  ```
	- 3.2 显示详细信息
	  ```
	  # gluster snapshot info <snapname>
	  ```
	- 3.3 显示快照中每个brick的信息
	  ```
	  # gluster snapshot status <snapname>
	  ```
- 调度快照
  - 配置快照调度
    - 1. 启用共享存储
      ```
	  # gluster volume set all cluster.enable-shared-storage enable
	  ```	
    - 2. 启用 cron_system_cronjob_use_shares 
      <pre><code># setsebool -P cron_system_cronjob_use_shares 1 </code></pre>
	  
	- 3. 初始化快照调度程序目录
      ```
	  # snap_scheduler.py init
      ```	
    - 4. 启用快照调度程序
      <pre><code># snap_scheduler.py enable</code></pre>	  
  - 调度快照
    - 添加新作业
	  ```
	  # snap_scheduler.py add "<JOBNAME>" "<SCHEDULE>" <VOLUME> //JOBNAME唯一标识符，SCHEDULE表示cron语法包含的五个字段
	  ```
	- 管理作业
	  - 显示作业列表
	    ```
		# snap_scheduler.py list
		```
      - 删除作业
	    ```
		# snap_scheduler.py delete "<JOBNAME>" 
		```
	  - 修改作业
	    ```
		# snap_scheduler.py edit "<JOBNAME>" "<SCHEDULE>" "<VOLUME>"
		```
	  - 快照日志存储在
		`/var/log/glusterfs/snap_scheduler.log`
		`/var/log/glusterfs/gcron.log`
	- 配置快照行为
	  - 通过以下命令可以配置快照行为
	    ```
		# gluster snapshot config <rate>
		```
		|rate|作用|
		|:-|:-|
		|active-on-create|新快照将自动激活，默认为disable|
		|snap-max-hard-limit|任何一个卷锁允许得快照数量上限，默认256|
		|snap-max-soft=limit|发出告警前卷上允许得快照数量，默认为硬限制得90%|
		|auto-delete|设置为enable，则到达软限制时快照会自动删除最旧的|
	
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  