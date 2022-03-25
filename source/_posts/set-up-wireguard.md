---
title: set-up-wireguard
date: 2022-03-23 15:24:17
tags: 
  - centos7
  - wireguard
categories: 
  - configuration
---

# 在centos7配置wireguard

## wireguard 服务端配置
- 1. 下载必要软件包
```
# yum install epel-release https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
# yum install yum-plugin-elrepo
# yum install kmod-wireguard wireguard-tools
```

- 2. 配置内核参数
```
# echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
# echo "net.ipv4.conf.all.proxy_arp = 1" >> /etc/sysctl.conf
# sysctl -p /etc/sysctl.conf
```

- 3. 配置iptables转发
```
# iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
# iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
# iptables -A FORWARD -i wg0 -o wg0 -m conntrack --ctstate NEW -j ACCEPT
# iptables -t nat -A POSTROUTING -s 192.0.2.0/24 -o eth0 -j MASQUERADE
```

- 4. 生成wg秘钥
```
wg genkey > example.key
wg pubkey < example.key > example.key.p
```

- 5. 安装docker，docker-compose
```
详细步骤可以参考docker官方网站，不建议使用centos编译版本
# yum install -y yum-utils
# yum-config-manager     --add-repo     https://download.docker.com/linux/centos/docker-ce.repo
# yum-config-manager --enable docker-ce-nightly
# yum install docker-ce docker-ce-cli containerd.io
# systemctl enable docker --now

安装docker-compose
# DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
# mkdir -p $DOCKER_CONFIG/cli-plugins
# curl -SL https://dl.mdbg.workers.dev/down/https://github.com/docker/compose/releases/download/v2.2.3/docker-compose-linux-x86_64 -o $DOCKER_CONFIG/cli-plugins/docker-compose
# cd $DOCKER_CONFIG/cli-plugins/
# chmod +x docker-compose
# mv docker-compose /usr/bin/
# docker-compose -v
```

- 6. 使用`docker-compose`启动wireguard-web
  - 检查docker0 网桥的地址
```
# ip a
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:34:35:b6:d5 brd ff:ff:ff:ff:ff:ff
    inet <font color=red>172.17.0.1</font>/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:34ff:fe35:b6d5/64 scope link 
       valid_lft forever preferred_lft forever
```
  - compose文件中的红色`ip`改为`docker0`桥的地址
```
# cat docker-compose.yaml 
version: '3.6'
services:
  wg-gen-web:
    image: vx3r/wg-gen-web:latest
    container_name: wg-gen-web
    restart: always
    expose:
      - "8080/tcp"
    ports:
      - 80:8080
    environment:
      - WG_CONF_DIR=/data
      - WG_INTERFACE_NAME=wg0.conf
      - OAUTH2_PROVIDER_NAME=fake
      - WG_STATS_API=http://172.17.0.1:8182
    volumes:
      - /etc/wireguard:/data
    network_mode: bridge
  wg-json-api:
    image: james/wg-api:latest
    container_name: wg-json-api
    restart: always
    cap_add:
      - NET_ADMIN
    network_mode: "host"
    command: wg-api --device wg0 --listen 172.17.0.1:8182
```
  - 启动docker-compose
```
# docker-compose up -d
```

- 7. 通过网页对wireguard网关进行配置
  - `server interface address` 为wireguard网段的网关地址
    ![3-23-wg1](/images/3-23/3-23-wg1.jpg)
  - `public endpoint`为当前wireguard主机的地址| `Default allowd ip`处填写与`1`中网关相同的网段
    ![3-23-wg2](/images/3-23/3-23-wg2.jpg)
  - 对应图中的内容进行修改
    ![3-23-wg3](/images/3-23/3-23-wg3.jpg)
    ```
	1为：
	iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
	2为：
	iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
	修改完成后点击3保存备份。
	保存后，点击 UPDATE SERVER CONFIGURATION 
	```
  - 配置完成后，检查对应 `/etc/wireguard/wg0.conf` 配置文件是否更改
    ```
	# cat /etc/wireguard/wg0.conf
	[Interface]
    Address = 10.6.6.1/24
	ListenPort = 9999
	PrivateKey = aaaaaaaaaaaaaaaaaaaaaaaaa
	
	PreUp = echo WireGuard PreUp
	PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
	PreDown = echo WireGuard PreDown
	PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
	```
- 8. 配置系统自动加载配置文件、自动生成配置文件
  - 加入ExecReload在不中断活跃连接的情况下重新加载配置文件
