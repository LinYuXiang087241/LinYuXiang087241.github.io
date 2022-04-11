---
title: k3s安装
date: 2022-03-28 18:14:44
tags: 
  - k3s
categories: 
  - installation
---

# 安装K3S
docs.rancher.cn/docs/k3s/quick-start/_index
<!-- more -->
- 1. 脚本安装
  - 常用环境变量
    - `INSTALL_K3S_SKIP_DOWNLOAD`
	   仅用于离线安装，设置为`true`则不会下载k3s二进制执行文件
	   安装前，将`k3s`二进制文件复制到`/usr/local/bin`下
	- `INSTALL_K3S_MIRROR`
	   如果值为`cn`，则改为国内镜像源
	- `INSTALL_K3S_SYMLINK`
	  若对应路径中二进制命令并不在，则创建符号链接
	  `skip`不会创建，`force`会创建并覆盖
	- `INSTALL_K3S_SKIP_ENABLE`
	  设置为`true`则默认不开机启用`K3S`服务
	- `INSTALL_K3S_SKIP_START`
	  设置为`true`，将不会启动`k3s`服务
	- `INSTALL_K3S_VERSION`
	  不指定则下载`stable`版本
	  `INSTALL_K3S_VERSION="v1.19.9+k3s1" `
	- `INSTALL_K3S_BIN_DIR`
	  配置安装`k3s`二进制文件的目录，默认是`/usr/local/bin`
	- `INSTALL_K3S_SYSTEMD_DIR`
	  安装`systemd`服务的目录，默认是`/ect/systemd/system`
	- `INSTALL_K3S_EXEC`
	  安装时额外启动参数，参考rancher官方文档
	- `INSTALL_K3S_NAME`
	  创建systemd的服务名，默认是k3s，以agent方式便是`k3s-agent`  
	- `INSTALL_K3S_SKIP_SELINUX__RPM`
	  如果设置为`true`，跳过 k3s RPM 自动安装
	- `INSTALL_K3S_CHANNEL_URL`	
	  K3S频道版本对应的网址url
	- `INSTALL_K3S_CHANNEL`
	  K3S安装的频道，默认是`stable`
	- `K3S_CONFIG_FILE`
	  指定配置文件位置，默认为`/etc/rancher/k3s/config.yaml`
	- `K3S_TOKEN`
	  用于server或agent加入集群  
	  默认token目录为`/var/lib/rancher/k3s/server/token`
	  添加k3s agent
	  ```
	  # curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://<server_ip>:6443 K3S_TOKEN=<server_token> sh -
	  ```
	- `K3S_TOKEN_FILE`
	  指定token的文件目录
---
# K3S SERVER 常用配置
- `K3S_KUBECONFIG_OUTPUT`
  指定k3s config文件目录，默认为 `/etc/rancher/k3s/k3s.yaml`
  可以将其复制到`/root/.kube/config`
- `INSTALL_K3S_EXEC="--docker"`
  默认使用docker启动k3s
- `INSTALL_K3S_EXEC="--advertise-address value"`
  手动指定向集群内发布的IP地址
- `INSTALL_K3S_EXEC="--node-ip value"`
  指定节点使用的IP
  添加agent节点时， 也需要指定ip
  ```
  # curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn INSTALL_K3S_EXEC="--node-ip <value>" K3S_URL=https://<server_ip>:6443 K3S_TOKEN=<server_token> sh -
  ```
- `INSTALL_K3S_EXEC="--node-external-ip value"`
  配置k3s对外发布的ip节点
- `INSTALL_K3S_EXEC="--tls-san value"`
  指定可以通过哪个IP进行访问，普遍应用于公有云
- `INSTALL_K3S_EXEC="--data-dir=value"`
  指定k3s数据存储目录，默认是`/var/lib/rancher/k3s`
- `INSTALL_K3S_EXEC="--disable value"`
  禁用组件 traefik coredns servicelb local-storage metrics-server
- `--node-label` `--node-taint`
  添加标签与污点
---
# k3s 网络选项
- Flannel 选项 配置文件地址为 /var/lib/rancher/k3s/agent/etc/flannel/net-conf.json
  - `INSTALL_K3S_EXEC="--flannel-backend=value"` 修改默认k3s backend
    种类有 ipsed / host-gw / wireguard
  - directrouting 启用
    - 1. 配置默认flannel的配置文件到别的目录（默认路径重启后会被默认配置覆盖）
	  ```
	  INSTALL_K3S_EXEC="--flannel-conf=/etc/net-conf.json"
	  ```
    - 2. 配置net-conf.json中添加以下内容
	  ```
	  "Directrouting": true
	  ``` 
  - 使用自定义CNI calico
    - 安装k3s时，不安装flannel插件
	  ```
	  INSTALL_K3S_EXEC="--flannel-backend=none"
	  ```
	- 修改calica yaml中添加
	  <html><pre><code>"container_settings": {
	                                 "allow_ip_forwarding": true
									 }</html></code></pre>
	- 将yaml apply至集群
  - 配置k3s mirror加速下载，与阿里云配置docker加速方式相同
  - 配置proxy，目录位于/etc/systemd/system/docker.server.d/http-proxy.conf
    ```
	[Service]
	Environment="HTTP_PROXY=http://ip:port"
	```


