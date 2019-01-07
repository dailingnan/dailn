---
title: 啃透Java并发之原子操作类(AtomicLong源码分析)和非阻塞算法篇
date: 2019-01-06 17:13:43
categories: "Java并发编程"
tags: [Java,并发,多线程]
---

<Excerpt in index | 首页摘要> 

# 背景

​	近年来，在并发算法领域的大多数研究都侧重于非阻塞算法，这种算法用底层的原子机器指令（例如比较并发交换指令）代替锁来确保数据在并发访问中的一致性。非阻塞算法被广泛的用于在操作系统和JVM中实现线程/进程调度机制、垃圾回收机制以及锁和其他并发数据结构。

<!-- more -->
<The rest of contents | 余下全文>

​	与基于锁的方案相比，非阻塞算法在设计和实现上都要复杂的多，但他们在可伸缩性和活跃性上却拥有巨大的优势，由于非阻塞算法可以使多个线程在竞争相同数据时不会发生阻塞，因此它能在粒度更细的层次上面进行协调，并且极大的减少调度开销。锁虽然Java语言锁定语法比较简洁，但JVM操作和管理锁时，需要完成的工作缺并不简单，在实现锁定时需要遍历JVM中一条复杂的代码路径，并可能导致操作系统级的锁定、线程挂起以及上下文切换等操作。



# 非阻塞算法

​	在基于锁的算法中可能会发生各种活跃性故障，如果线程在持有锁时由于阻塞I/O，内存页缺失或其他延迟执行，那么很可能所有线程都不能继续执行下去。如果在某种算法中，一个线程的失败或挂起不会导致其他线程也失败或挂起，那么这种算法就称为非阻塞算法。如果在算法的每个步骤中都存在某个线程能够执行下去，那么这种算法也被称为无锁算法。如果在算法中仅将CAS用于协调线程之间的操作，并且能够正确的实现，那么他既是一种无阻塞算法，又是一种无锁算法。

​	Java对非阻塞算法的支持：从Java5.0开始，底层可以使用原子变量类（例如AtomicInteger和AtoMicReference）来构建高效的非阻塞算法，底层实现采用的是一个比较并交换指令（CAS）。



# 比较并交换（CAS）

​	CAS包括了三个操作数，需要读写的内存位置V，进行比较的值A和拟写入的新值B。当且仅当V的值等于A时，CAS才会通过原子方式用新值B来更新A的值，否则不会执行任何操作。无论V的值是否等于A，都将返回V原有的值。CAS的含义是：我认为V的值应该是A，如果是那么将V的值更新为B，否则不修改并告诉V的值实际为多少。



# 原子变量类

​	原子变量（对应内存模型中的原子性）比锁的粒度更细。量级更轻，并且对于在多处理器系统上实现高性能的并发代码来说是非常关键的。原子变量将发生竞争的范围缩小到单个变量上面，这是你获得的粒度最细的情况。更新原子变量的快速（非竞争）路径不会被获得锁的快速路径慢，并且通常会更快，而它的慢速路径肯定比锁的慢速路径块，因为他不需要挂起或者重新调度线程。在使用基于原子变量而非锁的算法中，线程在执行时更不易出现延迟，并且如果遇到竞争，也更容易恢复过来。



# Java中的13个原子操作类

​	Java从JDK1.5开始提供了java.util.concurrent.atomic包（以下简称Atomic包），这个包中的原子操作类提供了一种用法简单、性能高效、线程安全地更新一个变量的方式。因为变量的类型有很多种，所以在Atomic包里一共提供了13个类，属于4种类型的原子更新方式，分别是原子更新基本类型、原子更新数组、原子更新引用和原子更新属性（字段）。Atomic包里的类基本都是使用Unsafe实现的包装类。

1. 原子更新基本类型类：使用原子的方式更新基本类型，Atomic包提供了以下3个类。
   * AtomicBoolean：原子更新布尔类型。
   * AtomicInteger：原子更新整型。
   * AtomicLong：原子更新长整型。
2. 原子更新数组：通过原子的方式更新数组里的某个元素，Atomic包提供了以下4个类。
   * AtomicIntegerArray：原子更新整型数组里的元素。
   * AtomicLongArray：原子更新长整型数组里的元素。
   * AtomicReferenceArray：原子更新引用类型数组里的元素。
3. 原子更新引用类型：原子更新基本类型的AtomicInteger，只能更新一个变量，如果要原子更新多个变量，就需要使用这个原子更新引用类型提供的类。Atomic包提供了以下3个类。
   * AtomicReference：原子更新引用类型。
   * AtomicReferenceFieldUpdater：原子更新引用类型里的字段。
   * AtomicMarkableReference：原子更新带有标记位的引用类型。可以原子更新一个布尔类型的标记位和引用类型。构造方法是AtomicMarkableReference（V initialRef，boolean initialMark）。
