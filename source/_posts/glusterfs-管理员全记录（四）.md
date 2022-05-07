---
title: glusterfs-管理员全记录（四）
date: 2022-04-23 18:19:55
tags:
  - gluster
categories:
  - storage
  - 分布式存储
---

<h3>glusterfs 学习记录 <font color=yellow>四</font></h3>

![22-412-g1](/images/22412/g1.png)
<html><pre><code>前提：基于红帽商业版glusterfs 3.5
本文主要内容：client端管理</code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

---

### 原生客户端升级
- 不同版本的repo
  - rhel6
    - rhel-6-server-rpms 
	- rhel-6-server-rhs-client-1-rpms
  - rhel7
    - rhel-7-server-rpms
	- rh-gluster-3-client-for-rhel-7-server-rpms
  - rhel8
    - rh-gluster-3-client-for-rhel-8-x86_64-rpms
- 升级前提请 umount  gluster 挂载点
- 升级 glusterfs 客户端
  ```
  # yum update glusterfs glusterfs-fuse
  ```
---
  
### 原生客户端glusterfs 常用挂载选项
- `backup-volfile-servers=serverb:serverc:serverd`\
  备用的卷服务器列表，当主服务器出现故障时，会依次尝试从所有备用服务器进行挂载
- `log-level`
  配置日志等级
- `log-file`
  日志存储的位置
- `transport-type`
  客户端传输类型，默认是 tcp 
- `dump-fuse`
  配置glusterfs客户端生成dump文件
- `ro`
  只读
- `background-qlen=n`
  客户端n个请求后的请求将会被拒绝。
- `enable-ino32`
  启用使文件系统展示32位inode 而不是64位
- `reader-thread-count=n`
  添加 n 个读取线程，从而提供更好io性能，默认为1
- `lru-limit`
  在到达 inode 限制后，此mount会从最近最少使用 lru 列表中清除 inode
- `acl`
  启用 posix 访问控制列表，可配置acl权限

- 挂载原生客户端，除mount命令外，可以配置 `fstab` 文件:
```
servera:/testvol /test glusterfs defaults 0 0
```
  - 如果是子目录，则
    ```
	servera:/testvol/subdir /test glusterfs defaults 0 0
	```
  
---

### NFS 配置 CTDB 

- 1. Gluster NFS
  - 1.1 gluster 卷开启 nfs 功能
    ```
	# gluster volume set testvol nfs.disable off
	```
	
  - 1.2 配置 nfs 被指定网段访问
    ```
	# gluster volume set testvol nfs.rpc-auth-allow '172.25.250.*'
	```
  - 1.3 配置 nfs acl 权限
    ```
	# gluster volume set testvol nfs.acl on
	```
- 2. 为 Gluster NFS 配置 CTDB
  - 2.1 为所有 NFS 服务器的节点安装 CTDB
    ```
	# yum install CTDB
	```
  - 2.2 修改配置文件（针对rhel7） 并重启nfs服务
    ```
	# sed -i '/STATD_PORT/s/^#//' /etc/sysconfig/nfs
	# systecmctl restart nfs-config
	# systecmctl restart rpc-statd
	# systecmctl restart nfs-mountd
	# systecmctl restart nfslock
	```
  - 2.3 配置 CTDB
      - 2.3.0 为ctdb单独配置卷
      - 2.3.1 配置以下文件
	    ```	
/var/lib/glusterd/hooks/1/start/post/S29CTDBsetup.sh
/var/lib/glusterd/hooks/1/stop/pre/S29CTDB-teardown.sh	   
	    ```
      - 2.3.2 将 META='all' 替换为ctdb卷名 META='ctdb'
        ```
		# sed -i 's/META="all"/META="ctdb"/g' /var/lib/glusterd/hooks/1/start/post/S29CTDBsetup.sh
        ```	
      - 2.3.3 启动CTDB 卷
        ```
		# gluster volume start ctdb	
		```

	  - 2.3.4 检查配置文件是否存在
	    ```
		# cat /etc/sysconfig/ctdb
		```
	  - 2.3.5 添加节点ip至所有节点配置文件
	    ```
# cat /etc/ctdb/nodes
192.168.31.110
192.168.31.111
192.168.31.112
		```
	  - 2.3.6 配置所有节点需您ip用于故障迁移
	    ```
# cat /etc/ctdb/public_addresses
192.168.31.32/24 ens33
192.168.31.33/24 ens33
		```
    - 2.4 gluster nfs 的ctdb仅提供节点级别高可用，无法检测NFS故障，如果NFS服务的节点仍处于运行状态，则不会提供高可用
  
