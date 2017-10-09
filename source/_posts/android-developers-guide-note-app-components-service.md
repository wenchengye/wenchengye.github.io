---
title: Android Developer Guide中的Service
subtitle: Android官方guide随笔 - App Components：Service
catalog: true
header-img: /img/header_img/android-note-header.jpg
tags:
  - Android
categories:
  - 读书笔记
date: 2017-08-04 15:23:41
---

## 展开点

* 在进程回收之后，系统什么情况下会重启Service。
* onRebind()的应用场景。
* 客户端服务端AIDL冲突的情况。
* Binder in Detail。
* 对象经过IPC方法传递后，是否还是同一个对象。


## Caution

* 如果不想Service被其他App使用，应该在Manifast里面将android:exported属性声明为false。
* 用户可以主动终止正在运行的Service，为了避免用户误杀，最好在`<service>`标签的android:description属性中描述Service的用途。
* onUnbind()回调是在**所有**Client都调用UnbindService()之后调用。
* 为了简单性尽量使用Messenger而不是AIDL，尽管Messenger是基于AIDL的。
* AIDL应当向后兼容（兼容旧版本）。
* 应当把AIDL的方法参数的in & out属性控制在最小权限，因为序列化的代价很大。
* 当IPC的客户端和服务端AIDL文件存在冲突时，IPC方法会抛出SecurityException异常。
* "四大组件"中，receiver不能bind Serivce；其他三位都可以，包括provider。
* 即便bindService()方法返回false，也需要主动调用unbindService()，否则Service不会停止运行。

## Services

根据启动方式，或者说执行方式的不同，Service可以被分为Scheduled、Started、Bound三类：

* Scheduled，通过JobScheduler启动的Serivce类型是Scheduled。JobScheduler从Android 5.0（21）开始被引入。
* Started，被startService()调用过的Service类型是Started；如果不主动结束(使用stopSelf()或者stopService())，Started的Service不会停止运行从而被正常销毁（除了进程回收）。
* Bound，被 bindService()调用过的Serivce类型是Bound；Bound Service用于IPC，当所有Client Component从Serivce Unbind之后Service停止运行从而被销毁。

Service可以即处于Started状态又处于Bound状态。

### 基础    

#### 重要回调

Service类几个重要的需要override的回调方法分别是onStartCommand()、onBind()、onCreate()、onDestroy()。
onCreate()和onDestroy()方法分别在Service创建&销毁时被调用，适用于setup和cleanup之类的工作。
onBind()在Service中是abstract的方法所以必须被继承，但如果不想支持bind就使onBind()返回null。

onStartCommand()会返回一个整数，代表Servcie应对进程回收的方式，返回值的值域有：

* START_NOT_STICKY，进程回收后除非有未处理的Intent(pending intent)，否则不再重启Servcie。
* START_STICKY，进程回收后重启Service，但重启时不会将最后一次的Intent作为参数传给onStartCommand()而是对Intent参数传null，如果有pending intent则传pending intent。
* START_REDELIVER_INTENT，进程回收后重启Service，并将最后一次的Intent作为参数传给onStartCommand()，如果有pending intent在这之后会一次传递pending intent。

#### 在manifest中声明   

在`<application>`标签下通过`<service>`标签声明Serivce，`<service>`标签中唯一必填的属性就是android:name来指明Service的class。
通过设置android:exported属性可以将Service声明为私有的，android:exported为false的Service仅能被UID相同的App访问；和Activity相同，android:exported的默认值在无intent-filter时为false，反之为true。

### 创建Started类型的Service   

使用Started类型的Service时通常有两个继承选择，继承Service或者继承IntentService。

#### 继承IntentService   

IntentService是系统提供的便利工具类，使用单独的线程响应startService()，使用者为一必要做的就是实现onHandleIntent()；如果要override其他方法，需要调用super。

IntentService做了以下的工作：

* 自动管理线程，使用一个单独的线程对每个传入onStartCommand()的Intent调用 onHandleIntent()。
* 自动管理声明周期，在所有onHandleIntent()执行结束后，调用stopSelf()。
* 提供了默认的onBind()方法返回null。
* 提供了默认onStartCommand()方法。

#### 继承Service

很多时候继承Service就像实现一个自己的IntentService一样，所以如果能用IntentService那么就不要自己实现它。

#### Start Service

Intent & startService()。

#### Stop Service

有一个带整形参数的关闭Service的方法Service.stopSelf(int)，这个参数和onStartCommand()传入的startId一致；当调用带参数stopSelf时，仅当使用最新的startId时Service才会被关闭。

