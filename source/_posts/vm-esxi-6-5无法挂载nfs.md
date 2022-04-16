---
title: vm-esxi-6.5无法挂载nfs
date: 2022-04-16 11:16:08
tags:
  - esxi
categories: 
  - configruation
---

# esxi 6.5 无法挂载NFS: 
<pre><code>报错如下:
Operation failed, diagnostics report: Unable to get console path for volume xxx</code></pre>
<!-- more -->

### 修复
- 这或许由于esxi nfs 最大挂载数限制有关。 
  ESXI 6.5 最大 nfs 挂载数为8，由参数 ` NFS.MaxVolumes` 控制，最高可配置为 `256`
- 参考如下图：
  ![1](/images/22416/1.png)
  - 1. `manager` 下找到 ` NFS.MaxVolumes`  配置项目
  - 2. 点击 `Edit` 增加数值，可以看到默认为8