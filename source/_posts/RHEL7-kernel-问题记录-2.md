---
title: RHEL7-kernel-问题记录-2
date: 2022-03-30 23:30:46
tags: 
  - kernel
  - RHEL7
categories: 
  - bug
---

# RHEL 7 kernel 问题记录-2
- 由于infiniband错误导致系统重启
  - 调查
    检查vmcore
<html><pre><code>[exception RIP: <font color=red>isert_login_recv_done</font>+40]</html></code></pre>
  - 修复
    - 对于`RHEL 7.5`将版本升级至`RHEL 7.9`及以上  
  - 参考文档
    [<font color=red>**NULL pointer dereference at (null) at isert_login_recv_done+0x28/0x190 [ib_isert]**</font>](https://access.redhat.com/solutions/6780051)
---
- 当发生 IO 超时时，Linux 内核 SCSI 错误处理程序逻辑会继续执行一系列恢复方法，以尝试恢复失败的设备或传输，同时尽可能少地对系统上发生的其他 IO 造成中断。
  每当恢复尝试失败或后续的 SCSI 测试单元就绪 (TUR) 命令失败时，都会按顺序执行标准恢复级别，并升级到下一个级别：
  - 以下是标准恢复过程中的步骤:
    ```
中止超时命令并尝试使设备联机(Abort timed out commands and attempt to bring device online)
为每个故障设备发出 SCSI 设备重置任务管理功能(Issue SCSI device Reset task management function for each failing device)
为每个失败的目标发出 SCSI 目标重置任务管理功能(Issue SCSI target reset task management function for each failing target)
为每个故障总线发出 SCSI 总线重置（模拟为光纤通道环境的一系列端口重置）[Issue SCSI bus reset for each failing bus (emulated as a series of port resets for Fibre Channel environments)]
发出 SCSI 主机总线适配器 (HBA) 重置(Issue SCSI host bus adapter (HBA) reset)
    ```
  - 看到正在尝试重置目标。但是，恢复过程应尽早完成，因此在这种情况下建议进行以下设置
  ```
eh_deadline = 20
eh_timeout = 10
lpfc_task_mgmt_tmo = 20
  ```
  - 还可以考虑更新 HBA 固件
  - 参考文档，限制 SCSI 设备的路径故障转移时间
    [<font color=red>**Limiting path failover time for SCSI devices**</font>](https://access.redhat.com/solutions/627903)
---
**<u><font color=red>仅用作经验记录</font></u>**