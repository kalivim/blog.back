---
layout: post
title: RedHat6/CentOS6 安装Metasploit 
tags: Kali Hacker Metasploit
categories: Hacker Kali 
---

* TOC 
{:toc}

![metasploit](https://github.com/kalivim/kalivim.github.io/raw/master/images/2016-12-06/banner.png)

> 本章记录在RedHat6/CentOS6上安装Metasploit，因为坑队友给了台Redhat6系统的云主机。只能自己搞起来了





## 1. Metasploit

**msf这部分`msfinstall`脚本会自动来安装，所以需要配置的也比较少**

下载官方提供的安装脚本，并执行

``` shell
curl https://raw.githubusercontent.com/rapid7/metasploit-omnibus/master/config/templates/metasploit-framework-wrappers/msfupdate.erb > msfinstall &&   chmod 755 msfinstall &&   ./msfinstall
```

> 静静等待它下载完。。。`msfinstall`脚本只会安装metasploit，不会安装postgresql


## 2. Postgresql

`yum -y install postgresql postgresql-server`

> 这里我使用的是epel源
>  `rpm -ivh http://dl.fedoraproject.org/pub/epel/6Server/x86_64/epel-release-6-8.noarch.rpm`
>  `yum clean all && yum makecache `


## 3. 配置Postgresql

#### (1)添加数据库，用户

**初始化启动 :**

`service postgresql initdb`

`service postgresql start`

``` shell

#切换到普通用户
su - postgres

#创建数据库和用户
psql

postgres=# create user "youruser" with password 'yourpasswd' nocreatedb;
CREATE ROLE

postgres=# create database "msf4" with owner="youruser";
CREATE DATABASE

postgres=# \q
退出
```

#### (2)修改服务配置文件

`切换回root`

`vim /var/lib/pgsql/data/postgresql.conf`

> 在我的机器上是这个文件，有可能会不同，名字一样自行查找

``` shell
#共修改2处

更改 #listen_addresses = ‘localhost’
为 listen_addresses = ‘127.0.0.1’

更改 #password_encryption = on
为 password_encryption = on
```

`vim /var/lib/pgsql/data/pg_hba.conf`

``` shell
#文件末尾处
更改 host    all         all         127.0.0.1/32          ident
为 host    all         all         127.0.0.1/32          md5
```

#### (3)重启服务

`service postgresql restart`

`psql -U youruser -h 127.0.0.1`

**测试是否配置成功**


## 4. 配置Metasploit

`vim /root/.msf4/database.yml`

**添加如下内容**

``` shell
production:
  adapter: "postgresql"
  database: "msf4"
  username: "youruser"
  password: "yourpasswd"
  host: "127.0.0.1"
  port: 5432
  pool: 5
  timeout: 5
```

`cp /root/.msf4/database.yml /var/lib/pgsql/.msf4/database.yml`

> msf初始化数据库

`su - postgres`

`msfdb init`


## 5. 运行msf

`msfconsole`

**查看数据库连接状态**

`msf > db_status`

`[*] postgresql connected to msf4`#表示连接成功


> 当使用search搜索模板会提示`[!] Module database cache not built yet, using slow search`意思是数据库没有建立缓存

`msf > db_rebuild_cache`

这样就会在后台建立数据库缓存了，至此全部安装配置已完成。后面会写一写关于metasploit的应用。
