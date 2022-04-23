---
title: glusterfs-学习记录 <font color=blue>一</font>
date: 2022-04-12 23:45:03
tags:
  - gluster
categories:
  - storage
  - 分布式存储
---

<h2>glusterfs 学习记录 <font color=red>一</font></h2>

![22-412-g1](/images/22412/g1.png)
<html><pre><code>前提：基于红帽商业版glusterfs 3.5
本文主要内容：安装/启用web console，客户端配置</code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->
---

##  环境介绍
- glusterfs 集群环境如下
   | 主机名 | IP | 角色 |
   |--- | --- | --- |
   | servera.glusterfs.linuxone.in | 192.168.31.110 | prod |
   | serverb.glusterfs.linuxone.in | 192.168.31.111 | prod |
   | serverc.glusterfs.linuxone.in | 192.168.31.112 | prod |
   | serverb.glusterfs.linuxone.in | 192.168.31.113 | prod |
   | servere.glusterfs.linuxone.in | 192.168.31.114 | 异地复制 |
   | manager.glusterfs.linuxone.in | 192.168.31.109 | 管理/监控 |
   | client.glusterfs.linuxone.in  | 192.168.31.108 | client | 
   
---

##  安装 glusterfs
- 前提
  - rhel 6 开启的repo
    ```
    rhel-6-server-rpms
    rhel-scalefs-for-rhel-6-server-rpms
    rhs-3-for-rhel-6-server-rpms
	samba需要开启的repo:  rh-gluster-3-samba-for-rhel-6-server-rpms
	rhel 6 不支持 nfs
	```
	
  - rhel 7 
    ```
	rhel-7-server-rpms
	rh-gluster-3-for-rhel-7-server-rpms
	samba需要开启 或 需要 CTDB 的repo: rh-gluster-3-samba-for-rhel-7-server-rpms
	nfs需要开启repo: rh-gluster-3-nfs-for-rhel-7-server-rpms  rhel-ha-for-rhel-7-server-rpms
	rhel-7-server-ansible-2-rpms
	```

  - rhel 8 
    ```
	rhel-8-for-x86_64-baseos-rpms
	rhel-8-for-x86_64-appstream-rpms
	rh-gluster-3-for-rhel-8-x86_64-rpms
	samba需要开启repo: rh-gluster-3-samba-for-rhel-8-x86_64-rpms
	nfs需要开启repo: rh-gluster-3-nfs-for-rhel-8-x86_64-rpms rhel-8-for-x86_64-highavailability-rpms
	ansible-2-for-rhel-8-x86_64-rpms 
	```
	
- 安装 `Red Hat Gluster Storage`
  ```
  # yum install redhat-storage-server
  ```
   
---
   
##  创建加密集群
- 1.1 server`a` / `b` / `c` / `d` / `e` / `client` 节点创建自签名证书
  - 进入对应目录
    ```
	# cd /etc/ssl
	```

  - 创建key
    ```
	# openssl genrsa -out glusterfs.key 2048
	```
	
  - 创建pem ，注意glusterfs中必须为`.pem`结尾，并复制一份附件
    ```
	# openssl req -new -x509 -key glusterfs.key -subj "/CN=servera" -out glusterfs.pem
	# cp glusterfs.pem servera.pem   <<<仅以servera为例
	```
	
  - 将对应主机的附件上传至一台主机同一目录
    ```
	# scp ./servera.pem manager.glusterfs.linuxone.in:~  <---此处以servera为例
	```

  - 复制完成后，在manager节点创建ca证书，并将`glusterfs.ca`复制回对应主机的`/etc/ssl`目录
    ```
	# cat servera.pem serverb.pem serverc.pem serverd.pem servere.pem client.pem > glusterfs.ca
	# for i in servera serverb serverc serverd servere client
	  do
	  scp glusterfs.ca root@$i:/etc/ssl
	  done
	```
	
- 1.2 在所有主机创建 `secure-access` 文件并重启`glusterd`服务
    ```
	# for i in servera serverb serverc serverd servere client
	  do
	  ssh root@$i touch /var/lib/glusterd/secure-access && systemctl restart glusterd
	  done
	```
	
