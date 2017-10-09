---
title: 自定义同步工具类
subtitle: JCIP读书笔记第十四章
catalog: true
header-img: /img/header_img/java-note-header.jpg
tags:
  - Java
categories:
  - 读书笔记
date: 2017-09-29 14:05:13
---


## caution

## 概述

自定义同步工具类最简单的方式就是基于已有的同步工具类；当然你也可以借助"内置Condition Queue"、"显示Condition Queue"以及AQS(AbstractQueuedSynchronizer)。

## 条件依赖操作与同步

如果不借助Condition Queue，Blocking等待条件成立写起来往往很麻烦，比如：

```java
    acquire lock on object state
    while (precondition does not hold) {
        release lock
        wait until precondition might hold
        optionally fail if interrupted or timeout expires reacquire lock
    }
    perform action release lock
```

在依赖条件不满足时，可以选择抛出异常、返回错误码、等待条件成立等不同策略。

### 当没有Condition Queue

这里在不使用Condition Queue的情况下，提供两个BlockingQueue的简单实现，并指出其不方便或没效率的缺点。

### 当有了Condition Queue

Condition Queue提供了“挂起线程，等待条件成立”和“提醒等待线程条件成立”的一组操作。所以在Condition Queue这个概念中，Queue中的元素是等待的线程。

就如同Object可以作为内置锁一样，Object也可以作为内置的Condition Queue，并提供了wait()、notify()、notifyAll等API。
同时内置锁和内置CQ有紧密的联系，在调用内置CQ的API时必须持有同一个对象的内置锁。
在调用wait()之后，当前线程自动释放了相同对象的内置锁，并要求操作系统挂起当前线程；当从wait()返回时，回自动获取相同对象的内置锁。

使用CQ实现Blocking可以获得更高的效率、更快的响应速度；使用fair CQ还可以获得额外的调度策略。

## CQ的使用

这一章介绍一些使用CQ的模式和准则，以防止开发者错误的使用CQ。

### 等待条件

CQ往往是在等待某个条件成立，为了正确使用CQ，应当用文档(注释)记录与CQ相关的条件，以及会影响这些条件(使条件成立)的操作。
CQ和等待条件与锁紧密相关，在执行wait()时必须持有相关的锁，而这个锁往往是用来保护等待条件中的变量的。

### CQ被反复唤醒

与一个CQ关联的等待条件可能有多个，所以CQ被唤醒(并重新活得锁)不意味着等待条件一定成立。等待CQ的经典写法如下:

```java
void stateDependentMethod() throws InterruptedException {
    // condition predicate must be guarded by lock
    synchronized(lock) {
        while (!conditionPredicate())
            lock.wait();
        // object is now in desired state
    } 
}
```

在编写等待CQ的代码注意如下条件:

* 一定要拥有一个等待条件。
* CQ每次被唤醒时需要检查等待条件。
* 将等待操作和条件检查置于循环中。
* 确保保护等待条件的锁和CQ关联的锁是同一个。
* 在调用CQ相关API时必须持有CQ关联的锁。

### 唤醒CQ

当某个操作会让CQ的一个等待条件变为成立时，记得在操作的同时唤醒CQ。(在开发中，这件事情似乎很容易发生疏漏)
notify()会选择唤醒一个等待线程；notifyAll会唤醒全部的等待线程。
使用notify()时容易出错，例如当CQ关联多个等待条件时，条件A发出的提醒可能唤醒等待条件B的线程，而等待条件A的线程会错过这次提醒。所以使用notify()而不是notifyAll()时要格外谨慎，以下是使用notify()的准则：

* CQ仅仅关联一个等待条件，并且等待线程都执行相同的操作。
* 每次提醒能且仅能满足一个等待线程完成操作。

### 继承和CQ/封装和CQ

当继承发生在使用CQ的类上时有两种选择：父类将等待策略(等待条件、提醒策略)和相关对象(CQ对象、锁对象、状态对象)完整的暴露给子类，并提供足够的文档(注释)；或者可以选择，禁止继承/对子类隐藏所有和相关对象(CQ对象、锁对象、状态对象)。

和继承同理，使用CQ的类应该将CQ相关对象封装在类内部，以防止类的使用者干扰等待策略。

## 显式的CQ对象

内置CQ的一个缺陷是一个内置锁只能关联一个内置CQ，这样导致了一个CQ关联多个等待条件的情况十分常见，也就难以满足使用notify()的条件。
显式CQ被称为Condition，它与一个显式锁Lock相关联；Condition可以通过Lock.newCondition()方法创建，一个Lock可以关联多个Condition；Condition的公平性与其关联锁的公平性一致(公平锁创建的Condition也是公平的)。

**注意：不要误用Condition对象自身的内置CQ。**

这一节还给出了一个用显式CQ实现的BlockingQueue的例子，可以发现当有等待条件时Condition比内置CQ有更好的可读性，切更容易使用单一提醒来提升效率。

## AQS(AbstractQueuedSynchronizer)

AQS是大多同步工具类的基础组件，通过继承AQS来实现同步工具。
在AQS内部存在一个整数变了，用于表示同步工具的状态；AQS提供了一组方法，包括原子读写状态、原子CAS状态用于修改状态，线程独占或线程共享的Acquire/Release方法用于提供安全方法；开发者需要继承tryAcquire/tryAcquireShared和tryRelease/tryReleaseShared来决定同步工具的具体行为。

本章没有展开介绍AQS的使用细节，如果想深入了解的话，参考`java.util.concurrent`包中同步工具类的实现。

## 在Java标准库中的AQS

本章介绍了一些`java.util.concurrent`中类的实现，这些类是基于AQS的。








