---
title: RH403_satellite6_部署升级_一
date: 2022-06-18 21:04:42
tags:
  - RH403_Satellite6
categories:
  - installation
  - upgrade
---

![22-619-g1](/images/22619/1.png)
<html><pre><code> 红帽 Satellite6.8 
部署、更新、升级satellite / capsule 6.8 RHEL7 -> 6.9 -> 6.10 -> 6.11 RHEL 8</code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->
---

##  环境介绍
- Satellite 环境如下
   | 主机名 | IP | lifecycle env | 
   |--- | --- | --- | 
   | satellite.linuxone.in | 192.168.31.70 | default | 
   | capsule.linuxone.in | 192.168.31.71 | capsule |
   | capsule2.linuxone.in | 192.168.31.72 | kickstart |
   | rhel7u6.linuxone.in | 192.168.31.76 | capsule |
   
## 部署 Satellite 6.8
- 配置防火墙
```
# firewall-cmd \
--add-port="80/tcp" --add-port="443/tcp" \
--add-port="5647/tcp" --add-port="8000/tcp" \
--add-port="8140/tcp" --add-port="9090/tcp" \
--add-port="53/udp" --add-port="53/tcp" \
--add-port="67/udp" --add-port="69/udp" \
--add-port="5000/tcp"
# firewall-cmd --runtime-to-permanent
```

- 确认 hostname 可以 ping 通，且可以解析
```
# ping satellite.linuxone.in
# nslookup satellite.linuxone.in
```

- 注册后开启以下 repo
```
# subscription-manager repos --enable=rhel-7-server-rpms \
--enable=rhel-7-server-satellite-6.8-rpms \
--enable=rhel-7-server-satellite-maintenance-6-rpms \
--enable=rhel-server-rhscl-7-rpms \
--enable=rhel-7-server-ansible-2.9-rpms

部署 satellite 6.9
--enable=rhel-7-server-satellite-6.9-rpms

部署 satellite 6.10 
--enable=rhel-7-server-satellite-6.10-rpms 
```

- 如果网络速度较慢，可以开启红帽 china.cdn
```
subscription-manager config --rhsm.baseurl=https://china.cdn.redhat.com
subscription-manager refresh
```

- 安装 Satellite 6.8 
```
 yum install satellite
```

- 部署 satellite 6.8
```
# satellite-installer --scenario satellite \
--foreman-initial-organization "org" \
--foreman-initial-location "location" \
--foreman-initial-admin-username admin \
--foreman-initial-admin-password redhat
```

- 创建订阅分配并导出 manifest 文件
```
Red Hat Customer Portal > subscriptions > subscription allocation > create / add subscriptions / export manifest
```

- Satellite 导入文件
```
web gui > content > subscriptions
注意，请关闭 sca 功能 
manage manifest > 点击按钮关闭 sca
```

---

## 部署 capsule 6.8
- satellite 同步以下 repo
```
--enable=rhel-7-server-rpms \
--enable=rhel-7-server-satellite-capsule-6.8-rpms \
--enable=rhel-7-server-satellite-maintenance-6-rpms \
--enable=rhel-7-server-satellite-tools-6.8-rpms \
--enable=rhel-server-rhscl-7-rpms \
--enable=rhel-7-server-ansible-2.9-rpms
```

- 将部署 capsule 主机注册到 satellite（lifecycle env / cv / activation key 创建方式不进行介绍）
```
capsule 主机：
# curl --insecure --output katello-ca-consumer-latest.noarch.rpm https://satellite.linuxone.in/pub/katello-ca-consumer-latest.noarch.rpm
# yum localinstall katello-ca-consumer-latest.noarch.rpm
# subscription-manager register --org="org" --activationkey="capsule-deploy"
```

- capsule 主机防火墙配置：
```
# firewall-cmd --add-port="53/udp" --add-port="53/tcp" \
--add-port="67/udp" --add-port="69/udp" \
--add-port="80/tcp" --add-port="443/tcp" \
--add-port="5000/tcp" --add-port="5647/tcp" \
--add-port="8000/tcp" --add-port="8140/tcp" \
--add-port="8443/tcp" --add-port="9090/tcp"

# firewall-cmd --runtime-to-permanent
```

