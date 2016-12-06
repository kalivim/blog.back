---
layout: post
title: Python黑客之旅0x5
categories: Python Hacker
tags: Python黑客之旅
---

* TOC 
{:toc}



## 使用PyInstaller生成可以执行程序

这一章是教大家如何把自己的python脚本编译成windows下可执行文件，它可以让你的python脚本跨平台去运行，并且不需要去安装python解释器。首先我们需要下载依赖包,cygwin(或者其他的工具也可以，这里我们使用Pywin).





# Python转exe

### 安装


Linux: `sudo apt-get install python2.7 build-essential python-dev zlib1g-dev upx`

Windows: http://www.activestate.com/activepython (fully packaged installer file)

安装 [Pywin32](http://sourceforge.net/projects/pywin32/), [Setuptools](https://pypi.python.org/pypi/setuptools#downloads), [PyInstaller](http://www.pyinstaller.org/)

### 安装完成之后

下一步我们就运行python命令生成可执行文件:

``` bash
python pyinstaller.py --onefile <scriptName>
```

执行上面的命令之后，导入依赖文件并且生成一个新的文件，这个文件里面包含了三个文件`<scriptName>.txt,<scriptName>.spec`和`<scriptName>.exe`文件，其中.txt与.spec可以删除掉，而***.exe***的文件就是你需要的执行程序.

### 完整的封装执行程序

Python脚本现在已经被编译成了windows PE文件，并且不需要Python解释器就能够在windows下面独立运行，这可以让你更轻松的把脚本迁移到windows上面而且不用担心依赖包缺失的问题.

一个简单的脚本:

``` python
#!/usr/bin/python
 
import os
 
os.system("echo Hello World!")
```

现在我们把上面这个脚本编译成为一个可以执行的文件:

**Windows**

``` python
c:\PathToPython\python.exe pyinstaller.py --onefile helloWorld.py
 
> helloWorld.exe
Hello World!
```
**Linux**

``` bash
pytnstaller --onefile helloworld.py
```
> 生成后的可执行文件在`dist`文件夹中

> 如果你想更详细的了解这个过程，可以参考[BACK TO THE SOURCE CODE – Forward/Reverse Engineering Python Malware](http://www.primalsecurity.net/back-to-the-source-code-forwardingreverse-engineering-python-malware/)

把你的python脚本编译成一个可以在windows上面可以执行的可执行程序是很有用的，因为它不需要你安装python解释器还有依赖包

大家可以尝试一下**0x3**中的例子，把那个脚本编译成可执行程序。下一章
