---
title: 反编译Android厂商的ROM
catalog: true
tags:
  - Android
categories:
  - 经验
date: 2018-05-04 15:28:59
header-img: /img/header_img/tips-header.jpg
---

为了解释某些厂商ROM反直觉的现象，或者绕过厂商ROM埋的坑，有时反编译ROM源码是一条不错的途径。

### 获取Rom包

google 搜索 + 下载；也许需要百度网盘。

### 解压System.img文件

参考[解压 Android 系统中的 system.img](https://www.jianshu.com/p/db70835d41c8)

如果Rom包中不包含system.img文件，而是有system.new.dat、system.patch.dat、system.transfer.list三个文件，需要使用[sdat2img工具](https://github.com/xpirt/sdat2img)生成system.img。

使用file命令确定system.img文件的文件类型，如果是“Linux rev 1.0 ext4 filesystem data”文件可以使用[ext4fuse工具](https://github.com/gerard/ext4fuse)在MacOS上挂载。

成功挂载之后就可以读取system.img中的文件了。

### 在System.img中寻找代码文件

不同rom的system.img中的文件布局各异，以vivo X9S为例，代码文件通常是在/framework目录下的.odex文件或.oat文件。

### 反编译.odex或.oat文件成.jar文件

.odex&.oat -> .smali -> .dex ：使用[smail工具](https://bitbucket.org/JesusFreke/smali/overview)，baksmali.jar可以将.odex&.oat转成.smali，smali.jar可以将.smali转成.dex。
.dex -> .jar：dex2jar工具。

有了.jar文件之后就可以开心的看厂商ROM都留了哪些坑了。


