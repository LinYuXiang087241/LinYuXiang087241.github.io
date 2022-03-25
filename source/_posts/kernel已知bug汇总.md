---
title: kernel已知bug汇总
date: 2022-03-24 21:27:07
tags: 
  - kernel
categories: 
  - bug
---

# RHEL 7
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