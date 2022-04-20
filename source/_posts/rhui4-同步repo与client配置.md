---
title: rhui4-同步repo与client配置
date: 2022-04-20 16:34:07
tags:
  - RHUI4
categories:
  - usage
---

<h2>RHUI4 同步repo与客户端配置 </h2>

<html><pre><code>基于红帽 RHUI 安装手册安装</code></pre></html>

![22-420-1](/images/22420/1.png)

**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

## 前提
确保所有节点已经正常使用

## 添加软件包频道
- `r` manage repositories
   ![22-420-1](/images/22420/1.png)
- `a` add a new Red Hat content repository
   ![22-420-2](/images/22420/2.png)
- 可以自行选择，由于当前为rh员工账号，所以可用的repo很多
  `1` 会导入所有的repo
  可以根据 `2` 和 `3` 自定以导入，此次测试我选择的`2`
  ![22-420-3](/images/22420/3.png)

- 此次文旦计划导入以下 `435` 的软件包频道`Single Sign-On 7.4 for RHEL 8 x86_64 (RPMs) from RHUI`
  - 输入 `435`并回车，此时该软件包频道前有一个 `x`
    ![22-420-5](/images/22420/5.png)
  - 输入`c`确认，并输入`y`确认
    ![22-420-6](/images/22420/6.png)
- 输入`l`，可以查看是否已经启用该repo
  ![22-420-7](/images/22420/7.png)
如上图已经顺利将`Single Sign-On 7.4 for RHEL 8 x86_64 (RPMs) from RHUI` 添加到`RHUI`

---

## 同步该软件包频道
- `s` synchronization status and scheduling
  ![22-420-8](/images/22420/8.png)
- `sr` 立即开始同步
  ![22-420-9](/images/22420/9.png)
  - 选择要同步的软件包频道，上图可以看到是`2`，输入后该repo前`x`，输入`c`，输入`y`，输入后开始同步
    ![22-420-10](/images/22420/10.png)
- 检查同步进度`rr`
  ![22-420-11](/images/22420/11.png)
- 检查同步结果`dr`
  ![22-420-12](/images/22420/12.png)

---

## 创建产品证书与客户端rpm
- 创建证书
  - `e` 进入 `e` generate an entitlement certificate
  - 选择`所有想用`并选择`c`
    ![22-420-13](/images/22420/13.png)
  - 然后输出证书名 / 密钥名 / 时间 / 密钥路径 如下图：
    ![22-420-14](/images/22420/14.png)
- 创建客户端rpm
  - `e` 进入 `c` 输入文件保存的本地路径 / RPM 名称 及之前创建的证书名等信息
    ![22-420-15](/images/22420/15.png)
	
- 测试该rpm 是否可用
  - 客户端安装此`test-2.0-1.noarch.rpm`
    ```
    # yum localinstall test-2.0-1.noarch.rpm
	```
	检查 yum repolist 是否存在之前的repo
	![22-420-16](/images/22420/16.png)
  








  
  