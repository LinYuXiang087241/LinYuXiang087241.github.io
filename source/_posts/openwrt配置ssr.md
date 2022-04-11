---
title: openwrt配置ssr
date: 2022-04-11 21:59:22
tags:
  - openwrt
  - vpn
categories:
  - usage
---
# openwrt配置ssr
<html><pre><code>前提：V2rayN已经导入订阅，将该订阅通过base64加密后导入openwrt ssr</code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

- 前期准备
  - 有订阅
  - 内网一台http主机

- v2rayN订阅导出
  选中所需导出的订阅，选择`服务器` > `批量导出分享url至剪切板` 粘贴保存。
- 进行base64加密
  百度搜索 base64 加密，进行网页加密
  ![22-411-5](/images/22411/5.png)   
  (1)粘贴之前复制的url
  (2)点击加密
  (3)复制加密后的结果，并粘贴到本地http主机 /var/www/html/index.html中
- 启动http主机`httpd`服务
- openwrt如下配置
  - 1. 添加ssr订阅
       服务 > ShadowSocksR Plus+ 设置 > 服务器节点
	   ![22-411-6](/images/22411/6.png)   
	   -(1) 点击添加链接
       -(2) http链接，注意一定要加上index.html
       -(3)	点击更新
	   -(4) 页面最下方，保存
	   -(5) 再次刷新订阅，检查是否可用
  - 2. 启用ssr
       服务 > ShadowSocksR Plus+ 设置 > 客户端
	   ![22-411-7](/images/22411/7.png)   
	   -(1) 主服务器选择导入的订阅节点
	   -(2) 点击`保存&应用`