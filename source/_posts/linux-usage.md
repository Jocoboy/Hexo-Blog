---
title: Linux常用命令
date: 2024-08-16 10:54:30
categories:
- Operation-System
tags:
- Linux
---

Linux系统基本概念、安装配置、各级目录含义、常用命令，以及VI/VIM、GUN nano编辑器使用方法。

<!--more-->

## 前言

Linux系统相比于Windows而言，开源，运行更稳定和安全。Linux系统是一个多层次的结构，包含了内核、系统库、shell以及应用程序。

## 基本概念

### Linux发行版


Linux发行版是Linux内核与软件包、系统工具、库文件等组成的一个完整的操作系统，它提供了一个预先配置好的Linux环境，使我们能更方便的安装、配置、使用Linux系统。常见的发型版有Redhat、CentOS、Ubuntu、Alpine等。

## 安装配置Linux

Linux系统可通过虚拟机软件(如VMWare/VirtualBox/Multipass)、容器(如Docker)安装，也可通过云服务器(如AWS/Azure/阿里云)直接使用。

Windows系统中推荐使用WSL安装适用于Linux的Windows子系统，初次安装可使用命令`wsl --install`，可使用命令`wsl --update`更新至最新版。在Microsoft Store中也可直接下载Linux的发行版(如Ubuntu)。

此外，还可以[下载Unbuntu镜像文件]((https://cn.ubuntu.com/download))，然后通过VMWare安装Unbuntu。

## VI/VIM编辑器

Linux系统中没有图形化编辑器，通常在服务器环境上需要借助VIM快速编辑修改文件配置。

VI是Unix操作系统和类Unix操作系统中最通用的文本编辑器。VIM编辑器是从VI发展出来的一个性能更强大的文本编辑器。可以主动的以字体颜色辨别语法的正确性，方便程序设计。VIM与VI编辑器完全兼容。

VIM包含一般模式、编辑模式、命令模式等，相应的语法详见VIM使用参考手册。

## GNU nano编辑器

nano是一个字符终端文本编辑器，有点像DOS下的editor程序。它比VI/VIM要简单得多，比较适合Linux初学者使用。

nano命令可以打开指定文件进行编辑，默认情况下它会自动断行，可使用`nano -w`禁止自动换行。

像Pico一样，nano是面向键盘的，它通过Ctrl键来控制。比如Ctrl+O保存当前文件(会让你输入文件名，按下Enter即可)，Ctrl+X退出编辑器。nano在屏幕底部显示两行快捷键，列出了当前状态下能用的命令。更多用法详见nano使用参考手册。

## Linux常用命令

### 文件

按照修改时间逆序显示文件的详细信息

`ls -ltr`

查看文件内容

`cat [file]`

创建空文件

`touch [file]`

创建文件并写入文件内容

`echo [content] > [file]`

创建硬链接文件(源文件删除不影响目标文件)

`ln [target_file] [source_file]`

创建软链接文件(源文件删除会影响目标文件)

`ln  -s [target_file] [source_file]`

删除文件

`rm [file]`

以索引方式删除文件(乱码文件)

`ls -i`

`find -inum [index] -delete`

复制文件

`cp [source_file] [target_file]`

重命名文件

`mv [source_file] [target_file]`

给文件添加权限

`chmod [u/g/o/a]+[w/r/x] [file]`

给文件添加最高权限

`chmod 777 [file]`

### 目录和文件夹

Linux系统中各级目录含义如下图所示。

{% asset_img linux_dir.png Linux系统中各目录含义 %}

查看当前目录

`pwd`

切换到根目录

`cd /`

创建目录

`mkdir [dir]`

创建多级目录

`mkdir -p [dir1]/[dir2]/..`

复制目录

`cp -r [source_dir] [target_dir]`

查看目录结构(文件大小)

`du` 

`sudo apt install tree`

`tree`

删除目录

`rm -r [dir]`

### dotnet程序包更新

切换到root用户(使用`su - [username]`返回当前用户)

`sudo su -`

列出系统中所有进程的详细信息

`ps -ef | grep dotnet`

关闭进程

`sudo kill -9 [pid]`

查找进程文件夹

`ll /proc/[pid]`

进入进程文件夹并覆盖式上传程序包

`cd /opt/vhosts/...`

`rz -y`

解压程序包

`unzip -o [*.zip]`

启动程序

`nohup dotnet [app].Web.dll --urls=http://localhost:[port] &`


## 参考文档

- [如何使用WSL在Windows上安装Linux](https://learn.microsoft.com/zh-cn/windows/wsl/install)

- [VIM使用参考手册](https://vimcdoc.sourceforge.net/doc/editing.html)

- [GNU nano使用参考手册](https://www.nano-editor.org/dist/latest/nano.html)

- [Linux Tutorial for Beginners](https://info-ee.surrey.ac.uk/Teaching/Unix/index.html)