---
title: cl260-离线部署ceph5
date: 2022-08-12 23:25:36
tags:
  - CL260_ceph5
categories:
  - installation
---

![22-812-g1](/images/22812/1.png)
<html><pre><code> 红帽 Ceph5 离线部署
镜像仓库选用quay 使用 cephadm 进行部署 </code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

---
## 环境介绍
- Ceph5 环境如下
   | 主机名 | IP | 角色 | 
   |--- | --- | --- | 
   | quay.linuxone.in | 192.168.31.250 | registry | 
   | ceph5n0.linuxone.in | 192.168.31.50 | _admin mon mgr prometheus |
   | ceph5n1.linuxone.in | 192.168.31.51 | mon osd mgr |
   | ceph5n2.linuxone.in | 192.168.31.52 | mon osd |
   | ceph5n3.linuxone.in | 192.168.31.53 | mon osd _admin |
   所有主机都有 sdb sdc sdd 三块硬盘用于创建 osd
   
---

## Quay 构建离线 registry
- 1. Quay 的搭建过程请参考之前文档
- 2. 将所需的 image 拉到本地，并更改tag 上传至 quay
  - 2.1 将镜像拉到本地
```
podman pull registry.redhat.io/rhceph/keepalived-rhel8:latest
podman pull registry.redhat.io/rhceph/rhceph-haproxy-rhel8:latest
podman pull registry.redhat.io/rhceph/rhceph-5-rhel8:latest
podman pull registry.redhat.io/rhceph/rhceph-5-dashboard-rhel8:latest
podman pull registry.redhat.io/openshift4/ose-prometheus-node-exporter:v4.6
podman pull registry.redhat.io/openshift4/ose-prometheus-alertmanager:v4.6
podman pull registry.redhat.io/rhceph/snmp-notifier-rhel8:latest
podman pull registry.redhat.io/openshift4/ose-prometheus:v4.6
```
  - 2.2 为镜像更改 tag
  ```
podman tag registry.redhat.io/rhceph/keepalived-rhel8:latest quay.linuxone.in/rhceph/keepalived-rhel8:latest
podman tag registry.redhat.io/rhceph/rhceph-haproxy-rhel8:latest quay.linuxone.in/rhceph/rhceph-haproxy-rhel8:latest
podman tag registry.redhat.io/rhceph/rhceph-5-rhel8:latest quay.linuxone.in/rhceph/rhceph-5-rhel8:latest
podman tag registry.redhat.io/rhceph/rhceph-5-dashboard-rhel8:latest quay.linuxone.in/rhceph/rhceph-5-dashboard-rhel8:latest
podman tag registry.redhat.io/openshift4/ose-prometheus-node-exporter:v4.6 quay.linuxone.in/openshift4/ose-prometheus-node-exporter:v4.6
podman tag registry.redhat.io/openshift4/ose-prometheus-alertmanager:v4.6 quay.linuxone.in/openshift4/ose-prometheus-alertmanager:v4.6
podman tag registry.redhat.io/rhceph/snmp-notifier-rhel8:latest quay.linuxone.in/rhceph/snmp-notifier-rhel8:latest
podman tag registry.redhat.io/openshift4/ose-prometheus:v4.6 quay.linuxone.in/openshift4/ose-prometheus:v4.6
  ```

  - 2.3 将镜像移除签名后上传至 quay
  ```
podman push quay.linuxone.in/rhceph/keepalived-rhel8:latest --remove-signatures
podman push quay.linuxone.in/rhceph/rhceph-haproxy-rhel8:latest --remove-signatures
podman push quay.linuxone.in/rhceph/rhceph-5-rhel8:latest --remove-signatures
podman push quay.linuxone.in/rhceph/rhceph-5-dashboard-rhel8:latest --remove-signatures
podman push quay.linuxone.in/openshift4/ose-prometheus-node-exporter:v4.6 --remove-signatures
podman push quay.linuxone.in/openshift4/ose-prometheus-alertmanager:v4.6 --remove-signatures
podman push quay.linuxone.in/rhceph/snmp-notifier-rhel8:latest --remove-signatures
podman push quay.linuxone.in/openshift4/ose-prometheus:v4.6 --remove-signatures
  ```

