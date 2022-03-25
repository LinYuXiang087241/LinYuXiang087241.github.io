---
title: server-add-disk
date: 2022-03-23 10:47:47
tags: 
  - x86_64
  - H700 H310 H710
categories: 
  - usage
---

# 通过H710 raid卡为服务器直通硬盘
- 准备：
  - 服务器关机
  - 为服务器添加硬盘
- 详细步骤：
  1. 重启服务器通过ctrl+R进入raid卡配置界面
  ![3-23-1](/images/3-23/3-23-1.jpg)
  2. 进入raid卡H710管理界面，其中`Unconfigured Physical Disk`显示的便是新加入的硬盘
  ![3-23-2](/images/3-23/3-23-2.jpg)
  3. 按`F2`进入操作，选择`Create New VD`
  ![3-23-3](/images/3-23/3-23-3.jpg)
  4. 通过下图配置硬盘为raid0直通服务器
  ![3-23-4](/images/3-23/3-23-4.jpg)
  5. 保存配置后，退出重启服务器。
  - 配置成功后会告知已经配置OK
    ![3-23-5](/images/3-23/3-23-5.jpg)
  - 配置完成后的结果图
    ![3-23-6](/images/3-23/3-23-6.jpg)

**<u><font color=red>仅用作经验记录</font></u>**	
  

