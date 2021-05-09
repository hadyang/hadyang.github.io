---
title: Linux 性能调试工具
draft: true
date: 2021-05-06
categories: 
    - Linux
---

最近在逛博客时，发现有个海外程序员总结的 Linux 性能调试工具汇总，包含应用层、文件系统、内存、CPU等一系列的调试工具。这篇博客就介绍下这些工具，使用场景、概念以及说明。

![](assists/linux_performance_tools.png)

## 系统函数库

### [ldd](https://man7.org/linux/man-pages/man1/ldd.1.html)


### [ltrace](https://man7.org/linux/man-pages/man1/ltrace.1.html)


## 系统调用


### [strace](https://man7.org/linux/man-pages/man1/strace.1.html)


### [perf trace](https://man7.org/linux/man-pages/man1/perf-trace.1.html)


## 文件系统

### [lsof](https://man7.org/linux/man-pages/man8/lsof.8.html)


### [df](https://man7.org/linux/man-pages/man1/df.1.html)

文件系统有自己的类型，包括 vfat、ext4等等

文件系统需要 mount 到目录结构上，才能被使用

https://superuser.com/questions/335252/dev-sda1-tmp-dev-shm


## 网络协议栈


## 进程资源



## IO设备



## 网络设备


## CPU