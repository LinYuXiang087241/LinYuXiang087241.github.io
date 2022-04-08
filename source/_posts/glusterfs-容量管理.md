---
title: glusterfs-容量管理
date: 2022-04-04 18:31:15
tags: 
  - glusterfs
categories: 
  - storage
  - 分布式存储
---

# glusterfs 扩展卷
### 扩大卷
- 如果扩展副本设置为2的卷，必须以2的倍数添加brick
- 扩大步骤
  - 执行扩大命令
    ```
	# gluster volume add-brick <volname> replica <NEW-REPLICA-COUNT> <brick>
	```
  - 重新平衡卷
    ```
	# gluster volume rebalance <volname> start
	# gluster volume rebalance <volname> status
	```
  - 平衡策略选项
    | 选项 | 内容|
	|:-----|:-----|
	| lazy | 每个节点仅允许一次迁移一个文件 |
	| normal | 如果vcpu=4，每次迁移二个文件；如果vcpu=16，一次移动六个；计算(逻辑CPU数-4)/2|
	| appressive | 每个节点每次移动四个文件，计算(逻辑CPU数-4)/2 |
  -	配置重平衡策略
	```
	# gluster volume set <volname> cluster.rebal-throttle aggressive
	```
### 缩小卷
- 2x2分布式复制卷
  - 启动移除操作并监控
    ```
	# gluster volume remove-brick <volname> <bricks> 
	# gluster volume remove-brick <volname> <bricks> status 
	```
  - 将移除提交至卷
    ```
	# gluster volume remove-brick <volname> <bricks> commit
	```