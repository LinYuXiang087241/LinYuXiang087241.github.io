---
title: glusterfs-管理员全记录：<font color=orange>五</font>
date: 2022-05-07 16:26:21
  - gluster
categories:
  - storage
  - 分布式存储
---

![22-412-g1](/images/22412/g1.png)
<html><pre><code>前提：基于红帽商业版glusterfs 3.5
本文主要内容：快照功能，配额限制，异地备份</code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

---
### 管理快照
- 1. 先决条件
  ```
  卷的每一个 brick 必须为精简配置 lv
  单个 lv 仅提供 1 个 brick
  如果配置有异地复制，在快照前需要关闭
  拍摄快照时，不应进行其他操作，如: rebalance 
  ```
- 2. 创建快照
     卷快照创建会创建包含 LVM 元数据信息的块快照池，拍摄后如果有新数据写入，快照池将会被覆盖，更改将复制到主卷。
	 因此，如果快照后，数据变化，快照池会消耗更多元数据空间。
  - 2.1 创建不含时间戳的快照
    ```
	# gluster snapshot create snap1 testvol no-timestamp
	```
  - 2.2 gluster 创建的快照默认创建时不会被激活，要配置默认就被激活，参考以下
    ```
	# gluster snapshot config activate-on-create enable
	```
  - 2.3 快照激活后，检查快照信息。 
        卷的快照会创建一个只读的 Red Hat Gluster Storage 卷。此卷将具有与原始/父卷相同的配置，挂载点也如下
    ```
# gluster snapshot status snap2

Snap Name : snap2
Snap UUID : 07b4b3d3-1d79-4272-ae03-09cc0908dcbc

	Brick Path        :   servera:/run/gluster/snaps/7b110933bf054efc97eff04fff524f8c/brick1/a  <<<
	Volume Group      :   vg_bricks
	Brick Running     :   Yes
	Brick PID         :   3426
	Data Percentage   :   0.54
	LV Size           :   2.00g


	Brick Path        :   serverb:/run/gluster/snaps/7b110933bf054efc97eff04fff524f8c/brick2/b  <<<
	Volume Group      :   vg_bricks
	Brick Running     :   Yes
	Brick PID         :   2946
	Data Percentage   :   0.54
	LV Size           :   2.00g
	```
  - 2.4 手动激活快照
    ```
	# gluster snapshot activate snap1
	```
- 3. 克隆快照
  - 3.1 在 activate 的快照上创建克隆
    ```
	# gluster snapshot clone clone-test snap2
	```
  - 3.2 创建的克隆卷 clone-test 在启动后，可以挂载用于文件恢复
    ```
	# gluster volume start clone-test
	```
- 4. 可用快照列表
  ```
  # gluster snapshot list 
  ```
- 5. 查看可用快照的信息
  ```
  # gluster snapshot info [(<snapname> | volume VOLNAME)]
  ```
- 6. 查看可用快照的状态
  ```
  # gluster snapshot status [(<snapname> | volume VOLNAME)]
  ```
- 7. 配置快照 config
     快照有以下内容
	- 1. `snap-max-hard-limit`
	  快照数量到达此数值将不允许继续创建快照，最大 256 个。
	- 2. `snap-max-soft-limit`
	  默认为硬限制的 90%，如果启用自动删除，则到达先之后会自动删除创建时间最久的快照
	- 3. `auto-delete`
	  启用自动删除功能
	- 4. `activate-on-create`
	  快照创建时激活，默认为不激活
	- 显示快照配置:
	  ```
	  # gluster snapshot config
	  ```
	- 修改快照配置
	  ```
	  # gluster snapshot config [VOLNAME] ([snap-max-hard-limit <count>] [snap-max-soft-limit <percent>]) | ([auto-delete <enable|disable>]) | ([activate-on-create <enable|disable>])
	  ```
	  如果不添加 volname ，则默认为系统配置
- 8. 停用快照
  ```
# gluster snapshot deactivate <snapname> 
  ```
- 9. 删除快照
  ```
# gluster snapshot delete <snapname>
  ```
  - 9.1 删除系统上所有快照
    ```
# gluster snapshot delete all
	```
  - 9.2 删除一个卷的所有快照
    ```
# gluster snapshot delete volume <volname>
	```
