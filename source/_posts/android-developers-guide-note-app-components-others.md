---
title: Android Developer Guide中的其他话题
subtitle: Android官方guide随笔 - App Components：Others
catalog: true
header-img: /img/header_img/android-note-header.jpg
tags:
  - Android
categories:
  - 读书笔记
date: 2017-08-21 17:19:18
---


## 展开点

* [Background Optimizations](https://www.youtube.com/watch?v=vBjTXKpaFj8&feature=youtu.be)。
*  foreground broadcast and receiver；FLAG_RECEIVER_FOREGROUND。
*  custom permission的定义App和使用App的安装顺序；如果在permission还没定义时使用会发生什么。
*  explicit intent with receiver。
*  local broadcast and receiver。
*  [Intelligent Job-Scheduling](https://developer.android.google.cn/topic/performance/scheduling.html)。
*  Resize App Widget。
*  ViewStub。
*  AppWidgetManager in Detail。
*  orderedBroadcast。 
*  [Launcher](https://android.googlesource.com/platform/packages/apps/Launcher3/)。

## Cautions

* 所有的系统Broadcast记录在Android SDK的BROADCAST_ACTIONS.TXT文件中。
* 通过context注册的receivers的作用时间和context的生命周期一样长，例如Activity和Application。

## Broadcasts

### 系统广播   

Android 7.0之后系统不再发ACTION_NEW_PICTURE和ACTION_NEW_VIDEO广播；CONNECTIVITY_ACTION只能用动态注册的receiver接受。

### 接收广播

App通过manifest的`<receiver>`标签生命receiver，package manager会在app安装的时候注册receiver，receiver仅在onReceive执行时处于active状态。

通过Context.registerReceiver方法动态注册receiver；也可以通过LocalBroadcastManager.registerReceiver动态注册local receiver。
local broadcast执行效率更高，无需跨越app的时候使用local broadcast。

#### goAsync()

调用goAsync()可以使receiver在onReceive返回之后仍处于active状态，处于active状态不会超过receiver的10秒限制，可以通过goAsync()返回的PendingResult对象的PendingResult.finish()方法终结receiver的active状态(在后台线程里)。

### 发送广播

使用sendBroadcast(Intent)或者LocalBroadcastManager.sendBroadcast。
Android 4.0之后，通过Intent.setPackage()限制哪些应用可以收到这次广播。

### 最佳实践

* 无需跨app时使用local broadcast，执行效率更高。
* 如果可以动态注册receiver就不要静态注册，以免拖慢系统。
* 确保广播的action是unique的。
* 不要在onReceiver里直接启动线程，除非使用goAsync()或者Service。
* 不要通过receiver启动activity。

## App Widget

App Widget就是可以被嵌入其他应用中的一块View。

### 基础

构建App Widget需要以下工作：

* AppWidgetProviderInfo，xml形式的metadata文件，用于配置AppWidget的属性。
* AppWidgetProvider的实现类。
* layout文件。
* AppWidget的configuration Activity，是AppWidet的可选组成部分。

### 在Manifest中  

* `<receiver>`标签：AppWidgetProvider继承自receiver，在manifest文件中通过`<receiver>`标签来声明AppWidgetProvider。
* `<intent-filter>`标签：`<intent-filter>`必须包含声明ACTION_APPWIDGET_UPDATE的`<action>`标签。
* `<meta-data>`标签：用于声明AppWidgetProviderInfo，androidLname值为android.appwidget.provider，android:resource为metadata的xml文件的位置。

### MetaData

AppWidgetProviderInfo包含了AppWidget的配置信息，通过res/xml下包含单独` <appwidget-provider>`标签的xml文件定义。

`<appwidget-provider>`的属性：

* minWidth & minHeight，描述widget占据的默认空间，系统会将其值近似到整cell。
* minResizeWidth & minResizeHeight，配合Android 3.1的缩放widget功能，描述widget的最小尺寸。
* updatePeriodMillis，调用AppWidgetProvider.update()的时间间隔，“0”代表从不。
* initialLayout，layout文件。
* configure，configuration Activity。
* previewImage，预览图。
* resizeMode，缩放方式例如"vertical"、"horizental"、"none"。
* widgetCategory，widget的显示位置例如"home_screen"或"keyguard"；在5.0后不支持锁屏widget。

Widget的update事件会将睡眠的设备唤醒，从而增加电量消耗。如果不需要再睡眠期间update widget可以使用AlarmMananger来代替（AlarmMananger不会唤醒设备）。

### Widget布局文件   

#### 设置margin 

widget需要和其他widget & app icon有一些margin，不然很丑。
在Android 4.0之后会自动给widget加入margin，所以targetSdkVersion=14。
如果需要兼容旧系统，可以在layout中设置padding，padding值在大于等于14时为0即可。

### 实现AppWidgetProvider类

需要实现的回调接口：

* onUpdate()，如果没有定义configuration activity，在第一次创建Widget的时候也会回调，所以需要负责setup工作。
* onAppWidgetOptionsChanged()，在widget实例第一次被放置或者widget被缩放时调用。
* onDeleted()，任意widget实例被删除时调用。
* onEnabled & onDisabled，第一个widget实例被创建时 & 最后一个widget实例被删除时调用。

AppWidgetProvider仅仅是个工具类，开发者也可以直接继承receiver来辅助widget。

### Configuration Activity

Configuration Activity会在Widget创建的时候被Widget host调用；Configuration Activity在manifest中必须包含action=ACTION_APPWIDGET_CONFIGURE的intent-filter。
Configuration Activity需要返回result；result和启动的Intent都需要包含
EXTRA_APPWIDGET_ID，代表widget id。

tip：在启动时就setResult中加入RESULT_CANCELED和EXTRA_APPWIDGET_ID，以防activity通过其他逻辑分支退出。

### 预览图 

Android模拟器包含应用"Widget Preview"，方便制作widget预览图。

### AppWidget显示列表

//TODO:加入展开列表

## App Widget Host

//TODO:加入展开列表

## Processes and Threads

默认情况下，同一个App的Component都是运行在同一个进程的主线程上。

通过Component标签的android:process属性可以使得同一个App的Component运行在不同进程中，或者使不同App的Component运行在同一进程中。
借助`<application>`的android:process属性设置App的默认进程名。

