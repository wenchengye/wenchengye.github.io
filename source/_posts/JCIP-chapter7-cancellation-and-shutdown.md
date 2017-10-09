---
title: 中止线程
subtitle: JCIP读书笔记第七章
catalog: true
header-img: /img/header_img/java-note-header.jpg
tags:
  - Java
categories:
  - 读书笔记
date: 2017-10-09 15:09:58
---


## 概述

中止线程比启动线程要复杂，Java没有提供之间中止线程的机制(初期的中止线程方法被弃用)，而是提供了interruption机制提醒线程自己进行中止。用interruption机制来代替中止线程的原因是，如果线程被类似stop()的方法立即中止可能会打断正在进行的操作，而使变量处于非法的状态。

## 任务中止

中止任务的需求有很多，例如用户请求、任务时限、发生错误、服务关闭等情况。没有主动中止任务方法情况下--就像Java--可以通过设置中止标志位，并由任务代码自身定期检查中止标志来执行中止；interruption机制正是使用了类似中止标志位的协议。

### 中断(Interruption)

Interruption是一个线程通过Interrupted标志位通知另一个线程的一种线程间交流机制，Interruption不一定要和中止线程的语意绑定，然而在实践中Interruption通常只用于中止线程的场景；将Interruption用于中止线程以外的场景，都被证明会使程序变得脆弱且难以维护。

每个线程都有一个interrupted标志位，interrupt某个线程就是将其interrupted置为true。Thread.interrupt()、Thread.isInterrupted()、Thread.interrupted()是与Interruption机制相关的三个方法。
其中Thread.interrupted()容易使人迷惑，他会返回interrupted标志位**并将其置为false**，这也是唯一将interrupted标志位置回false的方法。

Java库中对interruption的响应方式是将interrupted置回false，并抛出InterruptedException。

### 中断策略

通常来说，任务执行代码不应该假设其被执行的线程的中断策略，因为任务执行代码很可能在线程池的worker线程中被执行；所以在任务处理中断并中止任务后，应当保留当前线程的被中断状态(抛出InterruptedException或者保留interrupted为true)。

线程的持有者应当将任务中止操作封装为类似cancel方法；与此同时非线程持有者在中止任务时应当调用封装方法，而不是直接Thread.interrupt();

### 响应中断

处理InterruptedException的方法有两种，将InterruptedException抛出；或者将interrupted置为true。永远不要仅仅捕获InterruptedException并什么都不做。
线程拥有者通常是创建线程的对象，它通常还会继承Thread对象。只有线程的拥有者，才有权中止interrupted状态的传播；其他任务或者工具代码在处理中断时都应继续传播中断状态。

### 使用Future提供的中止任务的功能

Future.canel(boolean)可以用于中止任务，其包含布尔型参数mayInterruptIfRunning代表cancel方法是否会中断worker线程。

### 处理不可中断的阻塞

有一些阻塞的操作不响应中断，需要采用其他的方法来中止这些操作：

* 同步的IO操作，基于Socket的阻塞IO可以通过关闭Socket使其抛出SocketException来中止。
* 等待获取内置锁，等待内置锁的阻塞是无解的，如果有中止的需求可以改用显式锁。


### 重写ThreadPoolExecutor的newTaskFor方法

通过重写ThreadPoolExecutor的newTaskFor方法，可以返回自定义的FutureTask类；通过重写返回的FutureTask的cancel方法，可以hook中止任务的过程(例如加入关闭IO的操作)。

## 关闭服务(包含线程的服务)

通常应用程序都会使用一些包含线程的服务(例如任务系统、线程池)，这些服务通常都是常驻内存的，所以在应用结束之前应当使用适当的方法来关闭这些服务，以防止线程泄漏。
中止线程的工作应该有线程的拥有者，也就是服务本身来完成，所以类似这些服务通常应该提供shutdown方法。ExecutorService就定义了shutdown和shutdownNow两个关闭方法。

### 一个Log服务

本章将以一个使用单独线程的Log服务作例子，讲解关闭服务的方式。

