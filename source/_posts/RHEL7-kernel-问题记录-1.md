---
title: RHEL7-kernel-问题记录-1
date: 2022-03-24 21:27:07
tags: 
  - kernel
  - RHEL7
categories: 
  - bug
---

# RHEL 7 kernel 问题记录-1
- 1. kernel panic内容: `kernel BUG at fs/xfs/xfs_aops.c:1062!`
  - 调查
    通过检查 vmcore 即可发现
<pre><code>[1004630.854317] kernel BUG at <font color=red>fs/xfs/xfs_aops.c:1062!</font> </code></pre>
  - 修复
      - 对于`RHEL 7`升级内核版本至`kernel-3.10.0-693.el7`及以上
      - 对于`RHEL 7.3`升级内核版本至`kernel-3.10.0-514.16.1.el7`及以上
	  - 缓解办法： 如果客户机上 Java `JVM` 中使用`hsperf`功能，请<font color=red>**禁用**</font>
  - 参考文档
    [<font color=red>**RHEL7: kernel crash in xfs_vm_writepage - kernel BUG at fs/xfs/xfs_aops.c:1062!**</font>](https://access.redhat.com/solutions/2779111)
    
---	
- 2. slub内存泄露`SLUB: Unable to allocate memory on node -1 (gfp=0x80d0)`
  - 调查
    检查`/var/log/messages`可以发现
<pre><code>$ grep "Unable to allocate\|mock" var/log/messages
kernel: SLUB: Unable to allocate memory on node -1 (gfp=0x80d0)
kernel: SLUB: Unable to allocate memory on node -1 (gfp=0x80d0)
kernel: SLUB: Unable to allocate memory on node -1 (gfp=0x80d0)</code></pre>
  - 修复
      - 对于`RHEL 7`升级内核版本至`kernel-3.10.0-1062.4.1.el7`及以上
	  - 缓解办法：完全关闭内存“kernel memory accounting altogether”
	    在内核启动参数中添加以下参数`cgroup.memory=nokmem`
  - 参考文档
    [<font color=red>**SLUB: Unable to allocate memory on node -1 (gfp=0x20)**</font>](https://access.redhat.com/solutions/4088471)
   
---
- 3. vmcore中检测到`mem_cgroup_css_offline`崩溃
  - 调查
    调查vmcore
<pre><code>[344254.090710] BUG: unable to handle kernel NULL pointer dereference at 00000000000000b8
[344254.090865] IP: [<ffffffff811f375b>] <font color=red>mem_cgroup_css_offline+0xfb</font>/0x140   <<<
... ...
[344254.093838] RIP: 0010:[<ffffffff811f375b>]  [<ffffffff811f375b>] <font color=red>mem_cgroup_css_offline+0xfb</font>/0x140  <<< </code></pre>
  - 修复
      - 对于`RHEL 7.7`将内核版本升级至`kernel-3.10.0-1062.4.1.el7`及以上
	  - 对于`RHEL 7.6`将内核版本升级至`kernel-3.10.0-957.38.1.el7`及以上
	  - 对于`RHEL 7.5`将内核版本升级至`kernel-3.10.0-862.44.2.el7`及以上
	  - 对于`RHEL 7.4`将内核版本升级至`kernel-3.10.0-693.61.1.el7`及以上
  - 参考文档
     [<font color=red>**Kernel panic with exception RIP: mem_cgroup_css_offline caused by a possible slab leak**</font>](https://access.redhat.com/solutions/3291481) 
---
- 4. kernel bug: "kernel BUG at mm/mmap.c:738!"	  
  - 调查
    vmcore中显示：
<pre><code>RELEASE: <font color=red>3.10.0-693.5.2.el7.x86_64</font>                 <--- 
VERSION: #1 SMP Fri Oct 13 10:46:25 EDT 2017
MACHINE: x86_64  (1995 Mhz)
MEMORY: 511.8 GB
PANIC: <font color=red>"kernel BUG at mm/mmap.c:738!"</font>              <---</code></pre>
  - 修复
      - 对于`RHEL 7`将内核版本升级至`kernel-3.10.0-957.el7`及以上
	  - 对于`RHEL 7.5`将内核版本升级至`kernel-3.10.0-862.20.2.el7`及以上
	  - 对于`RHEL 7.4`将内核版本升级至`kernel-3.10.0-693.46.1.el7`及以上
  - 参考文档
    [<font color=red>**System unexpectedly reboots or panics with "kernel BUG at mm/mmap.c:738"**</font>](https://access.redhat.com/solutions/3392791)  
---
- 5. 系统重启于`fsnotify()`
  - 调查
    vmcore显示：
<html><pre><code>[exception RIP: <font color=red>fsnotify</font>+0x24b]
... ...
#4 [ffff96b6b5a13df0] <font color=red>fsnotify</font> at ffffffffb306186b</pre></code></html>
  - 修复
    - 对于`RHEL 7.6`将内核版本升级至`kernel-3.10.0-957.el7`及以上  
	- 对于`RHEL 7.5`将内核版本升级至`kernel-3.10.0-862.9.1.el7`及以上  
	- 可以将内核版本降低到`kernel-3.10.0-862`
  - 参考文档
    [<font color=red>**Soft lockups occur in fsnotify() after updating to RHEL 7.5**</font>](https://access.redhat.com/solutions/3441101)
---
- 6. 系统由于`Bad page state/map or Bad rss-counter state`导致重启
  - 调查
    vmcore显示：
<html><pre><code>crash> log|grep -E 'BUG: '
[30921.876412] BUG: <font color=red>Bad page state</font> in process oraagent.bin  pfn:325ab7c
[30921.879416] BUG: <font color=red>Bad rss-counter state</font> mm:ffff9a3cfca2c4c0 idx:1 val:1</code></pre></html>
  - 修复
    - 对于`RHEL 7`将内核版本升级至最新版本  
	- 进行内存硬件诊断
    - 可能由于第三方模块引起
	- 与<font color=blue>7</font> 问题类似
  - 参考文档
    [<font color=red>**Bad page state/map or Bad rss-counter state followed by kernel crash or soft lockup Red Hat Enterprise Linux**</font>](https://access.redhat.com/solutions/3605891)
---
- 7. 系统出现`'BUG: Bad page state in process'`并崩溃在`down_read_trylock`之前
  - 调查
    检查vmcore
<html><pre><code>[589570.661039] BUG: Bad page state in process cas  pfn:2207dff
[625759.760775] BUG: Bad page map in process python  pte:80003fddf8200904 pmd:205c25d067
[625759.880053] BUG: Bad page state in process python  pfn:2207dff
[625760.834004] WARNING: CPU: 19 PID: 76354 at lib/list_debug.c:62 __list_del_entry+0x82/0xd0
... ...
[531007.980839] BUG: unable to handle kernel NULL pointer dereference at 0000000000000008 [531008.020282] IP: [<ffffffffa66c0519>] <font color=red>down_read_trylock</font>+0x9/0x50</html></code></pre>
  - 修复
    - 对于`RHEL 7`将内核版本升级至`kernel-3.10.0-1062.el7`及以上  
	- 对于`RHEL 7.6`将内核版本升级至`kernel-3.10.0-957.27.2.el7`及以上  
	- 对于`RHEL 7.5`将内核版本升级至`kernel-3.10.0-862.37.1.el7`及以上  
	- 对于`RHEL 7.4`将内核版本升级至`kernel-3.10.0-693.55.1.el7`及以上  
	- 对于`RHEL 7.3`将内核版本升级至`kernel-3.10.0-514.66.2.el7`及以上  
	- 对于`RHEL 7.2`将内核版本升级至`kernel-3.10.0-327.79.2.el7`及以上  
  - 参考文档
    [<font color=red>**'BUG: Bad page state in process' event is logged just before crash in down_read_trylock, Red Hat Enterprise Linux 7**</font>](https://access.redhat.com/solutions/4094221)
---
- 8. 系统`kmem_cache_alloc` 从 `mempool_alloc_slab` 引用无效内存地址引起crash
  - 调查
    - 检查vmcore
<html><pre><cod>[14787601.353261] RIP: 0010:[]  [] <font color=red>kmem_cache_alloc</font>+0x75/0x1d0
... ...
[14787601.353705]  [] <font color=red>mempool_alloc_slab</font>+0x15/0x20
[14787601.353728]  [] mempool_alloc+0x69/0x170</code></pre></html>
    - 这是由于内核中的驱动模块（internal模块以及第三方模块）的"use-after-free"或者"double-free"的bug所导致
  - 修复
    - 对于`RHEL 7`将内核版本升级至最新
    - 继续调查这个问题，内核启动参数中添加"slub_debug=FZP"，开启slub debug功能。注意，这会造成主机生产减缓
    - 内核启动参数中添加"slub_debug=F"作为一个绕行方案，这样当再次出现freelist指针损坏的问题时，系统不会panic重启
  - 参考文档
    [<font color=red>**Kernel panic due to "kmem_cache_alloc+117 from mempool_alloc_slab" on RHEL 7**</font>](https://access.redhat.com/solutions/2149041)
---
- 9. 内核由于BUG`kernel BUG at drivers/ata/libata-sff.c:1350!`引起重启
  - 调查
    检查vmcore
<html><pre><code>[5252829.523223] <font color=red>kernel BUG at drivers/ata/libata-sff.c:1350!</font>
... ...
[5252829.524184] task: ffff9b46dfe8c100 ti: ffff9b4501e58000 task.ti: ffff9b4501e58000
[5252829.524232] RIP: 0010:[<ffffffffc03760cc>]  [<ffffffffc03760cc>] <font color=red>ata_sff_pio_task</font>+0x1cc/0x1d0 <font color=red>[libata]</font></code></pre></html>
  - 修复
    - 缓解办法：增加底层设备的默认命令超时时间
      ```
	  # echo "120" > /sys/block/xxx/device/timeout
	  ```
  - 参考文档
    [<font color=red>**kernel panic with error 'kernel BUG at drivers/ata/libata-sff.c:1350!' in virtual machine environment**</font>](https://access.redhat.com/solutions/5364781)
---
- 10. 由于模块`fileaccess_mod`分配slab 内存时在`kmem_cache_alloc`内核panic
  - 调查
    检查vmcore
<html><pre><code>
# crash> bt
    [exception RIP: <font color=red>kmem_cache_alloc</font>+118]
... ...
#12 [ffff8817d196be80] <font color=red>scanCache</font>_init at ffffffffa044a4c0 [<font color=red>fileaccess_mod</font>]
#13 [ffff8817d196bea0] hook_init at ffffffffa0444b83 [<font color=red>fileaccess_mod</font>]
#14 [ffff8817d196bec0] init_module at ffffffffa005a072 [<font color=red>fileaccess_mod</font>]
# crash> mod -t
NAME                       TAINTS
<font color=red>fileaccess_mod_1005031646  OE</font></html></pre></code>
  - 修复
    - 对于`fileaccess_mod`模块为`McAfee`模块，联系厂商调查
  - 参考文档
    [<font color=red>**Server panics at kmem_cache_alloc while allocating slab memory due to unsigned kernel module fileaccess_mod**</font>](https://access.redhat.com/solutions/3380141)
---
- 11. 系统由于`not syncing: Fatal Machine check or Machine Check Exception` 重启
  - 调查
    检查vmcore
<html><pre><code>Hardware event. This is not a software error.
Corrected error
Transaction: Memory scrubbing error
Memory ECC error occurred during scrub
Memory corrected error count (CORE_ERR_CNT): 1
Memory DIMM ID of error: 1
Memory channel ID of error: 2
Hardware event. This is not a software error.</html></code></pre>
  - 修复
    - 联系硬件厂商进行调查
	  ```
	  内存 DIMM 故障。
      内存控制器故障（通常是板载的）。
      主板内存线故障。
      BIOS 故障。
      过热系统。
      RAM 潜在连接故障（来自用户的静电放电）。
      电源问题或短路。
	  ```
	- 检查系统BIOS，升级至最新
	- 参考文档进行[**<font color=blue>内存检测</font>**](https://access.redhat.com/solutions/15693)
  - 参考文档
    [<font color=red>**"Kernel panic - not syncing: Fatal Machine check or Machine Check Exception (MCE)" in /var/log/messages**</font>](https://access.redhat.com/solutions/18723)
---
- 12. 系统由于硬件CPU运行问题重启
  - 调查
    检查vmcore
<html><pre><code> crash> bt
    [exception RIP: <font color=red>skb_release_head_state</font>+<font color=blue>0x18</font>]
对该函数进行反汇编<font color=red>skb_release_head_state</font>，并无<font color=blue>0x18</font>偏移量
crash> dis -rl ffffffffafc3c178
/usr/src/debug/kernel-3.10.0-1127.el7/linux-3.10.0-1127.el7.x86_64/include/net/dst.h: 300
... ...
0xffffffffafc3c17a <skb_release_head_state+0x1a>:	je     0xffffffffafc3c208 <skb_release_head_state+0xa8></html></code></pre>
  - 修复
    - 该问题只可能由于CPU硬件运行故障或计算错误引起，硬件厂商对CPU深入检查
  - 参考文档
    [<font color=red>**Intel microcode for Skylake/Cascade Lake CPUs can cause stability issues**</font>](https://access.redhat.com/solutions/5639591)
---
- 13. 内核BUG`"kernel BUG at mm/page_alloc.c:1656!"`
  - 调查
    检查vmcore
<html><pre><code>[310377.349350] <font color=red>kernel BUG at mm/page_alloc.c:1656</font>
... ...
[310377.349406] RIP: 0010:[<ffffffff875c458e>]  [<ffffffff875c458e>] <font color=red>move_freepages</font>+0x15e/0x160</html></code></pre>
  - 修复
    - 对于`RHEL 7.9`将内核版本升级至`kernel-3.10.0-1160.11.1.el7`及以上  
    - 对于`RHEL 7.7`将内核版本升级至`kernel-3.10.0-1062.43.1.el7`及以上  
  - 参考文档
    [<font color=red>**RHEL7: Kernel crashes in move_freepages() with a message "kernel BUG at mm/page_alloc.c:1656!"**</font>](https://access.redhat.com/solutions/5486951)	
---
- 14. 内核BUG`kernel BUG at mm/page_alloc.c:1389!`
  - 调查
    检查vmcore
<html><pre><code>[23774.452193] <font color=red>kernel BUG at mm/page_alloc.c:1389!</font></html></code></pre>
  - 修复
    - 对于`RHEL 7`将内核版本升级至`kernel-3.10.0-862.el7`及以上  
	- 对于`RHEL 7.4`将内核版本升级至`kernel-3.10.0-693.33.1.el7`及以上  
  - 参考文档
    [<font color=red>**RHEL 7.4 server panics with message "kernel BUG at mm/page_alloc.c:1389!"**</font>](https://access.redhat.com/solutions/3208581)
---
- 15. 内核中出现以下告警`WARNING: at fs/xfs/xfs_aops.c:1244 xfs_vm_releasepage`	
  - 调查
    检查系统messages
<html><pre><code><font color=red>WARNING: at fs/xfs/xfs_aops.c:1244 xfs_vm_releasepage</font>+0xcb/0x100 [xfs]()
... ...
[ffffffffa045556b] <font color=red>xfs_vm_releasepage</font>+0xcb/0x100 [xfs]  <<---</html></code></pre>
  - 修复
    - 对于`RHEL 7`将内核版本升级至`kernel-3.10.0-693.el7`及以上  
  - 参考文档
    [<font color=red>**RHEL7: WARNING: at fs/xfs/xfs_aops.c:1244 xfs_vm_releasepage in logs**</font>](https://access.redhat.com/solutions/2893711)
---
- 16. 系统崩溃出现以下日志`Timeout: Not all CPUs entered broadcast exception handler`	
  - 调查
    检查vmcore
<html><pre><code># cat vmcore-dmesg.txt
[60187489.667365] <font color=red>Kernel panic - not syncing: Timeout: Not all CPUs entered broadcast exception handler</font></html></code></pre>
  - 修复
    - 如果是物理机，请检查硬件尤其是CPU
    - 如果是虚拟机，请检查`Hypervisor`	
  - 参考文档
    [<font color=red>**System keeps crashing with message "Timeout: Not all CPUs entered broadcast exception handler"**</font>](https://access.redhat.com/solutions/3144261)
---
- 17. 系统hung死前日志记录`e1000e 0000:00:1f.6 enp0s31f6: Detected Hardware Unit Hang:`
  - 调查
    检查系统message日志
<html><pre><code>[963569.248045] e1000e 0000:00:1f.6 enp0s31f6: <font color=red>Detected Hardware Unit Hang</font>:
... ...
[963691.387160] e1000e 0000:00:1f.6 enp0s31f6: <font color=red>Detected Hardware Unit Hang:</font></html></code></pre>
  - 修复
    - 对于`RHEL 7.8`将内核版本升级至`kernel-3.10.0-1127.el7`及以上  
	- 强制 PCIe ASPM 进入 Performance 模式可以避免这个问题。在内核启动参数中添加以下配置
	```
	pcie_aspm=force pcie_aspm.policy=performance
	```
  - 参考文档
    [<font color=red>**log message 'e1000e Detected Hardware Unit Hang' before system hangs**</font>](https://access.redhat.com/solutions/4297061)
---
- 18. 系统重现出现日志`Power key pressed`
  - 调查
    检查系统message日志
<html><pre><code>localhost systemd-logind: <font color=red>Power key pressed</font>.
... ...
localhost systemd-logind: Powering Off...
... ...
localhost systemd-logind: System is powering down.</html></code></pre>
  - 修复
    - 检查系统是否被故意关闭
	- 检查硬件控制台是否有关闭、报错相关日志
  - 参考文档
    [<font color=red>**Server shutdown/reboot with "Power key pressed" messages**</font>](https://access.redhat.com/solutions/2349931)
---
- 19. 系统hung死`netlink_compare`
  - 调查
    检查vmcore
<html><pre><code># crash> bt
... ...
[exception RIP: <font color=red>netlink_compare+11</font>]
RIP: ffffffff8155654b  RSP: ffff88070d253b90  RFLAGS: 00010246</html></code></pre>
  - 修复
    - 对于`RHEL 7`将内核版本升级至`kernel-3.10.0-327.54.1.el7`及以上  
  - 参考文档
    [<font color=red>**Kernel panic or soft lockup or hung in netlink_compare in Red Hat Enterprise Linux 7**</font>](https://access.redhat.com/solutions/2647381)	
---
- 20. 内核位于`inet_twsk_schedule()`崩溃
  - 调查
    检查vmcore
<html><pre><code># crash> bt
... ...
[exception RIP: <font color=red>inet_twsk_schedule</font>+177]</html></code></pre>
  - 修复
    - 对于`RHEL 7.4`将内核版本升级至`kernel-3.10.0-693.el7`及以上  
	- 对于`RHEL 6.10`将内核版本升级至`kernel-2.6.32-754.el6`及以上  
	- 缓解办法，使用`sysctl`添加以下内核参数
	  ```
	  net.ipv4.tcp_tw_recycle = 0
	  ```
  - 参考文档
    [<font color=red>**RHEL: Kernel crash in inet_twsk_schedule()**</font>](https://access.redhat.com/solutions/1390823)
---
- 21. 系统由于第三方内核模块重启
  - 调查
    检查vmcore
<html><pre><code>crash> bt
    [exception RIP: <font color=red>pal_lru_find_dirty_sippet</font>+0x5a]
crash> sym <font color=red>pal_lru_find_dirty_sippet</font>
ffffffffc0b2e1a0 (t) <font color=red>pal_lru_find_dirty_sippet</font> [pal] 
crash> mod -t
NAME             TAINTS
... ...
<font color=red>pal</font>              POE    <==</html></code></pre>
  - 修复
    - 联系第三方模块供应商进行排查
--- 
- 22. 关于系统进入sleep状态
  - 需求
    - "Freeze / Low-Power Idle" 状态 "Suspend-to-Idle" ("s2idle")即相同的 "ACPI state: S0"
    - 可以使用以下方式进入
	```
    # echo "freeze" > /sys/power/state
	```
  - 参考文档
    https://access.redhat.com/labs/rhcb/RHEL-7.8/kernel-3.10.0-1127.19.1.el7/source/blob/Documentation/power/states.txt
---
- 23. 内核记录`power_meter ACPI000D:00: Found ACPI power meter`
  - 调查
    检查vmcore
<html><pre><code>kernel: <font color=red>power_meter ACPI000D:00: Found ACPI power meter.</font></html></code></pre>
  - 修复
    - 由于内核中的 acpi 驱动程序已成功检测到 ACPI 功率计设备，所以忽略此日志
  - 参考文档
    [<font color=red>**kernel log shows "power_meter ACPI000D:00: Found ACPI power meter."**</font>](https://access.redhat.com/solutions/3339131)
---	
- 24. 系统日志中记录`Abort command Issued`
  - 调查
    检查系统message日志
<html><pre><code>kernel: qla2xxx 0000:46:00.0: scsi(1:0:105): <font color=red>Abort command issued</font> -- 1 3e7bbc46 2002.</html></code></pre>
  - 修复
    - 该报错为SAN环境返回报错
    - 检查SAN环境中设备 FC 交换机、FC 布线、分区或存储阵列是否存在任何问题
	- 存储供应商查看交换机日志，以验证 FC 交换机日志中是否有任何错误计数器、CRC 错误
  - 参考文档
    [<font color=red>**"Abort command Issued" messages appear in /var/log/messages file**</font>](https://access.redhat.com/solutions/27624)
---
- 25. 内核由于`not syncing: GAB: Port d halting system due to network failure at [XX:XXXX]`引起恐慌
  - 调查
    检查vmcore
<html><pre><code><font color=red>LLT</font> INFO V-14-1-10205 link 0 (eth1) node 0 in trouble LLT INFO V-14-1-10205 link 1 (eth3) node 0 in trouble LLT INFO V-14-1-10032 link 0 (eth1) node 0 inactive 8 sec (696686965) LLT INFO V-14-1-10032 link 1 (eth3) node 0 inactive 8 sec (696614993) LLT INFO V-14-1-10032 link 0 (eth1) node 0 inactive 9 sec (696686965) LLT INFO V-14-1-10032 link 1 (eth3) node 0 inactive 9 sec (696614993) LLT INFO V-14-1-10032 link 0 (eth1) node 0 inactive 10 sec (696686965) LLT INFO V-14-1-10032 link 1 (eth3) node 0 inactive 10 sec (696614993) LLT INFO V-14-1-10032 link 0 (eth1) node 0 inactive 11 sec (696686965) LLT INFO V-14-1-10032 link 1 (eth3) node 0 inactive 11 sec (696614993) LLT INFO V-14-1-10032 link 0 (eth1) node 0 inactive 12 sec (696686965) LLT INFO V-14-1-10032 link 1 (eth3) node 0 inactive 12 sec (696614993) LLT INFO V-14-1-10032 link 0 (eth1) node 0 inactive 13 sec (696686965) LLT INFO V-14-1-10510 sent hbreq (NULL) on link 0 (eth1) node 0. 4 more to go.
<font color=red>LLT</font> INFO V-14-1-10510 sent hbreq (NULL) on link 0 (eth1) node 0. 3 more to go.
... ...
GAB INFO V-15-1-20040 Port h[GAB_USER_CLIENT (refcount 0)] gen   263813    visible 0
GAB INFO V-15-1-20033 Port d[GAB_LEGACY_CLIENT (refcount 0)] nid 0     [3:0:0:0]    [3:0:0:0]    [3:0:0:0]    [1:0:0:0]    [0:0:0:0]  22   26380f   20 9
GAB INFO V-15-1-20033 Port d[GAB_LEGACY_CLIENT (refcount 0)] nid 1     [3:0:0:0]    [3:0:0:0]    [3:0:0:0]    [2:0:0:0]    [0:0:0:0]  22   26380f   20 37
Kernel panic - <font color=red>not syncing: GAB: Port d halting system due to network failure at [14:2166] [..]</font></html></code></pre>
  - 修复
    - 验证系统上的所有网络配置。
	- 检查任何有故障的网络接口卡 (NIC)、交换机端口或电缆
	- 联系VCS` Veritas Cluster Server `供应商进行进一步检查
  - 参考文档
    [<font color=red>**Kernel panic - not syncing: GAB: Port d halting system due to network failure at [XX:XXXX]**</font>](https://access.redhat.com/solutions/976043)	
---
- 26. 虚拟机汇报软锁定`soft lockup`
  - 调查
    检查系统的message日志
<html><pre><code>有以下记录
1-> BUG: <font color=red>soft lockup</font> - CPU#0 stuck for 64s! [proceessname:15789]
2-> kernel: NMI watchdog: BUG: <font color=red>soft lockup</font> - CPU#6 stuck for 25s! [ksoftirqd/6:38]
3-> <font color=red>hrtimer: interrupt took NUMBER ns</font></html></code></pre>
  - 修复
    - 检查主机或虚拟机管理程序是否有工作过载的情况
	- 为避免重启，将`kernel.softlockup_panic`值设置为`0`
	- 增加软锁定报告时间的阈值，上限`60S`
	  <html><pre><code>kernel.watchdog_thresh
	  旧版RHEL为 kernel.softlockup_thresh</code></pre></html>
  - 参考文档
    [<font color=red>**Virtual machine reports a "BUG: soft lockup" (or multiple at the same time)**</font>](https://access.redhat.com/solutions/1503333)
---
- 27. 升级RHEL7.6后，QLogic FC HBA 的系统引导时崩溃，通过 QLogic 提供的LUN无法被识别
  - 调查
    检查系统启动日志
<html><pre><code>kernel: scsi host15: qla2xxx
kernel: qla2xxx [0000:2f:00.0]-00fb:15: QLogic QLE2692 - QLogic 16Gb FC Dual-port HBA.
kernel: qla2xxx [0000:2f:00.0]-00fc:15: ISP2261: PCIe (8.0GT/s x8) @ 0000:2f:00.0 hdma+ host#=15  fw=8.08.05 (d0d5).
...
kernel: <font color=red>TECH PREVIEW</font>: NVMe over FC may not be fully supported.</html></code></pre>
  - 修复
    - 对于`RHEL 7.6`将内核版本升级至`RHEL 7.8`及以上  
	- 升级HBA固件到`v8.08.204`或更高版本
	- 有三种缓解办法
	  - 1. 使用RHEL 7.5 的内核启动
	  - 2. 将 QLogic HBA 卡的固件降级到以前的版本
	  - 3. 修改内核模块配置
	    - 3.1 为内核模块添加以下参数
		  ```
		  $ cat > /etc/modprobe.d/qla2xxx.conf
          options qla2xxx ql2xnvmeenable=0
		  ```
		- 3.2 重新生成initramfs文件
		- 3.3 使用新创建的initramfs启动系统
  - 参考文档
    [<font color=red>**After upgrading to RHEL 7.6, systems with QLogic FC HBA crash on boot, OR LUNs presented to QLogic are only partially discovered or not at all**</font>](https://access.redhat.com/solutions/3776921)
---
- 28. 系统自动重启，日志中记录`systemd: Received SIGINT`
  - 调查
    检查系统message日志
<html><pre><code>systemd: <font color=red>Received SIGINT</font></html></code></pre>
  - 修复
    - 该`SIGINT`从用户或软件发出
	- 无法从消息获知谁发出的
	- 通过禁用[**<font color=blue>Ctrl-Alt-Del.target</font>**](https://access.redhat.com/solutions/1123873)来防止将来发生这些事件
  - 参考文档
    [<font color=red>**Why does server reboot with "systemd: Received SIGINT" in /var/log/messages?**</font>](https://access.redhat.com/solutions/3353861)	
---
- 29. `vim`崩溃后`ABRT`导致内核泄露
  - 调查
    检查系统日志
<html><pre><code>abrt-hook-ccpp: Process <PID> (vim) of user <UID> killed by SIGSEGV - dumping core</html></code></pre>
  - 修复
    - 将`VIM`版本升级至`vim-7.4.160-5.el`及以上
    - 将`ABRT`版本升级至`abrt-2.1.11-55.el7`及以上	
  - 参考文档
    [<font color=red>**ABRT causes out-of-memory condition after vim crash**</font>](https://access.redhat.com/solutions/4082081)
---
- 30. 内核BUG`mm/huge_memory.c:1989!`
  - 调查
    检查vmcore
<html><pre><code>[22320502.735528] mapcount 0 page_mapcount 1
[22320502.735572] ------------[ cut here ]------------
[22320502.735587] <font color=red>kernel BUG at mm/huge_memory.c:1989!</font>
... ...
[22320502.740331] RIP  [<ffffffff811ed5c8>] <font color=red>__split_huge_page</font>+0x738/0x750
</html></code></pre>
  - 修复
    - 对于`RHEL 7`将内核版本升级至`kernel-3.10.0-1127.el7`及以上  
	- 缓解办法：关闭[**<font color=blue>Huge Pages</font>**](https://access.redhat.com/solutions/1320153)，添加以下内容到grub启动内核参数中并重建`grub.cfg`文件，重启系统
	  ```
	  GRUB_CMDLINE_LINUX="... transparent_hugepage=never"
	  ```
  - 参考文档
    [<font color=red>**System crash due to kernel BUG at mm/huge_memory.c:1989!**</font>](https://access.redhat.com/solutions/5227771)
---
- 31. 系统日志中记录`kernel: mm/memory.c:120: bad pmd`	
  - 调查
    检查系统日志
<html><pre><code><font color=red>kernel: mm/memory.c:120: bad pmd</font> ffff81034d728070(00000013e64000e7</html></code></pre>
  - 修复
    - 在当前系统上运行[**<font color=blue>故障内存检查</font>**](https://access.redhat.com/solutions/15693) 
	- 如果硬件出现问题，有硬件更换
  - 参考文档
    [<font color=red>**'kernel: mm/memory.c:120: bad pmd' gets reported in the system logs**</font>](https://access.redhat.com/solutions/41282)
---
- 32. 系统日志中记录`kernel: WARNING: CPU: 0 PID: XXX at ./arch/x86/include/asm/thread_info.h:301 sigsuspend+0x6d/0x70`
  - 调查
    检查系统message日志
<html><pre><code>kernel: WARNING: CPU: 2 PID: 15713 at <font color=red>./arch/x86/include/asm/thread_info.h:301 sigsuspend+0x6d/0x70.</font></html></code></pre>
  - 修复
    - 对于`RHEL 7`将内核版本升级至`kernel-3.10.0-1160.11.1.el7`及以上  
  - 参考文档
    [<font color=red>**Getting warnings in /var/log/messages: kernel: WARNING: CPU: 0 PID: XXX at ./arch/x86/include/asm/thread_info.h:301 sigsuspend+0x6d/0x70.**</font>](https://access.redhat.com/solutions/5666021)	
---
- 33. 内核由于驱动导致panic`intel_pstate_timer_func()`	
  - 调查
    检查vmcore
<html><pre><code>[4067605.161646] <font color=red>divide error: 0000 [#1] SMP</font> 
... ...
[4067605.162343] RIP: 0010:[<ffffffff814a9279>]  [<ffffffff814a9279>] <font color=red>intel_pstate_timer_func</font>+0x179/0x3d0</html></code></pre>
  - 修复
    - 对于`RHEL 7.1`将内核版本升级至`kernel-3.10.0-229.20.1.el7`及以上  
    - 对于`RHEL 7.2`将内核版本升级至`kernel-3.10.0-327.el7`及以上 
    - 对于`RHEL 6`将内核版本升级至`kernel-2.6.32-642.el6`及以上
  - 参考文档
    [<font color=red>**Divide by zero error in intel_pstate_timer_func() [ inline s64 div_s64_rem() ]**</font>](https://access.redhat.com/solutions/1471663)
---
- 34. 系统因`hpwdt`崩溃
  - 调查
    检查vmcore
<html><pre><code>crash> bt
PID: 0      TASK: ffff95cbb4d7d140  CPU: 7   COMMAND: "swapper/7"
... ...
 #4 [ffff95d31fcc8e40] hpwdt_pretimeout at ffffffffc034d594 <font color=red>[hpwdt]</font></html></code></pre>
  - 修复
    - 硬件问题，联系硬件厂商调查  
	- HPE iLO 4 Firmware Version 低于1.51，存在BUG会导致这个问题
	  https://support.hpe.com/hpesc/public/docDisplay?docId=emr_na-c04332584
  - 参考文档
    [<font color=red>**Why does the system crash with HP NMI Watchdog [hpwdt]?**</font>](https://access.redhat.com/solutions/707563)
---
- 35. `polkitd`，`dbus-daemon`，`accounts-daemon`进程占用CPU高
  - 调查
    检查系统ps，对应进程的负载
  - 修复
    - 将`accountsservice`软件包版本升级至`accountsservice-0.6.50-5.el7`及以上  
  - 参考文档
    [<font color=red>**polkitd and accounts-daemon process consuming a large amount of CPU resources on Red Hat Enterprise Linux 7**</font>](https://access.redhat.com/solutions/3819902)	
---
- 36. 内核位于`prb_retire_rx_blk_timer_expired`出现软锁导致崩溃
  - 调查
    检查vmcore
<html><pre><code>[14797.183134] Call Trace:
[14797.183136]  <IRQ> 
[14797.183141]  [<ffffffffba0af27c>] mod_timer+0x10c/0x230
[14797.183146]  [<ffffffffba755670>] ? prb_retire_current_block+0x110/0x110
[14797.183148]  [<ffffffffba755706>] <font color=red>prb_retire_rx_blk_timer_expired</font>+0x96/0x140      <--</html></code></pre>
  - 修复
    - 对于`RHEL 7`将内核版本升级至`kernel-3.10.0-1127.el7`及以上  
  - 参考文档
    [<font color=red>**kernal panic - NMI watchdog: BUG: soft lockup / prb_retire_rx_blk_timer_expired**</font>](https://access.redhat.com/solutions/6751311)
---
- 37. 内核由于`crash_save_cpu+0x92/0x1d0`崩溃
  - 调查
    检查vmcore
<html><pre><code>dmesg logs :
[121730736.448991] BUG: unable to handle kernel NULL pointer dereference at           (null)
[121730736.448996] IP: [<ffffffff81106992>] <font color=red>crash_save_cpu+0x92/0x1d0</font>
crash> bt
[exception RIP: <font color=red>intel_idle+213</font>]
NMI exception stack
#12 [ffff8b00fb16fe20] <font color=red>intel_idle</font> at ffffffff816ab6a5</html></code></pre>
  - 修复
    - 缓解办法如下
<html><pre><code>1 设置 numa=off
2 关闭 Transparent Hugepage
3 移除 3rd party modules
4 升级内核至 kernel-3.10.0-862.el7 或更高
</html></code></pre>	  
  - 参考文档
    [<font color=red>**Kernel panic with RIP crash_save_cpu+0x92/0x1d0**</font>](https://access.redhat.com/solutions/3598891)
---
- 38. 内核BUG`kernel BUG at block/blk-core.c:2500!`
  - 调查
    检查vmcore
<html><pre><code>[536700.120045] <font color=red>kernel BUG at block/blk-core.c:2500!</font> </html></code></pre>
  - 修复
    - 对于`RHEL 7`将内核版本升级至`kernel-3.10.0-229.el7.x86_64`及以上  
    - 对于`RHEL 7.0`将内核版本升级至`kernel-3.10.0-123.13.1.el7.x86_64`及以上 
  - 参考文档
    [<font color=red>**[RHEL7] kernel BUG at block/blk-core.c:2500!**</font>](https://access.redhat.com/solutions/1289343)	
---	
- 39. 内核位于`not syncing: Timeout: Not all CPUs entered broadcast exception handler`崩溃
  - 调查
    检查vmcore
<html><pre><code>PANIC: <font color=red>"Kernel panic - not syncing: Timeout: Not all CPUs entered broadcast exception handler"</font>
crash> bt
... ...
 #3 [ffff9fc6fe8cbdc8] mce_panic at ffffffffad248c45        <<< 
... ...
 #7 [ffff9fc6fe8cbf50] machine_check at ffffffffad96b7ae    <<< machine_check() 在正常情况下不会发生。当它被调用时，意味着硬件级别有问题。
    [exception RIP: cpu_idle_poll+73]</html></code></pre>
  - 修复
    - 联系硬件供应商检查CPU
	- 如果是虚拟机，联系虚拟化供应商检查Hypervisor
  - 参考文档
    [<font color=red>**System keeps crashing with message "Timeout: Not all CPUs entered broadcast exception handler"**</font>](https://access.redhat.com/solutions/3144261)
---
- 40. 系统日志中记录`rsyslogd was HUPed`
  - 调查
    检查vmcore
<html><pre><code>grep -i "rsyslogd was HUPed" /var/log/messages
rsyslogd: [origin software="rsyslogd" swVersion="5.8.10" x-pid="1457" x-info="http://www.rsyslog.com"] <font color=red>rsyslogd was HUPed</font>
rsyslogd: [origin software="rsyslogd" swVersion="5.8.10" x-pid="22972" x-info="http://www.rsyslog.com"] <font color=red>rsyslogd was HUPed</font>
rsyslogd: [origin software="rsyslogd" swVersion="5.8.10" x-pid="22972" x-info="http://www.rsyslog.com"] <font color=red>rsyslogd was HUPed</font></html></code></pre>
  - 修复
    - `HUP`是`logrotate`服务发送的信号
    - 向`syslogd`进程发出`HUP`信号以告知写入新文件，因为旧日志文件现在已被重命名。这是预期的行为。
  - 参考文档
    [<font color=red>**Getting messages 'rsyslogd was HUPed' in /var/log messages**</font>](https://access.redhat.com/solutions/1345383)
---
- 41. 系统日志中记录`'qla2xxx: Invalid completion handle`	
  - 调查
    检查系统messages日志
<html><pre><code>qla2xxx [0000:2f:00.0]-5032:15: Invalid completion handle (5df) -- timed-out.</html></code></pre>
  - 修复
    - 将`HPE`的固件升级到最新
	- 检查硬件
  - 参考文档
    [<font color=red>**System log shows error - 'qla2xxx: Invalid completion handle' , F/W Error, and/or heartbeat failure events**</font>](https://access.redhat.com/solutions/4709121)
---
- 42. 内核恐慌`not syncing: Machine check`	
  - 调查
    检查vmcore
<html><pre><code>[Hardware Error]: CPU 13: Machine Check Exception: 4 Bank 5: be00000000800400
... ...
[Hardware Error]: Machine check: Processor context corrupt</html></code></pre>
  - 修复
    - 内核打印的信息来自硬件，提供给硬件支持人员进行分析 
	- 联系硬件供应商检查硬件
  - 参考文档
    [<font color=red>**RHEL server crashed with error " Kernel panic- not syncing: Machine check "**</font>](https://access.redhat.com/solutions/188383)	
---
- 43. 由于第三方内核模块`vmsecmod`出现死锁
//稍后解析
  - 参考文档
    [<font color=red>**The 3rd-party module "vmsecmod" caused a deadlock scenario between mm_update_next_owner() and taskexit_event_handler()**</font>](https://access.redhat.com/solutions/6766801)	

---
- 44. 内核日志`SLUB: Unable to allocate memory on node -1 `
  - 调查
    检查系统messages日志
<html><pre><code>SLUB: Unable to allocate memory on node -1 (gfp=0x200020)
SLUB: Unable to allocate memory on node -1 (gfp=0x20)
SLUB: Unable to allocate memory on node -1 (gfp=0x20)
</html></code></pre>
  - 修复
    - 对于`RHEL 7`将内核版本升级至`kernel-3.10.0-1062.4.1.el7`及以上  
	- 缓解办法：完全禁用内存记账，添加如下`grub`内核参数，重建`grub`后并启动
	  ```
	  cgroup.memory=nokmem
	  ```
  - 参考文档
    [<font color=red>**SLUB: Unable to allocate memory on node -1 (gfp=0x20)**</font>](https://access.redhat.com/solutions/4088471)
---
- 45. 使用`GPFS`的系统7.3频繁重启
  - 调查
    检查vmcore
<html><pre><code>crash> ps -p 6318 
PID: 0      TASK: c000000001358180  CPU: 0   COMMAND: "swapper/0"
 PID: 1      TASK: c000001ef9d00000  CPU: 31  COMMAND: "systemd"
  PID: 41045  TASK: c000000e9e8d5300  CPU: 10  COMMAND: "runmmfs"
   PID: 68294  TASK: c000000e9f766670  CPU: 12  COMMAND: "mmfsd"        <
    PID: 6296   TASK: c000000f4da56830  CPU: 66  COMMAND: "mmcommon"
     PID: 6317   TASK: c000001ee9befac0  CPU: 41  COMMAND: "umount"
      PID: 6318   TASK: c000001dc63779e0  CPU: 57  COMMAND: "umount.nfs"     < 
</html></code></pre>
  - 修复
    - 联系`GPFS`供应商进行进一步分析，`gpfs​​`模块`mmfs26`和 `mmfslinux` 涉及崩溃上下文
  - 参考文档
    [<font color=red>**GPFS (Spectrum Scale) 7.3 ppc64 lpars are crashing**</font>](https://access.redhat.com/solutions/3321231)
**<u><font color=red>仅用作经验记录</font></u>**