---
layout: post
title: 一次 Native Crash(file descriptor leaks) 探究过程
subtitle: Android native crash because of file descriptor leaks
tags: [Android, fd泄漏, Native Crash]
---

> CRASH: com.process.name (pid 8088)
<br>Short Msg: Native crash
<br>Long Msg: Native crash: Aborted
<br>ABI: 'arm'
<br>pid: 8088, tid: 8088, name: com.process.name  >>> com.process.name <<<
<br>signal 6 (SIGABRT), code -6 (SI_TKILL), fault addr --------
<br>Abort message: 'FORTIFY: FD_SET: file descriptor >= FD_SETSIZE'

&emsp;&emsp;Android开发者对Native Crash问题可能并不陌生，但是谁也不愿意遇到它，毕竟此类问题十分棘手。

...未完待续
