---
title: ezctl部署高可用K8S集群
date: 2022-04-16 23:20:45
tags:
  - kubernetes
  - ezctl
categories: 
  - deployment
---

# ezctl部署高可用K8S集群
<html><pre><code>基于ezctl构建外置Etcd集群的高可用K8S
ezctl版本: 3.2.0 | k8s版本: 1.23.1
OS版本: Centos 7.9 kernel 5.14</code></pre></html>

![2](/images/22416/2.png)

**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

### 集群节点:
   | 主机名 | IP | 角色 |
   |--- | --- | --- |
   | etcd1.k8s.linuxone.in | 192.168.31.120 | etcd |
   | etcd2.k8s.linuxone.in | 192.168.31.121 | etcd |
   | etcd3.k8s.linuxone.in | 192.168.31.122 | etcd |
   | master1.k8s.linuxone.in | 192.168.31.123 | master |
   | master2.k8s.linuxone.in | 192.168.31.124 | master |
   | master3.k8s.linuxone.in | 192.168.31.125 | master |
   | harbor1.k8s.linuxone.in  | 192.168.31.126 | registry | 
   | harbor2.k8s.linuxone.in  | 192.168.31.127 | registry | 
   | ha1.k8s.linuxone.in  | 192.168.31.128 | ha | 
   | ha2.k8s.linuxone.in  | 192.168.31.129 | ha | 
   | worker1.k8s.linuxone.in  | 192.168.31.130 | worker | 
   | worker2.k8s.linuxone.in  | 192.168.31.131 | worker | 
   | worker3.k8s.linuxone.in  | 192.168.31.132 | worker | 
   | VIP | 192.168.31.119 | cluster_VIP | 
   
### centos 7 k8s集群节点准备
- 1. 集群节点配置以下内核参数
  ~~~
  # cat sysctl.conf
  net.ipv4.ip_forward=1
  vm.max_map_count=262144
  kernel.pid_max=4194303
  fs.file-max=1000000
  net.ipv4.tcp_max_tw_buckets=6000
  net.netfilter.nf_conntrack_max=2097152
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  vm.swappiness=0
  ~~~

- 2. 系统配置以下限制

  ~~~
  # cat /etc/security/limits.conf 
  *             soft    core            unlimited
  *             hard    core            unlimited
  *	      soft    nproc           1000000
  *             hard    nproc           1000000
  *             soft    nofile          1000000
  *             hard    nofile          1000000
  *             soft    memlock         32000
  *             hard    memlock         32000
  *             soft    msgqueue        8192000
  *             hard    msgqueue        8192000
  ~~~

- 3. 准备软件包
  - 移除 kernel 相关工具
    ```
	# yum remove -y kernel-tools kernel-tools-libs
	```
  - 开启`elrepo`，安装新版 `kernel`
    ```
	# yum install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
	# yum --disablerepo=\* --enablerepo=elrepo-kernel install -y kernel-lt.x86_64 kernel-lt-tools.x86_64
	```
  - 配置系统从新版内核启动
    - 列出当前所有可用内核
	  ```
	  # awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg
	  ```
	- 配置第一个为默认启动内核
	  ```
	  # grub2-set-default 0
	  ```
  - 开启`epel`，安装ansible
    ```
	# yum install python3 epel-release
	# yum install ansible
	```
  - 安装 docker 
    参考官方文档: https://docs.docker.com/engine/install/centos/

---

### 安装 HA 节点
- 1. 主节点 `ha1.k8s.linuxone.in`
  - 1.1 配置 `keepalived` 服务
    - 1.1.1 安装 `keepalived`
      ```
	  # yum install keeplived
      ```	
    - 1.1.2 配置文件复制到对应目录
      ```
	  cp /usr/share/doc/keepalived/keepalived.conf.vrrp keepalived.conf
	  ```	
	  
	- 1.1.3 参考以下进行配置
	
	  ```
	  vrrp_instance VI_1 {
		  state MASTER     <<<
		  interface ens192
		  garp_master_delay 10
          smtp_alert
          virtual_router_id 51
          priority 100    <<<
          advert_int 1
          authentication {
            auth_type PASS
            auth_pass 1111
          }  
          virtual_ipaddress {
             192.168.31.119 dev ens192 label ens192:0   <<<
             192.168.31.118 dev ens192 label ens192:1   <<<
             192.168.31.117 dev ens192 label ens192:2   <<<
    }
}
      ```
  - 1.2 配置`haproxy`服务
    - 1.2.1 安装 `haproxy`
	  ```
	  # yum install haproxy
	  ```
	- 1.2.2 配置文件`default`添加以下配置
	  ```
	  listen k8s-api-6443
      bind 192.168.31.119:6443
      mode tcp
      server master1.k8s.linuxone.in 192.168.31.123:6443 check inter 3s fall 3s rise 1
      server master2.k8s.linuxone.in 192.168.31.124:6443 check inter 3s fall 3s rise 1
      server master3.k8s.linuxone.in 192.168.31.125:6443 check inter 3s fall 3s rise 1
	  ```
	  
	- 1.3 添加完成后，启动服务并检查`VIP`是否已经可用
	  - 1.3.1 启动服务
	    ```
		# systemctl enable haproxy --now
		# systemctl enable keepalived --now
		```
	  - 1.3.2 检查`VIP`
	    ![3](/images/22416/3.png)
		
