---
title: Git代理设置，加速clone
date: 2016-02-18
categories:
  - Git

tags:
    - 教程
    - ShadowSocks
---

由于在项目中经常使用到git来获取源码，但是clone的速度确实不敢恭维。本文主要通过ss进行代理，访问速度有500kb/s（个人的网比较渣）。

<!--more-->

首先你得有ss客户端和账号：
客户端下载接：https://shadowsocks.org/en/download/clients.html
账号可以在网上找免费的。


下面进入正题：

 1. 确保你的ss客户端正常连接
 2. 打开命令行输入以下代码：
```
git config --global http.proxy 'socks5://127.0.0.1:1080'
git config --global https.proxy 'socks5://127.0.0.1:1080'

```

1080是ss的默认端口，如果你修改过，只需要把上面代码里的1080替换掉。

到此完成配置，尽情享用。
