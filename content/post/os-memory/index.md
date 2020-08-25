---
title: 【操作系统】地址空间
draft: true
date: 2021-01-15
categories: 
    - OS
---

## CPU的历史

在讨论 CPU 之前，我们先框定下范围，本文仅限于 Intel 的 x86 系列 CPU。

目前生活中最常见的就是 Intel x86 系列的 CPU。x86 系列从 8086 开始，中间经历几个大版本：80286、80386，到这里 x86 系列的基础已经奠定了。

8086 是 16bit CPU，即其寄存器大小为 16bit。8086 拥有地址总线大小为 20bit，即其物理地址空间为 $2^20 byte = 1MB$。

80286 新增了保护模式，其地址总线的大小也扩展到 24bit



## BUS

system bus



## 参考

https://en.wikipedia.org/wiki/Virtual_memory
https://en.wikipedia.org/wiki/Memory_map
https://superuser.com/questions/595672/how-is-memory-mapped-to-certain-hardware-how-is-mmio-accomplished-exactly
https://en.wikipedia.org/wiki/PCI_configuration_space
https://techterms.com/definition/ioaddress
https://freemandealer.github.io/2016/10/07/io-memory/
https://en.wikichip.org/wiki/intel/microarchitectures/skylake_(client)