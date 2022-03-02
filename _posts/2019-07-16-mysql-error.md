---
layout: post
title:  "linux 环境下安装 Mysql, 以及 mysql 运程连接"
date:   2019-07-16 16:01:01 +0800
categories: [Tech]
tag: 
  - Mysql
---

### linux 环境安装 mysql

```bash
sudo apt update
sudo apt install mysql-server
```

### MySql 运程连接

#### 环境和 mysql 版本

MySql: `mysql  Ver 14.14 Distrib 5.7.26, for Linux (x86_64) using  EditLine wrapper`

linux: `Ubuntu 18.04.2 LTS`

#### 第一步

`grant all privileges on *.* to 'root'@'%' identified by 'password';`

`flush privileges;`

* 第一个\* 是数据库名称，可以改成允许访问的数据库名称

* 第二个\* 是数据库的表名称(\*代表允许访问任意的表和数据库)

* root代表远程登录使用的用户名，可以自定义

* % 代表允许任意ip登录，如果你想指定特定的IP，替换掉就可以

* password 是运程连接密码

* 第二条命令能让上面的命令立即生效

#### 第二步

修改`mysql.cnf`文件.在`bind-address = 127.0.0.1`前面加`#`

目录: `/etc/mysql/mysql.conf.d/mysqld.cnf`

命令:

`sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf`

注意:

`bind-address = 127.0.0.1`可能不在这个文件中,可以通过查找`find /* -name my.cnf`,这个文件里可能就有,如果没有,可以查看这个文件包含的路径,如:

```bash
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
```

这两个路径下面的文件不多,可以一个一个找.
