---
title: glusterfs-卷分层
date: 2022-04-05 19:44:42
tags:
  - glusterfs
categories: 
  - storage
  - 分布式存储
---

# glusterfs 卷分层
---
#### 管理分层卷
- 1. 创建分层卷
  ```
  # gluster volume tier testvol attach replica 2 brick1 brick2
  ```

- 2. 检查额外信息与统计信息
  ```
  # gluster volume tier testvol status
  # gluster volume tier testvol info
  ```
  
- 3. 断开热层
  - 3.1 先断开热层
    ```
    # gluster volume tier testvol detach start
    ```
  - 3.2 检查状态
    ```
	# gluster volume tier testvol detach status
	```
  - 3.3 状态均为completed时，断开热层
    ```
	# gluster volume tier testvol detach commit
	```
---
#### 配置文件冷层热层升降等级
- 可通过以下参数确定文件是否可被升级/降级
  - `cluster.watermark-lo`
    热层数据使用量没有达到此数值（默认75%），文件只能被升级
  - `cluster.watermark-hi`
    数据使用量达到此数值（默认90%），文件只能被降级
- 配置升级降级频率
  - `cluster.tier-promote-frequency`
    默认值为120秒，如果某一文件，在该秒内至少被访问一次，则符合升级条件
  - `cluster.tier-demote-frequency`
	默认为3600秒，如果某一文件，在该范围内未曾被访问过，则符合降级条件
  - `cluster.read-freq-threshold`
  - `cluster.write-freq-threshold`
    值为0为不启用，1-1000代表需要被访问多少次才符合被升级的次数
- 配置数据移动的限制
  - `cluster.tier-max-files`
    配置一个周期可移动的文件数，默认为10000个
  - `cluster.tier-max-mb` 
    限制一个周期可移动的数据量，默认为4000MiB
---
#### 分层卷的扩展
- 扩容
  - 断开热层
    ```
	# gluster volume tier testvol detach start
	# gluster volume tier testvol detach commit
    ```	
  - 将新brick添加到冷层
    ```
	# gluster volume add-brick testvol bricks
	```
  - 进行重平衡，直至completed
    ```
	# gluster volume rebalance testvol start
	# gluster volume rebalance testvol status
	```
  - 重新添加热层，若是添加热层，可以在断开后直接执行该步骤添加
    ```
	# gluster volume tier testvol attach replica x bricks
	```
- 缩容
  - 1. 断开热层
    ```
	# gluster volume tier testvol detach start
	# gluster volume tier testvol detach commit
    ```	
  - 2. 删除brick
    ```
	# gluster volume remove-brick testvol brick start
	# gluster volume remove-brick testvol brick commit
    ```	
  - 3. 重新连接热层
  - 如果要缩小热层，则在断开热层后，重新将热层attach到冷层
	