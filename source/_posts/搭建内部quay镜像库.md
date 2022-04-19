---
title: 搭建内部quay镜像库
date: 2022-04-11 22:39:41
tags:
  - registry
  - container
  - quay
categories:
  - configruation
---

# 搭建内部quay镜像库
![22-411-17](/images/22411/17.png)
<html><pre><code>前提：部署https内部quay镜像仓库
由于内网quay主机配置不够，仅提供配置步骤，不会保留该服务</code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**

<!-- more -->

#### 前提
  - 1. 移除系统自带`runc`/`podman`/`conmon`
  - 2. 关闭selinux
  - 3. 关闭firewalld
  - 4. 完成后重启  
  - 5. 域名解析要配置完成，为确保不出错，可以配置到`/etc/hosts`文件中
---
#### 开始安装
- 1. 安装必要软件包
  ```
  # yum install podman container-tools -y
  ```
- 2. podman 登录官方镜像库 registry.redhat.io
  ```
  # podman login registry.redhat.io
  ```
- 3. 创建文件夹
  ```
  # mkdir -p $QUAY/config
  # mkdir -p $QUAY/postgres-quay
  # mkdir -p $QUAY/storage
  ```
- 4. 启动postgres
    - 4.1 启动postgres容器
      ```
      # podman run -d --rm  --name postgresql-quay -e POSTGRESQL_USER=quayuser \
        -e POSTGRESQL_PASSWORD=quaypass -e POSTGRESQL_DATABASE=quay \
	    -e POSTGRESQL_ADMIN_PASSWORD=adminpass -p 5432:5432 -v $QUAY/postgres-quay:/var/lib/pgsql/data:z registry.redhat.io/rhel8/postgresql-10:1
      ```
	- 4.2 确保 `pg_trgm` 模块安装
      ```	
      # podman exec -it postgresql-quay /bin/bash -c 'echo "CREATE EXTENSION IF NOT EXISTS pg_trgm" | psql -d quay -U postgres'
      ```
- 5. 启动redis
  ```
  podman run -d --rm --name redis -p 6379:6379 -e REDIS_PASSWORD=strongpassword registry.redhat.io/rhel8/redis-5:1
  ```
- 6. 配置证书，步骤较多，命令如下
  - 6.1 创建证书存放目录
    ```
    # mkdir /etc/crts
    ```
  - 6.2. 创建证书
    - 6.2.1 创建ca:
	```
    # openssl genrsa -out /etc/crts/cert.ca.key 4096
    # openssl req -x509 -new -nodes  -key /etc/crts/cert.ca.key  -sha256 -days 36500  -out /etc/crts/cert.ca.crt  -subj /CN="Local Red Hat Ren Signer"  -reqexts SAN  -extensions SAN  -config <(cat /etc/pki/tls/openssl.cnf  <(printf '[SAN]\nbasicConstraints=critical, CA:TRUE\nkeyUsage=keyCertSign, cRLSign, digitalSignature'))
    ```

	- 6.2.2 创建key，其中红色部分替换为当前quay域名:
	```
    # openssl genrsa -out /etc/crts/cert.key 2048
    # openssl req -new -sha256 -key /etc/crts/cert.key -subj "/O=Local Cert/CN=<font color=red>quay.linuxone.in</font>" -reqexts SAN  -config <(cat /etc/pki/tls/openssl.cnf  <(printf "\n[SAN]\nsubjectAltName=DNS:<font color=red>quay.linuxone.in</font>\nbasicConstraints=critical, CA:FALSE\nkeyUsage=digitalSignature, keyEncipherment, keyAgreement, dataEncipherment\nextendedKeyUsage=serverAuth")) -out /etc/crts/cert.csr
    # openssl x509 -req -sha256  -extfile <(printf "subjectAltName=DNS:<font color=red>quay.linuxone.in</font>\nbasicConstraints=critical, CA:FALSE\nkeyUsage=digitalSignature, keyEncipherment, keyAgreement, dataEncipherment\nextendedKeyUsage=serverAuth")  -days 3650  -in /etc/crts/cert.csr  -CA /etc/crts/cert.ca.crt  -CAkey /etc/crts/cert.ca.key  -CAcreateserial -out /etc/crts/cert.crt
    # openssl x509 -in /etc/crts/cert.crt -text
	```
	
	- 6.2.3. 将证书移动到以下目录并信任:
	```
    # cp cert.ca.key  /etc/pki/ca-trust/source/anchors/
    # update-ca-trust extract</code></pre></html>
    ```
	 
