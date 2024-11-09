
---
layout: post
tags: Java
categories: 经验心得
title:  "java中使用ReentrantReadWriteLock锁机制来控制多个线程对共享资源的互斥访问"
---

> Java 提供了两种锁机制来控制多个线程对共享资源的互斥访问，第一个是 JVM 实现的 synchronized，而另一个是 JDK 实现的 ReentrantLock。

### 1. `synchronized`

`synchronized`是一个修饰关键词, 可以同步一个代码块, 一个类, 一个方法(含静态方法), 细粒度很不错, 使用方法是: 直接在作用域关键词后面添加`synchronized`就可以了

```java
// 静态方法
public synchronized static void fun() {
    // bla.. bla..
}
```

```java
// 类
public void func() {
    synchronized (Demo.class) {
        // ...blabla
    }
}
```

### 2. `ReenTrantLock`

重点说一下这种方法, `ReenTrantLock`是`JUC(java util concurrent)`包下的锁,它衍生出的有`ReentrantReadWriteLock` 这种机制很简单, 打个比方说现在有个List集合里面存着一批id, 有点类似工厂模式, 有生产id和消费id的方法, 算了, 用代码来说事儿吧!

```java
public Demo class {
    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    private List<String> ids = new ArrayList<>();
    public void saveId() {
        lock.writeLock().lock();    // 先上锁, 一个以上的线程会在此处等待
        try {
            for(int i = 0; i<5; i++) {
                ids.add(i);
            }
        } catch(Exception e) {
            
        } finally {
            // 无论执行成功与否, 都要释放锁, 否则会出现死锁
            lock.writeLock().unlock();
        }
    }
}
```

### 3. 我们该选哪个使用?

对比两者各有优缺点:
- synchronized 是 JVM 实现的，而 ReentrantLock 是 JDK 实现的。
- 新版本 Java 对 synchronized 进行了很多优化，例如自旋锁等，synchronized 与 ReentrantLock 大致相同。
- 当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情。ReentrantLock 可中断，而 synchronized 不行。
- 公平锁是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁。synchronized 中的锁是非公平的，ReentrantLock 默认情况下也是非公平的，但是也可以是公平的。
- 一个 ReentrantLock 可以同时绑定多个 Condition 对象。

除非需要使用 `ReentrantLock` 的高级功能，否则优先使用 `synchronized`。这是因为 `synchronized` 是 JVM 实现的一种锁机制，JVM 原生地支持它，而 `ReentrantLock` 不是所有的 JDK 版本都支持。并且使用 synchronized 不用担心没有释放锁而导致死锁问题，因为 JVM 会确保锁的释放。