- capsule 主机开启以下 repo
```
# subscription-manager repos --enable=rhel-7-server-rpms \
--enable=rhel-7-server-satellite-capsule-6.8-rpms \
--enable=rhel-7-server-satellite-maintenance-6-rpms \
--enable=rhel-7-server-satellite-tools-6.8-rpms \
--enable=rhel-server-rhscl-7-rpms \
--enable=rhel-7-server-ansible-2.9-rpms

6.9 capsule开启
--enable=rhel-7-server-satellite-capsule-6.9-rpms \
--enable=rhel-7-server-satellite-tools-6.9-rpms \

6.10 capsule开启
--enable=rhel-7-server-satellite-capsule-6.10-rpms \
--enable=rhel-7-server-satellite-tools-6.10-rpms \
```

- satellite 端为 capsule 创建证书并将证书拷贝至 capsule：
```
# mkdir /root/capsule_cert
# capsule-certs-generate \
--foreman-proxy-fqdn capsule.linuxone.in \
--certs-tar /root/capsule_cert/capsule_certs.tar

会有类似输出：
  satellite-installer \
                    --scenario capsule \
                    --certs-tar-file                              "/root/capsule_certs.tar"\
                    --foreman-proxy-content-parent-fqdn           "satellite.linuxone.in"\
                    --foreman-proxy-register-in-foreman           "true"\
                    --foreman-proxy-foreman-base-url              "https://satellite.linuxone.in"\
                    --foreman-proxy-trusted-hosts                 "satellite.linuxone.in"\
                    --foreman-proxy-trusted-hosts                 "capsule.linuxone.in"\
                    --foreman-proxy-oauth-consumer-key            "aTupLLeC4bihuUiVUVaEKyLybnXzWUrd"\
                    --foreman-proxy-oauth-consumer-secret         "x22RfhnZuyQAsXZ3bXoRhaPXvNAcsJgk"\
                    --puppet-server-foreman-url                   "https://satellite.linuxone.in"
# scp /root/capsule_cert/capsule_certs.tar root@capsule.linuxone.in:/root/
```

- capsule 安装所需软件包
```
# yum install satellite-capsule
```

- 在 capsule 端执行证书创建时的输出
  安装完成后会有如下显示
  ![22-619-g2](/images/22619/2.png)
  
- 为 capsule 添加 lifecycle env 
  - 1. web gui > infrastructure > capsules > 对应 capsule edit
    ![22-619-g3](/images/22619/3.png)
  - 2. `Lifecycle EnvironmentsLibrarycapsule-deploy` 标签下进行附加
    ![22-619-g4](/images/22619/4.png)
	
- 手动进行 capsule 同步
  - 1. web gui > infrastructure > capsules > 点击 capsule > sync 
    ![22-619-g5](/images/22619/5.png)
  
---

## client 注册到 satellite

- client 执行以下命令
```
# curl --insecure --output katello-ca-consumer-latest.noarch.rpm https://satellite.linuxone.in/pub/katello-ca-consumer-latest.noarch.rpm
# yum localinstall katello-ca-consumer-latest.noarch.rpm
# subscription-manager register --org="org" --activationkey="capsule-deploy"
```

- 安装必须的 satellite-tools 组件 （将会在 satellite 7 中移除）
```
# yum install katello-agent katello-host-tools-tracer
```

## 更新 satellite 6.8 / capsule 6.8 / client
- satellite
  - 开启以下 repo
    ```
# subscription-manager repos --enable \
rhel-7-server-satellite-maintenance-6-rpms
	```
  - 安装必须软件包
    ```
# yum install rubygem-foreman_maintain
	```
  - 检查可用更新的版本
    ```
# satellite-maintain upgrade list-versions
	```
	![22-619-g6](/images/22619/6.png)
  - 更新选择 `6.8.z` 版本。检查当前系统是否已经做好 `update` 准备
    ```
# satellite-maintain upgrade check --target-version 6.8.z
	```
  - 开始更新
    ```
# satellite-maintain upgrade run --target-version 6.8.z
	```
  - 更新完成，重启 satellite
    ![22-619-g7](/images/22619/7.png)
    ```
# satellite-maintain service stop
# reboot
	```
