---
title: Android Developer Guide中的系统结构
subtitle: Android官方guide随笔 - Platform Architecture
header-img: "/img/header_img/android-note-header.jpg"
catalog: true
tags: [Android]
categories: [读书笔记]
---

## Platform Architecture   

### 展开点   

* 为什么选用DEX bytcode。
* Build toolchains。
* AOT & JIT。
* Dalvik & ART。
* 如何调用Android native libraries。
* 使用Java feature，other then using Jack。

## Use Java 8 language features   

## Verifying App Behavior on the Android Runtime (ART)   

### 展开点   

* [Debugging Android JNI with CheckJNI](https://android-developers.googleblog.com/2011/07/debugging-android-jni-with-checkjni.html)。
* [JNI Local Reference Changes in ICS](https://android-developers.googleblog.com/2011/11/jni-local-reference-changes-in-ics.html)。
* [mockito](https://github.com/mockito/mockito)。
* adb bugreport。

### caution

*  调用System.getProperty("java.vm.version")判断runtime的类型，如果大于"2.0.0"则是ART。
*  Dalvik的Java栈和Native栈是分开的，ART的Java栈和Native栈是统一的(为了提升局部性)。