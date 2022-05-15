---
title: glusterfs-管理员全记录：<font color=cyan>六</font>
date: 2022-05-14 23:17:25
  - gluster
categories:
  - storage
  - 分布式存储
---

![22-412-g1](/images/22412/g1.png)
<html><pre><code>前提：基于红帽商业版glusterfs 3.5
本文主要内容：glusterfs卷管理、gluster存储日志管理、BITROT检测、分层卷（弃用）、
监控工作负载、性能优化 </code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

---

### 卷支持的配置项
参考官方文档`SUPPORTED VOLUME OPTIONS`
https://access.redhat.com/documentation/en-us/red_hat_gluster_storage/3.5/html/administration_guide/volume_option_table

- 以只读的方式挂载卷
  ```
# gluster volume set volname read-only enable
  ```
  
- 当 brick 可用的空间小于等于用户声明的保留空间时，不会创建新文件，只可进行内部操作，如重平衡、自愈等。
  ```
# gluster volume set volname storage.reserve percentage
  ```

- 1. 扩大卷（不支持开启异地复制的卷）
  - 1.1 扩展复制卷，复制卷应扩大对应副本数量的 brick ，扩展后应立即执行 rebalance
    ```
# gluster volume add-brick test-volume server5:/rhgs/brick5/ server6:/rhgs/brick6/
	```
  - 1.2 扩展分布式分散卷，如果配置为 6+2 的卷，则必须添加 6 的倍数的 brick
  - 1.3 扩展基础逻辑卷 lvm
    - 1.3.1 强制关闭 glusterd 服务
	  ```
# gluster volume stop volname
# kill -9 brick-process-id
	  ```
	- 1.3.2 扩展逻辑卷，扩展后重新启动卷
	  ```
# pvcreate /dev/PHYSICAL_VOLUME_NAME
# vgextend VOLUME_GROUP_NAME /dev/PHYSICAL_VOLUME_NAME
# lvextend -L+nG VOLUME_GROUP_NAME/POOL_NAME
# lvextend -L+nG /dev/mapper/ LOGICAL_VOLUME_NAME-VOLUME_GROUP_NAME
# xfs_growfs /dev/VOLUME_GROUP_NAME/LOGICAL_VOLUME_NAME
# mount -o remount /dev/VOLUME_GROUP_NAME/LOGICAL_VOLUME_NAME /bricks/path_to_brick
# gluster volume start VOLNAME force
	  ```
- 2. 缩小卷
  - 2.1 收缩复制卷，必须收缩 replica 的倍数
    ```
# gluster volume remove-brick test-volume server2:/rhgs/brick2 start
	```
	- 2.1.1 检查移除状态
	  ```
# gluster volume remove-brick VOLNAME BRICK status
	  ```
	- 2.1.2 显示状态为迁移完成，提交移除
	  ```
# gluster volume remove-brick VOLNAME BRICK commit
	  ```
	- 2.1.3 停止移除操作
	  ```
# gluster volume remove-brick VOLNAME BRICK stop
      ```
	  
- 3. 迁移卷
  - 3.1 复制卷执行新 brick 替换旧 brick
	- 3.1.1 执行替换
	  ```
# gluster volume replace-brick test-volume server0:/rhgs/brick1 server5:/rhgs/brick1 commit force
	  ```
	- 3.1.2 新brick 在线后，检查自愈
	  ```
# gluster volume heal VOL_NAME info
	  ```
- 4. 重平衡 rebalance
  - 4.1 启动重平衡，最好将所有挂载点都取消后进行
    ```
# gluster volume rebalance VOLNAME start
	```
  - 4.2 重平衡速率
    ```
# gluster volume set VOLNAME rebal-throttle lazy|normal|aggressive
	```	
  - 4.3 显示重平衡状态
    ```
# gluster volume rebalance VOLNAME start
	```	
  - 4.4. 停止重平衡
    ```
# gluster volume rebalance VOLNAME stop
	```
- 5. 管理脑裂 split-brain  
  - 5.1 防止脑裂
    - 5.1.1 服务器端配置
	  - 5.1.1.1 为所有卷开启超过一半节点在线
      ```
# gluster volume set all cluster.server-quorum-ratio 51%
	  ```
	  - 5.1.1.2 特定卷需要以下配置以参与服务器仲裁
	    ```
#gluster volume set testvol cluster.server-quorum-type server
		```
	- 5.1.2 客户端配置
	  - 5.1.2.1 cluster.quorum-count 允许写入时必须可用的最小brick 数量
	  - 5.1.2.2 cluster.quorum-type 配置写入行为， 
	    - 1. fixed 则只要副本集中可用的砖块数量大于或等于该cluster.quorum-count选项的值，就允许写入。
		- 2. auto 则当副本集中至少 50% 的砖块可用时，允许写入。
  - 5.2 从脑裂中恢复文件
    参考官方文档 `Recovering from File Split-brain`
	https://access.redhat.com/documentation/en-us/red_hat_gluster_storage/3.5/html/administration_guide/sect-managing_split-brain
	  
---