- 3. 客户端挂载，仅支持 v3
  - 3.1 命令行挂载
  ```
# mount -t nfs -o vers=3 servera:/ctdb xxx/
  ```
  - 3.2 开机自动挂载
  ```
# cat /etc/fstab
servera:/ctdb /mnt/xxx nfs defaults,_netdev 0 0
  ```
  - 3.3 对nfs精细控制
    - `nfs.export-dirs`
      默认开启，允许客户端在nfs卷创建子目录，提供nfs服务时可以无需指定子目录
	- `nfs.export-dir`
	  若配置为`on`，允许您指定一个或多个导出子目录，参考如下命令
	  ```
	  # gluster set ctdb volume nfs.export-dir /dir1,/dir2(192.168.31.0/24)
	  ```

---

### NFS Ganesha
- 1. 准备工作(rhel 7)
  - 1 配置 NFS Ganesha
    - 1.1 安装并配置修改以下内容
  ```
  # yum install glusterfs-ganesha -y
  # sed -i '/STATD_PORT/s/^#//' /etc/sysconfig/nfs
  # systemctl restart nfs-config
  # systemctl restart rpc-statd
  # sed -i '/STATD_PORT/s/^#//' /etc/sysconfig/nfs
  # sed -i '/LOCKD_TCPPORT/s/^#//' /etc/sysconfig/nfs
  # sed -i '/LOCKD_UDPPORT/s/^#//' /etc/sysconfig/nfs
  # systemctl restart nfs-config
  # systemctl restart rpc-statd
  # systemctl restart nfslock
  ```
      NFS-Ganesha 支持最小节点数为3，最大节点是8
    - 1.2 禁用 kernel-nfs
    ```
# systemctl stop nfs-server
# systemctl disable nfs-server
	```
    - 1.3 开启全局共享存储
    ```
	# gluster volume set all cluster.enable-shared-storage enable
	```
    - 1.4 修改配置文件 `ganesha-ha.conf`
    <pre><code>HA_NAME="ganesha-ha-360"
HA_CLUSTER_NODES="servera.glusterfs.linuxone.in,serverb.glusterfs.linuxone.in,serverc.glusterfs.linuxone.in"
VIP_servera_glusterfs_linuxone_in="192.168.31.114"
VIP_serverb_glusterfs_linuxone_in="192.168.31.100"
VIP_serverc_glusterfs_linuxone_in="192.168.31.99"</code></pre>
    - 1.5 将配置文件 `ganesha.conf` & `ganesha-ha.conf` 复制到目标目录`/var/run/gluster/shared_storage/nfs-ganesha`
    ```
	# mkdir /var/run/gluster/shared_storage/nfs-ganesha
	# cp /etc/ganesha/ganesha.conf /etc/ganesha/ganesha-ha.conf /var/run/gluster/shared_storage/nfs-ganesha/
	```
    - 1.6 配置以下服务开机启动
    ```
	# systemctl enable glusterfssharedstorage.service
	# systemctl enable nfs-ganesha
	```
