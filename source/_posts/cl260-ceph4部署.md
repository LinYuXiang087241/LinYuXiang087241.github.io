---
title: cl260-ceph4部署
date: 2022-11-10 10:11:47
tags:
  - CL260_ceph
categories:
  - installation
---


![22-812-g1](/images/22812/1.png)
<html><pre><code> 红帽 Ceph4 部署
使用 cockpit / ceph-ansible 方式进行部署 </code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

---

## 环境准备
- Ceph4 环境如下
   | 主机名 | IP | 角色 | 
   |--- | --- | --- | 
   | cepha.linuxone.in | 192.168.31.50 | _admin mon mgr |
   | cephb.linuxone.in | 192.168.31.51 | mon osd mgr |
   | cephc.linuxone.in | 192.168.31.52 | mon osd |
   所有主机都有 sdb sdc sdd 三块硬盘用于创建 osd
   
- Ceph4 主机所需的 repo 如下
  - RHEL 7:
    ```
rhel-7-server-rhceph-4-tools-rpms
rhel-7-server-ansible-2.9-rpms
rhel-7-server-rhceph-4-mon-rpms
rhel-7-server-rhceph-4-osd-rpms
	```
  - RHEL 8:
    ```
rhceph-4-tools-for-rhel-8-x86_64-rpms
ansible-2.9-for-rhel-8-x86_64-rpms
rhceph-4-mon-for-rhel-8-x86_64-rpms
rhceph-4-osd-for-rhel-8-x86_64-rpms
	```

- Ceph4 使用容器化部署，需要下载的容器镜像为：
  ```
registry.redhat.io/rhceph/rhceph-4-rhel8:latest
registry.redhat.io/openshift4/ose-prometheus-node-exporter:v4.6
registry.redhat.io/rhceph/rhceph-4-dashboard-rhel8:latest
registry.redhat.io/openshift4/ose-prometheus:v4.6
registry.redhat.io/openshift4/ose-prometheus-alertmanager:v4.6
  ```
  
- 防火墙配置：
  ```
  MON 配置
# firewall-cmd --zone=public --add-port=3300/tcp
# firewall-cmd --zone=public --add-port=3300/tcp --permanent
# firewall-cmd --zone=public --add-port=6789/tcp
# firewall-cmd --zone=public --add-port=6789/tcp --permanent
# firewall-cmd --permanent --add-service=ceph-mon
# firewall-cmd --add-service=ceph-mon

  OSD 配置
# firewall-cmd --zone=public --add-port=6800-7300/tcp
# firewall-cmd --zone=public --add-port=6800-7300/tcp --permanent
# firewall-cmd --permanent --add-service=ceph
# firewall-cmd --add-service=ceph
  
  RGW 配置
# firewall-cmd --zone=public --add-port=8080/tcp
# firewall-cmd --zone=public --add-port=8080/tcp --permanent
# firewall-cmd --zone=public --add-port=80/tcp
# firewall-cmd --zone=public --add-port=80/tcp --permanent
# firewall-cmd --zone=public --add-port=443/tcp
# firewall-cmd --zone=public --add-port=443/tcp --permanent
# firewall-cmd --zone=public --add-port=9090/tcp
# firewall-cmd --zone=public --add-port=9090/tcp --permanent
  ```
---

## 使用 ceph-cockpit 安装集群
- 1. 安装 cockpit
  ```
# dnf install cockpit -y
# systemctl enable --now cockpit.socket
# dnf install cockpit-ceph-installer
  ```
- 2. 运行 ansible 以下容器
  ```
# ansible-runner-service.sh -s
最后的输出如下：
The Ansible API container (runner-service) is available and responding to requests

Login to the cockpit UI at https://cepha.linuxone.in:9090/cockpit-ceph-installer to start the install
Linking the runner service inventory to ceph-ansible hosts
- ansible hosts linked to runner-service inventory
  ```

- 3.  将 Cockpit Ceph Installer SSH 密钥复制到集群中的所有节点
  ```
ssh-copy-id -f -i /usr/share/ansible-runner-service/env/ssh_key.pub root@cepha
  ```
- 4. 完成 Cockpit Ceph Installer 的 Environment 页面
  ![22-1110-g1](/images/221110/1.png)
  - 4.1  参考图中配置，基于 RPM 进行安装
  ![22-1110-g2](/images/221110/2.png)
  
- 5. 完成 Cockpit Ceph Installer 的 Host 页面
  - 5.1 点击 `add host` 添加主机
  ![22-1110-g3](/images/221110/3.png)
  - 5.2 添加主机
  ![22-1110-g4](/images/221110/4.png)
  - 5.3 验证主机
  ![22-1110-g5](/images/221110/5.png)
  - 5.4 配置网络
  ![22-1110-g6](/images/221110/6.png)
  - 5.5 最后检查，点击 `save` 开始部署
  ![22-1110-g7](/images/221110/7.png)
  
- 6. 部署完成后，点击 complete 出现集群信息
  ![22-1110-g8](/images/221110/8.png)
  部署完成后会是 warn 状态，禁用不安全模式：
  ```
# ceph config set mon auth_allow_insecure_global_id_reclaim false
  ```

---
## 使用 ceph-ansible 容器化部署 ceph 集群
- 1. 安装 ceph-ansible 软件包
  ```
# dnf install ceph-ansible 
  ```
