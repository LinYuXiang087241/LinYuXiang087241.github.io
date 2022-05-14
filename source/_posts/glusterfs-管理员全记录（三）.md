---
title: glusterfs-管理员全记录：<font color=blue>三</font>
date: 2022-04-21 22:22:58
tags:
  - gluster
categories:
  - storage
  - 分布式存储
---

![22-412-g1](/images/22412/g1.png)
<html><pre><code>前提：基于红帽商业版glusterfs 3.5
本文主要内容：卷管理</code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

---

## 创建一个精简配置的卷
- 创建`pv` `vg`=`vg_bricks` lvpoo  `lv`=testvol 并格式化为`xfs`

  ```
  # pvcreate /dev/sdb
  # vgcreate vg_bricks /dev/sdb
  # lvcreate -L 10G -T vg_bricks/brickpool
  # lvcreate -V 2G -T vg_bricks/brickpool -n testvol
  # mkfs.xfs -i size=512 /dev/vg_bricks/testvol
  ```
  
  
- 创建挂载点，并将`lvm`挂载

  ```
  # mkdir -p /bricks/test
  # echo "/dev/vg_bricks/testvol /bricks/test xfs defaults 0 0" >> /etc/fstab
  # mount -a
```
  
- 使用挂载目录下子目录创建卷

  ```
  # gluster volume create testvol serverb:/bricks/test/xx
  ```
  
- 如果想要使用之前删除的卷的 brick，使用前可以进行格式化

  ```
  # # mkfs.xfs -f -i size=512 /dev/vg_bricks/testvol
  ```

---

## 分布式卷`Distributed Volume`
分布式卷的架构类似于磁盘阵列`raid 0`，目录内容随机分部在所有`brick`中
- 创建卷
  ```
  # gluster volume create testvol servera:/bricks/test/testvol_n1 serverb:/bricks/test/testvol_n2
  ```
  
- 在创建卷时，建议关闭`performance.client-io-threads`参数，以提高性能
  ```
  # gluster volume set testvol performance.client-io-threads off
  ```

- 创建卷时可以配置 `transport ` 参数，配置传输类型。默认`tcp`，或可启用`rdma`
  ```
  #  gluster volume create testvol transport rdma servera:/bricks/test/testvol_n1 serverb:/bricks/test/testvol_n2   <<< 创建infiniband卷
  ```
  
---

## 创建复制卷`Replicated Volume`
复制卷要求三副本，建议卷的 brick 分别来自不同的服务器，且 brick 的数量必须等于复制卷的副本数。

- 创建 three-way 复制卷:
  ```
  # gluster volume create testvol replica 3 servera:/bricks/test/testvol_n1 serverb:/bricks/test/testvol_n2 serverb:/bricks/test/testvol_n3
  ```

- 创建分片复制卷 (sharded replicated volume)
启用分片后，写入卷的文件会被分成几部分，文件每一部分在各个 brick 之间均匀分部，可显著提高卷性能。
  - 创建 three-way 复制卷
  - 为复制卷开启 `features.shard enable`
    ```
	# gluster volume set testvol features.shard enable
	```
  - 配置每一个分片的大小，支持到 `512MB` ，分片配置仅影响启用后的文件
    ```
	# gluster volume set testvol features.shard-block-size 30MB
	```
- 要了解文件分片后是如何分部
  - 检查每一个卷的 gfid
<html><pre><code># getfattr -d -m. -e hex /bricks/test/testvol_n1/xxx.zip
# file: bricks/test/testvol_n1/xxx.zip
trusted.afr.dirty=0x000000000000000000000000
trusted.gfid=0x<font color=red>0aae99637522460f92627915a34d7088</font>
trusted.gfid2path.39420c22983a0807=0x30303030303030302d303030302d303030302d303030302d3030303030303030303030312f7878782e7a6970
trusted.glusterfs.shard.block-size=0x0000000001e00000
trusted.glusterfs.shard.file-size=0x00000000077713e20000000000000000000000000003bbca0000000000000000</code></pre></html>

  - 执行以下命令检查是如何分部的
    ```
# ls /bricks/test/testvol_n1/.shard/ -lh | grep 0aae9963-7522-460f-9262-7915a34d7088
-rw-r--r-- 2 root root 30M Apr 22 22:49 0aae9963-7522-460f-9262-7915a34d7088.1
-rw-r--r-- 2 root root 30M Apr 22 22:50 0aae9963-7522-460f-9262-7915a34d7088.2
-rw-r--r-- 2 root root 30M Apr 22 22:50 0aae9963-7522-460f-9262-7915a34d7088.3
	```

---