### 存储日志管理
- 1. 日志轮换
  每周轮换一次，每两周以 gzip 格式压缩一次，保存 52 周的日志

- 2. 配置日志格式
  - 为卷配置
    ```
日志包含 msg id
# gluster volume set VOLNAME diagnostics.brick-log-format with-msg-id
日志不包含 msg id
# gluster volume set testvol diagnostics.brick-log-format no-msg-id
	```
  - 为客户端配置
    ```
日志包含 msg id
# gluster volume set testvol diagnostics.client-log-format with-msg-id
日志不包含 msg id
# gluster volume set testvol diagnostics.client-log-format no-msg-id
	```
  - 为 glusterd 服务配置
    ```
日志包含 msg id
# glusterd --log-format=with-msg-id
日志不包含 msg id
# glusterd --log-format=no-msg-id
	```
- 3. 日志等级
  默认为`info`，即仅记录`CRITICAL` `ERROR` `WARNING` `INFO` 的信息
  - 3.1 将 brick 日志等级提升为 warning
    ```
# gluster volume set testvol diagnostics.brick-log-level WARNING
	```
  - 3.2 将 brick 的 syslog 日志等级提升为 warning
    ```
# gluster volume set testvol diagnostics.brick-sys-log-level WARNING
	```
  - 3.3 将客户端日志等级调整为 error
    ```
# gluster volume set testvol diagnostics.client-log-level ERROR
	```
  - 3.4 客户端 syslog 日志等级调整为 error
    ```
# gluster volume set testvol diagnostics.client-sys-log-level ERROR
	```
  - 3.5 为 `glusterd` 服务配置日志级别 `/etc/sysconfig/gluster`
	```
LOG_LEVEL='value'
	```
	配置完成后重启`glusterd`服务
  - 3.6 以日志级别为 error 的方式运行`volume status`
    ```
# gluster --log-level=ERROR volume status
	```	
	  
---

### BITROT检测
- 1. 启用或禁用 BitRot 检测
  ```
gluster volume bitrot VOLNAME enable
gluster volume bitrot VOLNAME disable
  ```
- 2. 清理文件
  - 2.1 启动按需清理
    ```
gluster volume bitrot VOLNAME scrub ondemand
	```	
  - 2.2 暂停清理
    ```
gluster volume bitrot VOLNAME scrub pause
	```	
  - 2.3 恢复卷清理
    ```
gluster volume bitrot VOLNAME scrub resume
	```	
  - 2.4 卷清理状态
    ```
gluster volume bitrot VOLNAME scrub status
	```	
  - 2.5 配置清理的速录，默认`lazy` `normal` `aggressive`
    ```
gluster volume bitrot VOLNAME scrub-throttle rate
	```	
  - 2.6 配置清理频率 `daily` `weekly` 默认`biweekly` `monthly`
    ```
gluster volume bitrot VOLNAME scrub-frequency frequency
	```	
	  
---

### 分层卷（弃用）	  
- 1. 将层附加到卷
  ```
# gluster volume tier test-volume attach replica 3 server1:/rhgs/brick5/b1 server2:/rhgs/brick6/b2
  ``` 
- 2. 要设置文件升级和降级的百分比，请运行以下命令：
```
# gluster volume set VOLNAME cluster.watermark-hi value
# gluster volume set VOLNAME cluster.watermark-low value  
  ``` 

- 3. 显示分成卷的信息
  ```
# gluster volume tier VOLNAME status
  ```
- 4. 分层卷分离
  - 4.1 启动分离
    ```
# gluster volume tier VOLNAME detach start  
    ```
	
  - 4.2 检查分离状态
    ```
# gluster volume tier VOLNAME detach status
	```
  - 4.3 成功分离后，提交层分离
    ```
# gluster volume tier VOLNAME detach commit
	```

---

### 监控工作负载	
- 1. 卷分析
  - 1.1 启动卷分析
  ```
# gluster volume profile VOLNAME start 
  ``` 
  - 1.2 显示信息
  ```
# gluster volume profile VOLNAME info 
  ```  
  - 1.3 停止卷分析
  ```
# gluster volume profile VOLNAME stop
  ```  
  - 1.4 显示挂载点的所有信息
  ```
# setfattr -ntrusted.io-stats-dump -v output_file_id  mount_point
  ```  
	
---

### 性能优化	
- 1. 小文件性能增强
  - 1.1 为客户端配置时间线程，最多可用同时处理n个网络连接
    ```
# gluster volume set test-vol client.event-threads 4
	```	
  - 1.2 为访问卷的服务器调整事件线程
    ```
# gluster volume set test-vol server.event-threads 4
	```
  - 1.3 启用查找优化
    ```
# gluster volume set VOLNAME cluster.lookup-optimize <on/off>
    ```	
- 2. 目录操作
  - 2.1 启用元数据存储
    ```
# gluster volume set <volname> group metadata-cache
    ```	
  - 2.2 增加可以缓存的文件数量，增加会增加 brick 进程内存占用
    ```
# gluster volume set <VOLNAME> network.inode-lru-limit <n>
    ```	
	  
	  
	  
	  
	  
  