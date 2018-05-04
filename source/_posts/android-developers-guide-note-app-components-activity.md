---
title: Android Developer Guide中的Activity
subtitle: Android官方guide随笔 - App Components：Activity
catalog: true
header-img: /img/header_img/android-note-header.jpg
tags:
  - Android
categories:
  - 读书笔记
date: 2017-08-03 15:30:44
---


## 展开点   

* [Things That Cannot Change](https://android-developers.googleblog.com/2011/06/things-that-cannot-change.html)。
* AsyncQueryHandler。
* [Multitasking the Android Way](https://android-developers.googleblog.com/2010/04/multitasking-android-way.html)。
* Activity‘s Task in Detail。
* allowTaskReparenting的应用场景。
* Parcel in Detail。
* [Document-centric model](https://plus.google.com/+DianneHackborn/posts/4QWEQgkB1v2)。
* ActivityManager.AppTask。

## Caution   
* Activity的android:name一旦被声明之后，再后续更新中不应改变，否则一些功能例如shortcut会出错。详见[Things That Cannot Change](https://android-developers.googleblog.com/2011/06/things-that-cannot-change.html)。
* Activity.isFinish()可以用来判断finish()方法是否被调用过（例如在onPause、onStop这些callback里面）。
* 如果startActivityForResult()启动的Activity异常退出，发出请求的Activity会收到RESULT_CANCELED。
* 由Intent指定的launchMode的优先级高于（会覆盖）在manifest中声明的launchMode。
* 7.0以上的系统中，通过Intent传递的数据过大会抛出TransactionTooLargeException异常；7.0以下仅在logcat中输出警告。这个限制的根源是，Binder transaction buffer通常有1MB左右的大小限制，每个进程有单独的Binder transaction buffer，buffer被进程内的所有binder transactions共享。


## Introduction to Activities    

这一节介绍了Activity的概念，给初学者关于Activity大致的印象。并对如何在Manifest中声明Activity进行说明，并简述了Activity的生命周期。

### Activity的概念

Activity是什么我们都是知道的。

### 在Manifest中声明Activity

通过在Manifest中声明`<activity>`标签来声明Activity。`<activity>`标签中唯一的必填属性是android:name；`<activity>`可以包含`<intent-filter>`标签；可以通过`<activity>`标签的android:permission属性来声明使用activity所需的权限。

## The Activity Lifecycle

管理Activity生命周期的关键，是在正确的生命周期回调中作正确的事。

本节介绍了Activity生命周期的概念、各个生命周期回调、提及了Activity状态和系统杀进程间的关系、以及保存Activity状态的方法。

### Activity生命周期的概念   

Activity生命周期包括了Activity的状态转移和6个核心回调：onCreate()、onStart()、onResume()、onPause()、onStop()、onDestroy()。

家喻户晓的Activity状态转移图：   
![Activity状态转移图](https://developer.android.google.cn/guide/components/images/activity_lifecycle.png) 

### 生命周期回调   

**onCreate**
当Activity进入*Created*状态之后，onCreate被回调。savedInstanceState会做为参数传入onCreate中，和传入onRestoreInstanceState()的savedInstanceState是同一个Bundle对象。
在onCreate中，适宜做一些初始化工作（初始化变量、启动线程、初始化UI对象）以及调用setContentView()。

**onStart**
当Activity进入*Started*状态之后（进入可视范围），onStart被回调。
在onStart中，适宜注册各类监听（listener和Receiver）、同步UI；重建在onStop中释放的对象。

**onResume**
当Activity进入*Resumed*状态之后，即成为当前活跃的Activity之后，onResume被回调。
在onResume中，适宜启动动画，以及Activity进入活跃状态才开启的工作；重建在onPause中释放的对象。

**onPause**
当Activity进入*Paused*状态，即离开活跃状态，onPause被回调。
在onPause中应该停止动画、暂停音视频播放等；同时释放在Paused状态下不需要的资源（例如sensors）以节约耗电。

**onStop**
当Activity进入*Stopped*状态，即完全不可见状态，onStop被回调。
在onStop中应该释放所有可能造成内存泄露的资源，因为在进程被系统回收时onDestroy不保证被调用；同时在onStop中适宜进行结束前的持久化工作。 

**onDestroy**
在Activity被销毁之前回调，在这里可以释放在onStop没被释放的资源（那些不会泄露的资源）。

### 进程回收与Activity状态

Android系统的资源回收是以进程为单位，不会单独作用于Activity。但是Activity状态回影响进程回收优先级，进程回收优先级从高到低为 Destroyed > Stopped > Pauseed > Created & Started & Resumed。

### 启动其他Activity

onActivityResult的参数ResultCode可以使用RESULT_CANCELED、RESULT_OK以外的自定义值，自定义的ResultCode应当大于整型常量RESULT_FIRST_USER。

当Activity A 启动 Activity B时，生命周期回调顺序如下：   
Activity A的onPause() -> Activity B的onCreate()、onStart()、onResume() -> Activity A的onStop()。

### 保存和恢复Activity状态

系统会对Activity进行状态保存及恢复的情况有两种，因进程回收销毁Activity和因configuration变化销毁Activity。
当Activity的状态保存被触发时，系统会用一个Bundle对象（即SaveInstanceState）来保存Activity状态，以便再次进入Activity时通过这个Bundle对象恢复Activity状态。
依靠默认行为，Activity会保存和恢复View的状态。

Activity通过onSaveInstanceState()方法保存状态，重写它时需要调用super，super方法里会进行保存View状态发生的工作。

SaveInstanceState会作为参数传递给onCreate()和onRestoreInstanceState()。onRestoreInstanceState()的调用顺序在onStart()之后，重写它时需要调用super，和onSaveInstanceState()同理。

onSaveInstanceState()大概是在onPause和onStop之间调用，onRestoreInstanceState()在onStart()之后被调用。

## Activity State Changes

本节列出了一些会造成Activity状态发生改变的情况。

* Configuration change，会引发Activity的销毁和重建，为了应对Configuration change应该实现onSaveInstanceState()来保存恢复状态。
* 7.0以上推出的multi-window模式。
* 新的Activity或Dialog被激活。
* 用户按下Back按键，按照默认行为会销毁当前Activity（如果没有Fragment从中作梗）。

## Tasks and Back Stack

本节讲解Task的概念，以及一些通过Manifest和Intent Flag变更Task和Back Stack默认行为方法。

### 定义Launch Mode

定义Activity的Launch Mode的方式有两种，在Manifest中或通过Intent Flag。在Intent Flag中定义的Launch Mode会覆盖Manifest中的定义。

Manifest中`<activity>`标签的launchMode属性可以定义著名的四种launchMode：

* standard，就如同Android系统的默认行为，也是launchMode的默认值。
* singleTop，Activity如果在当前Task的顶部，则不会新建Activity而是调用已有Activity的onNewIntent()。
* singleTask，Activity会被创建在一个新的Task中，但如果Activity已经存在于某个Task中则不会新建Activity而是调用已有Activity的onNewIntent()。同一时间singleTask的Activity仅会有一个实例。
* singleInstance，跟singleTask一样除了Activity会独占Task。

通过Intent Flag中可以定义如下Launch Mode：   

* FLAG_ACTIVITY_NEW_TASK，和singleTask一致。
* FLAG_ACTIVITY_SINGLE_TOP，和singleTop一致。
* FLAG_ACTIVITY_CLEAR_TOP，如果Activity存在于当前Task则不会新建，并且销毁所有在Back Stack中位于其之上的Activity，常与FLAG_ACTIVITY_NEW_TASK一起使用。如果Activity的launchMode是standard的，那么Activity会在Back Stack的原位置上销毁并重建。

### affinity   

affinity代表Activity倾向于存在的Task，默认情况下同一个App的所有Activity的affinity相同。
开发者可以通过`<activity>`标签中的taskAffinity的属性来修改affinity，Activity的affinity的默认值为包名。

affinity发生作用的两个场景：   

* 当以 FLAG_ACTIVITY_NEW_TASK 启动Activity时，Activity会在一个“新”Task中启动。实际情况是，如果已经存在一个和Activity具有相同affinity的Task存在，那么Activity会在那个Task中启动；否则才会新建一个Task。
* 当Activity的 allowTaskReparenting 属性被设为true时，如果Activity原本不存在于其affinity Task中，当它的affinity Task激活时，Activity会被移动到affinity Task中。

### 清理Back Stack   

当Task太久没被激活时，系统会清理除了Task Root以外的所有Activity（真的吗？），有一下几个标志位可以控制清理Back Stack的行为。

* alwaysRetainTaskState，当Task的root Actvity包含该标志位时，默认的清理Back Stack行为不会发生。
* clearTaskOnLaunch，和 alwaysRetainTaskState 刚好相反，当Task的root Actvity包含该标志位时，一旦用户离开这个Task，默认的清理Back Stack行为马上发生。
* finishOnTaskLaunch，该标志作用于单个Activity，当用户离开Task之后，Task中包含该标志位的Activity会从Back Stack中移除。

## Processes and Application Lifecycle

本节介绍进程回收时，系统判定不同进程的优先级。

进程重要程度降序如下：

* 前台进程(foreground process)，包括处于Resumed(Created、Started)状态的Activity；生命周期回调正在执行的Activity、Service；onReceive()正在执行的BroadcastReceiver。
* 可见进程(visible process)，包括处于Paused状态的Activity；由Service.startForeground()启动的Service等。
* 服务进程(service proces)，通过startService()启动的Service。当Service运行超过30分钟之后，服务进程可能会被降级成缓存进程(cached process)。
* 缓存进程(cached process)，包括处于Stopped状态的Activity。缓存进程由LRU列表管理，随时可能会被杀死。
* 空进程(empty process)。

## Parcelables and Bundles   

Parcelable对象和Bundle是设计用于在进程间传递信息的数据结构，Android的进程间通信是通过Binder transaction完成。

## Recents Screen

TODO。