- 1.3 在servera添加节点
    ```
	# gluster peer probe serverb
	# gluster peer probe serverc
	# gluster peer probe serverd
	```
	
- 1.4 检查是否添加成功
    ```
	# gluster peer status
	Number of Peers: 3

    Hostname: serverb
    Uuid: 41663cd0-0de9-4a5d-a516-4130821b07a5
    State: Peer in Cluster (Connected)
    
    Hostname: serverc
    Uuid: 1610842b-6e87-458e-b66c-0459e07e1c95
    State: Peer in Cluster (Connected)

    Hostname: serverd
    Uuid: d9381deb-e2b1-4ea3-a8ae-a75d2e3ecef0
    State: Peer in Cluster (Connected)

	# gluster pool list
	UUID					Hostname 	State
    41663cd0-0de9-4a5d-a516-4130821b07a5	serverb  	Connected 
    1610842b-6e87-458e-b66c-0459e07e1c95	serverc  	Connected 
    d9381deb-e2b1-4ea3-a8ae-a75d2e3ecef0	serverd  	Connected 
    5bf6133c-b535-42c7-be31-79e063237f44	localhost	Connected 
	```
	
- 1.5 移除节点
    ```
	# gluster peer detach serverb <---这里以移除serverb节点为例子
	```

---

##  安装 web console
- 前提
  - gluster web console 节点开启以下软件包频道
    ```
	# rhel-7-server-rpms
	# rhel-7-server-ansible-2-rpms
	# rh-gluster-3-web-admin-server-for-rhel-7-server-rpms
	```
	
  - gluster 存储节点开启以下软件包频道
    ```
	# rhel-7-server-rpms
	# rhel-7-server-ansible-2-rpms
	# rh-gluster-3-for-rhel-7-server-rpms
	# rh-gluster-3-web-admin-agent-for-rhel-7-server-rpms
	```

  - 创建挂载点。对于glusterfs，安装前要创建单独挂载点
    - 小型集群，节点数 < 8
	  - `/var/lib/etcd` `xfs`  > 20GB
      - `/var/lib/carbon` `xfs` > 200GB	
    - 中型集群，节点数 >9 而 < 16 ，
	  - `/var/lib/etcd` `xfs`  > 20GB
      - `/var/lib/carbon` `xfs` > 350GB	
	- 大型集群，节点数 >17 而 <24
	  - `/var/lib/etcd` `xfs`  > 20GB
      - `/var/lib/carbon` `xfs` > 500GB	
  
	
- 安装步骤
  - 1. 配置所有节点 root ssh 免密登录，并于manager节点安装必要软件包
    ```
	# yum install tendrl-ansible
	```

  - 2. 创建如下示例 `inventory` 文件
    <html><pre><code>[tendrl_server]
    manager.glusterfs.linuxone.in
    [gluster_servers]
    servera.glusterfs.linuxone.in
    serverb.glusterfs.linuxone.in
    serverc.glusterfs.linuxone.in
    serverd.glusterfs.linuxone.in
    [all:vars]
    etcd_ip_address=192.168.31.109
    etcd_fqdn=manager.glusterfs.linuxone.in
    graphite_fqdn=manager.glusterfs.linuxone.in
    <font color=red>configure_firewalld_for_tendrl=false</font>   <<< 由于之前关闭了firewalld，所以需要配置该选项</code></pre></html>

  - 3. 将playbook复制到当前目录
    ```
	# cp /usr/share/doc/tendrl-ansible-1.6.3/site.yml .
	# cp /usr/share/doc/tendrl-ansible-1.6.3/prechecks.yml .
	```
	
  - 4. 分别执行playbook，等待安装完成
    ```
	# ansible-playbook -i invntory prechecks.yml
	# ansible-playbook -i invntory site.yml 
	```
	
  - 5. 安装完成，manager 页面登录 `username`: `admin` | `password`: `adminuser`
    同时会在当前目录生成 ` grafana_admin_passwd` 文件，里面记录了 grafana 的密码
	![22-412-g2](/images/22412/g2.png)
	
---

