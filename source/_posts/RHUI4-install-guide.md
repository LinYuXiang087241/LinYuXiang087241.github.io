---
title: RHUI4-install-guide
date: 2022-04-18 15:37:28
tags:
  - RHUI4
categories:
  - installation
---

<h2>RHUI4 安装 </h2>

<html><pre><code>基于红帽 RHUI 安装手册安装</code></pre></html>

![22-418-1](/images/22418/1.png)

**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

## 准备主机
   | 主机名 | IP | 角色 |
   |--- | --- | --- |
   | rhua.linuxone.in | 192.168.31.107 | rhua |
   | cds1.linuxone.in | 192.168.31.106 | cds |
   | cds2.linuxone.in | 192.168.31.104 | cds |
   | haproxy.linuxone.in | 192.168.31.105 | haproxy |
   | quay.linuxone.in | 192.168.31.250 | nfs | 
 
## 前提
- 所有主机防火墙必须正确配置，必须正确配置，必须正确配置。

## 配置 RHUA 主机
- 1. 附加正确订阅
- 2. 开启以下软件包频道
  ```
  # subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rhui-rpms --enable=rhel-8-for-x86_64-appstream-rhui-rpms --enable=ansible-2-for-rhel-8-x86_64-rhui-rpms --enable=rhui-4-for-rhel-8-x86_64-rpms
  ```
- 3. 安装ansible 和 rhui-installer
  ```
  # dnf install ansible rhui-installer
  ```
- 4. 配置 RHUA 主机到其他所有节点（包括 RHUA）的 ssh 密钥登录
  ```
  # ssh-keygen -t ecdsa   <<< 一定要使用加密的 ssh key
  # ssh-copy-id root@rhua.linuxone.in  <<< cds 节点与 haproxy 节点也需要
  ```

## 配置 `nfs` 存储主机
- 使用`nfs`作为共享存储
  - 1.1 创建文件夹
    ```
    # mkdir /exports
	```	
  - 1.2 添加`exports`
    ```
# cat /etc/exports
/exports rhua.linuxone.in(rw,no_root_squash) cds1.linuxone.in(rw,no_root_squash) cds2.linuxone.in(rw,no_root_squash)
	```
  - 1.3 防火墙配置放行
    ```
	# firewall-cmd --permanent --add-service=mountd
	# firewall-cmd --permanent --add-service=nfs
	# firewall-cmd --permanent --add-service=rpc-bind
	# firewall-cmd --reload
	```
  - 1.4 启动服务
    ```
	# systemctl enable nfs-server --now
	# systemctl enable rpcbind --now
	```

## 配置 `haproxy` 主机
- 附加正确订阅

## 配置 `cds` 主机
- 附加正确订阅

## 开始安装
- 1. 命令行执行
  ```
  # rhui-installer --remote-fs-server quay.linuxone.in:/exports --rhua-hostname rhua.linuxone.in --cds-lb-hostname haproxy.linuxone.in
  ```

- 2. 安装完成后，密码文件在`/etc/rhui/rhui-subscription-sync.conf`
- 3. 使用以下命令登录
  ```
  # rhui-manager
  ```
  ![22-418-2](/images/22418/2.png)
- 4. 修改admin 密码
  - u
  ![22-418-3](/images/22418/3.png)
  - p ，输入新的密码即可重置
  ![22-418-4](/images/22418/4.png)
- 5. 添加 `cds` 节点
  - c > a
  ![22-418-5](/images/22418/5.png)
    - 1 处填写 cds hostname url ，请确保该 url 可以解析
    - 2 处填写 ssh 到该节点的用户，确保用户可以免密`sudo`
    - 3 处填写 rhua 到该节点ssh 免密登录的加密私钥所在位置	
    配置完成，成功结果如下：
	![22-418-6](/images/22418/6.png)
  - 5.1 检查当前节点是否已经加入rhui集群
    - 5.1.1 通过 df -h 命令检查nfs共享存储是否已经挂载
	  ![22-418-9](/images/22418/9.png)
	- 5.1.2 检查`rhui-tools.conf`是否存在
	  ```
# ls /etc/rhui/
rhui-tools.conf
      ```

- 6. 添加`haproxy`节点
  - h > a 步骤参考 `5`
    配置完成，结果如下：
	![22-418-7](/images/22418/7.png)
  - 登录`haproxy` 节点检查配置是否正确，注意，此配置为 rhui 自动化配置，不需要自己手动配置
	![22-418-8](/images/22418/8.png)
	
关于 RHUI 4 内容管理将会后续发布博客

NOTE: 请注意，此博客仅用作经验总结，不构成任何意见
  
参考文档为红帽官方管理文档:[**<font color=red>INSTALLING RED HAT UPDATE INFRASTRUCTURE</font>**](https://access.redhat.com/documentation/en-us/red_hat_update_infrastructure/4/html-single/installing_red_hat_update_infrastructure/index)