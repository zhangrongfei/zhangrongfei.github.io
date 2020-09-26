---
title: Java线程的6种状态分析
description: 在Java编码过程和问题定位中，线程的状态是大家避免不了需要了解和关注的，根据源码，我们可知Java里面存在NEW、RUNNABLE、BLOCKED、WAITING、TIMED_WAITING、TERMINATED这6中状态，在这篇简文中，和大家一起介绍一下这6种状态的发生原因和状态之间的切换。
date: 2019-03-29 20:21:56
categories:
- Foo
- Bar
- Baz
---

想起来写一下Java线程状态，还是源起与最近的一次问题定位，当时碰到一个偶先超时的问题，使用jstack命令打印出堆栈信息之后，例如
```

"transport-vert.x-eventloop-thread-11" #37 prio=5 os_prio=0 tid=0x00007f628d0f8800 nid=0x2a32 runnable [0x00007f62631f8000]
   java.lang.Thread.State: RUNNABLE
        at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
        at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:269)
        at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:93)
        at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
        - locked <0x00000006f3cb4310> (a io.netty.channel.nio.SelectedSelectionKeySet)
        - locked <0x00000006f3cb43d8> (a java.util.Collections$UnmodifiableSet)
        - locked <0x00000006f3cb42c8> (a sun.nio.ch.EPollSelectorImpl)
        at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:97)
        at io.netty.channel.nio.SelectedSelectionKeySetSelector.select(SelectedSelectionKeySetSelector.java:62)
        at io.netty.channel.nio.NioEventLoop.select(NioEventLoop.java:755)
        at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:410)
        at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:884)
        at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30)
        at java.lang.Thread.run(Thread.java:748)
```
通过定位线程的状态，找到了错误的原因，也稍微费了些劲，所以想到把线程的一些状态再这里面总结一下
## 线程的6种状态
Java线程`Thread`在`package java.lang;`中可以找到，通过源码，我们可以看到其状态有如下6种
* **NEW**
* **RUNNABLE**
* **BLOCKED**
* **WAITING**
* **TIMED_WAITING**
* **TERMINATED**

下面分别解释一下各种状态

### NEW
顾名思义，这个状态，只存在于线程刚创建，未`start`之前，例如
```java
        MyThread thread = new MyThread();
        System.out.println(thread.getState());
```
此时打印出来的状态就是NEW

### RUNNABLE
这个状态的线程，其正在JVM中执行，但是这个"执行"，不一定是真的在运行， 也有可能是在等待CPU资源。所以，在网上，有人把这个状态区分为READY和RUNNING两个，一个表示的start了，资源一到位随时可以执行，另一个表示真正的执行中，例如
```java
        MyThread thread = new MyThread(lock);
        thread.start();
        System.out.println(thread.getState());
```

### BLOCKED
这个状态，一般是线程等待获取一个锁，来继续执行下一步的操作，比较经典的就是`synchronized`关键字，这个关键字修饰的代码块或者方法，均需要获取到对应的锁，在未获取之前，其线程的状态就一直未BLOCKED，如果线程长时间处于这种状态下，我们就是当心看是否出现死锁的问题了。例如

```java
public class MyThread extends Thread {
    private byte[] lock = new byte[0];

    public MyThread(byte[] lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        synchronized (lock){
            try {
                Thread.sleep(10000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("done");

        }
    }
}
```
```java
    public static void main(String[] args) throws InterruptedException {
        byte[] lock = new byte[0];
        MyThread thread1 = new MyThread(lock);
        thread1.start();
        MyThread thread2 = new MyThread(lock);
        thread2.start();
        Thread.sleep(1000);//等一会再检查状态
        System.out.println(thread2.getState());
    }
```
此时我们看到的输出的第二个线程的状态就是BLOCKED