4. 原子更新字段类：如果需原子地更新某个类里的某个字段时，就需要使用原子更新字段类，Atomic包提供了以下3个类进行原子字段更新。
   * AtomicIntegerFieldUpdater：原子更新整型的字段的更新器。
   * AtomicLongFieldUpdater：原子更新长整型字段的更新器。
   * AtomicStampedReference：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于原子的更新数据和数据的版本号，可以解决使用CAS进行原子更新时可能出现的ABA问题。

# AtomicLong源码分析

​	上面的4种原子类型都是基于CAS实现，低层借助于unsafe实现原子操作。接下来结合源码，看一下比较有代表性的AtomicLong源码.

## 初始化

```java
//保存AtomicLong的实际值，用volatile 修饰保证可见性
private volatile long value;
 
// 获取value的内存地址的逻辑操作
    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicLong.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
 
 
//根据传入的参数初始化实际值，默认值为0
public AtomicLong(long initialValue) {
        value = initialValue;
    }
```

## 主要更新方法

```java
//以原子方式更新值为传入的newValue，并返回更新之前的值
public final long getAndSet(long newValue) {
        return unsafe.getAndSetLong(this, valueOffset, newValue);
    }
 
 
//输入期望值和更新值，如果输入的值等于预期值，则以原子方式更新该值为输入的值
public final boolean compareAndSet(long expect, long update) {
        return unsafe.compareAndSwapLong(this, valueOffset, expect, update);
    }
 
//返回当前值原子加1后的值
public final long getAndIncrement() {
        return unsafe.getAndAddLong(this, valueOffset, 1L);
    }
 
//返回当前值原子减1后的值
public final long getAndDecrement() {
        return unsafe.getAndAddLong(this, valueOffset, -1L);
    }
 
//返回当前值原子增加delta后的值
public final long getAndAdd(long delta) {
        return unsafe.getAndAddLong(this, valueOffset, delta);
    }
 
```

## unsafe.getAndAddLong

```java
 
public native long getLongVolatile(Object var1, long var2); 
 
 
public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);
 
/*
unsafe.getAndAddLong(this, valueOffset, 1L)
var1 当前值
var2 value值在AtomicLong对象中的内存偏移地址
*/
 
public final long getAndAddLong(Object var1, long var2, long var4) {
        long var6;
        do {
            //根据var1和var2得出当前变量的值，以便接下来执行更新操作
            var6 = this.getLongVolatile(var1, var2);
 
            //如果当前值为var6，则将值加var4，这样做是确保每次更新时，变量的值是没有被其他线
//程修改过的值，如果被修改，则重新获取最新值更新，直到更新成功
        } while(!this.compareAndSwapLong(var1, var2, var6, var6 + var4));
 
        return var6;
    }
 
```

从源码可以看出，获取当前值getLongVolatile方法，比较并交换compareAndSwapLong方法都是native方法。说明不是采用java实现原子操作的，具体各位同学可以继续去查看底层源码（应该是c++）实现，这里不在深入了（能力有限）。



# 比较并交换的缺陷

1. 通过源码可以看出，原子更新时，会先获取当前值，确保当前值没被修改过后在进行更新操作，这也意味着如果竞争十分激烈，CAS的效率是有可能比锁更低的（一般在实际中不会出现这种情况），JDK后面推出了LongAdd，粒度更小，竞争也会被分散到更低，具体实现各位同学可以自行了解。
2. ABA是谈到CAS不可避免的话题，比较并交换，会存在这样一个场景，当变量为值A时，将值执行更新。然而在实际中，有可能其他线程将值先改为B，然后又将值改回A，此时还是能够成功执行更新操作的（对于某些不在乎过程的没啥影响，对于链表之类的就不满足了）。解决方式是给变量打上版本号，如果版本号和值一致才执行更新操作（可使用AtomicStampedReference）。

# 总结

非阻塞算法通过底层的并发原语（例如比较交换而不是锁）来维持线程的安全性。这些底层的原语通过原子变量类向外公开，这些类也用做一种“更好的volatile变量”，从而为整数和对象引用提供原子的更新操作。



参考书籍：

[《JAVA并发编程实战》](https://www.baidu.com/s?wd=%E3%80%8AJAVA%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E5%AE%9E%E6%88%98%E3%80%8B&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)

[《JAVA并发编程的艺术》](https://www.baidu.com/s?wd=%E3%80%8AJAVA%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E7%9A%84%E8%89%BA%E6%9C%AF%E3%80%8B&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)