- 2. 修改 group_vars/all.yml 文件中的内容
  ```
# cat all.yml
---
dummy:
ntp_service_enabled: true
ceph_origin: repository
ceph_repository: rhcs
ceph_rhcs_version: 4
ceph_iscsi_config_dev: false
rbd_cache: "true"
rbd_cache_writethrough_until_flush: "false"
monitor_interface: ens33
public_network: 192.168.31.0/24
cluster_network: 192.168.31.0/24
ceph_conf_overrides:
  global:
    mon_clock_drift_allowed: 0.5
    mon_pg_warn_min_per_osd: 0
ceph_docker_image: "rhceph/rhceph-4-rhel8"
ceph_docker_image_tag: "latest"
ceph_docker_registry: "registry.redhat.io"
ceph_docker_registry_auth: true
ceph_docker_registry_username: <-- 
ceph_docker_registry_password: <--
containerized_deployment: True
dashboard_admin_user: admin
dashboard_admin_password: redhat
node_exporter_container_image: registry.redhat.io/openshift4/ose-prometheus-node-exporter:v4.6
grafana_admin_user: admin
grafana_admin_password: redhat
grafana_container_image: registry.redhat.io/rhceph/rhceph-4-dashboard-rhel8:4
prometheus_container_image: registry.redhat.io/openshift4/ose-prometheus:v4.6
alertmanager_container_image: registry.redhat.io/openshift4/ose-prometheus-alertmanager:v4.6
  ```

- 3. 修改 group_vars/osd.yml 中的内容
  ```
devices:
  - /dev/sdb
  - /dev/sdc
  - /dev/sdd
  ```

- 4. 编写 hosts 主机清单
  ```
# cat hosts 
[grafana-server]
cepha.linuxone.in

[mons]
cepha.linuxone.in
cephb.linuxone.in
cephc.linuxone.in

[mgrs]
cepha.linuxone.in
cephb.linuxone.in
cephc.linuxone.in

[osds]
cepha.linuxone.in
cephb.linuxone.in
cephc.linuxone.in
  ```

- 5. 运行安装 playbook
  ```
# cd /usr/share/ceph-ansible
# ansible-playbook site-container.yml -i hosts
  ```
- 6. 开启 dashboard 的 ssl 
  ```
# ceph dashboard create-self-signed-cert
# ceph config set mgr mgr/dashboard/ssl true
# ceph mgr module disable dashboard
# ceph mgr module enable dashboard
  ```

- 7. 在 cephb 和 cephc 节点上部署 RGW 网关
  - 7.1 修改 group_vars/all.yml 文件：
  ```
radosgw_interface: ens33
client.rgw.cepha.rgw0:
rgw_dns_name: cepha
  ```
  - 7.2 编辑 group_vars/rgws.yml 文件
```
# cp rgws.yml.sample rgws.yml
# vim rgws.yml
copy_admin_key: true
```

  - 7.3 hosts 文件添加 rgw 节点 
    ```
[rgws]
cephb.linuxone.in
cephc.linuxone.in
    ```
  
  - 7.4 执行部署
  ```
# ansible-playbook site-container.yml -i hosts --limit rgws
  ```
  
- 8. 部署 iscsi 网关
  - 8.1 编辑 all.yml ，添加以下调优
```
osd_client_watch_timeout : 15
osd_heartbeat_grace : 20
osd_heartbeat_interval : 5
```


  - 8.2 hosts中添加 iscsigws 
    ```
[iscsigws]
cephb.linuxone.in
cephc.linuxone.in
	```
  - 8.3 准备 iscsigws.yml 文件
    ```
# cp iscsigws.yml.sample iscsigws.yml
	```
  - 8.4 执行安装
    ```
# $ ansible-playbook site-container.yml --limit iscsigws  -i hosts
	```
  - 8.5 起用 iscsi 网关
    ```
# systemctl enable rbd-target-api --now
# gwcli ls
	```
	
- 9. 配置 cephfs
  - 9.1 hosts 中添加 mdss 
    ```
[mdss]
cepha.linuxone.in
	```
  - 9.2 执行安装
    ```
# ansible-playbook -i hosts site-container.yml --limit mdss
	```
---
## ceph 4 常用操作
- 1. 纠删码配置文件的操作
  ```
列出现有的配置文件
# ceph osd erasure-code-profile ls
查看现有配置文件的配置信息
# ceph osd erasure-code-profile get
移除现有的配置文件
# ceph osd erasure-code-profile rm        
  ```

- 2. 操作和管理池
  ```
重命名池
# ceph osd pool rename
删除池  
# ceph osd pool delete
防止删除特定的池
# ceph osd pool set pool_name nodelete true
查看所以后的池以及池配置
# ceph osd lspools 
# ceph osd pool ls detail
设置配额
# ceph osd pool set-quota
  ```

- 3. 示例 创建一个 xxxpool 具有 64 个 PG，副本数为 3 ， rbd 类型， pg_auto_scaple 模式开启
  ```
# ceph osd pool create xxxpool 64 64 replicated
# ceph osd pool set xxxpool size 3
# ceph osd pool application enable xxxpool rbd
# ceph osd pool set xxxpool pg_autoscale_mode on
  ```

- 4. ceph 获取 crushdump 并解译
  ```
# ceph osd getcrushmap -o ./map.bin
# crushtool -d ./map.bin -o ./map.txt
  ```
    显示 crush 可调项
  ```
# ceph osd crush show-tunables
  ```
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  