### WAITING
一个线程会进入这个状态，一定是执行了如下的一些代码，例如
* Object.wait()
* Thread.join()
* LockSupport.park()
当一个线程执行了Object.wait()的时候，它一定在等待另一个线程执行Object.notify()或者Object.notifyAll()。
或者一个线程thread，其在主线程中被执行了thread.join()的时候，主线程即会等待该线程执行完成。当一个线程执行了LockSupport.park()的时候，其在等待执行LockSupport.unpark(thread)。当该线程处于这种等待的时候，其状态即为WAITING。需要关注的是，这边的等待是没有时间限制的，当发现有这种状态的线程的时候，若其长时间处于这种状态，也需要关注下程序内部有无逻辑异常。例如
**LockSupport.park()**
```java
public class MyThread extends Thread {
    private byte[] lock = new byte[0];

    public MyThread(byte[] lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        LockSupport.park();
    }
}
```
```java
    public static void main(String[] args) throws InterruptedException {
        byte[] lock = new byte[0];
        MyThread thread1 = new MyThread(lock);
        thread1.start();
        Thread.sleep(100);
        System.out.println(thread1.getState());
        LockSupport.unpark(thread1);
        Thread.sleep(100);
        System.out.println(thread1.getState());
    }
```
输出WAITING和TERMINATED

**Object.wait()**
```java
public class MyThread extends Thread {
    private byte[] lock = new byte[0];

    public MyThread(byte[] lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        synchronized (lock){
            try {
                lock.wait(); //wait并允许其他线程同步lock
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
```java
    public static void main(String[] args)
        throws InterruptedException {
        byte[] lock = new byte[0];
        MyThread thread1 = new MyThread(lock);
        thread1.start();
        Thread.sleep(100);
        System.out.println(thread1.getState()); //这时候线程状态应为WAITING
        synchronized (lock){
            lock.notify(); //notify通知wait的线程
        }
        Thread.sleep(100);
        System.out.println(thread1.getState());
    }
```
输出WAITING和TERMINATED

**Thread.join()**

```java
public class MyThread extends Thread {
    private byte[] lock = new byte[0];

    public MyThread(byte[] lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
```java
public class MyThread1 extends Thread {

    Thread thread;

    public MyThread1(Thread thread) {
        this.thread = thread;
    }

    @Override
    public void run() {
        try {
            thread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
```java
public class Main {

    public static void main(String[] args)
        throws InterruptedException {
        byte[] lock = new byte[0];
        MyThread thread = new MyThread(lock);
        thread.start();
        MyThread1 thread1 = new MyThread1(thread);
        thread1.start();
        Thread.sleep(100);
        System.out.println(thread1.getState());
    }
}
```
输出为WAITING

###TIMED_WAITING
这个状态和**WAITING**状态的区别就是，这个状态的等待是有一定时效的，即可以理解为**WAITING**状态等待的时间是永久的，即必须等到某个条件符合才能继续往下走，否则线程不会被唤醒。但是**TIMED_WAITING**，等待一段时间之后，会唤醒线程去重新获取锁。当执行如下代码的时候，对应的线程会进入到**TIMED_WAITING**状态
* Thread.sleep(long)
* Object.wait(long)
* Thread.join(long)
* LockSupport.parkNanos()
* LockSupport.parkUntil()

代码示例
**Thread.sleep**
```java
public class MyThread3 extends Thread {
    @Override
    public void run() {
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
```java
        Thread thread = new MyThread3();
        thread.start();
        Thread.sleep(100);
        System.out.println(thread.getState());
```
输出为TIMED_WAITING

**Object.wait**
```java
public class MyThread4 extends Thread {
    private Object lock;

    public MyThread4(Object lock) {
        this.lock = lock;
    }

    @Override
    public void run() {

        synchronized (lock){
            try {
                lock.wait(1000);//注意，此处1s之后线程醒来，会重新尝试去获取锁，如果拿不到，后面的代码也不执行
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("lock end");
        }
    }
}
```

```java
        byte[] lock = new byte[0];
        MyThread4 thread = new MyThread4(lock);
        thread.start();
        Thread.sleep(100);
        System.out.println(thread.getState());
        Thread.sleep(2000);
        System.out.println(thread.getState());
```
输出
TIMED_WAITING
lock end
TERMINATED

其余方法类似

**TERMINATED**
这个状态很好理解，即为线程执行结束之后的状态

## 状态之间的切换


![线程状态切换.png](/images/thread-state/threadstate1.png)

以上这张图，能够较好的说明线程之前的状态切换


## 借鉴
[https://blog.csdn.net/pange1991/article/details/53860651](https://blog.csdn.net/pange1991/article/details/53860651)

感谢阅读



