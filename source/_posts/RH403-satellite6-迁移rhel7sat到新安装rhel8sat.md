---
title: RH403-satellite6-版本迁移_三
date: 2022-07-12 16:12:44
tags:
  - RH403_Satellite6
categories:
  - installation
  - migration
---

![22-619-g1](/images/22619/1.png)
<html><pre><code> 红帽 Satellite6.11
将 satellite 6.11 RHEL7 迁移至新安装 satellite 6.11 RHEL8 </code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

## 备份当前 satellite 6.11 + RHEL7 
- 为 satellie 和 capsule 分别创建完整离线备份
  ```
On satellite server:
# satellite-maintain backup offline /var/satellite-backup
On capsule server:
# satellite-maintain backup offline /var/capsule-backup
  ```
 
## 新安装 RHEL8 + satellite 6.11 与 capsule 6.11
- satellite 安装
  - 防火墙配置
    ```
# firewall-cmd --permanent \
--add-port="53/udp" --add-port="53/tcp" \
--add-port="67/udp" \
--add-port="69/udp" \
--add-port="80/tcp" --add-port="443/tcp" \
--add-port="5647/tcp" \
--add-port="8000/tcp" --add-port="9090/tcp" \
--add-port="8140/tcp"
	```
  - 开启以下 repo
    ```
# subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms \
--enable=rhel-8-for-x86_64-appstream-rpms \
--enable=satellite-6.11-for-rhel-8-x86_64-rpms \
--enable=satellite-maintenance-6.11-for-rhel-8-x86_64-rpms
	```
  - 启用模块
    ```
# dnf module enable satellite:el8
	```
  - 安装 satellite 软件包
    ```
# dnf install satellite
	```
  - 部署 satellite
    ```
# satellite-installer --scenario satellite \
--foreman-initial-organization "ORG" \
--foreman-initial-location "LOC" \
--foreman-initial-admin-username admin \
--foreman-initial-admin-password redhat
	```
	
- 安装 capsule
  - 防火墙配置
    ```
# firewall-cmd --permanent \
--add-port="53/udp" --add-port="53/tcp" \
--add-port="67/udp" \
--add-port="69/udp" \
--add-port="80/tcp" --add-port="443/tcp" \
--add-port="5647/tcp" \
--add-port="8140/tcp" \
--add-port="8443/tcp" \
--add-port="8000/tcp" --add-port="9090/tcp"
	```
  - satellite 开启以下 repo 并同步发布到 capsule 所在 cv
    ```
# subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms \
--enable=rhel-8-for-x86_64-appstream-rpms \
--enable=satellite-capsule-6.11-for-rhel-8-x86_64-rpms \
--enable=satellite-maintenance-6.11-for-rhel-8-x86_64-rpms
	```
  - 开启 capsule 所需要模块
    ```
# dnf module enable satellite-capsule:el8
	```
  - 安装 capsule 软件包
    ```
# yum install satellite-capsule
	```
  - 为 capsule 创建 ssl 证书
    ```
satellite 端执行创建证书文件夹
# mkdir /root/capsule_cert
创建证书
# capsule-certs-generate \
--foreman-proxy-fqdn capsule.linuxone.in \
--certs-tar /root/capsule_cert/capsule_certs.tar
将证书复制到 capsule 对应目录
# scp /root/capsule_cert/capsule_certs.tar root@capsule.linuxone.in:/root/capsule_certs.tar
	```
    `capsule-certs-generate` 会有如下输出，复制并在 capsule 端执行
	```
  satellite-installer \
                    --scenario capsule \
                    --certs-tar-file                              "/root/capsule_certs.tar"\
                    --foreman-proxy-register-in-foreman           "true"\
                    --foreman-proxy-foreman-base-url              "https://satellite.linuxone.in"\
                    --foreman-proxy-trusted-hosts                 "satellite.linuxone.in"\
                    --foreman-proxy-trusted-hosts                 "capsule.linuxone.in"\
                    --foreman-proxy-oauth-consumer-key            "KzSsuAt7J3D7xXKFtLn9pWwGqhSrDAnP"\
                    --foreman-proxy-oauth-consumer-secret         "pfxTvwMBbiU7XrhBw5Di98moNYNQ6bay"
	```
  - 为 capsule 附加 lifecycle 和 org location
	
## 从备份回复 satellite 和 capsule
- 将备份上传到新安装satellite /var/tmp 目录
```
scp -r satellite-backup/ root@satellite.linuxone.in:/var/tmp/
```
- 从备份中恢复 satellite
```
# satellite-maintain restore /var/tmp/satellite-backup/satellite-backup-2022-07-12-16-48-09/
```
  注意
  ```
WARNING: This script will drop and restore your database.
Your existing installation will be replaced with the backup database.
Once this operation is complete there is no going back.
Do you want to proceed?, [y(yes), q(quit)] y
  ```
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	