- capsule 
  - 升级并重启 `gofer` 服务
    ```
# satellite-maintain packages install gofer
# systemctl restart goferd
	```
  - 开启以下 repo
    ```
# subscription-manager repos --enable \
rhel-7-server-satellite-maintenance-6-rpms
	```
  - 检查可用更新的版本
    ```
# satellite-maintain upgrade list-versions
	```
  - 更新选择 `6.8.z` 版本。检查当前系统是否已经做好 `update` 准备
    ```
# satellite-maintain upgrade check --target-version 6.8.z
	```
  - 开始更新
    ```
# satellite-maintain upgrade run --target-version 6.8.z
	``` 
  - 停止 capsule 服务，并重启
    ```
# satellite-maintain service stop
# reboot
	```

- client
  - 升级 `gofer` 
    ```
# yum update gofer 
	```
  - 重启 `goferd` 服务
    ```
# systemctl restart goferd
	```
  - 更新后重启
    ```
# yum update
# reboot
	```

---

## 升级 satellite 6.8 至 6.9 （satellite / capsule /client）
- satellite 6.8 > 6.9
  - 确认开启以下 repo
    ```
# subscription-manager repos --enable \
rhel-7-server-satellite-maintenance-6-rpms
	```
  - 检查可用更新的版本
    ```
# satellite-maintain upgrade list-versions
	```
  - 更新选择 `6.9` 版本。检查当前系统是否已经做好 `upgrade` 准备
    ```
# satellite-maintain upgrade check --target-version 6.9
	```
  - 开始升级
    ```
# satellite-maintain upgrade run --target-version 6.9
	``` 
  - 停止 satellite 服务，并重启
    ```
# satellite-maintain service stop
# reboot
	```
- capsule 6.8 > 6.9
  - satellite 版本为6.9，且已经同步以下repo 并发布到 cv
    ```
rhel-7-server-satellite-capsule-6.9-rpms
rhel-7-server-satellite-tools-6.9-rpms
	```
  - 安装升级检测软件包：
    ```
# yum install rubygem-foreman_maintain
	```
  - 升级并重启 `gofer` 服务
    ```
# satellite-maintain packages install gofer
# systemctl restart goferd
	```
  - 检查可用更新的版本
    ```
# satellite-maintain upgrade list-versions
	```
  - 更新选择 `6.9` 版本。检查当前系统是否已经做好 `upgrade` 准备
    ```
# satellite-maintain upgrade check --target-version 6.9
	```
  - 开始升级
    ```
# satellite-maintain upgrade run --target-version 6.9
	``` 
  - 停止 satellite 服务，并重启
    ```
# satellite-maintain service stop
# reboot
	```
- client 升级步骤如上

---

## 升级 satellite 6.9 至 6.10 （satellite / capsule /client）
- satellite 6.9 > 6.10
  - 准备迁移到 pulp3
    - 1. 确保以下命令可以正确执行
      ```
# foreman-rake katello:correct_repositories COMMIT=true --trace
	  ```
	- 2. 更新文件权限
	  ```
# satellite-maintain prep-6.10-upgrade
	  ```
	- 3. 检查预迁移的内容
	  ```
# satellite-maintain content migration-stats
	  ```
	- 4. 准备迁移的内容
	  ```
# satellite-maintain content prepare
	  ```
	- 5. 检查迁移进度
	  ```
# satellite-maintain content migration-stats
	  ```
  - 升级 satellite 6.9 -> 6.10
    - 1. 确保以下条目被注释或不存在
	  ```
/etc/foreman-installer/custom-hiera.yaml
apache::purge_configs: false
	  ```
	- 2. 开启以下 repo
	  ```
# subscription-manager repos --enable \
rhel-7-server-satellite-maintenance-6-rpms
	  ```
  - 3. 检查可用更新的版本
    ```
# satellite-maintain upgrade list-versions
	```
  - 4. 更新选择 `6.10` 版本。检查当前系统是否已经做好 `upgrade` 准备
    ```
# satellite-maintain upgrade check --target-version 6.10
	```
  - 5. 开始升级
    ```
# satellite-maintain upgrade run --target-version 6.10
	``` 
  - 6. 停止 satellite 服务，并重启
    ```
# satellite-maintain service stop
# reboot
	```
  - 7. 迁移 pulp3 完成，执行以下命令清理 pulp2 中的内容
    ```
# satellite-maintain content remove-pulp2
	```
