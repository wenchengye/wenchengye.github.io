---
title: Android Developer Guide中的Intent
subtitle: Android官方guide随笔 - App Components：Intent
catalog: true
header-img: /img/header_img/android-note-header.jpg
tags:
  - Android
categories:
  - 读书笔记
date: 2017-07-26 18:13:13
---

## 展开点  
* MIME Type & URI结构。
* Intent Flags。
* CATEGORY_DEFAULT的意义。 
* 如何根据content provider类型URI推断Intent Data的MIME Type，对Intent Resolve有什么影响。
*  ClipData的用处。
* DocumentsProvider。

## Caution   
* 不要为Service添加Intent filter，不要使用implicit intent启动Service。在5.0以上，使用implicit intent调用bindService()，系统会抛出异常。
* Intent的静态常量中定义了一些通用的action、category和extra key字符串；Settings的静态常量定义了一些打开Settings App界面的action。
* Intent的setData方法和setType方法会相互将对方置空（setData会置空mType，setType会置空mData，优秀的设计...）；同时set需要调用setDataAndType。
* 在使用implicit intent时使用Intent.resolveActivity()检查有没有App响应这个Intent；如果没有Activity适合Intent的话，startActivity()会抛异常。
* （对于Activity而言）为了能收到implicit intent，在intent-filter中必须声明CATEGORY_DEFAULT，因为startActivity()和startActivityForResult()会默认添加CATEGORY_DEFAULT。


## Intents and Intent Filters   

Intent用于Component之间的交流、传递信息，依赖Intent的Component包括Activity、Servcie和Recevier（不包括Provider）。

### Intent的分类 

Intent分为Explicit intents和Implicit intents两类，其区别就是Intent中是否包含ComponentName。

### Intent组成    

Intent中包含的主要成分如下：

* Component name，区分Explicit intents和Implicit intents成分。Intent中的Component name属性是一个ComponentName对象。
* Action，表明Intent作希望执行的操作。Android系统有很多预定义的Action，同时Action通常也决定了Intent的结构(data、extra等)。
* Data，由URI和MIME type组成，做为Action的补充，代表Action的目标数据&数据类型。如果URI是content: URI，MIME type往往可以由URI推断出。
* Category，表明希望哪些类型的Component响应Intent。
* Extras，extras是一个Bundle对象，传递额外数据。
* Flags，一些标志位用于告知Android系统如何使用这个Intent。

Component name、Action、Data和Category这四种成分会影响Intent Resolve。 

借助Intent.createChooer()方法，可以在发送implicit intent时强制显示chooer dialog（尽管用户之前在chooer dialog选择了默认处理intent的App）。

### 接收implicit intent   

组件通过在manifest文件中定义`<intent-filter>`来接收implicit intent，组件可以定义多个`<intent-filter>`。

explicit intent将无视`<intent-filter>`以及由`<intent-filter>`定义的intent resolve规则。

`<intent-filter>`由三类标签组成：`<action>`、`<data>`、`<category>`。每类标签可以有一个或多个（或没有）。
当你定义多个`<action>`、`<data>`或`<category>`时，你所维护Component需要能够处理由这些标签任意组合而成的Intent；如果你只想处理特定组合，定义多个`<intent-filter>`。

在manifest中Component的android:exported属性可以控制该Component是否能被其他App调用。当android:exported为false时，Component仅能被相同UID的App调用。没有Intent-filter的Activity的android:exported属性默认值是false。

### 使用PendingIntent   

>The primary purpose of a PendingIntent is to grant permission to a foreign 
>application to use the contained Intent as if it were executed from your 
>app's own process.

### Intent resovle规则   

Intent若要通过Intent filter的检验，则需要通过Action检验、Data检验、Category检验三部分。

* Action检验。
    - 当Intent包含某个Intent filter中声明的action时可通过Action检验。
    - 如果Intent filter没有声明action，任何Intent都无法通过Action检验。
    - 如果Intent中不包含action，那么只要Intent filter声明了任意action，就可通过Acton检验。
* Category检验。
    - 当Intent中定义的所有category都在Intent filter中被声明，可通过Category检验。
* Data检验。Data检验包括URI检验和MIME type检验；URI检验仅检验Intent filter中定义的部分。
    - 如果Intent不包含URI和MIME type，仅在Intent filter没声明data条件时通过。
    - 如果Intent仅包含URI，仅在URI和Intent filter匹配，且Intent filter没声明type条件时通过。
    - 如果Intent仅包含MIME type，仅在type和Intent filter匹配，且Intent filter没声明URI条件时通过。
    - 如果Intent包含URI和MIME type，在URI和type均匹配时通过；或者在type匹配，Intent的URI为content:、file:类型，且Intent filter没有定义URI条件时通过。


## Common Intents   

介绍一些Android系统预先定义的Action，以及相应的Intent结构和经由onActivityResult返回Intent的结构。

### 通过adb命令测试Intent   
adb shell am start -a `<ACTION>` -t `<MIME_TYPE>` -d `<DATA>` -e `<EXTRA_NAME>` `<EXTRA_VALUE>` -n `<ACTIVITY>`。

