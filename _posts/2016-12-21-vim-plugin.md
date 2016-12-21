---
layout: post
title: 自定义VIM插件为IDE利器
categories: Vim	Linux
tags: Vim YouCompleteMe
---

* TOC 
{:toc}

![Vim演示](https://github.com/kalivim/kalivim.github.io/raw/master/images/2016-12-21/vim-plugins.gif "演示")

> 如图是VIM自定义插件及配置后的演示，虽然有Pycharm但是算不上轻量级，写个运维小脚本根本用不到，一般笔者只有在写Django或读python项目源码的时候会用，so我们开始吧







## 1. 安装插件管理工具[Vundle](https://github.com/VundleVim/Vundle.Vim "Vundle是Vim插件管理工具")

``` bash
git clone https://github.com/gmarik/vundle.git ~/.vim/bundle/vundle
```

编辑文件~/.vimrc

```
set nocompatible                    " be iMproved

filetype off                        " required!
set rtp+=~/.vim/bundle/vundle/
call vundle#rc()

"my Bundle here:
Bundle 'Valloric/YouCompleteMe'			#代码补全
Plugin 'Yggdroot/indentLine'			  #垂直对齐线
Plugin 'Chiel92/vim-autoformat'			#自动语法缩进

filetype plugin indent on


" hot key

```

修改完成后命令行执行 `vim +PluginInstall +qall` 等待安装完成，执行`vim +BundleInstall +qall` 下载**Valloric/YouCompleteMe**包（可能久些）




## 2. 编译安装[YouCompleteMe](https://github.com/Valloric/YouCompleteMe#ubuntu-linux-x64 "代码补全神器")

安装编译所需软件：

``` bash
 sudo apt-get install build-essential cmake
 sudo apt-get install python-dev python3-dev

```

### 可选编译支持代码补全

**1. 编译具有对C系列语言支持的YCM：**

``` bash
cd ~/.vim/bundle/YouCompleteMe
./install.py --clang-completer
```

**2. 编译YCM没有C语言的支持：**

```
cd ~/.vim/bundle/YouCompleteMe
./install.py
```

**3. 编译所有支持的语言：**

```
cd ~/.vim/bundle/YouCompleteMe
./install.py --all
```

> 脚本会自动下载所需包，笔者选择的是编译所有支持语言全部下来大小在400MB左右

## 3. 报错解决

报错如下：

``` bash
YouCompleteMe unavailable: requires Vim compiled with  Python 2.x support
```

原因：

执行`vim --version|grep python`

```
-python
-python3
```
vim编译安装的时候默认没有支持Python

解决：

在Debian中`sudo apt-get install vim-nox`

在Ubuntu中`sudo apt-get install vim-gnome-py2`

其他Linux发行版中如果没有上述软件包，需要重新下载vim源码重新编译，加选项`--enable-python3interp=yes`

更多解决方案：[stackoverflow](http://stackoverflow.com/questions/20160902/how-to-solve-requires-python-2-x-support-in-linux-vim-and-it-have-python-2-6-6)

> 只有个别发行版中Vim会不编译支持Python


## 附加： python交互模式中代码自动补全，及笔者`.vimrc`配置文件

家目录写入`.pythontab`文件

`vim ~/.pythontab`

```python
# python startup file
# -*- coding: utf8 -*-
import sys
import readline
import rlcompleter
import atexit
import os
# tab completion
readline.parse_and_bind('tab: complete')	#tab键可自定义
# history file
histfile = os.path.join(os.environ['HOME'], '.pythonhistory')
try:
    readline.read_history_file(histfile)
except IOError:
    pass
atexit.register(readline.write_history_file, histfile)


del os, histfile, readline, rlcompleter 

```

`chmod +x .pythontab`

编辑`~/.bashrc`文件加入环境变量

``` bash
echo "export PYTHONSTARTUP=~/.pythontab" >> ~/.bashrc && source ~/.bashrc
```

**测试：**

> 双击tab键补全,可更改在`~/.pythontab`文件中有做注释

``` bash
kionf@kionf-pc[18:23]~:$ python
Python 2.7.12+ (default, Aug  4 2016, 20:04:34) 
[GCC 6.1.1 20160724] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import random
>>> random.
random.BPF                  random.__reduce__(          random.betavariate(
random.LOG4                 random.__reduce_ex__(       random.choice(
random.NV_MAGICCONST        random.__repr__(            random.division
..........................................................................
random.__package__          random._warn(               
>>> random.

```

**.vimrc配置文件**

```bash
set nocompatible                " be iMproved

set tabstop=4					"tab键为4个空格
set shiftwidth=4
set expandtab
set nu						  	"显示行号，复制的时候可以:set nu!来取消显示
syntax on


filetype off                    " required!
set rtp+=~/.vim/bundle/vundle/
call vundle#rc()

"my Bundle here:
Bundle 'Valloric/YouCompleteMe'
Plugin 'Yggdroot/indentLine'
Plugin 'Chiel92/vim-autoformat'


filetype plugin indent on


" hot key
noremap <F3> :Autoformat<CR>			    "自动缩进
noremap <F2> <Esc>:w<CR>:!python %		"调试python文件vim下F2运行当前文件（F2后按回车）

```
