---
title: 原子变量与非阻塞同步
subtitle: JCIP读书笔记第十五章
catalog: true
header-img: /img/header_img/java-note-header.jpg
tags:
  - Java
categories:
  - 读书笔记
date: 2017-09-21 18:44:18
---


## Caution

* Float.floatToIntBits()/Double.doubleToLongBits()。
* AtomicFieldUpdater。

## 前言

`java.util.concurrent`中的一些线程安全类/同步工具类声称自己有更好的性能和伸缩性，这一章将介绍这种性能提升的基础——非阻塞同步。

**非阻塞同步算法的特点**
非阻塞同步基于硬件提供的一些原子指令(例如CAS)；非阻塞同步算法在操作系统、JVM这种高端大气的项目中很常见；非阻塞同步算法往往很难设计。

**非阻塞同步算法的好处**
没有锁，没有阻塞，大幅减少性能消耗；对死锁以及很多活跃性问题天然免疫。

## 锁的缺陷

**用锁同步可能会加重负载**。当获取锁失败后，线程会被挂起并在之后再恢复，当竞争严重时会反复的挂起与恢复，这中间涉及到了大量的中断和与性能消耗。对容器加锁时这种状况很容易出现。
**线程在等待锁时无法进行任何事**。同时因为等待锁的原因，线程的优先级也无法得到保证：高优先级的线程会因为低优先级的线程长时间持有锁而被挂起。
**volatile不提供原子性**。volatile变量没有过高的性能消耗，但是它不能保证原子性，例如`++i`对于`volatile i`就不是原子操作。

## 非阻塞同步的硬件支持  

如果说互斥锁是一种“悲观的”的技术：确保独占内存之后再去读写内存；那么非阻塞相关的指令就是“乐观的”技术：先去尝试修改内存，发生冲突再退回重试。
现代处理器提供了一些原子指令，例如CAS、load-linked/store-conditional，帮助实现非阻塞同步。

### CAS

就是CAS。

### 一个非阻塞计数器

这一节用仿造的CAS操作实现了一个非阻塞的递增计数器，并强调虽然写法复杂其实际性能比加锁递增好很多。当然在实际应用中无需这样写，使用Atomic变量就行了。
CAS的劣势是要求开发者自己处理冲突(重试、等待还是放弃)；实际上CAS最大的缺点是很难正确编写。

## 原子变量 

原子变量为开发者提供了非阻塞同步的基本工具，基于CAS的原理，原子变量提供了更好的性能，在无竞争情况下原子变量不差于加锁，在中等竞争时原子变量明显优于加锁。

由于Atomic变量只提供了AtomicInteger、AtomicLong、AtomicBoolean，如果要使用浮点型的原子变量需要借助floatToIntBits()／floatToIntBits()等方法。

### 原子变量是更好的volatile

因为原子变量提供了很多原子性接口，基于之前状态的符合操作(例如++i)，在使用原子变量时其原子性才能实现；仅仅使用volatile而不加锁无法实现原子性的复合操作。

### 锁和原子变量的性能对比

结论：在低竞争和中等竞争的情况下，原子变量的性能远好于锁；在竞争极高的情况下锁的戏能，锁的性能会优于原子变量。在高竞争情况下，锁的挂起线程机制会比使用CAS的CPU自旋机制性能更好。

## 非阻塞算法

非阻塞同步不会造成诸如死锁的活跃性问题，也不会造成线程优先级反转...文章开头说过了。
非阻塞同步算法常见于各种数据结构中，这一章介绍了一些非阻塞同步的例子。

### 非阻塞栈

```java
@ThreadSafe
public class ConcurrentStack <E> {
  AtomicReference<Node<E>> top = new AtomicReference<Node<E>>();
   
  public void push(E item) {
    Node<E> newHead = new Node<E>(item); Node<E> oldHead;
    do {
      oldHead = top.get();
      newHead.next = oldHead;
    } while (!top.compareAndSet(oldHead, newHead));
  }
    
  public E pop() {
    Node<E> oldHead;
    Node<E> newHead;
    do {
      oldHead = top.get(); if (oldHead == null) return null;
      newHead = oldHead.next;
    } while (!top.compareAndSet(oldHead, newHead)); return oldHead.item;
  }

  private static class Node <E> { 
    public final E item;
    public Node<E> next;
    public Node(E item) {
      this.item = item;
    } 
  }
} 
```

### 非阻塞链表

果然很难写，充满了trick，关键需要理清需一次操作要更新哪些Atomic变量，并保持对多个Atomic更新的一致性。如果下面算法看不懂，可以回去阅读原文的解释。

```java
@ThreadSafe
public class LinkedQueue <E> {
    private static class Node <E> { 
        final E item;
        final AtomicReference<Node<E>> next;
        public Node(E item, Node<E> next) {
            this.item = item;
            this.next = new AtomicReference<Node<E>>(next);
        }
    }

    private final Node<E> dummy = new Node<E>(null, null); 
    private final AtomicReference<Node<E>> head = new AtomicReference<Node<E>>(dummy);
    private final AtomicReference<Node<E>> tail = new AtomicReference<Node<E>>(dummy);

    public boolean put(E item) {
        Node<E> newNode = new Node<E>(item, null); 
        while (true) {
            Node<E> curTail = tail.get();
            Node<E> tailNext = curTail.next.get(); 
            if (curTail == tail.get()) {
                if (tailNext != null) {  
                    // Queue in intermediate state, advance tail 
                    tail.compareAndSet(curTail, tailNext); 
                } else {
                    // In quiescent state, try inserting new node
                    if (curTail.next.compareAndSet(null, newNode)) {
                        // Insertion succeeded, try advancing tail
                        tail.compareAndSet(curTail, newNode);
                        return true;
                    }
                }
            }
        }
    }
}
```

### Atomic field updater

AtomicFieldUpdater作为一个“Helper类”，可以为一般的volatile变量提供CAS操作。
与上一节的实现不同，在Java库中的非阻塞链表的实现里，Node类的next字段是一个volatile引用，并使用AtomicFieldUpdater更新；因为Node类对象会频繁的创建和销毁，这样做可以避免创建Atomic对象的消耗。

### ABA错误

在CAS类操作中出现的ABA错误可以描述如下：从变量中取出值位A -> 变量值从A变为B又变回A -> 通过CAS并没有发现变量经历的变化。在一些情况下只要变量值保持不变，就可以认为变量没有发生变化；但在另一些情况下，需要观察到这个变化。
比如说，想象一下在使用对象池的时候，对象地址虽然没变但可能在逻辑上已经是一个新的对象了。
通常解决方案就是在对象引用里加入一个tag，以标记引用是否经历过改变，`AtomicStampedReference`就提供了这个功能。