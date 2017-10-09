---
title: Android Developer Guide中的Manifest
subtitle: Android官方guide随笔 - Manifest
catalog: true
header-img: /img/header_img/android-note-header.jpg
tags:
  - Android
categories:
  - 读书笔记
date: 2017-09-09 17:57:22
---


# 展开点   

* `<activity-alias>`。
* `<instrumentation>`。
* [Google Cloud（backend）](https://cloud.google.com/tools/android-studio/app_engine/run_test_deploy)。
* App自身组件声明的permission对本app是否有限制作用；App自己定义的permission对本app是否有限制作用。

## App Manifest 

Manifest文件包含了App向系统注册的信息，相当于系统如何使用这个App的参考手册。

### Manifest文件书写惯例

#### 标签使用惯例 

* 必要标签，只有`<manifest>`和`<application>`标签在Manifest文件中必须有且仅有一个。
* 标签顺序，大部分平级标签是无序的，除了两个例外情况。
    - `<activity-alias>`标签必须在其引用的`<activity>`标签之后。
    - `<application>`标签必须是`<manifest>`标签下的**最后一个**标签。

#### 属性使用惯例

除了`<manifest>`中的几个属性，所有属性名都包含android:前缀。

#### 引用Java Class

在Manifest中引用Java Class需要使用包名+类名的完全格式；如果Java Class引用以`.`符号开头，会用`<manifest>`的packageName属性来补全`.`之前部分。
如果component标签中没有在android:name指定subclass，系统会使用baseClass来创建对象(即Activity，Service)。

#### 引用资源

和在资源文件中引用资源相同，格式为`@[package:]type/name`或者`?[package:]type/name`(in theme)。其中package必须是App包名或者`android`。

#### 字符串值

Manifest中的纯字符串值需要使用`\\`作为转意符，例如`\n`需要写作`\\n`。

### Manifest文件中的功能

#### Intent filters

见[Intent相关笔记](http://blog.overspark.me/2017/07/26/android-developers-guide-note-app-components-intents/)

#### Icon和Label

很多表情都包含android:icon和android:label属性，这两个属性的默认值会继承自父标签。这两个属性的默认值继承树的根节点一般是`<application>`标签(应用Icon和应用名)。有时候一些抽象标签，例如`intent-filter`的android:icon和android:label属性会被某些系统UI使用，例如Setting App。

### Manifest结构一览表

`<?xml version="1.0" encoding="utf-8"?>`

[`<manifest>`](https://developer.android.com/guide/topics/manifest/manifest-element.html)
 
&emsp;&emsp;[`<uses-permission />`](https://developer.android.com/guide/topics/manifest/uses-permission-element.html)   
&emsp;&emsp;[`<permission />`](https://developer.android.com/guide/topics/manifest/permission-element.html)   
&emsp;&emsp;[`<permission-tree />`](https://developer.android.com/guide/topics/manifest/permission-tree-element.html)   
&emsp;&emsp;[`<permission-group />`](https://developer.android.com/guide/topics/manifest/permission-group-element.html)   
&emsp;&emsp;[`<instrumentation />`](https://developer.android.com/guide/topics/manifest/instrumentation-element.html)   
&emsp;&emsp;[`<uses-sdk />`](https://developer.android.com/guide/topics/manifest/uses-sdk-element.html)   
&emsp;&emsp;[`<uses-configuration />`](https://developer.android.com/guide/topics/manifest/uses-configuration-element.html)  
&emsp;&emsp;[`<uses-feature />`](https://developer.android.com/guide/topics/manifest/uses-feature-element.html)   
&emsp;&emsp;[`<supports-screens />`](https://developer.android.com/guide/topics/manifest/supports-screens-element.html)   
&emsp;&emsp;[`<compatible-screens />`](https://developer.android.com/guide/topics/manifest/compatible-screens-element.html)    
&emsp;&emsp;[`<supports-gl-texture />`](https://developer.android.com/guide/topics/manifest/supports-gl-texture-element.html)   

&emsp;&emsp;[`<application>`](https://developer.android.com/guide/topics/manifest/application-element.html)   

&emsp;&emsp;&emsp;&emsp;[`<activity>`](https://developer.android.com/guide/topics/manifest/activity-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<intent-filter>`](https://developer.android.com/guide/topics/manifest/intent-filter-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<action />`](https://developer.android.com/guide/topics/manifest/action-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<category />`](https://developer.android.com/guide/topics/manifest/category-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<data />`](https://developer.android.com/guide/topics/manifest/data-element.html)  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`</intent-filter>`](https://developer.android.com/guide/topics/manifest/intent-filter-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<meta-data />`](https://developer.android.com/guide/topics/manifest/meta-data-element.html)   
 &emsp;&emsp;&emsp;&emsp;[`</activity>`](https://developer.android.com/guide/topics/manifest/activity-element.html)   

&emsp;&emsp;&emsp;&emsp;[`<activity-alias>`](https://developer.android.com/guide/topics/manifest/activity-alias-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<intent-filter> . . . </intent-filter>`](https://developer.android.com/guide/topics/manifest/intent-filter-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<meta-data />`](https://developer.android.com/guide/topics/manifest/meta-data-element.html)   
&emsp;&emsp;&emsp;&emsp;[`</activity-alias>`](https://developer.android.com/guide/topics/manifest/activity-alias-element.html)   

&emsp;&emsp;&emsp;&emsp;[`<service>`](https://developer.android.com/guide/topics/manifest/service-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<intent-filter> . . . </intent-filter>`](https://developer.android.com/guide/topics/manifest/intent-filter-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<meta-data/>`](https://developer.android.com/guide/topics/manifest/meta-data-element.html)   
&emsp;&emsp;&emsp;&emsp;[`</service>`](https://developer.android.com/guide/topics/manifest/service-element.html)   

&emsp;&emsp;&emsp;&emsp;[`<receiver>`](https://developer.android.com/guide/topics/manifest/receiver-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<intent-filter> . . . </intent-filter>`](https://developer.android.com/guide/topics/manifest/intent-filter-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<meta-data />`](https://developer.android.com/guide/topics/manifest/meta-data-element.html)   
&emsp;&emsp;&emsp;&emsp;[`</receiver>`](https://developer.android.com/guide/topics/manifest/receiver-element.html)   

&emsp;&emsp;&emsp;&emsp;[`<provider>`](https://developer.android.com/guide/topics/manifest/provider-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<grant-uri-permission />`](https://developer.android.com/guide/topics/manifest/grant-uri-permission-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<meta-data />`](https://developer.android.com/guide/topics/manifest/meta-data-element.html)   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[`<path-permission />`](https://developer.android.com/guide/topics/manifest/path-permission-element.html)   
&emsp;&emsp;&emsp;&emsp;[`</provider>`](https://developer.android.com/guide/topics/manifest/provider-element.html)   

&emsp;&emsp;&emsp;&emsp;[`<uses-library />`](https://developer.android.com/guide/topics/manifest/uses-library-element.html)   

&emsp;&emsp;[`</application>`](https://developer.android.com/guide/topics/manifest/application-element.html)   

[`</manifest>`](https://developer.android.com/guide/topics/manifest/manifest-element.html)   