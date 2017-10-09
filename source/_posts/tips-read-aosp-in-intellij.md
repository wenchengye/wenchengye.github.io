---
title: 使用Intellij/Android Studio阅读Android源码
catalog: true
tags:
  - Android
categories:
  - 经验
date: 2017-09-10 16:14:51
header-img: /img/header_img/tips-header.jpg
---


最近想看看AOSP，对于Java／Android开发者来说，最习惯的方式就是在Intellij／Android Studio中阅读源码了，记录一下折腾的过程。

## 硬件准备

### 移动硬盘

因为使用256G的MBP，硬盘空间常年吃紧，把AOSP直接下载到硬盘实在太奢侈了。所以准备了一块移动硬盘来放代码，这样大概会拖慢编译/IDE读工程的速度，但随后证明我这个决定是正确的。

### 需要留多少硬盘空间

因为需要作大小敏感的磁盘分区，所以要有分区空间预算，先说结论，对于master分支而言，120G+。

在写这句话之前，我double-check了一下以防冤枉，AOSP文中写到"完成编译至少需要 25GB 空间"...
最开始我留了50G，master分支还没拉完就阵亡了...
然后按照*2规则，我留了100G，master拉完看了一眼用了60G+，但是编译时又阵亡了...
然后改成200G，编译完成109G左右...

## 下载源码

master分支源码在我写这句话的时候大概60G，下载源码按照AOSP本来十分简单，阻力主要来自于GFW。

### repo init失败

第一次执行repo init的时候大概要去google服务器上下载最新的repo工具，然而连不上对吧。

解决方法：

去清华开源镜像站下载repo工具
>`git clone https://gerrit-google.tuna.tsinghua.edu.cn/git-repo`   

用git-repo/repo文件替换~/bin/repo；将git-repo重命名为repo替换<*你的aosp更目录*>/.repo/repo。
这样repo init可以正确执行了。

### Git访问失败

拉源码的过程中肯定是要开代理的，但是Git似乎不使用全局代理，所以访问google source会失败。

解决方法：

设置Git代理
Git代理可以通过以下命令来设置：
>`git config --global https.proxy https://<*你的代理IP*>:<*你的代理端口号*>`   

实际上Git代理的配置写在~/.gitconfig文件中，也可以直接改这个文件。
>我的gitconfig文件：
>
>`[https]`
>    `proxy = https://127.0.0.1:49689`
>`[http]`
>    `proxy = http://127.0.0.1:49689`
>`[color]`
>    `ui = auto`

`git config --list`可以查看当前的git config配置。

## 编译源码   

我就读个源码，为什么还要编译？实际上如果不编译，就没有idegen.jar文件，进而也就没发用intellij打开。

### 使用MacPorts获取Make、GPG失败

第一次使用MacPorts的话，需要先执行`port sync`。
执行`port sync`可能会失败，原因还是GFW，可以修改/opt/local/etc/macports/sources.conf文件中的同步地址。
在sources.conf文件中默认地址为：
> `rsync://rsync.macports.org/release/tarballs/ports.tar [default]`

可以尝试将其改为：
> `http://www.macports.org/files/ports.tar.gz [default]`  

或者将ports.tar下载到本地解压缩，再将sources.conf配置为：
> `file://*ports.tar的解压缩路径*`

### make时出错 

我在make时抛出如下错误：

> build/core/base_rules.mk:130: *** external/webrtc/src/system_wrappers/
> source:MODULE.TARGET.STATIC_LIBRARIES.libwebrtc_system_wrappers already 
> defined by external/webrtc/src/system_wrappers/source.  Stop.

解决方法：
将NDK_ROOT从环境变量里暂时移除...

## 用Intellij/Android Studio打开工程

想用intellij/Android Studio打开工程，需要编译生成的idegen.jar文件如果你make成功之后，发现在/out/host/linux-x86/framework目录下没有生成这个文件，那么可以自己手动编译：

> `cd development/tools/idegen/`
> `mm`

这之后就会生成idegen.jar。

然后在在源码根目录执行：

> `development/tools/idegen/idegen.sh`

过一会就会在根目录下生成两个文件，android.iml和android.ipr，这就是Intellij/Android Studio所需要的文件。

通过android.ipr打开工程，会有很长的时间来建立索引。

### 个人感觉Android Studio的浏览速度比Intellij快一些，也许是错觉

## 闲话

* 整个下载和编译过程，可以称为无痛了，只是占用硬盘空间大，等待时间较长。
* repo工具很好，有时间要学一下；之前有人黑Git带支持超大代码库时会变得非常慢，将代码库分解是或许是解决之道。
* 良心网站[清华开源镜像站](https://mirror.tuna.tsinghua.edu.cn/)。





