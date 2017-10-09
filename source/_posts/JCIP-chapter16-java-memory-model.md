---
title: Java内存模型
subtitle: JCIP读书笔记第十六章
catalog: true
header-img: /img/header_img/java-note-header.jpg
tags:
  - Java
categories:
  - 读书笔记
date: 2017-09-09 15:57:37
---


## Caution 

* 现代系统中，volatile变量读操作和普通变量读操作的代价几乎一样。

整本JCIP都在避免直接讲述Java内存模型，但是线程安全机制是以内存模型为基础的，所以了解一些内存模型的知识有利于明白线程安全规则为何要那样规定。

## Java内存模型概述(JMM)   

### 什么是Java内存模型

为什需要Java内存模型？
因为现代CPU的复杂优化，导致仅仅在单一线程上保证变量(抽象内存位置)的可见性；并且仅仅在单一线程上，指令的执行是可以等效成顺序执行(指令重排)的。

Java内存模型规定了什么？
Java内存模型规定了JVM的行为，描述JVM在什么情况下必须保证变量的修改对其他线程是可见的。

### 平台的内存模型   

平台(或者说cpu)会提供自己内存模型以及控制可见性特殊指令(例如memory barrier)。JVM会代理不同平台的内存模型，并抽象为统一的Java内存模型；各个平台的控制指令，也被抽象成了Java的同步机制。

### 指令重排   

众所周知，指令会重排；加上从缓存写会内存的时机不确定，跨线程分析变量的值可谓举步维艰(没有同步)。

### 所以Java内存模型是怎么说的

Java内存模型通过一种偏序关系来描述内存可见性规则，在这个规则中定义一种关系叫happens-before，如果语句A(或者说指令，在JMM规则里叫action)对于语句B是happens-before的关系，那么语句A对变量的修改对于语句B是可见的。

下面用一些规则什么情况下两条语句(指令，whatever)具有happens-before关系：   

* Program order rule。在同一线程上，所有语句按照代码顺序具有自然的happens-before关系；即同一线程上，直观上先执行的代码happens-before直观上后执行的代码。
* Monitor lock rule。对一个锁的释放操作happens-before对**同一个锁**的获取操作。注意两点，一定要求是同一个锁对象；规则同时适用于显式锁和内置锁。
* Volatile variable rule。对volatile属性的写操作happens-before对**同一个volatile属性**的读操作。注意两点，一定要求是同一个volatile属性；这条规则同时atomic变量。
* Thread start rule。启动线程操作happens-before被启动线程上的所有语句。
* Thread termination rule。一个线程上的所有语句happens-before发现这个线程结束的语句(发现线程结束的语句例如Thread.join()返回或者Thread.isAlive()返回false)。
* Interruption rule。线程A调用线程B的interrupt()happens-before线程B发现发现interrupt(例如调用isInterrupted)。
* Finalizer rule。对象构造函数的结束happens-before对象finalizer的开始。
* Transitivity。如果A事件happens-before B事件，且B事件happens-before C事件，那么A事件happens-before C事件。

也许Java内存模型的规则会让第一次看的人觉得晦涩，事实上整个规则就是以单线程的顺序可见性为基础，通过一些保证顺序可见行的关键操作(例如线程的开始结束，锁的获取和释放，volatile变量的读写)加上传递性，来维持多线程代码的偏序关系。进而通过这个偏序关系来描述可见性规则，即偏序关系在前的操作对偏序关系在后的操作可见。

### piggybacking on JMM(Trick的利用Java内存模型)

借助Java内存模型规则，可以利用微妙的happens-before传递来确保可见性，这种可见性保证不易发现并且需要推理，本书将这种现象叫piggybacking同步。

作者以FutureTask的实现为例(在最新的jdk中这个实现已经改变)，展示了piggybacking同步。简单来说可以表述为：

> `Thread A :`
>      `变量 X 的写操作`
>      `synchronized (变量 Y)` 
>          `操作变量 Y`
>          
> `Thread B :`
>     `synchronized (变量 Y)`
>         `操作变量 Y`
>     `变量 X 的读操作`   

