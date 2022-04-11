---
title: glusterfs-高可用
date: 2022-04-04 18:59:18
tags: 
  - glusterfs
categories: 
  - storage
  - 分布式存储
---

# gluster高可用配置
<!-- more -->
---
### gluster samba CTDB以启用Samba卷自动IP切换
- 1. 安装samba 和 ctdb 相关软件包
  repo channel: rh-gluster-3-samba-for-rhel-7-server-rpms
- 2. 更新 `/var/lib/glusterd/hooks/1/start/post/S29CTDBsetup.sh`
  `/var/lib/glusterd/hooks/1/stop/pre/S29CTDB-teardown.sh`
  使`META="all"`，变成META=ctdbmeta
- 3. 编辑 /etc/samba/smb.conf ，[global]块中添加以下
  ```
  clustering=yes
  ```
- 4. 确保所有节点 /etc/ctdb/nodes 中文件完全一致为节点ip
- 5. 创建以下文件 /etc/ctdb/public_addresses ，配置浮动ip地址
  ```
  ip/24 eth0
  ```
- 6. 启用ctdb服务
  ```
  # systemctl enable ctdb --now
  ```
- 7. 配置完成后，将卷通过samba导出
- 8. 管理CTDB
  ```
  # ctdb pnn 显示此主机的物理节点号
  # ctdb status 查看常规状态
  # ctdb ping -n all 对进群所有主机检测是否正常
  # ctdb ip 查看对此节点已知浮动ip列表 `-v`更详细的信息
  # ctdb ipinfo IP 显示指定浮动ip的详细信息以及报错哪一个节点在使用
  # ctdb ip IP PNN 将指定ip地址移到指定节点
  ```
  
---

### gluster nfs Canesha 配置
- 1. 开启以下软件包频道
  rh-gluster-3-nfs-for-rhel-7-server-rpms
  rhel-ha-for-rhel-7-server-rpms
  安装前请确保所有glusterd相关进程，已经终端glusterfs与glusterfsd进程
  ```
  # yum install glusterfs-ganesha
  ```
- 2. 编辑 `/etc/ganesha/ganesha-ha.conf`
  修改以下变量，并确保每一个节点上内容相同
  | 选项 | 内容 |
  |:-----|:-----|
  |HA_NAME|此名称要在运行的网络段内唯一|
  |HA_VOL_SERVER|指向gluster集群中的一个server|
  |HA_CLUSTER_NODES|显示`NFS-Ganesha HA`集群中主机列表|
  |VIP_HOSTNAME_WITH_DOTS_REPLACED_WITH_UNDERSCORES|为集群中的每个节点配置虚拟ip|
  - 示例:
    <pre><code>HA_NAME="ganesha-test"
    HA_VOL_SERVER="servera"
    HA_CLUSTER_NODES="servera,serverb"
    VIP_servera_lab_example_com="172.25.250.X"
    VIP_serverb_lab_example_com="172.25.250.Y"</code></pre>
- 3. 配置集群
  - 3.1 启动pacemaker 和pcsd 服务
    ```
	# systemctl enable pacemaker pcsd
	# systemctl start pcsd
	```
  - 3.2 对所有节点 pcs 通信进行身份验证
    ```
	# echo redhat | passwd ---stdin hacluster
	# pcs cluster auth -u hacluster -p redhat servera serverb
	```
- 4. 配置ssh key，启用nfs-ganesha免密码登录。
  - 4.1 创建ssh key 并且放置在 servera 和 server b 相同位置
    ```
    # ssh-keygen -f /var/lib/glusterd/nfs/secret.pem -t rsa -N ''
    ```
  - 4.2 配置可以免密登录两个节点
    ```
	# ssh-copy-id -i /var/lib/glusterd/nfs/secret.pem.pub root@servera
	# ssh-copy-id -i /var/lib/glusterd/nfs/secret.pem.pub root@serverb
	```
- 5. 配置集群启动glusterd以及启动共享存储
  ```
  # systemctl start glusterd
  # gluster volume set all cluster.enable-shared-storage enable
  ```
- 6. 配置nfs-ganesha，使用默认20048端口用于mountd模块
     /etc/ganesha/ganesha.conf 中 `NFS_core_Param`模块添加以下行
	 ```
	 MNT_Port = 20048;
	 ```
- 7. 启用nfs-ganesha，导出卷后可以进行vip挂载
  ```
  # gluster nfs-ganesha enable
  # gluster volume set <volname> ganesha.enable on
  ```