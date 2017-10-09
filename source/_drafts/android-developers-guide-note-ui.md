---
title: Android Developer Guide中的UI
subtitle: Android官方guide随笔 - UI
catalog: true
header-img: /img/header_img/android-note-header.jpg
tags:
  - Android
categories:
  - 读书笔记
---

## 展开点   

* [Say Goodbye to the Menu Button](https://android-developers.googleblog.com/2012/01/say-goodbye-to-menu-button.html)。
* [New Layout Widgets: Space and GridLayout](https://android-developers.googleblog.com/2011/11/new-layout-widgets-space-and-gridlayout.html)。
* [Customizing the Action Bar](https://android-developers.googleblog.com/2011/04/customizing-action-bar.html)。
* [Horizontal View Swiping with ViewPager](https://android-developers.googleblog.com/2011/08/horizontal-view-swiping-with-viewpager.html)。
* measuredSize和drawingSize的区别。
* [Holo Everywhere](https://android-developers.googleblog.com/2012/01/holo-everywhere.html)。

## Caution

* 系统提供的控件位于android.widget包下面。
* `<selector>`标签下`<item>`标签的顺序有意义，系统会从上到下选择第一个合适的`<item>`所以default的`<item>`需要处于最后一个。
* 系统会先调用Event Listeners之后调用Event Handlers，如果Event Listeners返回true，那么Event Handlers就不会被调用了。
* 在Touch Mode下isFocusableInTouchMode()为false的View(例如Button)是不会被focus的。
