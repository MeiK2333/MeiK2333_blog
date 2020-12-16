---
title: CentOS 配置 PHP5.2 + Apache + MySQL
date: 2018-03-23 16:10:22
tags: [Docker, 网站运维]
---

## 原材料
- Ubuntu16.04 服务器
- Docker
- CentOS:6 镜像

<!--more-->

## 没什么卵用的介绍
有个同学的需求，需要在一个 ubuntu16.04 服务器上部署一个 PHP 的网站，当我轻车熟路一顿 apt 之后……

![fatalerror.png](/images/centos-php5.2-apache-mysql/fatalerror.png)

代码中充满了 PHP5.2 版本才有的操作，包括且不限于 session_register、preg_match、ereg、mysql_connect 系列函数，我大概看了一下代码就放弃了帮他升级的想法 —— 升级到新版 PHP 的工作量不亚于写一个新的……

好吧，看来代码是不能动了，那就装一个 PHP5.2 吧。

经过一番搜索和考虑，我决定把 PHP5.2 装在 Docker 中，以 Docker 的形式来运行网站服务。一来是服务器上之前有什么东西我不知道，服务器上已经有 PHP7 了，我不想因为环境的修改导致其他的问题；二来就是，通过搜索，我发现，ubuntu16.04 上安装 PHP5.2 还没有先例……据某个网友说，ubuntu16.04 的系统无法兼容 PHP5 的 libphp5.so ，会导致 Apache 无法解析 PHP 文件。

## 安装 Docker 镜像

直接 pull 一个 CentOS:6 的镜像即可，我使用了国内源的加速。

```shell
docker pull registry.docker-cn.com/library/centos:6
docker run -it -p 80:80 centos:6
```

## 安装 MySQL

项目对 MySQL 的版本没有什么要求，因此直接 yum 安装即可。

```shell
yum install mysql mysql-server
```

## 安装 Apache

Apache 的版本同样没有什么要求，直接 yum 安装。
```shell
yum install httpd
```

## 安装 PHP5.2

### 准备工作

因为 yum 源里的 PHP 版本高于 5.2 ，因此需要手动下载编译 PHP 。

首先需要下载安装 PHP 必需的一些依赖。

```shell
yum groupinstall "Development tools"
useradd opt -d /opt/sbin
yum install wget
yum install epel-release
yum install gcc make httpd-devel libxml2-devel bzip2-devel openssl-devel curl-devel gd-devel libc-client-devel libmcrypt-devel libmhash-devel mysql55-devel aspell-devel libxslt-devel mysql-devel    
ln -s /usr/lib64/libjpeg.so /usr/lib/libjpeg.so
ln -s /usr/lib64/libpng.so /usr/lib/libpng.so
ln -s /usr/lib64/libXpm.so /usr/lib/libXpm.so
ln -s /usr/lib64/libc-client.so /usr/lib/libc-client.so
ln -s /usr/lib64/krb5 /usr/lib/krb5
ln -s /usr/lib64/libgssapi_krb5.so /usr/lib/libgssapi_krb5.so
ln -s /usr/lib64/libgssrpc.so /usr/lib/libgssrpc.so
ln -s /usr/lib64/libk5crypto.so /usr/lib/libk5crypto.so
ln -s /usr/lib64/libkadm5clnt.so /usr/lib/libkadm5clnt.so
ln -s /usr/lib64/libkadm5clnt_mit.so /usr/lib/libkadm5clnt_mit.so
ln -s /usr/lib64/libkadm5srv.so /usr/lib/libkadm5srv.so
ln -s /usr/lib64/libkadm5srv_mit.so /usr/lib/libkadm5srv_mit.so
ln -s /usr/lib64/libkdb5.so /usr/lib/libkdb5.so
ln -s /usr/lib64/libkrb5.so /usr/lib/libkrb5.so
ln -s /usr/lib64/libkrb5support.so /usr/lib/libkrb5support.so
ln -s /usr/lib64/mysql /usr/lib/mysql
usermod -aG wheel opt
echo "%wheel ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```

### 安装

```shell
wget http://museum.php.net/php5/php-5.2.17.tar.gz
tar xzf php-5.2.17.tar.gz
cd php-5.2.17

./configure --prefix=/opt/sbin/php --with-apxs2=/usr/sbin/apxs --with-config-file-path=/opt/sbin/etc/  --disable-posix   --enable-bcmath   --enable-calendar   --enable-exif   --enable-fastcgi   --enable-ftp   --enable-gd-native-ttf   --enable-libxml   --enable-magic-quotes   --enable-mbstring   --enable-pdo   --enable-soap   --enable-sockets   --enable-wddx   --enable-zip  --with-bz2   --with-curl   --with-curlwrappers   --with-freetype-dir   --with-gd   --with-gettext   --with-imap   --with-imap-ssl  --with-jpeg-dir  --with-kerberos   --with-libxml-dir  --with-libxml-dir   --with-mcrypt   --with-mhash   --with-mime-magic   --with-mysql  --with-mysqli   --with-openssl --with-openssl-dir --with-pcre-regex  --with-pdo-mysql   --with-pdo-sqlite   --with-pic   --with-png-dir   --with-pspell   --with-sqlite   --with-ttf   --with-xmlrpc   --with-xpm-dir  --with-xsl --with-zlib   --with-zlib-dir

cd php-5.2.17
sudo make install
libtool --finish /opt/sbin/php-5.2.17/libs
/opt/sbin/php/bin/php --version

    PHP 5.2.17 (cli) (built: Dec 28 2015 13:24:10)
    Copyright (c) 1997-2010 The PHP Group
    Zend Engine v2.2.0, Copyright (c) 1998-2010 Zend Technologies
```

## 配置 Apache 解析 PHP

将下面的代码添加到 `/etc/httpd/conf/httpd.conf` 中：

```
AddType application/x-httpd-php .php

LoadModule php5_module modules/libphp5.so
<IfModule mod_php5.c>

    AddType application/x-httpd-php .php
    AddType application/x-httpd-php-source .phps
</IfModule>
```

重启 Apache 和 MySQL

```shell
/etc/init.d/mysqld restart
/etc/init.d/httpd restart
```

此时访问服务器 IP ，已经在正常解析了。


## 总结

- 万恶的上古代码
- Docker 还是美滋滋啊
