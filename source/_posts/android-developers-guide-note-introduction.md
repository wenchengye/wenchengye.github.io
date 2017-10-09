---
title: Android Developer Guide中的Android概览
subtitle: Android官方guide随笔 - Introduction
header-img: "/img/header_img/android-note-header.jpg"
catalog: true
tags:
  - Android
categories:
  - 读书笔记
date: 2017-07-18 15:51:38
---

## 展开点

* Android app、Android进程、Linux UID、Android VM的对应关系。
* 令不同app使用同一个Linux UID的方法与用处（文中提到了share UID的app需要有相同签名）。
* Android各个组件状态(background、foreground、service是否被bind)和进程回收优先级的关系。
* JobScheduler、JobScheduler和Service的关系。
* [Android compatibility program](https://source.android.com/compatibility/overview)，Android系统对设备的要求。
* PackageManager的职责。
* google play filter
* [Google I/O 2015 - Android M Permissions](https://www.youtube.com/watch?v=f17qe9vZ8RM)。
* SYSTEM_ALERT_WINDOW和WRITE_SETTINGS这两个属于Special Permissions，不是normal or dangerous permissions。
* [Android系统的安全模型](https://source.android.com/security/)。
* 实验自动权限调整，查看自动加入的权限会否在安装时显示，实验自动权限调整是怎么影响dangerous权限的。
* 自定义权限和签名的关系（相同的签名的App可以自动获取权限？）& `<permission>`标签android:protectionLevel属性的取值，signature。
* 如何做验证签名，App-to-App，App-to-Server。
* Context.registerReceiver()时添加权限限制的方法。
* Context.checkCallingPermission()的用法（在IDL中的使用）。
* URI Permission的生命周期。
* URI Permission。
* 自定义Permission Group的方法与应用场景。

## Caution   

* (出于安全原因)对Service不能使用implicit intent
* `<user-feature>`标签中的feature IDs，定义在PackageManager的String类型静态常量中；可以调用PackageManager.hasSystemFeature()来查询feature是否可用。
* 某些`<user-permission>`会隐含`<user-feature>`，例如"android.permission.BLUETOOTH"会隐含"android.hardware.bluetooth"（PackageManager.FEATURE_BLUETOOTH）。如果需要避免`<user-permission>`去影响google play filter，需要显示声明`<user-feature>`并将require设为false。
* normal permission的permission group作用不大。
* 请求dangerous permission时的系统dialog***不能***自定义。
* Manifest类定义了permission常量(String)和permission group常量(String)。
* 如果两个不同App需要share UID的话，它们需要使用相同的签名；share UID通过manifest文件中的`<manifest>`标签的sharedUserId属性来定义。
* App A **签名级别**的自定义权限，App B需要有与A相同的签名才能获得。
* Android系统不允许不同的App(原文是package)定义相同名称的permisson，除非它们(apps)有相同的签名。如果已经安装一个定义permisson的App，那么再安装另一个定义同名permisson的App(不同签名)会安装失败。


## Application Fundamentals

首先介绍Android App和Android系统的关系。

* 一个Apk文件包括了一个Android App的全部内容。
* 每个Android App在Android Linux上使用独立UID，通过Linux权限系统来实现安全性（私有文件不会被其他app访问）。
* 每个Android进程运行在单独的Android VM上，每个Android app运行在自己的进程上，Android app所属的进程由Android系统负责。

然后介绍Android App所包含的几类内容。

* app components，即Android组件，定义了App的功能。
* manifest文件，声明App包含的所有app components；声明App对设备的筛选要求(sdk、软件&硬件功能等)。
* 资源，把资源和代码分离的重要目的是，便于在不同的系统配置下使用不同的资源。

文章的之后几节分类介绍了Android App的各类内容。

### *app components*   
“四大组件”。

### *manifest*   
介绍manifest包含的重点功能。

* 声明App包含的app components，通过`<activity>`、`<service>`、`<receiver>`、`<provider>`标签。
* 声明每个App component能力（即能执行哪些Action），通过intent-filter。
* 声明App对设备能力的筛选要求（sdk、软件&硬件功能等）。

### *资源*   
概述而已。

## Device Compatibility

支持Android操作系统的设备多种多样，App应该支持尽量多的设备；同时也有方法对App可以被安装到的设备进行筛选（借助Google Play）。本章介绍App筛选设备的方式。

### *基于技术原因筛选设备*   
借助google play，App可以依赖以下条件筛选设备，设备功能(device feature)、系统版本、屏幕规格。
这些筛选条件声明在manifest文件中。

* 设备功能，在mainfest中通过`<uses-feature>`标签增加对设备功能的筛选条件（例如需要蓝牙、重力感应、系统支持widget等）。
* 系统版本，在mainfest中通过`<uses-sdk>`标签的minSdkVersion属性和targetSdkVersion属性来筛选系统版本。minSdkVersion表示可安装的最低版本；targetSdkVersion表示App完全适配的最高版本，要求比targetSdkVersion更高版本的系统对App行为进行向前兼容。
* 屏幕规格，不能通过屏幕规格筛选设备，请进行适配。

### *基于非技术原因筛选设备*   
在Google Play控制台中，可以对App可安装的设备增加更多的条件，例如地区、年龄等，通常出于商业&产品的顾虑。
基数技术原因的筛选条件通常声明在Apk文件中（manifest）；基于非技术原因的筛选条件通畅声明在google play控制台。

## System Permissions——Requesting Permissions    

本章介绍如何请求Android系统的标准权限，下一章介绍如何自定义权限。

### *请求权限*   

App需要请求的权限都需要通过`<uses-permission>`标签在manifest文件中声明（无论normal还是dangerous），其中normal权限会自动获得；dangerous权限则需要用户显示授予。

大多情况下，违反权限规则的接口调用（还未请求权限就使用权限），会抛出异常&打印错误日志。   

### *动态请求权限*   

Dangerous权限和动态权限授予的功能是从6.0（23）加入Android系统的，所以Android对动态权限的兼容方式如下：

* App的targetSdkVersion大于等于23***且***系统大于等于6.0时，App需要在运行时请求dangerous权限（请求之后，系统弹出权限授予dialog，用户选择后回调App）；dangerous权限可以随时被剥夺（在 Settings -> Apps 中）。
*  App的targetSdkVersion小于等于22***且***系统大于等于6.0时，dangerous权限在App安装时请求用户授予，否则不能被安装；dangerous权限可以随时被剥夺（在 Settings -> Apps 中）。
* 系统小于等于5.1时，dangerous权限在App安装时请求用户授予，否则不能被安装；当系统小于等于5.1时dangerous权限只能在删除App之后被剥夺。

所以即便App的targetSdkVersion小于等于22，也要测试App在没有所需权限时能否适当的运行，因为系统大于等于6.0后用户可以在安装后剥夺dangerous权限。

使用ContextCompat和ActivityCompat中的方法来检查动态权限和请求动态权限。

如果用户在显示授予dialog拒绝授予权限并且勾选“don't show again”，再次请求这个权限会直接被拒绝（不弹出dialog）。

### *自动权限调整*   

当Android系统版本更新之后，可能会加入新的权限种类。当App的targetSdkVersion较小，并且后续Android版本加入新的权限种类时，Android会自动为App加入新的权限声明（就像App在manifest里声明了这些新权限一样）。

自动加入的权限，也会被列在Google Play的权限列表中。   

这种自动权限调整的行为很奇怪对不对，新权限不一定需要对不对，那么及时更新App的targetSdkVersion哦。（呵呵...）

### *查看权限列表*   
adb shell pm list permissions；Settings -> Apps。

### *permission group*   

对于同组的dangerous权限，如果App已经获得了其中一个，请求组内其他的权限时就无需用户显示授权，**但是在App代码中仍然需要请求这些同组权限**。

## System Permissions——Defining Permissons   

这一章简单介绍自定义权限，主要描述定义权限和要求权限的方式。

### *背景知识*   

自定义权限的背景知识是，Android系统包括UID和签名的安全模型。

### *定义权限*   

通过在mainfest文件中定义`<permission>`标签来定义权限，`<permission>`标签包含了protectionLevel、permissionGroup、label、description等属性。

* protectionLevel是必填字段，定义了权限类型，其值域包括normal、dangerous、signature、signatureOrSystem等。
* permissionGroup选填，仅在dangerous权限时有意义(大概)。可以通过`<permission-group>`可以自定义permissionGroup，不过大多数情况应该使用系统标准的permissionGroup。
* label和description用于提供权限的文字描述，按照惯例label为权限名字；description分为两句话，一句描述权限，一句恶意应用获得这个权限能造成的危害。

### *自定义权限的建议*   

* 小心评估是否需要自定义权限。
* 如果需要设计一组互相协作App，尽量让每个permission在这些App中只被定义一次（如果这些App的签名不同，那么必须只被定义一次）。
* 如果权限尽在共享签名的App之间使用，则无需自定义权限，尽在跨App请求时进行签名验证。
* 使用单独的App，不提供任何功能，仅定义权限。

### *要求权限* 

在manifest文件中的四大组件标签中声明android:permisson属性，可以要求在访问该组件时所需的权限。

* Activity的权限检查发生在Context.startActivity()和 Activity.startActivityForResult()调用时。无权限会抛出异常。
* Service的权限检查发生在Context.startService(), Context.stopService()和Context.bindService() 调用时。无权限会抛出异常。
* BroadcastReceiver 的权限检查发生在 Context.sendBroadcast() 之后，BroadcastReceiver 的权限定义为  是否有权限向 BroadcastReceiver 发送广播。无权限时不会抛出异常，仅仅是广播无效。
* ContentProvider 的权限分为 android:readPermission 和 android:writePermission ，在请求 ContentProvider 是检查权限。无权限会抛出异常。

### *广播接收权限*   

出了定义是否有权限向 BroadcastReceiver 发送广播，还可以定义 BroadcastReceiver 是否有权限接受广播。在 Context.sendBroadcast() 方法中加入 receiverPermission（String）参数可以定义接收者所需权限。

### *其他要求权限方法*   

 Context.checkCallingPermission()、Context.checkPermission(String, int, int)、Context.checkPermission(String, int, int)。

### *URI权限*   

通过URI权限，可以赋予App通向ContentProvider的特定URI的临时权限（尽管App不具有访问 ContentProvider的权限）。

通过Intent.FLAG_GRANT_READ_URI_PERMISSION或 Intent.FLAG_GRANT_WRITE_URI_PERMISSION向activity发Intent（包括返回result），activity将被赋予Intent所包含data URI的 URI权限。Context.grantUriPermission()方法也可以赋予URI权限，但尽量使用Intent来赋予URI权限，因为通过Intent方法赋予的URI权限有自动过期特性。

支持的URI权限的ContentProvider需要在`<provider>`标签中定义grantUriPermissions属性；或在`<provider>`标签先定义`<grant-uri-permissions>`标签。




















