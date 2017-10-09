---
title: 显式锁
subtitle: JCIP读书笔记第十三章
catalog: true
header-img: /img/header_img/java-note-header.jpg
tags:
  - Java
categories:
  - 读书笔记
date: 2017-09-29 15:58:59
---


## caution

## 开头

Java 5.0之前，Java只有内置锁和volatile；从Java 5.0开始，引入了显式锁来提供一些内置锁所不具备的功能。

## Lock和ReentrantLock

Lock是个接口，定义了显式锁的基本行为；ReentrantLock是Lock的常用实现，其同步语意和内置锁基本相同。

ReentrantLock相对于内置锁提供了更多的功能：可中断的获取锁的方法、如果失败立即返回(或超时返回)的获取锁的方法；可以在代码的任意部分释放锁，而不用局限在synchronized代码块中。
由于显式锁必须显示释放，所以绝大多数情况下，unlock()方法要写在finally代码块中以保证安全；如果不这样做，由于某种原因造成没有执行unlock()的错误，将十分难以追查。

Lock接口如下:

```java
public interface Lock { 
    void lock();
    void lockInterruptibly() throws InterruptedException; boolean tryLock();
    boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException; 
    void unlock();
    Condition newCondition(); 
}
```

## 性能对比

在Java 5.0时，ReentrantLock的性能比内置锁好很多；但从Java 6.0内置锁的性能被改进之后，ReentrantLock和内置锁就没有明显的性能差异了。

## 公平性

ReentrantLock提供了公平锁和非公平锁的功能选择。
公平锁会将请求锁的线程排队，然后实行先到先得方案；非公平锁在某个线程请求锁时会插队尝试是否能直接获取成功，只有请求不成功时才会将线程排队。(即便是公平锁，在使用tryLock()操作时也会插队)
。
在竞争较多时公平锁的性能远差于非公平锁；公平锁更适用于持有锁和获取锁时间都很长的情况。

## 内置锁还是ReentrantLock

除非需要使用ReentrantLock提供能附加功能(响应中断、公平性等等)，那么请选择内置锁。

## 读写锁

读写锁允许多个线程同时持有“读锁”，但仅有一个线程持有"写锁“(当然持有"写锁"时其他线程也不能获取"读锁")。其接口定义如下:

```java
public interface ReadWriteLock { 
    Lock readLock();
    Lock writeLock();
}
```

ReentrantReadWriteLock是读写锁的一个实现。
读写锁的使用是出于性能考虑，在读操作明显多于写操作、读操作执行时间较长的情况下，读写锁的性能优于独占锁；在其他情况下，读写锁的性能不及独占锁。当然，使用读写锁能否提升性能，还得看profiling的结果。

