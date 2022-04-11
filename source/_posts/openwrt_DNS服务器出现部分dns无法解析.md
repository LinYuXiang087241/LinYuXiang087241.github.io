---
title: openwrt DNS服务器出现部分dns无法解析
date: 2022-04-11 21:35:55
tags: 
  - openwrt
  - dns
categories:
  - configruation
---
# openwrt做DNS服务器出现部分内网dns无法解析
<html><pre><code>前提：使用cloudflare配置局域网解析，但是本地局域网内主机无法顺利解析</code></pre></html>
<!-- more -->

- 问题详述
  - 在cloudflare配置了dns解析，默认可以联网的主机是可以顺利解析
      - 以下是腾讯云主机成功解析
        ![22-411-1](/images/22411/1.png)
	  - 本地主机无法解析该域名，使用openwrt作为网关与dns服务器
	    ![22-411-2](/images/22411/2.png)
      - 但是本地主机可以解析其他cloudflare配置的域名
	    ![22-411-3](/images/22411/3.png)
- 修复
  - 可能由于openwrt配置了重绑定保护，关闭即可
    ![22-411-4](/images/22411/4.png)