- 7. 配置quay 
  - 7.1 启动quay config 容器
    ```
	# podman run --rm -it --name quay_config -p 8080:8080 -p 443:8443  registry.redhat.io/quay/quay-rhel8:v3.4.1 config secret  
	```
	![22-411-8](/images/22411/8.png)
  - 7.2 访问8080端口，用户名:`quayconfig`，密码:`secret`，进入配置页面
    ![22-411-9](/images/22411/9.png)
	- 7.2.1 配置`Server Configuration`
      ![22-411-10](/images/22411/10.png)
	  1. 填写quay fqdn
	  2. 选择为 `Red Hat Quay handles TLS`
	  3. 选择 `cert.crt` 文件
	  4. 选择 `cert.key` 文件
	- 7.2.2 配置`Database`
	  ![22-411-11](/images/22411/11.png)
	  1. 选择postgres
	  2. Sever 配置为当前quay主机fqdn
	  3. name配置为`4.1`中`POSTGRESQL_USER`
	  4. 密码配置为`4.1`中`POSTGRESQL_PASSWORD`
	  5. 数据库名配置为`4.1`中`POSTGRESQL_DATABASE`
	- 7.2.3 配置`Redis`
	  ![22-411-12](/images/22411/12.png)
	  1. 容器启动主机的fqdn
	  2. 密码为`5`中`REDIS_PASSWORD`
	- 7.2.4 配置 `Registry Storage`
	  ![22-411-13](/images/22411/13.png)
	  1. 可以配置为本地存储，选择`Locally mounted directory`
	  2. 配置目录请记住，默认为`/datastorage/registry`
	- 7.2.5 `Access Settings`配置超级用户，该步骤请于后续执行
	  ![22-411-14](/images/22411/14.png)
	  1. 填写用户名
	  2. 点击`add` 添加
	- 7.2.6 基本配置好后，点击`Validate Configuration Changes`，下载 quay-config.tar.gz 并解压到 `$QUAY/config` 目录中
	  ![22-411-15](/images/22411/15.png)
- 8. 启动quay
  ```
  # podman run -d --rm -p 80:8080 -p 443:8443     --name=quay    -v $QUAY/config:/conf/stack:Z    -v $QUAY/storage:/datastorage:Z    registry.redhat.io/quay/quay-rhel8:v3.6.3
  ```
    - 8.1 启动后点击 `Create Account` 创建用户，例`quayadmin`，密码为`password`
	  ![22-411-16](/images/22411/16.png)
	- 8.2 创建完成用户后，关闭该容器
	  ```
	  # podman rm -f quay
	  ```
	- 8.3 重新执行第7步，包括`7.2.5`，配置`quayadmin`为超级用户，配置完成后下载 quay-config.tar.gz，并覆盖之前`$QUAY/config` 目录中内容
	- 8.4 再次启动quay，使用quayadmin登录
- 9. 配置podman信任本地quay
  - 9.1 将ca证书复制到对应目录，如果不存在，则创建
    ```
	# cp /etc/crts/cert.ca.crt /etc/containers/certs.d/quay.linuxone.in/cert.ca.crt
	```
  - 9.2 创建完成后，信任该证书
    ```
	# update-ca-trust extract
	```
  - 9.3 测试登录
    ```
	# podman login quay.linuxone.in
	```

- 10. 配置容器重启后依旧生效
  - 10.1 podman自动生成service文件
    ```
# podman generate systemd --new --files --name redis
# podman generate systemd --new --files --name postgresql-quay
# podman generate systemd --new --files --name quay
	```
  - 10.2 将service文件复制到指定目录
    ```
# cp -Z container-redis.service /usr/lib/systemd/system
# cp -Z container-postgresql-quay.service /usr/lib/systemd/system
# cp -Z container-quay.service /usr/lib/systemd/system
	```
  - 10.3 使用`systemd`的方式进行管理
    ```
    # systemctl daemon-reload 
    # systemctl enable --now container-redis.service
    # systemctl enable --now container-postgresql-quay.service
    # systemctl enable --now container-quay.service
	```
	