- 2. 配置集群服务
    - 2.1 启用 pacemaker 和 pcsd 服务
	  ```
	  # systemctl enable pacemaker.service
	  # systemctl enable pcsd --now
      ```
	- 2.2 配置 `hacluster` 用户密码
	  ```
	  # echo redhat | passwd --stdin hacluster
	  ```
    - 2.3 为所有节点配置认证
	  ```
# pcs cluster auth servera.glusterfs.linuxone.in serverb.glusterfs.linuxone.in serverc.glusterfs.linuxone.in
Username: hacluster
Password: 
serverb.glusterfs.linuxone.in: Authorized
serverc.glusterfs.linuxone.in: Authorized
servera.glusterfs.linuxone.in: Authorized
	  ```
	- 2.4 配置 ssh 密钥(在 servera 上执行)
	  ```
	  # ssh-keygen -f /var/lib/glusterd/nfs/secret.pem -t rsa -N ''
	  # ssh-copy-id -i /var/lib/glusterd/nfs/secret.pem.pub root@serverb
	  # ssh-copy-id -i /var/lib/glusterd/nfs/secret.pem.pub root@serverc
	  # scp -i /var/lib/glusterd/nfs/secret.pem /var/lib/glusterd/nfs/secret.* root@serverb:/var/lib/glusterd/nfs/
	  # scp -i /var/lib/glusterd/nfs/secret.pem /var/lib/glusterd/nfs/secret.* root@serverc:/var/lib/glusterd/nfs/
	  ```