- capsule 6.9 > 6.10
  - 1. satellite 同步并开启以下 repo
    ```
rhel-7-server-satellite-capsule-6.10-rpms
rhel-7-server-satellite-tools-6.10-rpms
	```
  - 2. 确保以下条目被注释或不存在
	```
/etc/foreman-installer/custom-hiera.yaml
apache::purge_configs: false
	```
  - 3. 检查可用更新的版本
    ```
# yum install rubygem-foreman_maintain
# satellite-maintain upgrade list-versions
	```
  - 4. 更新选择 `6.10` 版本。检查当前系统是否已经做好 `upgrade` 准备
    ```
# satellite-maintain upgrade check --target-version 6.10
	```
  - 5. 开始升级
    ```
# satellite-maintain upgrade run --target-version 6.10
	``` 
  - 6. 停止 satellite 服务，并重启
    ```
# satellite-maintain service stop
# reboot
	```
  - 7. 迁移 pulp3 完成，执行以下命令清理 pulp2 中的内容
    ```
# satellite-maintain content remove-pulp2
	```
  - 8. capsule 升级完成后，请手动执行同步，因为内容的迁移并不会从 satellite 迁移过来。
  
- client 升级
  - 开启以下repo
    ```
rhel-7-server-satellite-tools-6.10-rpms
    ```	
  - 更新软件包
    ```
yum install -y katello-agent katello-host-tools-tracer
	```

## 升级 satellite 6.10 至 6.11 rhel8 （satellite / capsule /client）
- satellite 6.10 > satellite 6.11
  - 1. 开启以下 rpeo
    ```
# subscription-manager repos --enable rhel-7-server-satellite-maintenance-6.11-rpms
	```
  - 2. 检查可用版本
    ```
# satellite-maintain upgrade list-versions
	```
  - 3. 升级预检查
    ```
# satellite-maintain upgrade check --target-version 6.11
	```
  - 4. 升级，完成后重启 satellite
    ```
# satellite-maintain upgrade run --target-version 6.11
	```
	
- capsule 6.10 > 6.11
  - 1. satellite 同步并开启以下 repo
    ```
rhel-7-server-ansible-2.9-rpms
rhel-7-server-rpms/7Server
rhel-7-server-satellite-capsule-6.11-rpms
rhel-7-server-satellite-maintenance-6.11-rpms
rhel-server-rhscl-7-rpms
	```  
  - 2. satellite 重新为 capsule 创建 cert 文件
    ```
# capsule-certs-generate --foreman-proxy-fqdn "capsule.linuxone.in" --certs-update-all --certs-tar "~/capsule.linuxone.in-certs.tar"
# scp capsule.linuxone.in-certs.tar root@capsule.linuxone.in:~
	```
  - 3. 确保 capsule 可以访问以上 repo ，然后执行检查维护
    ```
# satellite-maintain self-upgrade
	```
  - 4. 检查 capsule 中配置的 satellite 是否已经指向 satellite
    ```
# grep foreman_url /etc/foreman-proxy/settings.yml
	```
  - 5. 检查运行状况确认是否可以升级
    ```
# satellite-maintain upgrade check --target-version 6.11
	```
  - 6. 将新生成的证书解压到 /root 目录
    ```
# tar xf capsule.linuxone.in-certs.tar
	```
  - 7. 执行升级
    ```
# satellite-maintain upgrade run --target-version 6.11
	```
	
