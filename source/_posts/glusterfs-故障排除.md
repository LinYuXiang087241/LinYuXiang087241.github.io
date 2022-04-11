---
title: glusterfs-故障排除
date: 2022-04-05 16:47:45
tags:
  - glusterfs
categories: 
  - storage
  - 分布式存储
---

# 故障排除
<!-- more -->
#### 自我修复
- 对于复制卷，需要执行自我修复后，才能重新同步所有副本，由`glusterfshd`进行
  - 触发自我修复
    ```
	# gluster volume heal <testvol> heal full   <<<加full，则执行完整修复，不加将根据更新时间进行普通修复
	```
  - 查看修复文件数量详细信息
    ```
	# gluster volume heal <testvol> statistics
	```
  - 查看出现脑裂的文件
    ```
	# gluster volume heal <testvol> info split-brain
	```

---
#### 更换brick
- 更换硬件时，需要添加新brick后再删除旧brick来更换brick
- 如果卷为复制卷
  - 执行以下命令，其中`OLD_BRICK`可以处于离线状态
    ```
	# gluster volume replace-brick <testvol> OLD_BRICK NEW_BRICK commit force
	```
  - 修复动作可以通过以下命令监控
    ```
	# gluster volume heal <testvol> info
	```
---
#### BitRot检测
- 会定期计算和存储的校验和，如果更改日志记录没有更改而计算校验和不符，则向以下目录下入报错
  `/var/log/glusterfs/bitd.log`
  `/var/log/glusterfs/scrub.log`
- 启用BitRot检测
  ```
  # gluster volume bitrot <testvol> enable
  ```
  禁用BitRot检测
  ```
  # gluster volume bitrot <testvol> disable
  ```
- 配置BitRot检测
    - 显示指定卷BitRot状态和设置，同时也列出检测到的错误文件
	  ```
	  # gluster volume bitrot <testvol> scrub status
	  ```
	- 暂停指定卷清理
	  ```
	  # gluster volume bitrot <testvol> scrub pause
	  ```
	- 恢复指定清理
	  ```
	  # gluster volume bitrot <testvol> scrub resume
	  ```
	- 配置每个节点可以同时清理的文件数量
	  ```
	  # gluster volume bitrot <testvol> scrub-throttle <Rate>
	  ```
	  <Rate> 可以为lazy/normal/aggressive，默认为lazy
	  lazy每次清理一个，normal每次允许清理两个或（vcpu数-4）/2个，aggressive 允许一次清理四个或（vcpu数-4）/2个
	- 配置校验和清理所有文件的频率
	  ```
	  # gluster volume bitrot <testvol> scrub-frequency <FREQUENCY>
	  ```
	  <FREQUENCY>可配置为 hourly/daily/weekly/biweekly/monthly
	- 恢复损坏的文件
	  ```
	  # find /path/to/brick/.glusterfs -name GFID
      /path/to/brick/.glusterfs/XX/YY/GFID
      # find /path/to/brick/ -samefile /path/to/brick/.glusterfs/XX/YY/
      GFID
      /path/to/brick/.glusterfs/XX/YY/GFID
      /path/to/brick/corruptedfile
	  ```
	  在找到损坏的文件后，可以删除该损坏文件，以及 brick 上 .glusterfs 中的相关文件。这两个文
      件都可以通过上例中最后的 find 命令找到。如果损坏的文件设有任何其他硬链接，也需要将它们删除。