##  web console 开启卷分析
- 前提
  - 创建 `lvm` ，挂载并启动卷
    ```
	# pvcreate /dev/sdb
    # vgcreate vg_bricks /dev/sdb
    # lvcreate -T vg_bricks/brickpool -l 100%FREE
    # lvcreate -T vg_bricks/brickpool -V 2G -n brick1
    # mkfs.xfs -i size=512 /dev/vg_bricks/brick1
    # mkdir /bricks/test
    # echo "/dev/vg_bricks/brick1 /bricks/test xfs defaults 0 0" >> /etc/fstab
    # mount -a
    # gluster volume create testvol servera:/bricks/test/testvol
    # gluster volume status
    # gluster volume start testvol
	```

- 在web console 开启卷分析`Enable Profiling`
  - `volumes` > `Enable Profiling`
    ![22-412-g3](/images/22412/g3.png)
  - 开启后，会对卷的使用数据进行分析
    ![22-412-g4](/images/22412/g4.png)
  - 关闭卷分析 `volumes` > `Disable Profiling`
    ![22-412-g5](/images/22412/g5.png)

---

##  web console 扩展集群增加节点

- 前提
  - 1. 节点已经配置完成
  - 2. 节点已经配置证书`glusterfs.ca` / `glusterfs.key` / `glusterfs.pem`
  - 3. 配置管理节点至扩展节点 ssh 免密码登录
- 通过 web console 扩展步骤
  - 1. 添加扩展节点至`inventory`
    ```
	# cat invntory 
	[gluster_servers]
	servera.glusterfs.linuxone.in
	serverb.glusterfs.linuxone.in
	serverc.glusterfs.linuxone.in
	serverd.glusterfs.linuxone.in
	servere.glusterfs.linuxone.in   <<<
	```
  - 2. 运行play book
    ```
	# ansible-playbook -i invntory site.yml
	```

  - 3. web console 进行扩展
    - 3.1 点击 expend
      ![22-412-g6](/images/22412/g6.png)   
	- 3.2 点击 expand
	  ![22-412-g7](/images/22412/g7.png)   

---

##  web console 取消管理集群
- 前提
  集群已经被 web console 管理
  
- web console 中点击 unmanage
  ![22-412-g8](/images/22412/g8.png)   

---

##  监控信息保存配置

- 1. 取消管理集群
- 2. 停止 `carbon-cache` 服务
  ```
  # systemctl stop carbon-cache
  ```
- 3. 修改配置文件:
  ```
  # vim /etc/tendrl/monitoring-integration/storage-schemas.conf
  [tendrl]
  pattern = ^tendrl\.
  retentions = 60:180d  <<<
  ```
  `60`为数据点
  `180d`为保存天数，数据点相加最多180天的数据
- 4. 重启 `carbon-cache` 服务
  ```
  # systemctl start carbon-cache
  ```
---

##  web 管理的日志等级
- 管理组件
  - web 管理组件
    - tendrl-monitoring-integration
	- tendrl-notifier
  - storage 节点管理组件
    - tendrl-gluster-intergration

- 日志等级
  - DEBUG
  - INFO
  - WARNING
  - ERROR(DEFAULT)
  - CRITICAL

- 修改日志等级
  - 配置文件 `/etc/tendrl/node-agent/node-agent_logging.yaml`  
	<html><pre><code># vim /etc/tendrl/node-agent/node-agent_logging.yaml
	handlers:
    console:
        class: logging.StreamHandler
        level: ERROR
        stream: ext://sys.stdout

    info_file_handler:
        class: logging.handlers.SysLogHandler
        facility: local5
        address: /dev/log
        level: ERROR
	root:
        level: ERROR
        handlers: [console, info_file_handler]</code></pre></html>

---

## 配置客户端
- 前提
  - RHEL 6
    ```
	rhel-6-server-rpms 
	rhel-6-server-rhs-client-1-rpms
    ```	
  - RHEL 7
    ```
	rhel-7-server-rpms
	rhel-7-server-rh-common-rpms
	rh-gluster-3-client-for-rhel-7-server-rpms
    ```	
  - RHEL 8
    ```
	rh-gluster-3-client-for-rhel-8-x86_64-rpms
    ```	
- 安装客户端
  ```
  # yum install glusterfs glusterfs-fuse
  ```
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	