- 10. 恢复快照
  - 10.1 恢复前提
    ```
被恢复的卷需要处于 stop 状态
快照必须存在
不应在被恢复的卷上进行添加卷等操作
	```
  - 10.2 恢复快照，快照恢复后将会被删除，建议恢复后重新拍摄快照
    ```
# gluster snapshot restore snap3
    ```
  - 10.3 异地复制卷的快照恢复
    - 10.3.1 关闭异地副本
	  ```
# gluster volume geo-replication MASTER_VOL SLAVE_HOST::SLAVE_VOL stop
	  ```
	- 10.3.2 关闭从卷后关闭主卷
	  ```
# gluster volume stop VOLNAME
	  ```
	- 10.3.3 恢复主卷 snapshot
	  ```
# gluster snapshot restore snapname
	  ```
	- 10.3.4 启动从卷然后启动主卷
	- 10.3.5 启动异地主从复制
	  ```
# gluster volume geo-replication MASTER_VOL SLAVE_HOST::SLAVE_VOL start
	  ```
	- 10.3.6 恢复异地主从复制
	  ```
# gluster volume geo-replication MASTER_VOL SLAVE_HOST::SLAVE_VOL resume
	  ```
- 11. 访问快照
      快照卷中内容为只读，不支持 nfs 和 cifs 方式挂载快照
	  客户端执行以下命令只读访问快照
	  ```
# mount -t glusterfs <hostname>:/snaps/<snapname>/parent-VOLNAME /mount_point
# mount -t glusterfs servera:/snaps/snap1/testvol ./nfs/
	  ```
- 12. 调度快照
  - 12.1 前提条件
    - 1. 设置共享存储卷
	  ```
# gluster volume set all cluster.enable-shared-storage enable
	  ```
    - 2. 在所有节点执行初始化程序
      ```
# snap_scheduler.py init
      ``` 

  - 12.2 启用计划快照
    ```
# snap_scheduler.py enable
	```
  - 12.3 禁用计划快照
    ```
# snap_scheduler.py disable
	```
  - 12.4 查看计划快照状态
    ```
# snap_scheduler.py status
	```
  - 12.5 添加计划快照
    ```
# snap_scheduler.py add "Job Name" "Schedule" "Volume Name"
# snap_scheduler.py add "Job1" "*/5 * * * *" "testvol"
snap_scheduler: Successfully added snapshot schedule   
    ```	
  - 12.6 编辑计划快照
    ```
# snap_scheduler.py edit "Job Name" "Schedule" "Volume Name"
# snap_scheduler.py edit "Job1" "*/1 * * * *" "testvol"
 
snap_scheduler: Successfully edited snapshot schedule
# snap_scheduler.py list
JOB_NAME         SCHEDULE         OPERATION        VOLUME NAME      
--------------------------------------------------------------------
Job1             */1 * * * *      Snapshot Create  testvol          
# gluster snapshot list
snap1
Scheduled-Job1-testvol_GMT-2022.05.07-11.00.01

	```
  - 12.7 显示所有计划快照计划
    ```
# snap_scheduler.py list
JOB_NAME         SCHEDULE         OPERATION        VOLUME NAME      
--------------------------------------------------------------------
Job1             */5 * * * *      Snapshot Create  testvol  
	```
  - 12.8 删除计划快照
    ```
# snap_scheduler.py delete "Job Name"
	```

- 13. 用户可维护的快照
  当文件很久之前删除，可以到 `.snaps` 目录中进行恢复。
  每个激活的快照由 User Serviceable Snapshot 初始化时，都会消耗一定内存，每个brick 大约 10MB。
  - 13.1 针对卷开启此功能
    ```
	# gluster volume set testvol features.uss enable
	```
  - 13.2 激活快照
  - 13.3 客户端挂载
  - 13.4 在卷中会存在 `.snaps`路径， `ls -a` 也不会显示该路径
    ```
# ls .snaps -p
snap1/
snap4_GMT-2022.05.07-11.09.15/
	```
  - 13.5 进入对应快照的目录，将对应文件复制出
  - 13.6 如果使用 cifs，必须进行以下配置，且 `.snaps`目录为隐藏目录，win需要开启查看隐藏目录。同时该目录不能在子文件夹中访问。
    ```
# gluster volume set volname features.show-snapshot-directory on
	```