- 2. 通过命令行配置 NFS-Ganesha
  - 2.1 配置 ha 集群
    ```
# gluster nfs-ganesha enable 
Enabling NFS-Ganesha requires Gluster-NFS to be disabled across the trusted pool. Do you still want to continue?
 (y/n) y
This will take a few minutes to complete. Please wait ..
nfs-ganesha : success 
    ```	
	- 如果出现失败的情况，检查 rpc-statd 服务是否监听在 662 端口
	  ```
	  # rpcinfo -p
	  ```
	- 如果不是，重启 rpc-statd 服务
	  ```
	  # systemctl restart rpc-statd
	  ```
  - 2.2 取消 ha 集群
    ```
	# gluster nfs-ganesha disable
	```
  - 2.3 检查集群状态
    ```
#  /usr/libexec/ganesha/ganesha-ha.sh --status /var/run/gluster/shared_storage/nfs-ganesha
Online: [ servera.glusterfs.linuxone.in serverb.glusterfs.linuxone.in serverc.glusterfs.linuxone.in ]

servera.glusterfs.linuxone.in-cluster_ip-1 servera.glusterfs.linuxone.in
serverb.glusterfs.linuxone.in-cluster_ip-1 serverb.glusterfs.linuxone.in
serverc.glusterfs.linuxone.in-cluster_ip-1 serverc.glusterfs.linuxone.in

Cluster HA Status: HEALTHY
	```
  - 2.4 配置卷通过 nfs-ganesha 共享
    ```
	# gluster vol set testvol ganesha.enable on
	```
  - 2.5 客户端调优
    ```
# sysctl -w sunrpc.tcp_slot_table_entries=128
# echo 128 > /proc/sys/sunrpc/tcp_slot_table_entries
# echo 128 > /proc/sys/sunrpc/tcp_max_slot_table_entries	
	```
  - 2.6 客户端尝试挂载nfs
    - 2.6.1 通过 nfs v3 进行挂载
	  ```
	  # mount -t nfs -o vers=3 192.168.31.114:/testvol ./nfs/
	  ```
	- 2.6.2 通过 nfs v4 进行挂载
	  ```
# mount -t nfs -o vers=4 192.168.31.114:/testvol ./nfs/
# df
Filesystem              1K-blocks    Used Available Use% Mounted on
devtmpfs                   914428       0    914428   0% /dev
tmpfs                      931516       0    931516   0% /dev/shm
tmpfs                      931516   10464    921052   2% /run
tmpfs                      931516       0    931516   0% /sys/fs/cgroup
/dev/mapper/rhel-root     8374272 4612860   3761412  56% /
/dev/sda1                 1038336  187480    850856  19% /boot
tmpfs                      186304       8    186296   1% /run/user/42
tmpfs                      186304       0    186304   0% /run/user/0
192.168.31.114:/testvol   2086912   54272   2032640   3% /root/nfs    <<<
	  ```
	- 2.6.3 nfs 服务端查找客户端的ip
	  ```
# dbus-send --type=method_call --print-reply --system --dest=org.ganesha.nfsd /org/ganesha/nfsd/ClientMgr org.ganesha.nfsd.clientmgr.ShowClients
method return time=1651848475.811716 sender=:1.73 -> destination=:1.91 serial=3293 reply_serial=2
   struct {
      uint64 1651848475
      uint64 811519666
   }
   array [
      struct {
         string "::ffff:192.168.31.108"   <<< 客户端 ip
         boolean true
         boolean true
         boolean false
         boolean false
         boolean false
         boolean true
         boolean false
         boolean false
         struct {
            uint64 1651848442
            uint64 829119794
         }
      }
      struct {
         string "::ffff:192.168.31.110"   <<< 服务端 ip
         boolean false
         boolean true
         boolean false
         boolean false
         boolean false
         boolean false
         boolean false
         boolean false
         struct {
            uint64 1651847866
            uint64 926745984
         }
      }
   ]
	  ```
  - 2.7 添加节点
    ```
	# /usr/libexec/ganesha/ganesha-ha.sh --add <HA_CONF_DIR> <HOSTNAME> <NODE-VIP>
	# /usr/libexec/ganesha/ganesha-ha.sh --add /var/run/gluster/shared_storage/nfs-ganesha serverd 192.168.31.98	
	```	
    共享存储的配置文件存放目录未来将会从`/var/run/gluster/ ` 更改为 `/run/gluster/`
  - 2.8 删除节点
    ```
	# /usr/libexec/ganesha/ganesha-ha.sh --delete <HA_CONF_DIR> <HOSTNAME>
	# /usr/libexec/ganesha/ganesha-ha.sh --delete /var/run/gluster/shared_storage/nfs-ganesha serverd 
	```	
  - 2.9 配置 nfs 默认 exports 的配置
    - 2.9.1 流程
	  - 修改exports 配置文件`/run/gluster/shared_storage/nfs-ganesha/exports/export.testvol.conf`
	  - 重新加载配置
	    ```
		# /usr/libexec/ganesha/ganesha-ha.sh --refresh-config <HA_CONF_DIR> <volname>
		#  /usr/libexec/ganesha/ganesha-ha.sh --refresh-config /run/gluster/shared_storage/nfs-ganesha/ testvol
		```
    - 2.9.2 为特定客户端进行配置
	  ```
client {
        clients = 192.168.31.108;  # 客户端 ip
        access_type = "RO"; # Read-only
        Protocols = "3"; # 仅支持 nfs v3
        anonymous_uid = 1440;
        anonymous_gid = 72;
  }	  
	  ```
	- 2.9.3 开启 nfs v4 acl 权限
	  ```
	  Disable_ACL = false;
	  ```
    - 2.9.4 配置 nfs v4 伪路径
	  ```
	  Pseudo = "pseudo_path"
	  ```
  - 2.10 配置 nfs 共享子目录
    - 2.10.1 方式一
	  - 2.10.1.1 配置单独共享 exports 文件，好处是不会影响已经连接到该nfs server 的客户端
        ```
# cat export.testvol-subdir.conf
        EXPORT{
              Export_Id = 3;    <<< id 要不同于其他
              Path = "/testvol/subdir";   <<< dir在 vol 下面的路径
              FSAL {
                   name = GLUSTER;
                   hostname="localhost";  
                  volume="testvol";
            volpath="/subdir";     <<< dir在 vol 中的路径
                   }
              Access_type = RW;
              Disable_ACL = true;
              Squash="No_root_squash";
              Pseudo="/testvol/subdir";   <<< dir在 vol 下面的路径
              Protocols = "3", "4";
              Transports = "UDP","TCP";
              SecType = "sys";
             }

		```
      - 2.10.1.2 配置文件`/etc/ganesha/ganesha.conf`添加条目  
	    ```
		%include "/var/run/gluster/shared_storage/nfs-ganesha/exports/export.testvol-subdir.conf"
		```
	  - 2.10.1.3 重新配置 nfs-ganesha
	    ```
# /usr/libexec/ganesha/ganesha-ha.sh --refresh-config /run/gluster/shared_storage/nfs-ganesha/ testvol-subdir
Refresh-config completed on serverb.
Refresh-config completed on serverc.
Success: refresh-config completed.
		```
	  - 2.10.1.4 客户端进行检测
	    ```
# showmount -e 192.168.31.114
Export list for 192.168.31.114:
/testvol        (everyone)
/testvol/subdir (everyone)     <<<
		```
    - 2.10.2 方式二
	  - 2.10.2.1 使用 subdir 条目编辑卷的 export 文件。此方法只会导出子目录而不是父卷。
	    ```
EXPORT{
      Export_Id = 4;
      Path = "/testvol/subdir";
      FSAL {
           name = GLUSTER;
           hostname="localhost";
          volume="testvol";
          volpath="/subdir";
           }
      Access_type = RW;
      Disable_ACL = true;
      Squash="No_root_squash";
      Pseudo="/testvol/subdir";
      Protocols = "3", "4" ;
      Transports = "UDP","TCP";
      SecType = "sys";
     }	
		```
    - 2.10.3 取消共享子目录
	  - 2.10.3.1 获取共享的 ID，如`2.10.1.1`共享id 为 `3`
	  - 2.10.3.2 删除配置文件
	    ```
		# rm -rf /var/run/gluster/shared_storage/nfs-ganesha/exports/
		```
	  - 2.10.3.3 移除配置文件中对应条目`/etc/ganesha/ganesha.conf`
	  - 2.10.3.4 执行以下命令移除
	    ```
# dbus-send --print-reply --system --dest=org.ganesha.nfsd /org/ganesha/nfsd/ExportMgr org.ganesha.nfsd.exportmgr.RemoveExport uint16:3
		```
  - 2.11 开启`all_squash`	   
    ```
	Squash = all_squash
	```	
