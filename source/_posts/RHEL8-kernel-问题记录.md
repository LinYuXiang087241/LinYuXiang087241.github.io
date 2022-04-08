---
title: RHEL8-kernel-问题记录
date: 2022-03-26 23:17:26
tags: 
  - kernel
  - RHEL8
categories: 
  - bug
---

# RHEL 8
- 1. kernel崩溃位于`n_tty_receive_buf_common()`
  - 调查
    检查vmcore
<html><pre><code>crash> bt
... ...
[exception RIP: <font color=red>n_tty_receive_buf_common</font>+0x61]</html></code></pre>
  - 修复
    - 对于`RHEL 8`将内核版本升级至`kernel-4.18.0-348.el8`及以上  
  - 参考文档
    [<font color=red>**Kernel panics in the n_tty_receive_buf_common() routine**</font>](https://access.redhat.com/solutions/5697911)
---
<!-- more -->

- 2. kernel崩溃由于`list_del`位于`LIST_POISON2`腐败
  - 调查
    检查vmcore
<html><pre><code>[570928.662632] <font color=red>list_del corruption</font>, ffff8b3c3b76b048->prev is <font color=red>LIST_POISON2</font> (dead000000000200)
[570928.662739] ------------[ cut here ]------------
[570928.662740] kernel BUG at <font color=red>lib/list_debug.c</font>:50!
</html></code></pre>
  - 修复
    - 对于`RHEL 8.4`将内核版本升级至`kernel-4.18.0-305.12.1.el8_4`及以上  
	- 缓解办法也可以在`grub`中添加以下参数
	  ```
	  cgroup_disable=memory
	  ```
  - 参考文档
    [<font color=red>**RHEL 8.4: kernel crashed due to list_del corruption with LIST_POISON2**</font>](https://access.redhat.com/solutions/6094611)
---
- 3. 内核位于`n_tty_set_termios`恐慌
  - 调查
    检查vmcore
<html><pre><code>crash> bt
PID: 1975792  TASK: ffff9fce316c9ec0  CPU: 2   COMMAND: <font color=red>"in.telnetd"</font>
... ...
[exception RIP: <font color=red>n_tty_set_termios</font>+48]</html></code></pre>
  - 修复
    - 对于`RHEL 8.4`将内核版本升级至`kernel-4.18.0-305.19.1.el8_4`及以上  
	- 对于`RHEL 8.1`将内核版本升级至`kernel-4.18.0-147.56.1.el8_1`及以上 
	- 缓解办法：关闭`telnetd`服务
  - 参考文档
    [<font color=red>**RHEL8: kernel panic at n_tty_set_termios+0x30**</font>](https://access.redhat.com/solutions/6101331)
**<u><font color=red>仅用作经验记录</font></u>**