- 14. 快照常见问题处理
  - 14.1 快照创建失败
    - 14.1.1. 检查 lv 名
	  ```
# lvs -o pool_lv /dev/mapper/vg_bricks-testvol 
  Pool     
  brickpool
	  ```
      - 确保 brick 精简配置。以上如果输出没有 `pool`，则表示并非精简配置lv。
	  - 解决办法：确保brick 是精简配置后重新尝试创建快照
    - 14.1.2 检查卷的状态，确认卷所有 brick 都是好的
      ```
# gluster volume status VOLNAME
      ```
	  
      - 解决办法：尝试重新强制启动卷并创建快照
        ```
# gluster volume start VOLNAME force
	    ```	
		
    - 14.1.3 检查节点状态，如果由出现问题的节点，恢复集群状态后重新拍摄快照
	- 14.1.4 检查卷 `rebalance` 是否正在进行，如果在，等待其结束
	  ```
# gluster volume rebalance VOLNAME status
	  ```
  - 14.2 快照删除失败，导致系统不一致
    - 14.2.1 关闭 glusterd 服务
    - 14.2.2 从问题节点删除特定快照存储	
	  ```
	  # rm -rf /var/lib/glusterd/snaps/snap1
	  ```
	- 14.2.3 启动 glusterd 服务

---

### 配额管理
	
- 1. 为卷启用或关闭配额
  - 1.1 启用
    ```
# gluster volume quota testvol enable
	```
  - 1.2 禁用
    ```
# gluster volume quota testvol disable
	```
- 2. 为卷中目录启用配额
  ```
# gluster volume quota VOLNAME limit-usage /dir hard_limit
  ```
- 3. 限制磁盘使用量
  - 3.1 硬限制整个卷大小 1 TB
    ```
# gluster volume quota testvol limit-usage / 1TB
	```
  - 3.2 配合以上将用量的 75% 设置为软限制
    ```
# gluster volume quota testvol limit-usage / 1TB 75
	```
  - 3.3 将每一个卷的默认软限制从 80% 增加为 90%
    ```
# gluster volume quota testvol default-soft-limit 90
	```
  - 3.4 显示当前卷所有的限制
    ```
# gluster volume quota testvol list 
	```
  - 3.5 默认 `df` 命令会显示卷所有可用空间，而不会按照配额为目录分配空间。通过以下配置使 `df` 查看配额大小
    ```
# gluster volume set testvol quota-deem-statfs on
	```
  - 3.6 gluster 通过定期检查磁盘的使用量
	- 3.6.1 软限制默认为 `60` 秒，也即实际使用量小于配置时
	  ```
# gluster volume quota testvol soft-timeout 60
	  ```
	- 3.6.2 硬限制默认是 `5` 秒，也即实际容量超过配额后
	  ```
# gluster volume quota testvol hard-timeout 5
	  ```
	- 3.6.3 使用量到达软限制后，系统记录的频率。警报频率默认一周
	  ```
gluster volume quota testvol alert-time 1w
	  ```
  - 3.7 移除卷配额
    ```
# gluster volume quota testvol remove /  <-- 移除卷根目录配额
	```
	
---

### 配置异地复制
- 1. 前提条件
  - 1.1 主节点与从节点 gluster 版本一致
  - 1.2 主卷与从卷需要分别属于两个可信存储池
  - 1.3 从卷禁用以下选项
    ```
# gluster volume set slavevol performance.quick-read off
	```
