---
layout: post
title: "ubuntu下安装Linux Apache MySQL PHP dbninja"
date: 2014-01-05 14:06
comments: true
categories: ubuntu
---

### About dbninja

[dbninja](http://www.dbninja.com/)是用基于web的一个开源免费的mysql数据库管理工具,有点类似于phpmyadmin,它的界面也是很美,功能很强大,安装很简单,我一般用来管理在rails下的mysql数据库,很方便

<!-- more -->

{% img /images/dbninja.png %}

### Set Up

使用它必须安装php,在ubuntu下我们用apt-get直接安装apache+php+mysql

我的环境是ubuntu 13.04

### Step One -- Install MySQL

``` bash
sudo apt-get install mysql-server
```

这样就可以了,它会把mysql-client这种包也装上的,还有一些其他依赖的包

### Step two -- Install Apache

``` bash
sudo apt-get install apache2
```

装完后,下次开机apache会自动启动的,如果要重启

``` bash
sudo /etc/init.d/apache2 restart
```

### Step three -- Install PHP

``` bash
sudo apt-get install php5 libapache2-mod-php5 php5-mysql
```

### Step four -- Install dbninja

apache默认放项目的目录在/var/www

只要把下载的dbninja解压放在/var/www下即可

然后

``` bash
sudo chmod 777 -R dbninja
```

这样在浏览器输入localhost/dbninja就可以访问了
