---
title: Java库中的线程安全类
subtitle: JCIP读书笔记第五章
catalog: true
header-img: /img/header_img/java-note-header.jpg
tags:
  - Java
categories:
  - 读书笔记
date: 2017-10-09 17:24:53
---


## 概览

最稳定的实现线程安全类的方式，就是借助已有的线程安全类。这一章将介绍Java库中的线程安全类，包括一些线程安全容器类和同步工具类。

## 同步容器

Java库中的同步容器包括Vector、Hashtable以及Collections.synchronized*系列容器，这些容器都采用内置锁来执行同步操作。

### 同步容器的问题

**client-side锁**
如果需要同步的组合操作(例如取得容器长度，然后删除最后一个元素)，需要使用client-side锁。事实上同步容器是支持client-side锁的，其同步使用的锁即自身对象，然而使用client-side锁往往基于对于容器实现代码的阅读。

### 同步容器抛出ConcurrentModificationException

当从同步容器获取迭代器之后如果同步容器被修改，迭代器的操作会抛出ConcurrentModificationException；这一特性是通过modification count计数器实现的。
即使在单线程情况下，也可能会抛出ConcurrentModificationException。

注意foreach语法也是通过迭代器实现的。

### 隐蔽的迭代器使用

迭代器的使用可能发生在某些被封装的方法中(例如toString)，使得调用者不易注意到ConcurrentModificationException可能被抛出。

## 并发容器

Java5.0之后提供了并发容器系列，所谓同步容器的改进或补充；并发容器不使用严格的互斥操作，提供了更好的性能和伸缩性。并发容器系列包括ConcurrentHashMap、 CopyOnWriteArrayList、ConcurrentLinkedQueue等。

因为性能大幅提升，尽量使用并发容器而不是同步容器；同步容器唯一的优势就是互斥操作(如果需要的话)。

### ConcurrentHashMap

ConcurrentHashMap使用了更细粒度的同步机制，允许多线程并发读取、同时读写以及一定数量的并发写入，大幅度提升了吞吐量和伸缩性。
ConcurrentHashMap和其他并发容器返回的迭代器不会抛出ConcurrentModificationException；ConcurrentHashMap的迭代器采用一种弱同步机制来同步容器元素的变化(不保证准确)。

### CopyOnWriteArrayList

CopyOnWriteArrayList线程安全依赖于不可变对象一定是线程安全的。人如其名，在每次发生写入时进行拷贝。

### 更多的原子操作

由于并发容器不能使用client-side锁，所以并发容器提供了一系列的原子操作(例如put-if-absent等)。

## 阻塞队列和生产者消费者模式

略

## 阻塞方法和可中断方法

见[中止线程](2017/10/09/JCIP-chapter7-cancellation-and-shutdown/)。

## 同步工具类

同步工具类是基于一些状态控制线程执行流程的工具类，它们通常都持有某种状态，并基于这个状态决定允许线程执行或阻塞线程。BlockingQueue是线程安全容器的同时，也可以视为同步工具。

同步工具还包括：Latch、FutureTask、Semaphore、Barrier等。