- 2. 配置环境
  - 2.1 以 root 用户进行配置，主卷 ssh-copy-id 到从卷 root 用户
    - 2.1.1 创建复制的 `pem pub` 前需要执行以下命令
	  ```
# gluster system:: execute gsec_create
	  ```
	- 2.1.2 也可以执行以下命令
	  ```
# gluster-georep-sshkey generate
+-----------+-------------+---------------+
|    NODE   | NODE STATUS | KEYGEN STATUS |
+-----------+-------------+---------------+
|  serverb  |          UP |            OK |
|  serverc  |          UP |            OK |
| localhost |          UP |            OK |
+-----------+-------------+---------------+
	  ```
	- 2.1.3 创建复制的 `pem pub`
	  ```
# gluster volume geo-replication testvol servere::testcopy create push-pem
	  ```
	- 2.1.4 为主卷与从卷启用共享存储
	  ```
# gluster volume set all cluster.enable-shared-storage enable
	  ```
	- 2.1.5 为异地从卷配置元数据卷
	  ```
# gluster volume geo-replication testvol servere::testcopy config use_meta_volume true
	  ```
	- 2.1.6 启动异地复制
	  ```
# gluster volume geo-replication testvol servere::testcopy start
	  ```
	- 2.1.7 检查异地复制状态
	  ```
# gluster volume geo-replication testvol servere::testcopy status
	  ```
  - 2.2 以非 root 用户进行配置，主卷 ssh-copy-id 到从卷 syncuser 用户
	- 2.2.1 在从卷节点创建 syncuser 用户 syncgrp 组
	  ```
# groupadd syncgrp
# useradd syncuser -G syncgrp 
# echo redhat | passwd --stdin syncuser 
	  ```
	- 2.2.2 创建 mountbroker 目录
	  ```
# gluster-mountbroker setup /var/mountbroker-root syncgrp
	  ```
	- 2.2.3 将卷与用户添加到 mountbroker 服务
	  ```
# gluster-mountbroker add copyvol syncuser
	  ```
	- 2.2.4 检查 mountbroker 状态，并重启 glusterd 服务
	  ```
#  gluster-mountbroker status
+-----------+-------------+---------------------------+-------------+--------------------+
|    NODE   | NODE STATUS |         MOUNT ROOT        |    GROUP    |       USERS        |
+-----------+-------------+---------------------------+-------------+--------------------+
| localhost |          UP | /var/mountbroker-root(OK) | syncgrp(OK) | syncuser(copyvol)  |
+-----------+-------------+---------------------------+-------------+--------------------+
# systemclt restart glusterd
	  ```
	- 2.2.5 配置主卷节点可以免密登录 servere 通过 syncuser账户
	- 2.2.6 创建密钥
	  ```
# gluster system:: execute gsec_create
	  ```
	- 2.2.7 推送密钥到 servere syncuser 用户
      ```
# gluster volume geo-replication prodvol syncuser@servere.glusterfs.linuxone.in::copyvol create push-pem
      ```	
    - 2.2.8 为主卷与从卷启用共享存储
	  ```
# gluster volume set all cluster.enable-shared-storage enable
	  ```	
	  
	- 2.2.9 运行 `/usr/libexec/glusterfs/set_geo_rep_pem_keys.sh` 
	  ```
# /usr/libexec/glusterfs/set_geo_rep_pem_keys.sh syncuser prodvol copyvol
	  ```
	- 2.2.10 为异地从卷配置元数据卷
	  ```
# gluster volume geo-replication prodvol syncuser@servere.glusterfs.linuxone.in::copyvol config use_meta_volume true
	  ```
	- 2.2.11 启动异地复制
	  ```
# gluster volume geo-replication prodvol syncuser@servere.glusterfs.linuxone.in::copyvol start
	  ```
	- 2.2.12 检查异地复制状态
	  ```
# gluster volume geo-replication prodvol syncuser@servere.glusterfs.linuxone.in::copyvol status
	  ```
	- 2.2.13 在取消异地复制后，需要在从卷节点删除 mountbroker 用户以及卷的信息：
	  ```
# gluster-mountbroker remove --volume copyvol --user syncuser
	  ```
  - 3. 异地复制配置
    - 3.1 配置异地复制方式使用 rsync
	  ```
# gluster volume geo-replication prodvol syncuser@servere.glusterfs.linuxone.in::copyvol config sync_method rsync
	  ```
	- 3.2 还原默认配置以 `!`开头，以重置日志等级为示例
	  ```
# gluster volume geo-replication prodvol syncuser@servere.glusterfs.linuxone.in::copyvol config '!log-level'
	  ```
	- 3.3 设置异地复制检查点，可以获得当时主卷上数据是否已复制到从卷的同步信息
	  ```
# gluster volume geo-replication prodvol syncuser@servere.glusterfs.linuxone.in::copyvol config checkpoint now
	  ```
	- 3.4 显示检查点的详细信息
	  ```
# gluster volume geo-replication prodvol syncuser@servere.glusterfs.linuxone.in::copyvol status detail
	  ```
	- 3.5 移除检查点
	  ```
# gluster volume geo-replication prodvol syncuser@servere.glusterfs.linuxone.in::copyvol config '!checkpoint'
	  ```
	- 3.6 删除异地复制会话
	  - 3.6.1 root 用户的复制会话
	    ```
#  gluster volume geo-replication Volume1 storage.backup.com::slave-vol delete
		```
	  - 3.6.2 普通用户的复制会话
	    ```
#  gluster volume geo-replication prodvol syncuser@servere.glusterfs.linuxone.in::copyvol  delete reset-sync-time
		```