- 2. 从节点`ha2.k8s.linuxone.in` 配置
  - 2.1 配置`keepalived`服务并启用
    ```
	vrrp_instance VI_1 {
    state BACKUP     <<< 模式
    interface ens192
    garp_master_delay 10
    smtp_alert
    virtual_router_id 51
    priority 80     <<< 调低
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
       192.168.31.119 dev ens192 label ens192:0
       192.168.31.118 dev ens192 label ens192:1
       192.168.31.117 dev ens192 label ens192:2
    }
}
    ```	
  
  - 2.2 参考以上配置`haproxy`服务
    - 2.2.1 添加以下内核参数后，启动服务
	  ```
	  net.ipv4.ip_nonlocal_bind = 1    <<< 允许监听在尚未启用的端口
	  ```

---

- 安装 k8s 集群，选则 master1 作为安装节点
  - 1. 配置 master1 到集群中所有节点 ssh 服务免密登录
  - 2. 下载 3.2.0 版本的 `ezdown`
    ```
	# export release=3.2.0
    # wget https://github.com/easzlab/kubeasz/releases/download/${release}/ezdown
	```
	
  - 3. 下载必须的组件
    ```
	# ./ezdown -D
	```
	
  - 4. 下载完成，至 `/etc/kubeasz` 目录创建集群
    ```
	# cd /etc/kubeasz
	# ./ezctl new k8s-01 
	```
 
  - 5. 进入集群文件加进行配置
    - 5.1 配置 hosts 文件
	```
	# cd cluster/k8s-01
	# vim hosts
	[etcd]
    配置etcd节点
    [kube_master]
    配置master节点
    [kube_node]
    配置worker节点
	[ex_lb]
	配置负载均衡节点
	CLUSTER_NETWORK=配置网络插件
	bin_dir=二进制文件所在位置
    ```

    参考文件
<html><pre><code>[etcd]
<font color=red>192.168.31.120</font>
<font color=red>192.168.31.121</font>
<font color=red>192.168.31.122</font>
# master node(s)
[kube_master]
<font color=red>192.168.31.123</font>
<font color=red>192.168.31.124</font>
# work node(s)
[kube_node]
<font color=red>192.168.31.130</font>
<font color=red>192.168.31.131</font>
# [optional] harbor server, a private docker registry
# 'NEW_INSTALL': 'true' to install a harbor server; 'false' to integrate with existed one
[harbor]
#192.168.1.8 NEW_INSTALL=false
# [optional] loadbalance for accessing k8s from outside
[ex_lb]
192.168.1.6 LB_ROLE=backup EX_APISERVER_VIP=<font color=red>192.168.31.119</font> EX_APISERVER_PORT=<font color=red>6443</font>
192.168.1.7 LB_ROLE=master EX_APISERVER_VIP=<font color=red>192.168.31.119</font> EX_APISERVER_PORT=<font color=red>6443</font>
# [optional] ntp server for the cluster
[chrony]
#192.168.1.1
[all:vars]
# --------- Main Variables ---------------
# Secure port for apiservers
SECURE_PORT="6443"
# Cluster container-runtime supported: docker, containerd
CONTAINER_RUNTIME="<font color=red>docker</font>"
# Network plugins supported: calico, flannel, kube-router, cilium, kube-ovn
CLUSTER_NETWORK="<font color=red>calico</font>"
# Service proxy mode of kube-proxy: 'iptables' or 'ipvs'
PROXY_MODE="ipvs"
# K8S Service CIDR, not overlap with node(host) networking
SERVICE_CIDR="<font color=red>10.68.0.0/16</font>"
# Cluster CIDR (Pod CIDR), not overlap with node(host) networking
CLUSTER_CIDR="<font color=red>172.20.0.0/16</font>"
# NodePort Range
NODE_PORT_RANGE="<font color=red>30000-65000</font>"
# Cluster DNS Domain
CLUSTER_DNS_DOMAIN="<font color=red>linuxone.in</font>"
# -------- Additional Variables (don't change the default value right now) ---
# Binaries Directory
bin_dir="<font color=red>/usr/local/bin</font>"
# Deploy Directory (kubeasz workspace)
base_dir="/etc/kubeasz"
# Directory for a specific cluster
cluster_dir="{{ base_dir }}/clusters/<font color=red>k8s-01</font>"
# CA and other components cert/key Directory
ca_dir="/etc/kubernetes/ssl"</code></pre></html>

