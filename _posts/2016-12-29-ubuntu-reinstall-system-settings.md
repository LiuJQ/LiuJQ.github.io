---
layout: post
title: Ubuntu 14.04 卸载ibus导致System Settings异常
tags: [Ubuntu, ibus]
---

### Uninstall ibus
> sudo apt-get remove ibus

### Uninstall ibus and it's dependent packages
> sudo apt-get remove --auto-remove ibus

### Purging ibus
(delete configuration and/or data files)
> sudo apt-get purge ibus

(delete configuration and/or data files and it's dependencies)
> sudo apt-get purge --auto-remove ibus

### 卸载ibus后，打开系统设置会发现系统设置里面的设置项不全
### 重新安装unity-control-center即可
> sudo apt-get install unity-control-center