- satellite 6.11 RHEL7 升级 RHEL8
  - 1. 开启以下 repo 
    ```
# subscription-manager repos --enable=rhel-7-server-extras-rpms
    ```	
  - 2. 安装所需的 leapp leapp-repository 软件包
    ```
# satellite-maintain packages install leapp leapp-repository
	```
  - 3. leapp 提前分析
    ```
# leapp preupgrade
	```
  - 4. 手动解决 leapp 分析的问题
    ```
# rmmod pata_acpi
# leapp answer --section remove_pam_pkcs11_module_check.confirm=True\
# echo PermitRootLogin yes | tee -a /etc/ssh/sshd_config
	```
  - 5. 由于 bug ，为 satellite 添加标记，并持续 preupgrade 直到没有 error
    ```
# subscription-manager repo-override --repo=satellite-6.11-for-rhel-8-x86_64-rpms --add=module_hotfixes:1   <<< 必须
# subscription-manager repo-override --repo=ansible-2.9-for-rhel-8-x86_64-rpms --add=module_hotfixes:1
# subscription-manager repo-override --repo=satellite-maintenance-6.11-for-rhel-8-x86_64-rpms  --add=module_hotfixes:1

# subscription-manager repo-override --repo=satellite-capsule-6.11-for-rhel-8-x86_64-rpms --add=module_hotfixes:1
	```
  - 6. 由于 BUG， 会出现 ansible 的报错，采用以下缓解办法
    ```
# rpm -e ansible ansible-test --nodeps
	```
  - 7. 升级过程
    - 7.1 leapp upgrade 命令无报错
    ![22-619-g35](/images/22619/35.JPG)
	- 7.2 重启主机，开始升级
    ![22-619-g36](/images/22619/36.JPG)
	      升级完成时会有以下显示
	![22-619-g37](/images/22619/37.JPG)
	- 7.3 升级完成自动重启，此时需要重新 label 各个文件系统，需要一定时间
	![22-619-g38](/images/22619/38.JPG)
	- 7.4 计算机启动后，检查当前系统版本与 satellite 状态
	![22-619-g39](/images/22619/39.JPG)
  - 8. 重新运行安装命令
    ```
# satellite-installer
	```
- capsule 6.11 RHEL7 升级 RHEL8  
  - 1. satellite 同步以下 repo 
    ```
satellite-capsule-6.11-for-rhel-8-x86_64-rpms 
satellite-maintenance-6.11-for-rhel-8-x86_64-rpms
ansible-2.9-for-rhel-8-x86_64-rpms
baseos 8.6
appstream 8.6
ha 8.6
	```  
  - 2. 开启以下 repo 
    ```
# subscription-manager repos --enable=rhel-7-server-extras-rpms
    ```	
  - 3. 安装所需的 leapp leapp-repository 软件包
    ```
# satellite-maintain packages install leapp leapp-repository
	```
  - 4. 由于 bug ，为 satellite 添加标记，并持续 preupgrade 直到没有 error
    ```
	# subscription-manager repo-override --repo=satellite-capsule-6.11-for-rhel-8-x86_64-rpms --add=module_hotfixes:1
	```
  - 5. 由于 BUG， 会出现 ansible 的报错，采用以下缓解办法
    ```
# rpm -e ansible ansible-test --nodeps
	```
  - 6. 参考以下只是文档，下载 `leapp-data17.tar.gz` 到 capsule
    ```
# tar -xzf leapp-data17.tar.gz -C /etc/leapp/files && rm leapp-data17.tar.gz
	```
    Leapp utility metadata in-place upgrades of RHEL for disconnected upgrades
    https://access.redhat.com/articles/3664871
  - 7. leapp 提前分析，直至 leapp preupgrade 没有报错
    ```
# leapp preupgrade --target 8.6
	```
  - 8. 手动解决 leapp 分析的问题
    ```
# rmmod pata_acpi
# leapp answer --section remove_pam_pkcs11_module_check.confirm=True
# echo PermitRootLogin yes | tee -a /etc/ssh/sshd_config
	```
  - 9. 进行升级
    - 9.1 leapp 进行升级，并重启
     ```
# leapp upgrade --target 8.6
# reboot
     ```
    - 9.2 从新内核启动
     ![22-619-g41](/images/22619/41.png)
    - 9.3 selinux 需要 relabel ，需要等待一段时间
	 ![22-619-g40](/images/22619/40.png)
  - 10. 重新检查 capsule 一致性
    ```
# satellite-installer
	```

至此， satellite/capsule 6.8 升级 6.9 / 6.10 / 6.11 + RHEL8 的所有流程已经整理完毕，所有过程仅供参考。
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  