---

## 预先配置各个ceph节点
- 1. 预先同步以下 repo 并提供给内网 ceph 节点
```
rhceph-5-tools-for-rhel-8-x86_64-rpms
ansible-2.9-for-rhel-8-x86_64-rpms
rhel-8-for-x86_64-baseos-rpms
rhel-8-for-x86_64-appstream-rpms
```

- 2. 安装 cephadm-ansible
```
dnf install cephadm-ansible
```

- 3. 以`ceph5n0` 为 `bootstrap` 节点进行点火安装 ceph。配置 ssh 密钥
```
# ssh-keygen
# ssh-copy-id root@ceph5n1.linuxone.in
# ssh-copy-id root@ceph5n0.linuxone.in
# ssh-copy-id root@ceph5n2.linuxone.in
# ssh-copy-id root@ceph5n3.linuxone.in
```

- 4. 运行预检查 playbook
  - 4.1 配置 inventory
  ```
# cd /usr/share/cephadm-ansible/
# cat hosts
ceph5n1.linuxone.in
ceph5n2.linuxone.in
ceph5n3.linuxone.in
ceph5n0.linuxone.in
  ```
  
  - 4.2 运行预检查 playbook
  ```
# cd /usr/share/cephadm-ansible/
# ansible-playbook  -i hosts cephadm-preflight.yml  --extra-vars "ceph_origin="
  ```

- 5. 配置 `ceph` 使用本地镜像
```
# ceph config set mgr mgr/cephadm/container_image_prometheus quay.linuxone.in/openshift4/ose-prometheus
# ceph config set mgr mgr/cephadm/container_image_grafana quay.linuxone.in/rhceph/rhceph-4-dashboard-rhel8
# ceph config set mgr mgr/cephadm/container_image_alertmanager quay.linuxone.in/openshift4/ose-prometheus-alertmanager
# ceph config set mgr mgr/cephadm/container_image_node_exporter quay.linuxone.in/openshift4/ose-prometheus-node-exporter
```

- 6. 配置 ceph 使用本地 quay
  - 6.1 将 quay 的证书拷贝到各个 ceph 节点的指定目录(quay证书为创建时自签名ca)
  ```
# mkdir /etc/containers/certs.d/quay.linuxone.in/
从quay节点复制证书：
# scp cert.ca.crt root@192.168.31.50:/etc/containers/certs.d/quay.linuxone.in/
  ```
  - 6.2 创建 quay 用户登录 `json` 文件，我使用的用户是 `quayadmin`
    - 6.2.1 登录 quay 页面 点击 `account Settings`
	![22-812-g2](/images/22812/2.png)
	- 6.2.2 点击 `Generate Encrypted Password`
    ![22-812-g3](/images/22812/3.png)	
    - 6.2.3 点击 `Docker Configruation` 下载 `quayadmin-auth.json`
	![22-812-g4](/images/22812/4.png)
    - 6.2.4 将 `quayadmin-auth.json` 文件上传至ceph各节点对应目录
	```
# mv /etc/ceph/podman-auth.json /etc/ceph/podman-auth.json.bak
# mv quayadmin-auth.json /etc/ceph/podman-auth.json
	```

---

## 在`ceph5n0`安装 bootstrap 节点
- 以下命令点火安装，配置管理端口密码为 `redhat`，并且首次登录 console 不需要重置密码
```
# cephadm --image quay.linuxone.in/rhceph/rhceph-5-rhel8:latest bootstrap --mon-ip=192.168.31.50 \
--initial-dashboard-password=redhat --dashboard-password-noupdate \
--allow-fqdn-hostname --registry-url=quay.linuxone.in --registry-username=quayadmin --registry-password=LINyuxiang111096! \
--yes-i-know --allow-overwrite
```
  执行以下命令后，会得到以下输出:
  ![22-812-g5](/images/22812/5.png)
  同时，使用红色框中命令登录管理 ceph 集群
  ![22-812-g6](/images/22812/6.png) 

---

