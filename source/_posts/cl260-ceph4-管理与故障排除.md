---
title: cl260-ceph4-管理与故障排除
date: 2022-11-10 22:48:49
  - CL260_ceph
categories:
  - troubleshooting
---

![22-812-g1](/images/22812/1.png)
<html><pre><code> 红帽 Ceph4 
基础的故障排除方式与集群管理 </code></pre></html>
**<u><font color=red>仅用作经验记录</font></u>**
<!-- more -->

---
## 管理 ceph 集群
- 1. 集群管理与监控
  ```
如果系统处于异常状态，会提供更多详细信息
$ ceph health detail
显示 ceph 集群中正在发生的事件和其他实时监控信息
$ ceph -w
  ```
  - 1.1 监控集群，查看 MON 仲裁状态
    ```
# ceph mon stat
# ceph quorum_status -f json-pretty
	```
  - 1.2 查看特定 ceph 守护进程的日志
    ```
$ journalctl -u ceph-osd@10
	```
  - 1.3 为特定守护进程启用日志记录
    ```
$ ceph config set mon.mon1 log_to_file true
	```
- 2. 分析 OSD 的使用情况
  - 2.1 常用命令
    ```
# ceph osd df    报告 osd 使用量
# ceph osd stat  显示 OSD 状态
# ceph osd find  显示 OSD 位置
# ceph osd perf  查看 OSD 性能统计数据
	```
  - 2.1 PG 问题
    ```
# ceph pg dump 
	```

- 3. 验证 dashboard 模块
  ```
# ceph mgr module ls
# ceph mgr services
  ```