~~~
# nano /usr/lib/systemd/system/wg-quick@.service
[Unit]
Description=WireGuard via wg-quick(8) for %I
After=network-online.target nss-lookup.target
Wants=network-online.target nss-lookup.target
PartOf=wg-quick.target
Documentation=man:wg-quick(8)
Documentation=man:wg(8)
Documentation=https://www.wireguard.com/
Documentation=https://www.wireguard.com/quickstart/
Documentation=https://git.zx2c4.com/wireguard-tools/about/src/man/wg-quick.8
Documentation=https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/wg-quick up %i
ExecStop=/usr/bin/wg-quick down %i
ExecReload=/bin/bash -c 'exec /usr/bin/wg syncconf %i <(exec /usr/bin/wg-quick strip %i)'
Environment=WG_ENDPOINT_RESOLUTION_RETRIES=infinity

[Install]
WantedBy=multi-user.target
~~~
  - 为wg-gen-web添加自动Reload
~~~
# nano /etc/systemd/system/wg-gen-web.service
[Unit]
Description=Restart WireGuard
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/systemctl reload wg-quick@wg0.service

[Install]
WantedBy=multi-user.target
~~~
  - 添加wg-gen-web监听路径
~~~
# nano /etc/systemd/system/wg-gen-web.path
[Unit]
Description=Watch /etc/wireguard for changes

[Path]
PathModified=/etc/wireguard

[Install]
WantedBy=multi-user.target
~~~
  - 配置完成后刷新配置
~~~
# systemctl daemon-reload
# wg-quick down wg0
# systemctl enable wg-gen-web.service wg-gen-web.path wg-quick@wg0 --now
~~~

- 9 加入客户端前的操作
  - 在wireguard网页点击`ADD NEW CLIENT`创建client
  - `Client friendly name`为必填项，填写完成点击`SUBMIT`创建
    ![3-23-wg4](/images/3-23/3-23-wg4.jpg)
	
## 加入客户端
### windows客户端
- 1. 登录wireguard.com 下载[win](https://www.wireguard.com/install/)专用的客户端
  ![3-23-wg5](/images/3-23/3-23-wg5.jpg)
- 2. 下载完成双击安装（安装过程需要可以访问外网）
- 3. 安装完成后，下载服务端对应client的配置文件，如图点击`DOWNLOAD`，下载下来的便是`xxxx.conf`
  ![3-23-wg6](/images/3-23/3-23-wg6.jpg)
- 双击打开安装好的wireguard客户端
  ![3-23-wg7](/images/3-23/3-23-wg7.jpg)
- 点击从文件导入隧道即可建立wiregurad连接
  ![3-23-wg8](/images/3-23/3-23-wg8.jpg)

### centos客户端  
- 1. 下载必要软件包
```
# yum install epel-release https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
# yum install yum-plugin-elrepo
# yum install kmod-wireguard wireguard-tools
```

- 2. 从服务端下载client端的配置文件`xxxx.conf`，并将其改名为`wg0.conf`
```
# mv xxxx.conf /etc/wireguard/wg0.conf
```

- 3. 通过服务的方式启动wireguard
```
# systemctl daemon-reload
# systemctl enable wg-quick@wg0 --now |systemctl reload wg-quick@wg0
```

## 安全加固
- 配置除了内网ip外，其他ip无法访问
```
# cat docker-compose.yaml 
version: '3.6'
services:
  wg-gen-web:
    image: vx3r/wg-gen-web:latest
    container_name: wg-gen-web
    restart: always
    expose:
      - "8080/tcp"
    ports:
      - 10.6.6.1:80:8080
    environment:
      - WG_CONF_DIR=/data
      - WG_INTERFACE_NAME=wg0.conf
      - OAUTH2_PROVIDER_NAME=fake
      - WG_STATS_API=http://172.17.0.1:8182
    volumes:
      - /etc/wireguard:/data
    network_mode: bridge
  wg-json-api:
    image: james/wg-api:latest
    container_name: wg-json-api
    restart: always
    cap_add:
      - NET_ADMIN
    network_mode: "host"
    command: wg-api --device wg0 --listen 172.17.0.1:8182
```
- 重新启动docker-compose
```
# docker-compose stop
# docker-compose up -d
```

## 鸣谢
[使用Wireguard基於國内網絡異地組網實踐](https://blog.madebug.net/ops/2021-07-04-use-wireguard-to-rebuild-a-intranet-in-china)
[WireGuard 配置教程：使用 wg-gen-web 来管理 WireGuard 的配置 – 云原生实验室 - Kubernetes|Docker|Istio|Envoy|Hugo|Golang|云原生](https://fuckcloudnative.io/posts/configure-wireguard-using-wg-gen-web/)

**<u><font color=red>仅用作经验记录</font></u>**	