## 在 bootstrap 安装 ceph 集群
- 1. 将其他节点添加到 ceph 集群
  - 1.1 创建 ceph 密钥，并复制到集群各个节点
  ```
# /usr/sbin/cephadm shell --fsid 9e26cb76-1a3b-11ed-8f8f-0050562d60d7 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring
ceph: # ceph cephadm get-pub-key > ~/ceph.pub
ceph: # ssh-copy-id -f -i ~/ceph.pub root@192.168.31.51
ceph: # ssh-copy-id -f -i ~/ceph.pub root@192.168.31.50
ceph: # ssh-copy-id -f -i ~/ceph.pub root@192.168.31.52
ceph: # ssh-copy-id -f -i ~/ceph.pub root@192.168.31.53
  ```
  - 1.2 将节点添加到 ceph 
  ```
ceph: # ceph orch host add ceph5n1 192.168.31.51
ceph: # ceph orch host add ceph5n2 192.168.31.52
ceph: # ceph orch host add ceph5n3 192.168.31.53
  ```
  - 1.3 检查节点状态信息
  ```
ceph: # ceph orch host ls
  ```
  - 1.4 删除集群中节点的顺序如下
    - 1.4.1 排空该节点所有守护进程
	```
ceph: # ceph orch host drain ceph5n1
	```
	- 1.4.2 检查节点 osd 状态，确认 osd 停用
	```
ceph: # ceph orch osd rm status
	```
	- 1.4.3 检查所有守护进程是否已经移除
	```
ceph: # ceph orch ps ceph5n1
	```
	- 1.4.4 确认守护进程已经移除，删除主机
	```
ceph: # ceph orch rm ceph5n1
	```
- 2. 部署 mon 守护进程
  - 2.1 为所有主机创建 mon 标签
  ```
ceph: # ceph orch host label add ceph5n1 mon
ceph: # ceph orch host label add ceph5n2 mon
ceph: # ceph orch host label add ceph5n3 mon
ceph: # ceph orch host label add ceph5n0 mon
  ```
  - 2.2 将所有带有 mon 标签的主机部署 mon 守护进程
  ```
ceph: # ceph orch apply mon --placement="label=mon"
  ```

- 3. 为对应主机部署 mgr 守护进程
  - 3.1 为所有主机创建 mgr 标签
  ```
ceph: # ceph orch host label add ceph5n1 mgr
ceph: # ceph orch host label add ceph5n0 mgr
  ```
  - 3.2 将所有带有 mon 标签的主机部署 mon 守护进程
  ```
ceph: # ceph orch apply mgr --placement="label=mgr"
  ```
- 4. 部署 osd 守护进程
```
ceph: # ceph orch daemon add osd ceph5n1:/dev/sdb
ceph: # ceph orch daemon add osd ceph5n1:/dev/sdc
ceph: # ceph orch daemon add osd ceph5n2:/dev/sdb
ceph: # ceph orch daemon add osd ceph5n2:/dev/sdc
ceph: # ceph orch daemon add osd ceph5n3:/dev/sdc
ceph: # ceph orch daemon add osd ceph5n3:/dev/sdb
ceph: # ceph orch daemon add osd ceph5n3:/dev/sdd
ceph: # ceph orch daemon add osd ceph5n2:/dev/sdd
ceph: # ceph orch daemon add osd ceph5n1:/dev/sdd
```

- 5. 创建新的 `_admin` 管理节点
  - 5.1 为 ceph5n3 节点添加标签
  ```
ceph: # ceph orch host label add ceph5n3 _admin
  ```
  - 5.2 移除 `_admin` 标签
  ```
ceph: # ceph orch host label rm ceph5n3 _admin
  ```

- 6. 部署完成检查 ceph 集群状态
  - 6.1 通过命令行检查
    ```
ceph: # ceph -s
ceph: # ceph health detail
	```
    ![22-812-g7](/images/22812/7.png) 
  - 6.2 通过网页查看
    ![22-812-g8](/images/22812/8.png) 
   
---

## 清除 ceph 集群
- 执行以下命令进行移除
```
# ansible-playbook -i hosts cephadm-purge-cluster.yml -e fsid=a6ca415a-cde7-11eb-a41a-002590fc2544 -vvv
```
   
   
   
   