---
title: Android Developer Guide中的Fragment和Loader
subtitle: Android官方guide随笔 - App Components：Fragment & Loader
catalog: true
header-img: /img/header_img/android-note-header.jpg
tags:
  - Android
categories:
  - 读书笔记
date: 2017-08-04 15:19:51
---


## 展开点   

* DialogFragment vs Dialog。
* FragmentManager in Detail。
* multiple fragments to the same container。 
* Fragment and Menus。
* Fragment lifecycle in Detail。

## Caution   

* 经过commit()方法提交Fragment更改应该发生在Activity进行SaveInstanceState之前，如果在SaveInstanceState之后这么做会抛出异常；如果一定要在SaveInstanceState之后提交Fragment更改，使用commitAllowingStateLoss()。
* 如果要Fragment能够收到onCreateOptionsMenu()回调，需要在Activity处于Created状态时调用setHasOptionsMenu()。

## Fragments

### 创建Fragment   

创建包含UI的Fragment有两种途径：

* 在layout文件中声明`<fragment>`标签，如同使用View一样；系统会用onCreateView()返回的View替换`<fragment>`标签。
* 使用FragmentTransaction.add(int, Fragment)方法提交fragment。

创建不包含UI的Fragment，需要使用FragmentTransaction.add(Fragment, String)方法，String参数代表了Fragment的Tag。因为不包含UI的Fragment不存在于View结构中，通过Activity查找它的唯一方法是findFragmentByTag()。

每个Fragment必须有一个唯一标识，系统寻找这个标识有三种方式：  

* android:id属性。
* android:tag属性。
* 如果前两条都不存在，系统会使用fragment的parent viewgroup的id。

### Fragment和Activity的交流

* Fragment中可以通过getActivity()或得所属的Activity。
* Fragment定义Listner类的Interface来给Activity回调。
* Fragment可以参与Options Menu和Context Menu。

### Fragment的生命周期

![Fragment生命周期图](https://developer.android.google.cn/images/activity_fragment_lifecycle.png)

与Activity相同，系统会Fragment的onSaveInstanceState()方法，savedInstanceState会在 onCreate()、onCreateView()、onActivityCreated()方法中被传入。

## Loader

### 为什么要使用Loader

* Loader运行在单独线程中（车有轮子...）。
* Loader简化了Worker线程和主线程之间的交流（AsyncTask也行）。
* Loader当Activity发生configuration changes时，加载结果会得以保存（都MVP了，不过在轻量级实现时有一定意义）。
* Loader会在数据源变化时产生响应，例如ContentProvider（有一定意义，跟LiveData有何不同）。
*

### 使用Loader

Loader相关的API主要包含在LoaderManager，LoaderManager.LoaderCallbacks和Loader三个class & interface中。

* LoaderManager，每个Activity & Fragment绑定唯一一个LoaderManager，可以通过getLoaderManager()获取。
* LoaderManager.LoaderCallbacks，绑定Loader回调接口，返回Loader的加载状态和加载结果。
* Loader，开发者自己实现Loader或者使用系统提供的工具Loader，实现具体的加载过程。




