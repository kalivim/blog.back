---
layout: post
title: Kickstart无人值守安装Linux
tags: 自动化运维 Kickstart Linux
categories: Linux Kickstart
---

* TOC 
{:toc}

作为中小公司的运维，经常会遇到一些机械式的重复工作，例如：有时公司同时上线几十甚至上百台服务器，而且需要我们在短时间内完成系统安装。

OK让我们来一起kickstart吧





## 1. 简介

`理论性知识建议大概看下,先按照步骤做一遍知到大概思路,在仔细看会更好(个人建议)`

### `1.1` 什么是PXE?

> PXE，全名`Pre-boot Execution Environment`，预启动执行环境；
通过网络接口启动计算机，不依赖本地存储设备（如硬盘）或本地已安装的操作系统；
由Intel和Systemsoft公司于1999年9月20日公布的技术；
`Client/Server的工作模式:`
PXE客户端会调用网际协议(IP)、用户数据报协议(UDP)、动态主机设定协议(DHCP)、小型文件传输协议(TFTP)等网络协议；
PXE客户端(client)这个术语是指机器在PXE启动过程中的角色。一个PXE客户端可以是一台服务器、笔记本电脑或者其他装有PXE启动代码的机器（我们电脑的网卡）。

### `1.2` PXE工作过程