在实现关闭服务操作时，需要留意几个问题：

* 是将现在正在排队的任务处理完再关闭服务(暂且称为安全关闭)，还是立即关闭服务。
* 如何在关闭服务过程中，处理外部程序调用服务功能。
* 在关闭服务时清理状态。

在这一节，给出了一个利用请求计数实现安全关闭例子。

### ExecutorService的关闭方法

ExecutorService的showdown方法提供了安全关闭；showdownNow方法提供了立即关闭。

这一节借助ExecutorService重写了Log服务，并利用ExecutorService的shutdown方法实现了安全关闭。

### Poison Pills

用于生产者消费者关系中的一种关闭方式，生产者在关闭之后将poison pill加入队列，消费者在获取到poison pill之后关闭自己。
poison pill方式只能用于生产者数量和消费者数量都确定的情况。

### 使用非常驻的ExecutorService

借助局部的ExecutorService，将所有任务submit之后执行shutdown和awaitTermination，可以达到等待所有任务完成之后再返回的效果。

### showdownNow(立即关闭)的局限性

ExecutorService的showdownNow方法会返回所有未完成任务的列表(Runnable的list)，但是无法区分哪些是还未开始的任务、哪些是执行到一半中止的任务。

## 处理线程的异常结束

线程可能会因为抛出异常(通常是RuntimeException)而提前结束，这种异常结束不易被其他线程发现(也许仅仅在控制台输出了异常栈)，从而产生了线程泄漏。

一种简单处理方式，在worker线程中捕获类似Runnable.run()方法抛出的unchecked Exception、结束线程、并通知worker线程的拥有者(通常是某种Service)线程异常结束的消息；尽管这种捕获unchecked Exception方法的安全性存在争议。

### Uncaught exception handler

Thread提供了UncaughtExceptionHandler相关的API，可以发现从线程中抛出的未捕获的异常。

最好为所有常驻线程添加UncaughtExceptionHandler。

## 关闭JVM
 
JVM有两种关闭方式：orderly或abrupt。
当JVM中最后一个非守护线程结束、System.exit或者其他关闭JVM的方法被调用时，触发orderly关闭；当Runtime.halt被调用或者关闭JVM进程时，触发abrupt关闭。

在orderly关闭过程中，JVM首先会执行所有的Shutdown hook；然后可能会执行对象的finalizer；然后关闭JVM。orderly关闭发生时，JVM不会去中断任何正在执行的线程，这些线程运行直到JVM关闭时停止。
在abrupt关闭过程中，JVM直接停止其他什么都不做。

### Shutdown hook

Shutdown hook时通过Runtime.addShutdownHook注册的；Shutdown hook运行在单独的线程中，所以需要是线程安全的。

一种相对轻松的使用Shutdown hook方式是，整个应用仅使用一个Shutdown hook，将所有的关闭工作在这个Shutdown hook中按某种顺序执行。

### 守护线程

JVM中的线程可以被分为两类，普通线程和守护线程，如果不想让某些常驻线程阻止JVM关闭，可以将这些线程设为守护线程；除了主线程之外，JVM启动时创建的所有线程都是守护线程。

当线程被创建时，其守护属性默认继承创建它的线程的守护属性，所以普通线程创建的线程默认是普通线程，守护线程创建的线程默认是守护线程。

守护线程可能在任何时刻突然停止，如果在守护线程中执行IO操作，那么就可能没有机会清理IO。所以，**任务类服务中的work线程不适合被创建成守护线程**，守护线程适合做一些没有生命周期属性的工作。

### finalizer

finalizer被JVM在单独上执行，所以需要是线程安全的；其被执行的时间、和是否会被执行都不确定。基于finalizer要求线程安全，重载finalizer的类会增加额外的同步，从而影响性能；同时finalizer很难被正确的编写；所以**尽量避免使用finalizer**

一个使用finalizer的例外是，在finalizer中检查对象是否已经释放其持有的资源，否则输出错误提示。



