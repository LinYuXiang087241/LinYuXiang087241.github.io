---
title: RH403_satellite6-主机调配_二
date: 2022-06-19 23:30:19
tags:
  - RH403_Satellite6
categories:
  - usage
---

![22-619-g1](/images/22619/1.png)
<html><pre><code> 红帽 Satellite6.10
调配主机 </code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

### satellite 调配主机（pxe）
- 1. 安装额外 capsule2 用于提供 pxe 服务(capsule2.linuxone.in)
- 2. 提前同步 kictstart 存储库，发布并推送到 capsule 所在的 lifecycle env
- 3. 配置 Hosts > Provisioning Templates 配置模板
  ![22-619-g8](/images/22619/8.png)
  - 3.1 点击 Kickstart default PXELinux
    - Associtation 包含需要安装的操作系统
      ![22-619-g9](/images/22619/9.png)
	- Locations 保持和用于调配 capsule 一致，我环境中 capsule2 所属 location 为 loc，确认后点击 `submit`
	  ![22-619-g10](/images/22619/10.png)
  - 3.2 点击  Kictstart default 进行与 `3.1`相同的配置
- 4. 确认分区表在当前位置 loc 可用
  - 4.1 Hosts > Partition Tables 点击 Kickstart default
	![22-619-g11](/images/22619/11.png)
  - 4.2 点击 Locations ，确保 loc 位置已经配置
    ![22-619-g12](/images/22619/12.png)
- 5. 确认希望安装的操作系统引用正确的调配模板和分区表
  - 5.1 Hosts > Operating Systems 点击需要安装操作系统，我测试的为 rhel8.1
    ![22-619-g13](/images/22619/13.png)
  - 5.2 确保 Partition Table 标签下已经选中 Kictstart default
    ![22-619-g14](/images/22619/14.png)
  - 5.3 确保 Templates 下 PXELinux template 已设为 Kickstart default PXELinux ，
        并且 Provisioning template 已设为 Kickstart default ， 并点击 submit 提交
	![22-619-g15](/images/22619/15.png)
- 6. 配置调配网络
  - 6.1 我测试环境中，capsule2 的信息如下，并创建 ks.linuxone.in 的域名用于调配
   | 参数 | 数值 |
   |--- | --- | 
   | DNS服务网卡 | ens33（vmware 虚拟机默认网卡名） | 
   | DNS 转发 | 192.168.31.1 | 
   | DNS ZONE | ks.linuxone.in | 
   | DNS reverse | 31.168.192.in-addr.arpa | 
   | DHCP服务网卡 | ens33（vmware 虚拟机默认网卡名） |
   | DHCP范围 | 192.168.31.20 - 192.168.31.35 |
   | DHCP主机 | capsule2的ip 192.168.31.72 |
   | DHCP网关 | 192.168.31.1 |

	在 capsule2 执行以下命令
	```
#  satellite-installer --scenario capsule \
 --foreman-proxy-dns true \
 --foreman-proxy-dns-interface ens33 \
 --foreman-proxy-dns-forwarders 192.168.31.1 \
 --foreman-proxy-dns-zone ks.linuxone.in \
 --foreman-proxy-dns-reverse 31.168.192.in-addr.arpa \
 --foreman-proxy-dhcp true \
 --foreman-proxy-dhcp-interface ens33 \
 --foreman-proxy-dhcp-range "192.168.31.20 192.168.31.35" \
 --foreman-proxy-dhcp-nameservers 192.168.31.72 \
 --foreman-proxy-dhcp-gateway 192.168.31.1 \
 --foreman-proxy-tftp true
    ```
	
  - 6.2 确认 capsule2 的 DNS、DHCP、TFTP 功能已经启用 Infrastructure > capsules > capsule2.linuxone.in 在 Active features 查看，如果没有请刷新尝试
    ![22-619-g16](/images/22619/16.png)
	
- 7. 创建 ks.linuxone.in DNS 域名
  - 7.1 Infrastructure > Domains 点击 Create Domain
    - 1. DNS Domain 输入 ks.linuxone.in
	- 2. DNS Capsule 输入 capsule2.linuxone.in
	![22-619-g17](/images/22619/17.png)
	- 3. 确认位置 loc 已经关联
	![22-619-g18](/images/22619/18.png)
	- 4. 确认组织 org 已经关联
	- 5. 点击 `submit` 提交
	![22-619-g19](/images/22619/19.png)

- 8. 创建 kickstart 专用子网，子网名 ks date center
  - 8.1 Infrastructure > Capsules 点击 capsule2.linuxone.in 
    ![22-619-g20](/images/22619/20.png)
  - 8.2 点击 Actions 并选择 Import IPv4 subnets
    ![22-619-g21](/images/22619/21.png)
  - 8.3 子网信息如下，并点击 `submit` 提交
    | 参数 | 数值 |
    |--- | --- | 
    | Name | ks date center |
	| Primary DNS Server | 192.168.31.72 |
	| IPAM | DHCP |
- 9. 修改子网配置
  - 9.1 Infrastructure > Subnets 点击 ks date center ，Subnet 标签根据之前 DHCP 时的规划填写
    ![22-619-g22](/images/22619/22.png)
  - 9.2 Domains 标签下，将该子网与 Domain ks.linuxone.in 关联
    ![22-619-g23](/images/22619/23.png)
  - 9.2 capsules 标签下，将所有可选项均修改为 capsule2.linuxone.in
    ![22-619-g24](/images/22619/24.png)
  - 9.3 Locations 与 Organizations 标签，请选择与 capsule2 相同的内容

- 10. 创建主机组
  - 10.1 Configure > host groups 点击 Create Host Groups 内容以下
    | 参数 | 数值 |
    |--- | --- | 
    | Name | ks host group (自定义即可) |
    | Lifecycle Env | kickstart(由之前创建) | 	
	| Content View | kictstart(由之前创建) |
	| Content Source | capsule2.linuxone.in |
    ![22-619-g25](/images/22619/25.png)
  - 10.2 Network 选项卡，将 Domain 配置为 ks.linuxone.in ，ipv4 subnet 配置为 ks date center
    ![22-619-g26](/images/22619/26.png)
  - 10.3 Operating System 标签卡进行如下配置
    ![22-619-g27](/images/22619/27.png)
  - 10.4 Locations 与 Organizations 标签，请选择与 capsule2 相同的内容，点击 `submit` 提交
  
- 11. 调配主机，pxe 创建 rh8u1.ks.linuxone.in 
  - 11.1 Hosts > Create host 创建以下信息
    | 参数 | 数值 |
    |--- | --- | 
    | Name | rh8u1 |
    | Host Group | ks host group | 	
	| Organization | org | 
    | Location | loc |
    ![22-619-g28](/images/22619/28.png)
  - 11.2 Operating Systemt 标签下，更改 root 密码为 redhat123 ，点击 resolve 可以验证 pxe 配置
    ![22-619-g29](/images/22619/29.png)
  - 11.3 配置 pxe 网卡
    - 1. Interfaces 标签，点击 Edit
	  ![22-619-g30](/images/22619/30.png)
	- 2. 其中 mac address 填写网卡 mac 地址，我使用的 vmware 虚拟机，则额外勾选 virtual NIC
	  ![22-619-g31](/images/22619/31.png)
  - 11.4 配置完成，点击 `submit` 提交
    ![22-619-g32](/images/22619/32.png)
  - 11.5 Vmware 启动虚拟机进行 pxe 自动安装
    ![22-619-g33](/images/22619/33.png)
  - 11.6 安装完成，检查 hostname 与 redhat-release 
    ![22-619-g34](/images/22619/34.png)
    