- 4. 异地复制配置定时作业
  - 4.1 仅需要时运行会话
    ```
#  python /usr/share/glusterfs/scripts/schedule_georep.py prodvol servere.glusterfs.linuxone.in copyvol
	```
  - 4.2 每天 20:30 运行异地复制
    ```
30 20 * * * root python /usr/share/glusterfs/scripts/schedule_georep.py --no-color Volume1 storage.backup.com slave-vol >> /var/log/glusterfs/schedule_georep.log 2>&1
	```
- 5. 灾备恢复
  - 5.1 故障出现，从卷提升为主卷
    - 5.1.1 禁用从卷的只读
	  ```
#  gluster volume set copyvol features.read-only off
	  ```
    - 5.1.2 运行以下命令，将从卷提升为主卷
	  ```
# gluster volume set copyvol geo-replication.indexing on
# gluster volume set copyvol changelog on
	  ```
    - 5.1.3 在应用配置使用从卷进行 I/O 操作
  - 5.2 主卷恢复，将主卷恢复到原始状态
    - 5.2.1 停止从原始主卷到原始从卷的异地复制会话(主卷执行)
	  ```
# gluster volume geo-replication prodvol syncuser@servere.glusterfs.linuxone.in::copyvol stop force
	  ```
    - 5.2.2 创建新的异地复制会话，将原始从卷作为新的主卷，原始主卷作为新的从卷，命令携带`force`选项进行创建，到步骤配置元数据卷操作后停下。
    - 5.2.3 启用特殊同步模式，该模式将会忽略原始从卷启动 `indexing` 模式前的文件，这将会加快恢复
	  ```
# gluster volume geo-replication  copyvol syncuser@servera.glusterfs.linuxone.in::prodvol config special-sync-mode recover
	  ```
    - 5.2.4 禁用以下选项
	  ```
# gluster volume geo-replication copyvol syncuser@servera.glusterfs.linuxone.in::prodvol config gfid-conflict-resolution false
	  ```
    - 5.2.5 启动从卷到主卷的异地复制
	- 5.2.6 通过配置检查点来检查异地复制是否完成
	- 5.2.7 检查点完成，停止并删除当前异地复制
	- 5.2.8 原始主卷禁用只读
	  ```
# gluster volume set prodvol features.read-only off
	  ```
    - 5.2.9 重置将从卷提升为主卷的选项
	  ```
# gluster volume reset copyvol geo-replication.indexing force
# gluster volume reset copyvol changelog
	  ```
	- 5.2.10 重新启动原始主卷到从卷的异地复制
- 6. 性能优化与故障分析
  - 6.1 开启以下选项优化性能
    ```
# gluster volume set SLAVE_VOL batch-fsync-delay-usec 0
	```
  - 6.2 更改日志选项调整异地复制性能
    - 6.2.1 更改日志速率，默认时间 15秒，异地复制推荐时间 10-15s，配置日志翻转时间
	  ```
# gluster volume set VOLNAME rollover-time 15
	  ```
	- 6.2.2 更改日志更新写入磁盘的频率，默认间隔为5
	  ```
# gluster volume set VOLNAME fsync-interval 5
	  ```
    - 6.2.3 触发 explicit sync 
	  ```
# setfattr -n glusterfs.geo-rep.trigger-sync -v "1" <file-path>
	  ```
    - 6.3 异地同步失败`Stable`，日志有以下记录
	  ```
[2011-05-02 13:42:13.467644] E [master:288:regjob] GMaster: failed to sync ./some_file`
	  ```
	   - 异地复制要求 rsync 版本不低于 v3.0.0
    - 6.4 异地同步失败`Faulty`，日志有类似记录
	  ```
012-09-28 14:06:18.378859] E [syncdutils:131:log_raise_exception] <top>: FAIL: Traceback (most recent call last): File "/usr/local/libexec/glusterfs/python/syncdaemon/syncdutils.py", line 152, in twraptf(*aa) File "/usr/local/libexec/glusterfs/python/syncdaemon/repce.py", line 118, in listen rid, exc, res = recv(self.inf) File "/usr/local/libexec/glusterfs/python/syncdaemon/repce.py", line 42, in recv return pickle.load(inf) EOFError
	  ```
	  - 检查ssh 连接
	  - 检查 fuse 是否安装
	
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
	
  
  