---
title: 【操作系统】从上电到 BootLoader
draft: true
date: 2021-01-14
categories: 
    - OS
---

## 实模式


分页


## 地址空间映射

CPU 只和内存交互

地址空间为 ROM 和硬件保留了一部分地址空间  memory-mapped I/O (MMIO)


PAE

https://zh.wikipedia.org/wiki/%E7%89%A9%E7%90%86%E5%9C%B0%E5%9D%80%E6%89%A9%E5%B1%95

https://stackoverflow.com/questions/20861032/who-loads-the-bios-and-the-memory-map-during-boot-up

## 寻址方式

寻址空间  物理地址空间&线性地址空间

物理地址空间是指 cpu 地址总线的宽度

线性地址空间 和分段寻址是对应的？ https://en.wikipedia.org/wiki/Flat_memory_model

寻址粒度 8byte？ memory chunk

分页是为了磁盘和内存的交换？


---

grep 'address sizes' /proc/cpuinfo

https://superuser.com/questions/944080/why-does-my-cpu-only-support-32gb-ram-when-it-has-39-address-bits

内存大小被 内存控制器 控制 https://zh.wikipedia.org/wiki/%E5%86%85%E5%AD%98%E6%8E%A7%E5%88%B6%E5%99%A8

so why not have just 34 physical address bits? What purpose does 39 bits serve? – Lord Loh. 

Not everything that has a physical address is RAM. Having more bits of physical address than you need for RAM allows, for example, I/O devices that present large amounts of memory-mapped address space without interfering with the amount of RAM you can use. I've also seen platforms that used some of the "extra" bits to specify types of access to I/O space (like byte- and word-sized access) which the processor couldn't otherwise support - this was specified via code in the driver support library and interpreted by the bus bridge. I don't know if this is done in current Intel platforms.

---

CS:IP


## 从上电到 BootLoader


传统BIOS缺点:16位寻址 https://superuser.com/questions/988308/why-does-bios-operate-in-16-bits-instead-of-32-64-bits

传统BIOS最大的缺点就是16位寻址模式，16位寻址最大支持1MB内存，远远小于现有主流计算机内存大小。CPU出于兼容性考虑，在BIOS引导过程中只能运行在实模式。


## 磁盘地址

[LAB](https://en.wikipedia.org/wiki/Logical_block_addressing)

磁盘地址空间，不包含在物理地址空间中。其中的数据也不是内存映射。https://superuser.com/questions/1442815/how-does-cpu-dma-access-hard-disk


## 参考资料

1. http://web.archive.org/web/20180713151039/https://manybutfinite.com/post/motherboard-chipsets-memory-map/
2. https://manybutfinite.com/post/how-computers-boot-up/
3. https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-1.html
4. https://www.cnblogs.com/zhuge2018/p/8466879.html
5. https://stackoverflow.com/questions/34183799/how-does-this-assembly-bootloader-code-work
6. https://stackoverflow.com/questions/20861032/who-loads-the-bios-and-the-memory-map-during-boot-up
7. https://web.archive.org/web/20180713151104/https://manybutfinite.com/post/memory-translation-and-segmentation/
8. https://web.archive.org/web/20180713151101/https://manybutfinite.com/post/cpu-rings-privilege-and-protection/
9. [0x7C00地址的由来](https://www.glamenv-septzen.net/en/view/6)


bffdbfff-00100000