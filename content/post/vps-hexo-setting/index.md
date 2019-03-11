---
title: VPS上搭建Hexo静态博客
date: 2016-06-23
categories:
  - VPS

tags:
    - 博客
    - 教程
---

最近在提交百度收录的时候发现github将百度的爬虫给禁掉了，百度收录不了github上的网站。后来试过用CDN但是没有适合，最终还是将博客重新部署在VPS上，这样就解决了百度爬虫的问题。

<!--more-->

### 准备

- 独立IP服务器，可以在[搬瓦工](http://bandwagonhost.com/)上购买。搬瓦工上的VPS可以用来翻墙而且性价比很高，很适合作为静态博客服务器。
- 域名，域名可以在[godaddy](https://sg.godaddy.com/zh?countryview=1)上购买，支持支付宝付款。godaddy是全球最大的域名注册商，最主要的是不需要备案。
- SSH密钥，用来登录VPS和上传网站。

### 服务器配置

1. 使用putty等工具连接上VPS，使用root权限添加用户，修改密码。

  ```
  [root@VM_32_44_centos /]# adduser hexo
  [root@VM_32_44_centos /]# passwd hexo
  ```

  Centos-minial版本本身没有sudo命令，为了方便我们要先安装sudo。

  ```
  [root@VM_32_44_centos www]# yum install sudo
  ```

  赋予hexo root权限，在/etc/sudoers中添加一行
  ```
  root        ALL=(ALL)       ALL
  hexo        ALL=(ALL)       ALL #这个是添加的
  ```


2. 将ssh公钥添加到/home/hexo/.ssh文件夹下面，并重命名为authorized_keys。

  > 注意`.ssh`文件夹的权限为700，`authorized_keys`的权限为600。

3. 添加nginx源，将以下代码添加到/etc/yum.repos.d/nginx.repo中。

  ```
  [nginx]
  name=nginx repo
  baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
  gpgcheck=0
  enabled=1
  ```

4. 安装nginx。

  ```
  [root@VM_32_44_centos /]# yum install nginx
  ```

5. 删除nginx虚拟主机配置文件

  ```
  [root@VM_32_44_centos /]# rm /etc/nginx/conf.d/*
  ```

6. 将以下内容添加到`/etc/nginx/conf.d/hexo.conf`中

  ```
  server {
    root /home/hexo/www;  #网站根目录
    index index.html index.htm;
    server_name www.astayang.com;   #你的域名
    location / {
      try_files $uri $uri/ /index.html;
    }
  }
  ```

7. 修改nginx配置文件，将/etc/nginx/nginx.conf中的`user`值修改为`hexo`。

8. 切换到hexo用户，并新建www文件夹，作为网站根目录。

  ```
  [astayang@VM_32_44_centos /]$ su hexo
  Password:
  [hexo@VM_32_44_centos /]$ cd ~
  [hexo@VM_32_44_centos ~]$ ls
  [hexo@VM_32_44_centos ~]$ mkdir www
  [hexo@VM_32_44_centos ~]$ ls
  www
  ```

9. 启动nginx服务，并验证nginx配置

  ```
  [hexo@VM_32_44_centos hexo]# sudo nginx -t
  nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
  nginx: configuration file /etc/nginx/nginx.conf test is successful
  ```

  如果输出和上面的一致就说明配置文件没有问题，接下来就启动nginx服务，并设置为开机启动。

  ```
  [hexo@VM_32_44_centos hexo]# sudo systemctl start nginx
  [hexo@VM_32_44_centos hexo]# sudo systemctl enable nginx
  ```

  在/home/hexo/www文件夹下添加index.html文件。

  ```
  [hexo@VM_32_44_centos www]$ echo "This is index.html" > index.html
  ```

  在浏览器中输入VPS的IP地址，这是会出现`This is index.html`字样，删除index.html。

10. 配置git
  ```
  [hexo@VM_32_44_centos www]$ sudo yum install git
  ```

  安装完成后，初始化git仓库
  ```
  [hexo@VM_32_44_centos ~]$ mkdir hexo.git
  [hexo@VM_32_44_centos ~]$ cd hexo.git/
  [hexo@VM_32_44_centos hexo.git]$ git init --bare
  Initialized empty Git repository in /home/hexo/hexo.git/
  ```

  将以下内容添加到`hexo.git/hooks/post-receive`中，并将post-receive文件权限改为755。
  ```
  #!/bin/bash
  GIT_REPO=/home/hexo/hexo.git
  TMP_GIT_CLONE=/tmp/HexoBlog
  PUBLIC_WWW=/home/hexo/www
  rm -rf ${TMP_GIT_CLONE}
  git clone $GIT_REPO $TMP_GIT_CLONE
  rm -rf ${PUBLIC_WWW}/*
  cp -rf ${TMP_GIT_CLONE}/* ${PUBLIC_WWW}
  ```

  ```
  [hexo@VM_32_44_centos ~]$ chmod 755 hexo.git/hooks/post-receive
  ```


以上就将服务器配置完毕，下面就需要配置客户端。

### 客户端

在hexo根目录_config.yml中添加部署配置

```
deploy:
- type: git
  repo: ssh://hexo@VPS的IP:ssh的端口/home/hexo/hexo.git
  branch: master
```

使用hexo命令将网站部署到VPS。
