---
title: Android Developer Guide中的ContentProvider
subtitle: Android官方guide随笔 - App Components：ContentProvider
catalog: true
header-img: /img/header_img/android-note-header.jpg
tags:
  - Android
categories:
  - 读书笔记
date: 2017-08-21 17:19:05
---


## 展开点   

* [AbstractThreadedSyncAdapter](https://developer.android.google.cn/training/sync-adapters/index.html)。
* search framework & Search Suggestions with ContentProvider；SearchRecentSuggestionsProvider。
* ContentProviderOperation & ContentProvider Batch Access。
* LevelDB。
* [protobuf](https://github.com/google/protobuf)。
* 使用ContentProvider分享文件。
* ContentProvider's Temporary permission。
* 如何限制列的访问。
* ContentProvider执行时发生异常，跨进程抛出异常。

## Caution

* Android Framework提供的ContentProvider的定义(Contract)存放于android.provider包里。
* Uri & Uri.Builder可以用来构建结构化Uri；ContentUris包括一些静态方法来处理content:类型的Uri中包含的id字段。
* 不要在主线程上执行ContentProvider操作，这些操作时同步的。
* 在SelectionClause中使用"?"通配符，可以避免SQL注入。
* 为了在SimpleCursorAdapter中使用Cursor，Cursor中必须包含_ID列。
* 可以使用BLOB类型字段，来实现兼容多种数据格式的表。
* 访问ContentProvider的getType()不需要权限，即便ContentProvider的exported为false。

## Content Provider Basics

这一节介绍如何使用已有的ContentProvider。

### 概述

#### 访问ContentProvider

Client Component通过ContentResolver来访问ContentProvider，ContentResolver包含了和ContentProivder一致的接口，Client Component通过ContentResolver的接口来访问ContentProivder实例的接口。

#### Content URI

Content URI的用于定位ContentProvider中的数据表或者数据行，其结构为`content://authority/path/id`。
ContentResolver通过authority在系统中解析Provider，ContentProvider通过path来选择将被访问的表。

### “查”    

“查”大致分为两步，申请权限(读权限) & 调用接口。

#### 查询操作的结果 

Provider的查询结果以Cursor对象的形式返回，如果没有符合条件的数据那么返回的Cursor.getCount()为0；如果在查询时发生错误，依赖Provider的实现有可能返回null或者抛出异常。

可以使用工具类SimpleCursorAdapter配合ListView来展示Cursor中所包含的数据；如果要这样做Cursor中必须包含`_ID`列。

### Provider的权限

如果Provider不定义权限，其他的App就无法访问Provider；Provider所属的App**总是**可以自由的访问Provider。

### “增”、“删”、“改”  

ContentResolver.insert()方法会返回刚刚插入的数据的URI。

### Provider的数据类型

Provider支持的基本类型包括：string、int、long、float、double和blob；其中blob为上限64KB的byte数组。通常每一列的类型会在Contract类型中注释。

同时，Provider还会提供所包含URI的MIME type；可以通过ContentResolver.getType()来查询URI的MIME type。

### Provider的其他访问方式

这里介绍批量访问(Batch)和借助其他App访问。

#### Batch操作

见展开点。

#### 借助其他App访问

即使不申请provider的读取权限，也可以通过StartActivityForResult()访问其他App，经由返回intent中携带的Uri来访问proivder；这么做需要借助URI权限。

通过Intent获取provider URI的常用协议为ACTION_PICK(android.intent.action.PICK)加上MIME type。

### Contract类    

Contract类用于定义(向使用者说明)Provider结构，并定义一些辅助常量。

### MIME Type

MIME type的结构为`type/subtype`。
对于自定义的MIME type而言，type字段永远是`vnd.android.cursor.dir`(表示一组自定义数据)或`vnd.android.cursor.item`(表示一条自定义数据)；subtype则自由定义。

尽管Android系统自定义的MIME type的subtype通常很短，开发者自定义的MIME type通常基于包名 &表名unique。

## Creating a Content Provider

### 实现前，想清楚

在实现ContentProvider之前，首先需要确定是否真的需要ContentProvider。

实现ContentProvider包括以下工作：

* 设计数据的存储结构，ContentProvider可以处理文件类型数据和结构化数据。
* 继承ContentProvider并实现接口。
* 编写contract类，定义authority，path，列名；定义Intent协议；定义权限常量，MIME type常量等。


### 设计contract

authority需要在系统中是独一无二的，通常采用`包名.Provider名`的格式。path类似于表名，path的路径可以有多层(无需每层都有意义)。

UriMatcher类可以作为识别URI的工具类。
UriMatcher使用uri pattern stirng，其中可以包含通配符`*`和`#`，`*`代表任意长度的字符串，`#`代表任意长度的数字。

### 继承&实现ContentProvider

实现ContentProvider需要实现：生命周期方法(onCreate())，获取MIME type方法(getType()等)以及增删改查方法。

增删改查方法有可能在任何线程上调用(八成跟Service的AIDL一样)，ContentProvider需要时线程安全的。
返回MIME type的方法——getType()等方法——是需要认真实现的。

即使在跨进程的情况下，也有方法从ContentProvider的增删改差方法中，向调用者抛出异常（方法见展开点）。

不要再onCreate()方法中执行耗时操作，因为系统在创建ContentProvider的时候立即调用onCreate()过长的创建时间会拖慢返回速度，延迟耗时的初始化到第一次使用的时候。

### 实现MIME type

ContentProvider通过`getType()`和`getStreamTypes()`返回table与文件的MIME type。

#### 返回属于table的MIME type

格式为`vnd.android.cursor.<item ro dir>/vnd.<package>.<typename>`。

#### 返回属于文件的MIME type

如果ContentProvider支持文件，就需要实现`getStreamTypes()`。

### 实现Provider的权限

通过权限控制其他应用对Provider的访问，那么实际存储数据的文件或Database应该是private的，否则就没有意义。

在Manifest中可以分不同粒度声明Proivder的所需权限：

* 整体权限，在`<provider>`标签的android:permission字段中声明Provider的整体权限。
* 读写权限，在`<provider>`标签的android:readPermission和android:writePermission分别声明读写权限。
* 表权限，可以通过`<provider>`的字标签` <path-permission>`为某些pattern的content uri声明读写权限。
* Uri权限(临时权限)，通过`<provider>`标签的android:grantUriPermissions属性和子标签`<grant-uri-permission>`来开启Provider的Uri权限。

### 关于`<provider>`标签的属性

* android:authorities，等价于Provider在系统中的id。
* android:name，实现Provider的class。
* permission相关属性。
* android:icon和android:label，也许会在Setting里显示。