## 创建分布式复制卷`Distributed Replicated Volumes`
brick 数量必须是副本数的倍数。指定的brick顺序，前x个会组成复制集，后x个组成复制集群。
- 创建 three-way 分布式复制卷:
  ```
  # gluster volume create testvol replica 3 servera:/bricks/test/testvol_n1 serverb:/bricks/test/testvol_n2 serverc:/bricks/test/testvol_n3 \
  serverd:/bricks/test/testvol_n4 servere:/bricks/test/testvol_n5 serverf:/bricks/test/testvol_n6
  ```
  如上创建的卷，server a b c 会成为一个三向复制集， server d e f 会成为另一个三向复制集

---

## 创建仲裁复制卷` Arbitrated Replicated Volumes`
仲裁复制卷，包含三个 brick，其中两个brick为完整复制副本，一个 brick 作为仲裁brick，仅存储文件名/结构/元数据。
- 1. 仲裁 brick 对于容量要求并不高，以下为容量计算公式:
```
 4 KB * ( 单个数据块的大小 kb / 文件平均大小 kb)
```
如果启用卷分片，则计算方式如下
```
 4 KB * ( 单个数据块的大小 kb / 单位分片大小 kb)
```

- 2. 仲裁逻辑
   | 状态 | 卷状态 |
   |--- | --- | 
   | 所有` brick `都可用 | 允许所有操作 |
   | 可用仲裁 `brick` 和一个数据`brick`  | 如果仲裁器不同意数据块可用，则写入会`ENOTCONN`失败，但是允许其他操作。当仲裁器元数据与可用数据节点一致，则允许所有 |
   | 仲裁器失效，其他可用 | 允许所有文件操作，仲裁器的记录会在其可用时修复 |
   | 仅有一个数据块可用 | 所有文件操作都会因`ENOTCONN`失败 |

- 3. 创建仲裁复制卷
  ```
  # gluster volume create testvol replica 3 arbiter 1 servera:/bricks/test/testvol_n1 serverb:/bricks/test/testvol_n2 serverb:/bricks/test/testvol_n3 \
  serverd:/bricks/test/testvol_n4 servere:/bricks/test/testvol_n5 serverf:/bricks/test/testvol_n6
  ``` 
  其中三个为一组创建仲裁复制卷，且第三个为仲裁器节点。

- 4. 在更少节点创建更多仲裁复制卷
  - 将一个数据 brick 和一个仲裁 brick 在一起，这对于写入繁重的工作负载有效。（单独配置仲裁节点效果最佳）
    ```
	 # gluster volume create testvol replica 3 arbiter 1 servera:/bricks/test/testvol_n1 serverb:/bricks/test/testvol_n2 serverc:/bricks/test/testvol_n3 \
     # gluster volume create secondvol replica 3 arbiter 1 serverd:/bricks/test/secondvol_n1 servere:/bricks/test/secondvol_n2 serverc:/bricks/test/secondvol_n3
	```
    通过以上命令， serverc 便集中为两个卷 teatvol 和 secondvol 的仲裁卷节点。
  - 链式仲裁卷，原理如官方图
    ![22-412-g4](/images/22423/4.png)
  
