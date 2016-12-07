title: Java中Synchronized、Volatile、Atomic的一些事
comments: true
date: 2016-12-04 10:43:13
updated: 2016-12-04 10:43:13
tags:
- Java
- synchronized
- volatile
- atomic
- thread safe
- concurrency
categories:
- 技术
- Concurrency
---
![旅途是积累，是发现未知，是发现自己；在这世外桃源般的诺邓古村 - By Muyoo](http://7vzs9m.com1.z0.glb.clouddn.com/%E5%A4%A7%E7%90%8697.jpg)

# Summary of synchronized、atomic、volatile in Java

在学习Java并发的过程中必然会遇到同步的问题；而且有时会在synchronized、volatile和atomic之间傻傻分不清楚。由此，专门去学习了一下这三者。本文就算是学习笔记。
*java版本1.8*
<!-- more -->

## 线程安全

>Thread Safe describe some code that can be called from multiple threads without corrupting the state of the object or simply doing the thing the code must do in right order. —— 《Java Concurrency in Practice》, Brian Goetz

直译过来的意思就是，当一段代码被多个线程调用时，其中的被使用的对象的状态不发生冲突，或按照正确的顺序执行任务。

一个比较简单的例子展示不是线程安全：多线程下执行```i++```的操作，

	// 1. 先取得 i 的值
	// 2. 执行 +1 
	tmp = i + 1;
	// 3. 赋值
	i = tmp;

如果是多线程的情况下，其实无法确定每个线程会在什么时候执行到上述的三个步骤，也就会导致每个线程获取到的i值不同，或是输出结果并不是我们想要的。

那么，如何做到线程安全呢？

目前能想到的思路有：

- 使用锁，synchronized 关键字
- 使用锁，Lock类
- 利用原子操作，atomic
- 利用Java提供的volatile

接下来，会就这三者写下一些记录。

## 锁，synchronized

在java中，任何对象都可以是锁，或者说，任何对象都可以被锁住。

>	对于同步方法，锁是当前实例对象。
	对于静态同步方法，锁是当前对象的Class对象。
	对于同步方法块，锁是Synchonized括号里配置的对象。    
	—— 《聊聊并发（二）—— Java SE1.6中的Synchronized》by 方腾飞
	
java的每个对象都有一个monitor对象，monitor对象有“进入数”这么一个值。

synchronized关键字的适用范围：

- 修饰代码块
- 修饰方法

下面就这两者分别记录一下：

- 修饰代码块

	synchronized里，这个monitor有两条指令：monitorentrer 和 monitorexit，它们是成对出现的。在JVM的规范中，对这两条指令是这么描述的：

	对于monitorenter：

	>Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes monitorenter attempts to gain ownership of the monitor associated with objectref, as follows:
	>• If the entry count of the monitor associated with objectref is zero, the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor.
	>• If the thread already owns the monitor associated with objectref, it reenters the monitor, incrementing its entry count.
	>• If another thread already owns the monitor associated with objectref, the thread blocks until the monitor's entry count is zero, then tries again to gain ownership.

	简单说，monitorenter指令是指线程尝试竞争获取monitor的所有权。当monitor的进入数为0时，线程进入monitor然后将其设置为1；如果线程已经占有这个monitor，就将进入数+1；如果有另一个线程尝试竞争monitor，就得一直阻塞着，直到进入数为0，再去竞争获取monitor所有权。

	对于monitorexit：

	>The thread that executes monitorexit must be the owner of the monitor associated with the instance referenced by objectref.
	The thread decrements the entry count of the monitor associated with objectref. If as a result the value of the entry count is zero, the thread exits the monitor and is no longer its owner. Other threads that are blocking to enter the monitor are allowed to attempt to do so.

	即是，将monitor的进入数-1，且只有拥有这个monitor的线程才能执行这个操作。

- 修饰方法
	synchronized修饰方法时，在字节码中这个方法会有一个ACC_SYNCHRONIZED标识。JVM会隐式执行enter和exit，不会有显式的monitorenter和monitorexit指令。实际上其过程与修饰代码块时是一样的。

好了，synchronized 用于锁一个对象的过程如上。但在上面的描述中有一个问题：线程是怎么竞争monitor的呢？

要回答这个问题，得先了解一下CAS操作和synchronized对应的3种有锁的状态：

- 偏向锁状态
- 轻量级锁状态
- 重量级锁状态

先来简单提一下CAS操作。

- CAS
	CAS全称是 Compare and Swap，比较并设置。这是在硬件层面上做的原子操作：比较目标值，如果与拿到的当前值一致，则修改为新值；不一致，则不修改。
	>In computer science, compare-and-swap (CAS) is an atomic instruction used in multithreading to achieve synchronization. It compares the contents of a memory location to a given value and, only if they are the same, modifies the contents of that memory location to a given new value. This is done as a single atomic operation. —— Wikipedia

刚才提到的设置进入数这样的操作，即是通过CAS来完成。

上述那3种状态是存储在java对象的对象头里，有具体的状态值表示这些状态，引入这些状态是为了减少获得锁和释放锁的性能消耗。
这3种状态之间的转换关系是：偏向锁 -> 轻量级锁 -> 重量级锁，随着竞争情况逐渐升级，但不能降级。

### 偏向锁状态

偏向锁中的“偏向”的含义其实是说“指向占用锁的线程”。在java对象头里，还有个值记录占有锁的线程ID号。

- 加锁：
	当有一个线程获取锁后，会将这个值设置成自己的线程ID号。然后有线程来尝试获取这个锁时，先比较这个线程ID号。如果与当前线程一致，则直接获取锁然后执行同步代码块，不需要执行CAS操作来加锁。如果不一致，则看一下偏向锁的标识是否设置成1，如果设置了则用CAS来尝试将线程ID指向自己；如果没有设置，则用CAS来竞争锁。
	
- 释放：
	
	持有锁的线程会等到有其它线程来竞争锁时才会释放锁。撤销时，会暂停持有锁的线程，然后将线程ID设置为空，然后唤醒被暂停的线程。
	其实在“设置线程ID为空”这一步时，还有细节的操作：
	>偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态，则将对象头设置成无锁状态，如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。    —— 《聊聊并发（二）——Java SE1.6中的Synchronized》by 方腾飞
	*为什么偏向锁是这么设置的呢？Hotspot的作者经过以往的研究发现大多数情况下锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。*

### 轻量级锁

JVM会创建用于存储锁记录的空间，并将对象头里锁的状态复制到这个空间里。（创建的地方是线程的栈帧中；这个过程参考官方的Displaced Mark Word。）
	
- 加锁：

	线程用CAS操作尝试将对象头中的锁状态替换为指向刚才提到的锁记录的指针。如果成功，则线程成功获得锁；如果失败，线程则用自旋来尝试获取锁。

- 释放：
	
	解锁过程即是线程用CAS操作来将锁记录中的值替换回对象头中。
	
在这个释放过程中，如果成功了，则说明没有竞争。如果失败了，则说明有竞争，锁状态则会升级成为重量级锁。这时当有其它线程尝试获取锁时，就会blocking。

### 重量级锁

其实即是开始时提到的enter、exit的基本过程：竞争锁、blocking的方式。
	
上述内容是目前我对synchronized的理解。

## Volatile

在记录Atomic前，需要先了解一下volatile关键字。volatile关键字在java中是用于修饰变量的。

多核的处理器每个核都有自己的cache；
对于普通的变量来说，cpu读取并修改这个变量值的过程是：

1. 从main memory中读取该变量的值，然后放在自己的缓存中；
2. 下次读的时候先从缓存中找

在写这个值的时候，cpu也是先将值写到缓存中，然后才会更新到内存。

在这样的过程中，多个线程同时从内存中取出某个变量值后，再更新、再读，这个过程在线程之间是不可见的，很可能造成内存中该变量的值错误。

而volatile关键字的作用，就会让这个变量不会从缓存中取出，更新时也不会更新到缓存，而是直接读写内存。这样，每个线程去读写这个值的时候，都能保证对其它线程来说是可见的。

>当把变量声明为volatile类型后，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其它内存操作仪器重排序。volatile变量不会缓存在寄存器或者对其他处理器不可见的地方，因此在读取volatile类型的变量时总会返回最新写入的值。 ——《Java Concurrency in Practice》

volatile相比synchronized而言，是一个轻量级的同步机制。但它的用法跟synchronized不同。
volatile只能保证可见性，不能保证原子性，而加锁可以同时保证这两者；相对地，使用volatile的开销也比加锁要轻许多。
因此，当使用volatile变量时，应注意需要满足以下几点：

1. 对变量的写入操作不依赖变量当前的值，或者只有一个线程能对其写。
2. 该变量不会与其它状态变量一起纳入不变性条件中。
3. 在访问变量时不需要加锁。




## Atomic，原子操作



## 参考资料

- [聊聊并发（二）——Java SE1.6中的Synchronized](http://www.infoq.com/cn/articles/java-se-16-synchronized)
- [Java并发编程：Synchronized及其实现原理](https://www.cnblogs.com/paddix/p/5367116.html)
- [比较并交换(compare and swap, CAS)](https://zh.wikipedia.org/wiki/%E6%AF%94%E8%BE%83%E5%B9%B6%E4%BA%A4%E6%8D%A2)