### Foreground Service

通过`startForeground(int NOTIFICATION_ID, Notification notification)`方法使Service运行在前台；注意NOTIFICATION_ID不能为0。

### Service的生命周期

![service生命周期图](https://developer.android.google.cn/images/service_lifecycle.png)

当Service同时被Started和Bound时，只有两类个类型的生命周期都结束时，Service才会被销毁；即stopSelf()被调用**且**所有Client Component都unbind。

## Bound Service   

Bound Service最经典的应用场景是IPC，但可以使同进程内的Client-Server式访问更加规范。

### 基础   

Service为了支持bind，需要在onBind()方法中返回一个非空的IBinder对象；需要bindService的Component需要实现ServiceConnection来监听Bind状态的变化。

Android系统会缓存Service第一次返回的IBinder对象，在其他Component再去BindService时，系统直接返回缓存的IBinder并不再调用onBind。

### 提供IBinder   

可以选用以下几种方式提供IBinder：

* 继承Binder(注意不是实现IBinder)，适用于没有跨进程需求的情况，通过返回的IBinder在同一进程内直接进行函数调用。
* 使用Messenger，Service端Messenger使用一个Hanlder做初始化，通过Messenger.getBinder()方法返回IBinder对象，Client端发送的消息会被返回给Hanlder；Messenger将所有消息由单线程返回；Messenger也是基于AIDL的。
* 使用AIDL。

#### 继承Binder

同进程，有如一般的callback一样，如你所想。

#### 使用Messenger   

如果需要Service给bind Component返回数据，在bind Component发送的Message对象的replyTo字段中，添加一个Service to Client的Messenger，以实现双向通信。

### 发起Bind请求   

即便bindService()方法返回false，也需要主动调用unbindService()，否则Service不会停止运行。

如果Bind Component销毁时仍与Service保持Bound状态，系统会自动unbind，但不要依赖这个行为unbind。

### 注意事项   

* 对象的引用计数是跨进程的（啊，目前这句话没发提供指导意义）。

## AIDL(Android Interface Definition Language)

AIDL用于在IPC过程中，定义Client和Service端交互的接口。

AIDL接口对于Client而言是同步的函数调用，对于Service而言函数调用实际发生的线程分情况而不同：

* 当Client和Service处于同一个进程的时候，AIDL接口的执行和调用处于同一个线程，即调用AIDL接口的线程；没有IPC，情况和直接的function call一样。
* 当IPC发生时，AIDL接口的运行被分发到由系统维持的线程池中，系统为每个进程维护一个这样的IPC线程池。

如果AIDL接口用oneway修饰，IPC情况下接口调用会立刻返回，接口执行随后发生在Binder线程池中；同进程情况下，这个接口调用依然是同步的。

AIDL接口运行过程中的抛出的异常**不会**返回给调用者。

### 使用AIDL

Client和Serivce两个Application中必须存放.aidl文件。Android SDK会为每个.aidl文件生成IBinder接口。

使用AIDL的步骤包括：编写.aidl文件、实现aidl接口(Stub)、返回AIDL接口给Client(Stub`s implementation)。

#### 编写.aidl文件

每个.aidl文件职能保护一个aidl接口。

AIDL支持以下数据类型：

* 基础类型。
* String & CharSequence。
* List & Map，经过IPC之后List的实现均为ArrayList，Map的实现均为HashMap。
* 由AIDL生成的接口 & 由AIDL声明的Parcelable。

即使在同一个包中，在.aidl文件中引用其他类型也需要显示import。

注意，在接口参数列表中的非基础类型，需要标注传递方向(in & out)；.aidl文件中的注释会存在于生成的.java文件中。

#### 实现AIDL接口    

由.aidl文件生成的.java文件中，会存在名称为Stub并继承自Binder的抽象子类；通过继承Stub类来实现AIDL接口。

#### 返回AIDL接口

将Stub类的实现作为IBinder从onBind()中返回；Client通过YourServiceInterface.Stub.asInterface(IBinder)获得AIDL接口进行调用。

### 通过AIDL接口传递复杂类型

通过返回的复杂类型必须实现Parcelable接口才能跨越进程；同时还需要在.aidl文件中将类型声明为Parcelable。

### 调用IPC接口

在调用IPC接口的时候，需要捕获DeadObjectException(RemoteException)，在绑定断开时抛出；和捕获SecurityException，当Client和Service的AIDL文件有冲突时抛出。