- 3. NFS Ganesha 停机维护
  - 3.1 ganesha 集群的宽限期最长为 20s，最大故障转移时间 20-22 秒，
  - 3.2 调整宽限期为 90秒
    ```
# vim /etc/ganesha/ganesha.conf
NFSv4 {
Grace_Period=<grace_period_value_in_sec>;
}
# systemctl restart nfs-ganesha
	```
- 4. NFS Ganesha 调整`Readdir`性能
  ```
NFS-Ganesha 进程在实例中读取目录的全部内容。该目录上的任何并行操作都会暂停，直到 readdir 操作完成。
在 Red Hat Gluster Storage 3.5 中，该Dir_Chunk参数允许在实例中以块的形式读取目录内容。
该参数默认启用。此参数的默认值为128。此参数的范围是1到UINT32_MAX。要禁用此参数，请将值设置为0
  ```
  修改配置文件
```
# vim /etc/ganesha/ganesha.conf
CACHEINODE {
        Entries_HWMark = 125000;
        Chunks_HWMark = 1000;
        Dir_Chunk = 128; # Range: 1 to UINT32_MAX, 0 to disable
}
# systemctl restart nfs-ganesha
```

- 5. Troubleshooting NFS Ganesha
  - 5.1 服务进行强制检查
    ```
# service nfs-ganesha status
# service pcsd status
# service pacemaker status
# pcs status
	```
  - 5.2 常用日志
    ```
/var/log/ganesha/ganesha.log
/var/log/ganesha/ganesha-gfapi.log
/var/log/messages
/var/log/pcsd.log
	```
  - 5.3 出现 nfs-ganesha HA 集群设置失败时需要清理。
    将机器复原成原始状态
	```
# /usr/libexec/ganesha/ganesha-ha.sh --teardown /var/run/gluster/shared_storage/nfs-ganesha
# /usr/libexec/ganesha/ganesha-ha.sh --cleanup /var/run/gluster/shared_storage/nfs-ganesha
# systemctl stop nfs-ganesha
	```
 