![PXE](http://ww1.sinaimg.cn/large/0060lm7Tgw1f6xsskxex0j30gn0fptar.jpg)

`1.` PXE Client向DHCP发送请求
PXE Client从自己的PXE网卡启动，通过PXE BootROM(自启动芯片)会以UDP(简单用户数据报协议)发送一个广播请求，向本网络中的DHCP服务器索取IP。

`2.` DHCP服务器提供信息
DHCP服务器收到客户端的请求，验证是否来至合法的PXE Client的请求，验证通过它将给客户端一个“提供”响应，这个“提供”响应中包含了为客户端分配的IP地址、pxelinux启动程序(TFTP)位置，以及配置文件所在位置。

`3.` PXE客户端请求下载启动文件
客户端收到服务器的“回应”后，会回应一个帧，以请求传送启动所需文件。这些启动文件包括：pxelinux.0、pxelinux.cfg/default、vmlinuz、initrd.img等文件。

`4.` Boot Server响应客户端请求并传送文件
当服务器收到客户端的请求后，他们之间之后将有更多的信息在客户端与服务器之间作应答, 用以决定启动参数。BootROM由TFTP通讯协议从Boot Server下载启动安装程序所必须的文件(pxelinux.0、pxelinux.cfg/default)。default文件下载完成后，会根据该文件中定义的引导顺序，启动Linux安装程序的引导内核。

`5.` 请求下载自动应答文件
客户端通过pxelinux.cfg/default文件成功的引导Linux安装内核后，安装程序首先必须确定你通过什么安装介质来安装linux，如果是通过网络安装(NFS, FTP, HTTP)，则会在这个时候初始化网络，并定位安装源位置。接着会读取default文件中指定的自动应答文件ks.cfg所在位置，根据该位置请求下载该文件。

>这里有个问题，在第2步和第5步初始化2次网络了，这是由于PXE获取的是安装用的内核以及安装程序等，而安装程序要获取的是安装系统所需的二进制包以及配置文件。因此PXE模块和安装程序是相对独立的，PXE的网络配置并不能传递给安装程序，从而进行两次获取IP地址过程，但IP地址在DHCP的租期内是一样的。

`6.` 客户端安装操作系统
将ks.cfg文件下载回来后，通过该文件找到OS Server，并按照该文件的配置请求下载安装过程需要的软件包。
OS Server和客户端建立连接后，将开始传输软件包，客户端将开始安装操作系统。安装完成后，将提示重新引导计算机。


## 2. 安装DHCP服务

**`如果是用虚拟机测试需要关闭虚拟网卡DHCP功能`**

### 1) 安装包

```bash 
[root@CentOS6.5 ~]# yum -y install dhcp
[root@CentOS6.5 ~]# rpm -ql dhcp |grep "dhcpd.conf"
/etc/dhcp/dhcpd.conf   # 查看配置文件位置
/usr/share/doc/dhcp-4.1.1/dhcpd-conf-to-ldap
/usr/share/doc/dhcp-4.1.1/dhcpd.conf.sample
/usr/share/man/man5/dhcpd.conf.5.gz
```

### 2) 编辑配置文件

`[root@CentOS6.5 ~]# vim /etc/dhcp/dhcp.conf`

```bash
default-lease-time 14400;
ddns-update-style none;
next-server 192.168.1.52;		#指定tftpd服务器ip
filename "pxelinux.0";
subnet 192.168.1.0 netmask 255.255.255.0 {		#地址池
        range 192.168.1.50 192.168.1.240;
        default-lease-time 14400;               # 设置默认的IP租用期限
        max-lease-time 172800;
}
```

`[root@CentOS6.5 ~]# vim /etc/sysconfig/dhcpd`

```bash
DHCPDARGS=eth1  # 指定监听网卡
```

`[root@CentOS6.5 ~]# service dhcpd restart`

## 3. 安装TFTP

### 1) TFTP简介:

`TFTP（Trivial File Transfer Protocol,简单文件传输协议）是TCP/IP协议族中的一个用来在客户机与服务器之间进行简单文件传输的协议，提供不复杂、开销不大的文件传输服务。端口号为69。`

### 2) TFTP安装配置

```bash
[root@CentOS6.5 ~]# yum -y install tftp-server
[root@CentOS6.5 ~]# vim /etc/xinetd.d/tftp
service tftp
{
        socket_type             = dgram
        protocol                = udp
        wait                    = yes
        user                    = root
        server                  = /usr/sbin/in.tftpd
        server_args             = -s /var/lib/tftpboot # 指定目录，保持默认，不用修改
        disable                 = no # 由原来的yes改为no
        per_source              = 11
        cps                     = 100 2
        flags                   = IPv4
}
[root@CentOS6.5 ~]# /etc/init.d/xinetd restart
Stopping xinetd:                                           [FAILED]
Starting xinetd:                                           [  OK  ]
[root@CentOS6.5 ~]# netstat -tunlp|grep 69
udp        0      0 0.0.0.0:69            0.0.0.0:*      1106/xinetd
```

## 4. 安装HTTP服务

**`可以用Apache或Nginx提供HTTP服务。Python的命令web服务不行，会有报错。`**

```bash
[root@CentOS6.5 ~]# yum -y install httpd
[root@CentOS6.5 ~]# sed -i "277i ServerName 127.0.0.1:80" /etc/httpd/conf/httpd.conf
[root@CentOS6.5 ~]# /etc/init.d/httpd start
[root@CentOS6.5 ~]# mkdir /var/www/html/CentOS-6.5
[root@CentOS6.5 ~]# mount /dev/sr0 /var/www/html/CentOS-6.5/ #sr0测试虚拟机环境线上会不同
mount: block device /dev/sr0 is write-protected, mounting read-only
[root@CentOS6.5 ~]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3        19G  2.4G   16G  14% /
tmpfs           491M   16K  491M   1% /dev/shm
/dev/sda1       190M   36M  145M  20% /boot
/dev/sr0        3.7G  3.7G     0 100% /var/www/html/CentOS-6.5
# 不管怎么弄，只要把安装光盘内容能通过web发布即可。因为是演示，如果复制镜像就有点浪费时间。但生产环境就一定要复制了，光盘读取速度有限。
```

浏览器访问测试配置是否成功`http://IP/CentOS-6.5`

## 5. 配置支持PXE的启动程序


`syslinux是一个功能强大的引导加载程序，而且兼容各种介质。SYSLINUX是一个小型的Linux操作系统，它的目的是简化首次安装Linux的时间，并建立修护或其它特殊用途的启动盘。如果没有找到pxelinux.0这个文件,可以安装一下。`

```bash
[root@CentOS6.5 ~]# yum -y install syslinux
[root@CentOS6.5 ~]# cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/
# 复制启动菜单程序文件
[root@CentOS6.5 ~]# cp -a /var/www/html/CentOS-6.5/isolinux/* /var/lib/tftpboot/
[root@CentOS6.5 ~]# ls /var/lib/tftpboot/
boot.cat  grub.conf   isolinux.bin  memtest     splash.jpg  vesamenu.c32
boot.msg  initrd.img  isolinux.cfg  pxelinux.0  TRANS.TBL   vmlinuz
# 新建一个pxelinux.cfg目录，存放客户端的配置文件。
[root@CentOS6.5 ~]# mkdir -p /var/lib/tftpboot/pxelinux.cfg
[root@CentOS6.5 ~]# cp /var/www/html/CentOS-6.5/isolinux/isolinux.cfg /var/lib/tftpboot/pxelinux.cfg/default
```

## 6. 测试手动网络安装



`开启一个新的虚拟机内存给1GB,不出错误会看到光盘安装界面, 接下来就是创建ks.cfg了. 关闭测试虚拟机,继续`

## 7. 创建ks.cfg文件

>通常，我们在安装操作系统的过程中，需要大量的和服务器交互操作，为了减少这个交互过程，kickstart就诞生了。使用这种kickstart，只需事先定义好一个Kickstart自动应答配置文件ks.cfg（通常存放在安装服务器上），并让安装程序知道该配置文件的位置，在安装过程中安装程序就可以自己从该文件中读取安装配置，这样就避免了在安装过程中多次的人机交互，从而实现无人值守的自动化安装。

#### **`生成kickstart配置文件的三种方法：`**

- 方法1、 每安装好一台Centos机器，Centos安装程序都会创建一个kickstart配置文件，记录你的真实安装配置。如果你希望实现和某系统类似的安装，可以基于该系统的kickstart配置文件来生成你自己的kickstart配置文件。（生成的文件名字叫anaconda-ks.cfg位于/root/anaconda-ks.cfg）
- 方法2、Centos提供了一个图形化的kickstart配置工具。在任何一个安装好的Linux系统上运行该工具，就可以很容易地创建你自己的kickstart配置文件。kickstart配置工具命令为rsystem-config-kickstart,网上有很多用CentOS桌面版生成ks文件的文章，如果有现成的系统就没什么可说。但没有现成的，也没有必要去用桌面版，命令行也很简单。
- 方法3、阅读kickstart配置文件的手册。用任何一个文本编辑器都可以创建你自己的kickstart配置文件。

#### `这里贴出我的ks.cfg,这个是CentOS6.8的,可以根据需求更改`

**`这里是官方文档`[CentOS](https://access.redhat.com/documentation/zh-CN/Red_Hat_Enterprise_Linux/6/html/Installation_Guide/s1-kickstart2-options.html)**

```bash
#platform=x86, AMD64, 或 Intel EM64T
#version=DEVEL
# Firewall configuration
firewall --enabled --http --ssh
# Install OS instead of upgrade
install
# Use network installation
url --url="http://192.168.1.52/CentOS6.5"
# Root password
rootpw --iscrypted $1$xn3sYOAA$jOfmmjxxYA/IRjFm0BM5O0
# System authorization information
auth  --useshadow  --passalgo=sha512
# Use graphical install
graphical
# System keyboard
keyboard us
# System language
lang zh_CN
# SELinux configuration
selinux --enforcing
# Do not configure the X Window System
skipx
# Installation logging level
logging --level=info

# System timezone
timezone  Africa/Libreville
# System bootloader configuration
bootloader --location=mbr
# Partition clearing information
clearpart --all  

%packages
@php

%end

```

### 8. 整合Default配置文件实现无人值守

`vim /var/lib/tftpboot/pxelinux.cfg/default`

```bash
default ks
prompt 0
label ks
  kernel vmlinuz
  append initrd=initrd.img ks=http://192.168.1.52/ks_config/CentOS-6.5-ks.cfg # 告诉安装程序ks.cfg文件在哪里
# append initrd=initrd.img ks=http://192.168.1.52/ks_config/CentOS-6.5-ks.cfg ksdevice=eth0
# ksdevice=eth0代表当客户端有多块网卡的时候，要实现自动化需要设置从eth1安装，不指定的话，安装的时候系统会让你选择，那就不叫全自动化了
```

### 9. 打开系统电源,出去吃个饭,过会回来,系统就以经装好了

**`关闭服务端的iptables,selinux`**

```bash
service iptables stop
setenforce 0
```
>起cobbler也可实现无人值守,属于工具