- 11. 配置容器镜像漏洞扫描服务`Clair`
  - 11.1 创建`Clair`容器所需的目录
    ```
	# mkdir -p $QUAY/postgres-clairv4/
	```
  - 11.2 启动 `Clair` 所需单独使用的 postgresql 数据库
    ```
	# podman run -d --rm --name postgresql-clairv4   -e POSTGRESQL_USER=clairuser   -e POSTGRESQL_PASSWORD=clairpass   -e POSTGRESQL_DATABASE=clair   -e POSTGRESQL_ADMIN_PASSWORD=adminpass   -p 5433:5432   -v $QUAY/postgres-clairv4:/var/lib/pgsql/data:Z   registry.redhat.io/rhel8/postgresql-10:1
	```
  - 11.3 为`postgresql` 添加 `Clair` 所需的数据库
    ```
	# podman exec -it postgresql-clairv4 /bin/bash -c 'echo "CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\"" | psql -d clair -U postgres'
	```
  - 11.4 启动 quay 配置容器，额外进行以下配置
    ```
	# podman run --rm -it --name quay_config   -p 80:8080 -p 443:8443   -v $QUAY/config:/conf/stack:Z   registry.redhat.io/quay/quay-rhel8:v3.6.4 config secret
	```
    在配置页面进行以下配置并保存配置文件 `quay-confg.tar.gz`
	![22-411-18](/images/22411/18.png)
  - 将对应配置解压到`$QUAY/config`目录，然后配置 `Clair` 配置文件
    ```
# cat /etc/clairv4/config/config.yaml
http_listen_addr: :8081
introspection_addr: :8089
log_level: debug
indexer:
  connstring: host=quay.linuxone.in port=5433 dbname=clair user=clairuser password=clairpass sslmode=disable   <<<
  scanlock_retry: 10
  layer_scan_concurrency: 5
  migrations: true
matcher:
  connstring: host=quay.linuxone.in port=5433 dbname=clair user=clairuser password=clairpass sslmode=disable   <<<
  max_conn_pool: 100
  run: ""
  migrations: true
  indexer_addr: clair-indexer
notifier:
  connstring: host=quay.linuxone.in port=5433 dbname=clair user=clairuser password=clairpass sslmode=disable   <<<
  delivery_interval: 1m
  poll_interval: 5m
  migrations: true
auth:
  psk:
    key: "aDFiMmY2ajk4NjI1OQ=="   <<<
    iss: ["quay"]
# tracing and metrics
trace:
  name: "jaeger"
  probability: 1
  jaeger:
    agent_endpoint: "localhost:6831"
    service_name: "clair"
metrics:
  name: "prometheus"
	```

  - 11.5 启动`Clair` 容器
    ```
	# podman run -d --rm --name clairv4   -p 8081:8081 -p 8089:8089   -e CLAIR_CONF=/clair/config.yaml -e CLAIR_MODE=combo   -v /etc/clairv4/config:/clair:Z   registry.redhat.io/quay/clair-rhel8:v3.6.4
	```
  - 11.6 然后启动 quay 容器
  - 11.7 由于 quay 容器依赖与`Clair`容器启动后启动，进行以下配置
    ```
# cat /usr/lib/systemd/system/container-quay.service
[Unit]
Description=Podman container-quay.service
Documentation=man:podman-generate-systemd(1)
Wants=network.target
After=container-clairv4.service   <<<

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
RestartSec=30
ExecStartPre=/bin/rm -f %t/container-quay.pid %t/container-quay.ctr-id
ExecStart=/usr/bin/podman run --conmon-pidfile %t/container-quay.pid --cidfile %t/container-quay.ctr-id --cgroups=no-conmon -d --rm -p 8080:8080 --name=quay -v /home/user1/quay/config:/conf/stack:Z -v /home/user1/quay/storage:/datastorage:Z registry.redhat.io/quay/quay-rhel8:v3.4.0
ExecStop=/usr/bin/podman stop --ignore --cidfile %t/container-quay.ctr-id -t 10
ExecStopPost=/usr/bin/podman rm --ignore -f --cidfile %t/container-quay.ctr-id
PIDFile=%t/container-quay.pid
KillMode=none
Type=forking

[Install]
WantedBy=multi-user.target default.target
	```

  -  11.8 关闭漏洞扫描的原因为，当前主机为我自己家用nas 4核心，开启漏洞扫描后负载为：
     ![22-411-18](/images/22411/19.png)
	 根本负担不起，所以就不开启此服务了。

---

#### 一些遇到的小问题
1. 不想进行安全验证，则添加`--tls-verify=false`
```
# podman login quay.linuxone.in --tls-verify=false
```

2. 镜像被签名，无法推送到本地quay，则添加`--remove-signatures`
```
# podman push --remove-signatures quay.linuxone.in/quay/postgresql-10:1
```

3. 配置podman对于docker.io加速
  - 3.1 已经有阿里云镜像加速器
  - 3.2 对应修改`registry.conf`文件
<html><pre><code># /etc/containers/registries.conf
unqualified-search-registries = ["registry.fedoraproject.org", "registry.access.redhat.com", "registry.centos.org", <font color=red>"docker.io"</font>]
<font color=red>[[registry]]
prefix = "docker.io"
location = "xacs4bss.mirror.aliyuncs.com"</font> </code></pre></html>

4. 本地配置https本地镜像库的证书
将ca证书复制到以下以镜像库`url`命令的目录，如果不存在，则创建

<html><pre><code># cp /etc/crts/cert.ca.crt /etc/containers/certs.d/<font color=red>quay.linuxone.in</font>/cert.ca.crt</code></pre></html>