---

### SMB 共享
- 1. ctdb 配置 smb 高可用
  - 1.1 安装 ctdb
    ```
	# yum install ctdb -y 
	```
  - 1.2 在 gluster 存储节点配置 ctdb
    - 1.2.1 创建 ctdb 卷
	  ```
	  # gluster volume create ctdb replica 3 192.168.31.112:/bricks/ctdb/b1 192.168.31.110:/bricks/ctdb/b2 192.168.31.111:/bricks/ctdb/b3
	  ```
    - 1.2.2 修改配置将 `META="all"` 修改为 `META="ctdb"`
	  ```
	  /var/lib/glusterd/hooks/1/start/post/S29CTDBsetup.sh
      /var/lib/glusterd/hooks/1/stop/pre/S29CTDB-teardown.sh
	  ```
	- 1.2.3 配置 `/etc/samba/smb.conf`
	  ```
	  clustering=yes
	  ```
    - 1.2.4 启动 ctdb 卷
	  ```
	  # gluster volume start ctdb
	  ```
    - 1.2.5 创建 `/etc/ctdb/nodes`
	  ```
# cat /etc/ctdb/nodes
192.168.31.110
192.168.31.111
192.168.31.112
	  ```
	- 1.2.6 配置虚拟IP `/etc/ctdb/public_addresses`
	  ```
# cat /etc/ctdb/public_addresses
192.168.31.114/24 ens33
	  ```
	- 1.2.7 启动 CTDB 服务
	  ```
	# systemctl start ctdb
	  ```
    - 1.2.8 检查 ctdb 服务状态
	  ```
# ctdb status
Number of nodes:3
pnn:0 192.168.31.110   OK (THIS NODE)
pnn:1 192.168.31.111   OK
pnn:2 192.168.31.112   OK
Generation:177693459
Size:3
hash:0 lmaster:0
hash:1 lmaster:1
hash:2 lmaster:2
Recovery mode:NORMAL (0)
Recovery master:1
	  ```

