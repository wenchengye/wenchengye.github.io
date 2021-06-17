---
title: Java 泛型:《Thinking in java》
subtitle: 《Thinking in java》微缩笔记 0x01
catalog: true
header-img: /img/header_img/java-note-header.jpg
tags:
    - Java
categories:
  - 读书笔记
---

## 简单泛型

Java 泛型的核心概念：告诉编译器想使用的类型，让编译器来检查类型和转型。

## 泛型接口

## 泛型方法

如果能用泛型方法替代，就不要使用泛型类。

泛型函数的泛型参数列表需要写在函数的返回类型**之前**

大部分情况下，泛型方法可以推断出泛型参数；但也可以在 `.` 操作符和方法之间加入 `<>`，显示指明泛型参数类型。  

## 泛型擦除

在泛型代码内部，无法获得任何有关泛型参数类型的信息。

泛型参数将被擦除到它的第一个边界。