- 5.2 配置 `config.yml`	  
```
MASTER_CERT_HOSTS: 配置 ha vip
```
![4](/images/22416/4.png)
  
- 6. 检查所有k8s节点 `docker` 默认`cgroup`驱动.
     `kubelet` 默认`cgroup` 为 `systemd` ，所以要对现有驱动进行修改
  - 6.1 确认 docker 默认驱动
    ```
	# docker info | grep Cgroup
    Cgroup Driver: cgroupfs   <<<
	```
  - 6.2 修改 `docker` 默认驱动为 `systemd`
    ```
	# cat /etc/docker/daemon.json 
    {
      "exec-opts": ["native.cgroupdriver=systemd"]   <<<
    }
	```
  - 6.3 重新启动`docker` 服务
    ```
	# systemctl daemon-reload
	# systemctl restart docker
	```
  - 6.4 再次确认

- 7. 开启安装集群
  - 7.1 预配置集群
    ```
	# cd /etc/kubeasz/
	将 lb与 chrony 相关的设置取消，与如下相似
	# vim playbooks/01.prepare.yml
	- hosts:
    - kube_master
    - kube_node
    - etcd
    roles:
    开始预准备
	# ./ezctl setup k8s-01 01
	```
  - 7.2 配置etcd
    ```
	# ./ezctl setup k8s-01 02
	```
    - 7.2.1 检查etcd集群是否健康
	  ```
	  # ETCDCTL_API=3 etcdctl --endpoints=https://192.168.31.120:2379 --cacert=/etc/kubernetes/ssl/ca.pem \
      --cert=/etc/kubernetes/ssl/etcd.pem --key=/etc/kubernetes/ssl/etcd-key.pem endpoint health
      # ETCDCTL_API=3 etcdctl --endpoints=https://192.168.31.121:2379 --cacert=/etc/kubernetes/ssl/ca.pem \
      --cert=/etc/kubernetes/ssl/etcd.pem --key=/etc/kubernetes/ssl/etcd-key.pem endpoint health
      # ETCDCTL_API=3 etcdctl --endpoints=https://192.168.31.122:2379 --cacert=/etc/kubernetes/ssl/ca.pem \
      --cert=/etc/kubernetes/ssl/etcd.pem --key=/etc/kubernetes/ssl/etcd-key.pem endpoint health
	  ```
  - 7.3 安装运行时
    ```
	# ./ezctl setup k8s-01 03
    ```	
 
  - 7.4 安装master节点
    ```
	# ./ezctl setup k8s-01 04
	```
  - 7.5 安装worker节点
    ```
	# ./ezctl setup k8s-01 05
	```
  - 7.6 安装calico插件
    ```
	# ./ezctl setup k8s-01 06
	```
	- 7.6.1 检查 calico 插件是否正常
	  ```
	  # calicoctl node status
Calico process is running.

IPv4 BGP status
+----------------+-------------------+-------+----------+-------------+
|  PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+----------------+-------------------+-------+----------+-------------+
| 192.168.31.124 | node-to-node mesh | up    | 13:12:19 | Established |
| 192.168.31.130 | node-to-node mesh | up    | 13:12:19 | Established |
| 192.168.31.131 | node-to-node mesh | up    | 13:12:18 | Established |
+----------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
	  ```
	  
  - 8. 检查集群网络
    起两个测试容器
	```
	# kubectl run net-test2 --image=centos sleep 36000
	# kubectl run net-test1 --image=centos sleep 36000
	```
	获取pod信息后登录其中一个进行网络测试
	```
	# kubectl get pod -o wide
	# kubectl exec -it net-test1 sh
	```
	可以参考以下图片信息
	![5](/images/22416/5.png)
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  