- 2. 通过 SMB 共享卷	
  - 2.1 默认共享卷的配置 /etc/samba/smb.conf
    ```
[gluster-VOLNAME]
comment = For samba share of volume VOLNAME
vfs objects = glusterfs
glusterfs:volume = VOLNAME
glusterfs:logfile = /var/log/samba/VOLNAME.log
glusterfs:loglevel = 7
path = /
read only = no
guest ok = yes
    ```	
  - 2.2 版本 samba-4.8.5-104 及以上进行以下配置
    - 2.2.1 为卷开启 user.cifs 及 user.smb
	  ```
# gluster volume set testvol user.cifs enable
# gluster volume set testvol user.smb enable
# gluster volume set testvol group samba
# gluster volume set testvol stat-prefetch off
# gluster volume set testvol server.allow-insecure on
# gluster volume set testvol storage.batch-fsync-delay-usec 0
	  ```
    - 2.2.2 添加 smb 用户
	  ```
# smbpasswd -a root
	  ```
	- 2.2.3 添加以下配置，并重启 glusterd 服务
	  ```
# vim /etc/glusterfs/glusterd.vol   <<< 所有节点进行配置
option rpc-auth-allow-insecure on
# systemctl restart glusterd
# gluster volume stop testvol
# gluster volume start testvol
	  ```
  - 2.3 客户端配置 smb 共享
    - 2.3.1 检测服务端 smb 共享
      ```
# smbclient -L 192.168.31.114 -U%

	Sharename       Type      Comment
	---------       ----      -------
	gluster-testvol Disk      For samba share of volume testvol  <<<
	IPC$            IPC       IPC Service (Samba Server Version 4.11.6)
Reconnecting with SMB1 for workgroup listing.
smbXcli_negprot_smb1_done: No compatible protocol selected by server.
protocol negotiation failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available
	  ```
	- 2.3.2 测试挂载 smb 共享
	  ```
# smbclient //192.168.31.114/gluster-testvol -U root%redhat
Try "help" to get a list of possible commands.
smb: \> mkdir test
smb: \> ls
  .                                   D        0  Sat May  7 14:28:05 2022
  ..                                  D        0  Sat May  7 14:28:05 2022
  test                                D        0  Sat May  7 14:28:05 2022

		2086912 blocks of size 1024. 2032632 blocks available
smb: \> exit
	  ```
    - 2.3.3 win 尝试挂载smb目录
	  - 2.3.3.1 进行以下配置
	    ![22-57-g5](/images/22423/6.png)
		- (1) 点击添加映射
		- (2) 输入共享路径
		  ```
		  \\192.168.31.114\gluster-testvol
		  ```
		- (3) 点击完成添加
      - 2.3.3.2 添加后可以打开该共享存储
	    ![22-57-g5](/images/22423/5.png)
  
- 3. 性能调整
  - 3.1 启用元数据缓存
    - 1. 启用元数据缓存
      ```
	  # gluster volume set <volname> group metadata-cache
	  ```	
    - 2. 增加缓存文件个数	  
      ```
	  # gluster volume set <VOLNAME> network.inode-lru-limit <n>
	  ```
	  设置为 50000，如果卷中活动文件数量非常多，可以增加，但是增加这个数字会增加 brick 进程的内存占用
  - 3.2 提高目录列表性能
    随着卷中 brick 数量增加，目录列表会变慢，但是文件/目录编号保持不变。通过启用`readdir`卷选项，
	目录列表的性能独立于卷中brick的数量。因此，卷规模增加便不会降低目录列表的性能。
	只有当卷的分发计数为 2 或更大并且目录的大小很小（< 3000 个条目）时，才能期望性能提高。
	卷（分布计数）越大，性能优势就越大。
	- 1. 启用 `readdir`
	  ```
	  # gluster volume set <VOLNAME> performance.readdir-ahead on
	  ```
	- 2. 启用 `parallel-readdri`
	  ```
	  # gluster volume set <VOLNAME> performance.parallel-readdir on
	  ```
    - 3. 如果卷有超过50个 brick，建议缓存增加到 10Mb以上
	  ```
	  # gluster volume set <VOLNAME> performance.rda-cache-limit <CACHE SIZE>
	  ```
  - 3.3  增强文件/目录创建性能
    在创建/重命名任何文件之前，会发送查找（在 SMB 中为 5-6）以验证文件是否已存在。
	通过在可能的情况下从缓存中提供这些查找，在 SMB 访问中将创建/重命名性能提高了多倍。
	启用缓存
	```
	# gluster volume set <volname> group nl-cache
	```
	启用后，缓存超时失效时间增加到10分钟
  
---

### 检查客户端
- 1. 检查客户端版本
  ```
  # gluster volume status volname clients
  ```
- 对于旧版本的 glusterfs （3.2 版本及以上）
  - 1. 为要检查客户端的卷进行状态转储
    ```
	# gluster volume statedump volname
	```
  - 2. 找到状态转储的目录
    ```
# gluster --print-statedumpdir
    ```
  - 3. 找到客户端信息
    ```
# grep -A4 "identifier=client_ip" statedumpfile
    ```
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  