- 5. 复制卷转换为仲裁卷
  - 5.1 将两副本复制卷转换为仲裁卷
    - 5.1.1 检查卷是否不在自愈中
	  ```
	  # gluster volume heal testvol info
	  ```
      ![22-423-g1](/images/22423/1.png)

    - 5.1.2 禁用并停止自我修复
	  ```
# gluster volume set testvol cluster.data-self-heal off
# gluster volume set testvol cluster.metadata-self-heal off
# gluster volume set testvol cluster.entry-self-heal off
# gluster volume set testvol self-heal-daemon off
	  ```
	
	- 5.1.3 为该卷添加仲裁卷
	  ```
	  # gluster volume add-brick testvol replica 3 arbiter 1 serverc:/bricks/test/arbiter_brick
	  ```
    
	- 5.1.4 等待完成，大约五分钟
	- 5.1.5 重新启用自愈
	  ```
# gluster volume set testvol cluster.data-self-heal on
# gluster volume set testvol cluster.metadata-self-heal on
# gluster volume set testvol cluster.entry-self-heal on
# gluster volume set testvol self-heal-daemon on
	  ```

	- 5.1.6 使用`5.1.1`的命令检查所有条目是否已经恢复

  - 5.2 将分布式三向复制卷转为仲裁卷（不支持配置异地复制的卷）
    - 前提：创建三副本分布式卷
	  ```
	  # gluster volume create testvol replica 3 servera:/bricks/test/testvol_n1 serverb:/bricks/test/testvol_n2 serverc:/bricks/test/testvol_n3 servera:/bricks/test/testvol_n4 serverb:/bricks/test/testvol_n5 serverc:/bricks/test/testvol_n6
	  ```
	  ![22-423-g2](/images/22423/2.png)
	  
	- 5.2.1 确认不在自愈中
	  ```
	  # gluster volume heal testvol info
	  ```
	  
	- 5.2.2 将卷的副本数减少为`2`
	  ```
	  # gluster volume remove-brick testvol replica 2 servera:/bricks/test/testvol_n1 servera:/bricks/test/testvol_n4  force
	  ```
    - 5.2.3 禁用并停止自我修复
	  ```
# gluster volume set testvol cluster.data-self-heal off
# gluster volume set testvol cluster.metadata-self-heal off
# gluster volume set testvol cluster.entry-self-heal off
# gluster volume set testvol self-heal-daemon off	
	  ```
	
	- 5.2.4 为卷添加仲裁 brick
	```
	#  gluster volume add-brick testvol replica 3 arbiter 1 servera:/bricks/arbiter_brick1 servera:/bricks/arbiter_brick2
	```
	  
	- 5.2.5 检查是否可用
	  ```
# gluster volume info testvol
# gluster volume status testvol
	  ```
	- 5.2.6 重新启用自愈
	  ```
# gluster volume set testvol cluster.data-self-heal on
# gluster volume set testvol cluster.metadata-self-heal on
# gluster volume set testvol cluster.entry-self-heal on
# gluster volume set testvol self-heal-daemon on
	  ```

	- 5.2.7 使用`5.2.1`的命令检查所有条目是否已经恢复	  

  - 5.3 将仲裁卷转为三副本卷（不支持配置异地复制的卷）
	- 5.3.1 确认不在自愈中
	  ```
	  # gluster volume heal testvol info
	  ```
	- 5.3.2 移除仲裁 brick
	  ```
# gluster volume remove-brick testvol replica 2 servera:/bricks/arbiter_brick1 servera:/bricks/arbiter_brick2 force
	  ```

    - 5.3.3 禁用并停止自我修复
      ```
# gluster volume set testvol cluster.data-self-heal off
# gluster volume set testvol cluster.metadata-self-heal off
# gluster volume set testvol cluster.entry-self-heal off
# gluster volume set testvol self-heal-daemon off		  
      ```
	  
	- 5.3.4 添加 brick

	  ```
# gluster volume add-brick testvol replica 3  servera:/bricks/test/testvol_n1  servera:/bricks/test/testvol_n4
	  ```
	  
	- 5.3.5 重新启用自愈
	  ```
# gluster volume set testvol cluster.data-self-heal on
# gluster volume set testvol cluster.metadata-self-heal on
# gluster volume set testvol cluster.entry-self-heal on
# gluster volume set testvol self-heal-daemon on
	  ```

	- 5.3.6 使用`5.3.1`的命令检查所有条目是否已经恢复	  
	  
  - 5.4 针对仲裁卷，如果有专用仲裁节点，则仲裁器可以配置为 JBOD，数据块可以使用 RAID6。链式仲裁卷则推荐 RAID6。
  
---
  
## 创建分散卷`Dispersed Volumes`	  
通过纠删码数据保护，以`n = k + m`表示，其中:
`n` 表示 brick 总数
`k` 表示 `k` 个 brick 中的数据可以恢复原数据
`m` 表示最多可以损坏的 brick 数量

- 分散卷支持以下配置
   | n = k + m | 冗余级别 |
   |--- | --- | 
   | 6 = 4 + 2 | 冗余级别 2 |
   | 10 = 8 + 2  | 冗余级别 2 |
   | 11 = 8 + 3 | 冗余级别 3 |
   | 12 = 8 + 4 | 冗余级别 4 |
   | 20 = 16 + 4 | 冗余级别 4 | 	  

- 创建 6 brick，冗余级别 2 的分散卷
  ```
# gluster v create glustervol disperse-data 4 redundancy 2 \
  servera:/bricks/test/testvol_n1 serverb:/bricks/test/testvol_n2 serverc:/bricks/test/testvol_n3 \
  servera:/bricks/test/testvol_n4 serverb:/bricks/test/testvol_n5 serverc:/bricks/test/testvol_n6 
  ```

- 如果使用`smb`协议进行分享分散卷，建议关闭`open-behind`，以避免出现大文件工作瓶颈
  ```
# gluster volume set testvol open-behind off
  ```

---

## 创建分布式分散卷`Distributed Dispersed Volumes`  
- 创建 6 brick，冗余级别 2 的2分布式分散卷，优化建议参考分散卷
  ```
# gluster v create glustervol disperse-data 4 redundancy 2 \
  servera:/bricks/test/testvol_n1 serverb:/bricks/test/testvol_n2 serverc:/bricks/test/testvol_n3 \
  servera:/bricks/test/testvol_n4 serverb:/bricks/test/testvol_n5 serverc:/bricks/test/testvol_n6 \
  servera:/bricks/test/testvol_n7 serverb:/bricks/test/testvol_n8 serverc:/bricks/test/testvol_n9 \
  servera:/bricks/test/testvol_n10 serverb:/bricks/test/testvol_n11 serverc:/bricks/test/testvol_n12   
  ```  	  	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  
	  

