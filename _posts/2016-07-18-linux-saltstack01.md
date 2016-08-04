---
layout: post
title: Linux 自动化运维之SlatStack01
category: Linux
tags: Linux 自动化运维
---

* TOC 
{:toc}





# saltstack基本原理：

> ` SaltStack`采用 C/S模式，server端就是salt的`master`，client端就是`minion`，minion与master之间通过ZeroMQ消息队列通信，**是一个同时对一组服务器进行远程执行命令和状态管理的工具。**

> minion上线后先与master端联系，把自己的`pub key`发过去，这时master端通过salt-key -L命令就会看到minion的key，接受该minion-key后，也就是master与minion已经互信

>  `master`可以发送任何指令让`minion`执行了，salt有很多可执行模块，比如说cmd模块，在安装minion的时候已经自带了，它们通常位于你的python库中，`locate salt | grep /usr/ `可以看到salt自带的所有东西。

>  这些模块是python写成的文件，里面会有好多函数，如cmd.run，当我们执行`salt '*' cmd.run 'uptime'`的时候，master下发任务匹配到的minion上去，minion执行模块函数，并返回结果。master监听**4505**和**4506**端口，4505对应的是ZMQ的PUB system，用来发送消息，4506对应的是REP system是来接受消息的。


### 具体步骤如下:

1. Salt stack的Master与Minion之间通过ZeroMq进行消息传递，使用了ZeroMq的发布-订阅模式，连接方式包括`tcp`，`ipc`

2. salt命令，将`cmd.run ls`命令从`salt.client.LocalClient.cmd_cli`发布到master，获取一个Jodid，根据jobid获取命令执行结果。

3. master接收到命令后，将要执行的命令发送给客户端minion。

4. minion从消息总线上接收到要处理的命令，交给`minion._handle_aes`处理

5. `minion._handle_aes`发起一个本地线程调用cmdmod执行ls命令。线程执行完ls后，调用minion._return_pub方法，将执行结果通过消息总线返回给master

6. master接收到客户端返回的结果，调用`master._handle_aes`方法，将结果写的文件中

7. `salt.client.LocalClient.cmd_cli`通过轮询获取Job执行结果，将结果输出到终端。

**下面让我们来使用它，才能更好的理解它的工作模式和原理**

## 安装
* CentOS6/Redhat6

```bash
sudo yum install https://repo.saltstack.com/yum/redhat/salt-repo-latest-1.el6.noarch.rpm
```

```bash
yum clean all
```

* 安装salt-minion, salt-master, 或者其它组件

```bash
sudo yum install salt-master			#server端
sudo yum install salt-minion			#client端
sudo yum install salt-ssh
sudo yum install salt-syndic
sudo yum install salt-cloud
sudo yum install salt-api
```
**服务端**

```shell
    sudo yum install salt-master 
```

**客户端**

```shell
    sudo yum install salt-minion
```

## 配置  :

### 服务端 `vim /etc/salt/master`

*#master消息发布端口 Default: 4505*

`publish_port: 4505`    

*#工作线程数，应答和接受minion Default: 5*

`worker_threads: 100`

*#客户端与服务端通信的端口 Default: 4506*

`ret_port: 4506`    

*# 自动接受所有客户端*

`auto_accept: True` 

*# 自动认证配置*   

`autosign_file: /etc/salt/autosign.conf`

### 客户端 `vim /etc/salt/minion`

*# master IP或域名*

`master: 10.0.0.1`

*# 客户端与服务端通信的端口。 Default: 4506*

`syndic_master_port: 4506`

*# 建议线上用ip显示*

`id: test`

>id minion的唯一标示。Default: hostname *
> minion id：minion的唯一标示，默认为minion的hostname，如果id修改了，master 需要重新认证。
> （通过tcpdump做了个实验，修改minion id后，master上会新增一个id，但老id也还在，执行salt '*' test.ping的时候，执行时间变长了，延迟时间约为14s，而且master会在发送命令后延迟10s再给每个已经执行成功的minion发送一个包并有minion有返回，如果没有老id存在不会发送，可以理解为mater在向每个minion寻求未连接的id的信息，minion的salt服务关闭也是这种情况，修改timeout值无效。）


## 测试命令 ： 

`测试环境中关闭 iptbles！！`

```shell
service iptables stop
```

**master端执行**

```shell
salt-key -L	
salt-key -A
```

*查看到minion端ip表示成功*

> Accepted Keys:
192.168.1.3
Denied Keys:
Unaccepted Keys:
Rejected Keys:

```shell
salt '*' test.ping
```

> 192.168.1.3:
    True


*Posted by : DY*