变量X在A线程上的读操作对B线程是可见的，尽管并没有针对X作任何同步，但是通过happens-before规则来保证了可见性。
piggybacking同步有时很显而易见(例如通过同步队列传递对象)，有时却很晦涩(例如上面的例子)。**piggybacking同步要求操作微妙的顺序，因此非常的脆弱易出错，除非非常追求性能否则不要使用**。

## 安全发布与安全构造

第三章讨论过安全发布，这一章借由Java内存模型旧话重提，来看看之前提到安全发布方式/安全构造方式是如何被Java内存模型保证的。顺便批判一下Double-checked实现的单例模式(被称为臭名昭注)，不是真的线程安全。

安全发布和安全构造都与可见性相关，且往往联系紧密，因为一般来说发布将紧随构造之后，但我们还是给这两个术语。安全发布是指，其他线程能够正确变量被修改；安全构造是指，其他线程不会看到未完成构造的对象。

### 不安全的发布 

先将竞态条件放到一边，仅仅讨论可见性。由于指令重排的存在，在其他线程看来，对发布引用的写入操作，可能会先于被发布对象的构造函数中的某些指令，于是导致其他线程读到未完成构造的对象。

必须通过happens-before规则来保证发布对象可见性和完整性(安全构造)；不可变对象是个例外，参见第三章。

### 安全发布

happens-before规则是第三章中介绍的安全发布规则的基础，但是那些规则比happens-before规则看起来更直观。

### 安全构造的一种方式(单例)

JVM保证了静态初始化的线程安全，被静态代码构造的变量对所有线程都是可见的，但是单纯的静态构造不能满足使用时才初始化的要求(lazy)；单纯的加锁构造 & 加锁访问也许会拖累线程。
借助class第一次被使用才会被JVM加载的规则，可以实现双赢(lazy、线程安全、不必每次都加锁)也较为优美的安全构造：

>`@ThreadSafe`
>`public class ResourceFactory {`
>&emsp;&emsp;`private static class ResourceHolder {`
>&emsp;&emsp;&emsp;&emsp;`public static Resource resource = new Resource();`
>&emsp;&emsp;`}`
>&emsp;&emsp;`public static Resource getResource() {` 
>&emsp;&emsp;&emsp;&emsp;`return ResourceHolder.resource;`
>&emsp;&emsp;`}` 
>`}`

### (批判)Double-checked单例构造

Double-checked的单例构造经常出现于各种面试题中，然而它实际上不是线程安全的，也不优美。其线程不安全的原因是，在第一次判没有使用同步，所以没有安全构造的保证，可能返回未完全构造的对象。尽管把对象的引用声明为volatile可以解决这个问题(如果对象是不可变的也不会有问题)，但这个写法远不如static-holder方式(上一节所述)简洁明了。

### Java构造安全规则

Java构造安全规则描述了在对象的构造过程中，什么样的属性的构造过程对其他线程是保证可见的，事实上这个规则在第三章也被提起过。这个规则和final属性息息相关。

当对象被正确构造时(对象没有在构造函数中逸出)，Java构造安全规则保证，其他线程能正确看到这个对象的所有final属性构造结果，并保证能正确看到由这些final属性可达的所有对象的构造结果。

这个规则的一个直观效果：不可变对象(所有属性都是final的)一定会被安全构造。

啊，上面话有点抽象了，换一种说法：JVM保证了在构造函数中所有对final属性可见性(即使没有使用同步)，并且JVM保证对final属性的所有操作不会被重排到构造函数返回之后。

啊，还是觉得很抽象的话，请看原文：

> Initialization safety guarantees that for properly constructed objects, all 
> threads will see the correct values of final fields that were set by  
> the constructor, regardless of how the object is published. Further, any 
> variables that can be reached through a final field of a properly 
> constructed object (such as the elements of a final array or the contents of 
> a HashMap refer- enced by a final field) are also guaranteed to be visible 
> to other threads.

> Initialization safety makes visibility guarantees only for the values that 
> are reachable through final fields as of the time the constructor finishes. 
> For values reachable through nonfinal fields, or values that may change 
> after construction, you must use synchronization to ensure visibility.
