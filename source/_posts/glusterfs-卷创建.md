---
title: glusterfs-卷创建
date: 2022-04-04 15:56:32
tags: 
  - glusterfs
categories: 
  - storage
  - 分布式存储
---

# gluster 安装所需开启repo
- 服务端
  rhel-7-server-rpms
  rh-gluster-3-for-rhel-7-server-rpms
  rh-gluster-3-nfs-for-rhel-7-server-rpms
  rh-gluster-3-samba-for-rhel-7-server-rpms
- 客户端
  rhel-x86_64-server-7-rh-gluster-3-client
# glusterfs 创建集群
- 添加节点
```
# gluster peer probe <node-name>
# gluster peer status
# gluster pool list
```

- 移除节点
```
# gluster peer detach <node-name>
```
---
# 创建卷
- 1. 分布式卷-将多个brick全部利用，当有一个brick不可用则整个存储文件不可访问
  - 应用场景： 空间比可靠性重要
  - 创建方式：
  ```
  # gluster volume create distributevol <bricks>
  ```
- 2. 复制卷-卷中brick总数应给为副本数的倍数，当文件写入brick时也会写入副本brick
  - 应用场景： 冗余能力和高可用
  - 创建方式：
  ```
  # gluster volume create replicavol replica x <bricks>    <---其中x为副本数
  ```
- 3. 离散式卷-基于纠删码提供类似于raid5和raid6的能力。
     `N=K+M`，其中N为总brick数，K为数据brick数，M为允许出现失败的brick数，也指有K个brick可用，则数据可恢复
  - 支持离散布局：
    6 = 4 + 2
	11 = 8 + 3
	12 = 8 + 4
  - 创建方式：
    disperse-data 为 K 数
    redundancy 为 M 数
  ```
  # gluster volume create dispersevol disperse-data 4 redundancy 2 <bricks>   
  ```
- 4. 分布式复制卷-形成多个复制式集合，集合数等于卷中brick 数除以副本数
  数据分布在X个bricks上，并有Y份副本，类似于raid 10
- 5. 分布式离散卷-在就删码的基础上增加整体的副本

---

# 客户挂载卷
- 使用glusterfs挂载
  - 1. 安装必须软件包
    ```
    # yum udpate glusterfs glusterfs-fuse
    ```

  - 2. fstab中挂载可选项
    | 选项 | 用途 |
    |:-----|:-----|
    | backup-volfile-servers=servera:serverb:serverc:serverd | 如果卷制定的地址不可用，则可以此列表寻找其他可用 |
    | ro | 挂载为只读 |
    | acl     | 可以实施与设置acl |
    | _netdev | 网络文件系统      |
	
- 使用nfs挂载
  - 1. 安装必须软件包
  ```
  # yum install nfs-utils
  ```
  - 2. 挂载
    使用NFS v3 则建议配置` vers=3 `到挂载项中
- 使用smb挂载
  - 服务端配置
    - 1. 禁用卷`stat-prefetch`
	  ```
	  # gluster volume set <volname> stat-prefetch off
	  ```
	- 2. 允许从无特权端口与组成此卷的brick通信
	  ```
	  # gluster volume set <volname> server.allow-insecure on
	  ```
    - 3. 将fsync延迟设置为0ms，确保适当的锁和I/O一致性
	  ```
	  # gluster volume set <volname> storage.batch-fsync-delay-usec 0
	  ```
    - 4. 将glusterd守护进程配置为允许来自无特权端口
	  ```
	  # cat /etc/glusterfs/gluster.vl
	  option rpc-auth-allow-insecure on
	  # systemctl restart glusterd
	  ```
    - 5. 重启后自动触发脚本 /var/lib/glusterd/hooks/1/post/S30samba-start.sh 向卷 /etc/samba/smb.conf 中添加导出
	- 6. 如果没有配置samba中央化用户信息与验证，请创建本地用户并配置samba密码
	  ```
	  # adduser -s /sbin/nologin smbuser
	  # smbpasswd -a smbuser
	  ```
	- 7. 如果为单独的卷配置禁用通过samba共享
	  ```
	  # gluster volume set <volname> user.cifs disable 
	  ```
  - 客户端配置
    - 1. 安装`cifs-utils`软件包
	- 2. 验证samba挂载点是否可用
	  ```
	  # smbclient -L servera -U smbuser%redhat
	  ```
	- 3. fstab挂载
	  ```
	  # cat /etc/fstab
	  //servera/volname /mnt/smbsata cifs user=smbuser,pass=redhat 0 0
	  ```
---
# 卷配置项
|选项 |默认值 |描述 |	
|:-----|:-----|:-----|
|auth.allow | * | 客户端以`,`分隔，配置可以连接到卷的客户端 |
|auth.reject||以`,`分隔，配置不允许访问的客户端，若同时拒绝与允许则默认拒绝|
|nfs.rpc-auth-allow|*|以`,`分隔，配置可以nfs访问卷的客户端| 
|nfs.rpc-auth-reject||以`,`分隔，配置不可以nfs访问卷的客户端|
|nfs.disable||如果配置为on，则不允许nfs v3分享|
|features.read-only|off|卷是否导出为对所有访问的客户端只读|
|server.root-squash|off|如果配置为on，客户端上root用户将映射为nfsnobody用户|

---

# 卷配额
- 启用配额
  ```
  # gluster volume quota <volname> enable
  ```
- 设置目录限制
  ```
  # gluster volume quota <volname> limit-usage <PATH-ON-VOLUME> <SIZE>  
  ```
- 配置超时与警告
  ```
  # gluster volume quota <volname> default-soft-limit <SOFTLIMIT-PERCENTAGE>
  ```
  一般软限制为硬限制的 `80%`，且默认60S
  ```
  # gluster volume quota <volname> hard-timeout <TIMEOUT>
  ```
  默认硬限制超时时间为5S
- 报告配额
  ```
  # gluster volume quota <VOLUME> list
  ```
  如果设置卷选项 `quota-deem-statfs on`，则` df `命令会将` hard-limit `报告为目录的可用空间
  ```
  # gluster volume set <volname> quota-deem-statfs on
  ```