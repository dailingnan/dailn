---
title: 啃透Java并发之死锁篇
date: 2019-01-06 17:13:43
categories: "Java并发编程"
tags: [Java,并发,多线程]
---

<Excerpt in index | 首页摘要> 

# 背景

​	在多线程中，我们使用加锁机制来确保线程安全，但如果使用不当，则可能导致死锁。JVM解决死锁问题方面，并不像数据库服务那么强大（数据库系统设计中考虑了检测死锁以及从死锁中恢复），当一组Java线程发生死锁时，"游戏"到此结束，这些线程永远不能再使用了。

<!-- more -->
<The rest of contents | 余下全文>

# 死锁的含义

​	当线程A持有锁L并想获得锁M的同时，线程B持有锁M并尝试所得锁L，那么这两个线程将永远的等待下去，这种情况就是最简单的死锁。

# 死锁产生的四个必要条件

* 互斥条件：进程要求对所分配的资源（如打印机）进行排他性控制，即在一段时间内某 资源仅为一个进程所占有。此时若有其他进程请求该资源，则请求进程只能等待。
* 不剥夺条件：进程所获得的资源在未使用完毕之前，不能被其他进程强行夺走，即只能 由获得该资源的进程自己来释放（只能是主动释放)。
* 请求和保持条件：进程已经保持了至少一个资源，但又提出了新的资源请求，而该资源 已被其他进程占有，此时请求进程被阻塞，但对自己已获得的资源保持不放。
* 循环等待条件：存在一种进程资源的循环等待链，链中每一个进程已获得的资源同时被 链中下一个进程所请求。

# 死锁示例

```java
package com.example.demo1.controller;
 
public class LockTest {
 
    private final Object left= new Object();
 
    private final Object right = new Object();
 
    public void  test() throws InterruptedException {
 
        new Thread(()->{
            synchronized (left){
                try {
                    Thread.sleep(2000);
                    System.out.println("获得left锁");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("持有left锁，请求获取right");
                synchronized (right){
                    System.out.println("获取right成功");
                }
            }
            System.out.println("left-right解锁完毕");
        }).start();
 
 
        new Thread(()->{
            synchronized (right){
                try {
                    Thread.sleep(2000);
                    System.out.println("获得right锁");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("持有right锁，请求获取left锁");
                synchronized (left){
                    System.out.println("成功获取left锁");
                }
            }
            System.out.println("right-left解锁完毕");
        }).start();
 
 
    }
 
    public static void main(String[] args) {
        LockTest lt = new LockTest();
        try {
            lt.test();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
 
    }
 
}
```

结果：

```   java
获得right锁
持有right锁，请求获取left锁
获得left锁
持有left锁，请求获取right
```

以上是一个简单的死锁例子，产生死锁的原因是因为两个线程试图以不同的顺序来获得相同的锁。如果按照相同的顺序来请求锁，那么就不会出现循环的加锁依赖性，因此就不会产生死锁。

死锁一旦发生，在java中就只能中断程序了，所以我们需要在实际中避免死锁的发生。

# 死锁的避免

* 加锁顺序：尽量让线程按照顺序加锁。
* 加锁时限：尝试使用lock.trylock代替内置锁，当然只是在特定的场合（需要使用到定时锁，或者可中断锁的时候），否则应当优先使用内置锁，内置锁1.6优化后内置锁效率接近显示锁。
* 加锁资源：避免一个线程在锁内同时占有多个资源，尽量保证每个锁只占有一个资源。
* 加锁数量：避免一个线程同时获取多个锁（上面的例子就是一个线程获取两个锁），有需要时应该尽量缩小锁的范围，能用同步块加锁就不要用同步方法加锁。尽量使用开放调用（在调用某个方法时不需要持有锁，那么这种调用被称为开放调用）。

------

感兴趣的朋友可以学习下银行家算法（避免死锁发生的著名算法）