---
title: Guava中RateLimiter（流控）简介
description: 一般的做后台服务的，都会接触到流控，一般的场景就是在流量异常，比如遭受攻击的时候，保障服务不过载，在可支持的范围内提供稳定的服务。比如我们的服务支持100QPS，当一下子来了1000个请求的时候，我们在可服务的范围内，每秒处理100个请求，这样在牺牲一些响应时效性的时候，可以保证服务不会crash。
date: 2019-11-22 23:30:09
categories:
- Foo
- Bar
- Baz
---

## 流控作用
一般的做后台服务的，都会接触到流控，一般的场景就是在流量异常，比如遭受攻击的时候，保障服务不过载，在可支持的范围内提供稳定的服务。比如我们的服务支持100QPS，当一下子来了1000个请求的时候，我们在可服务的范围内，每秒处理100个请求，这样在牺牲一些响应时效性的时候，可以保证服务不会crash。
## Guava中RateLimiter
### 示例
Guava给我们提供了好用的流控工具，简单使用场景如下
```java
    private static RateLimiter rateLimiter = RateLimiter.create(5);

    public static void main(String[] args) throws InterruptedException {
        while (true) {
            get(1);
        }
    }

    private static void get(int permits) {
        rateLimiter.acquire(permits);
        System.out.println(System.currentTimeMillis());
    }
```
运行这个简单的代码片段，从打印的时间戳可以看出来，每200ms打印一次，即正好控制QPS为5，同时保证稳定的速率。
### 原理
简单来说，就是当有大量请求进来的时候，限制请求的频率，维持其在一个稳定的区间。而其具体的方法，简单来说就是，根据上次处理的时间戳和允许的每秒允许的请求，来决定下次可以执行的时间。
而RateLimiter主要是利用了一个令牌桶的算法，如下


![令牌桶简单示意图](/images/ratelimiter/ratelimiter.png)


系统以恒定的速率产生令牌（permit），当来一个请求的时候，会请求一个或者多个令牌，当且仅当系统有这么多个令牌的时候，请求才被允许执行，否则就一直等待令牌的生成。

### 源码
相关源码基于guava-28.0-jre的版本
相关的核心类均在`com.google.common.util.concurrent`里面，可见这些方法都是线程安全的，具体有如下几个
* `RateLimiter`流控主类，也是一个抽象类
* `SmoothRateLimiter`平滑流控类，这是Guava默认实现的一种流控方式，保障服务器已稳定的速率处理请求或者获取资源
* `SmoothWarmingUp`该类实现一个热启动的功能，即流量由低到高，然后达到一个稳定的状态
* `SmoothBursty`该类支持突发请求的状况，支持一下子来很多请求（但是在可控范围内）的情况
* `SleepingStopwatch`实现一个不可中断的sleep的操作

下面简单介绍下几个类的关系，UML类图关系如下


![RateLimiter相关类图](/images/ratelimiter/ratelimiter-class.png)


#### RateLimiter
该类是对外暴露使用的类，根据注释我们知道它实现了如下功能
>一个RateLimiter，从概念上面说，是在一个指定的速率上分发许可（permit）。当每次来请求的时候，线程会阻塞，直到获取到可用的permit，使用完这些permit之后不需要进行释放的操作。
RateLimiter经常使用在需要限制物理或者逻辑资源的获取速率的时候。对比于`java.util.concurrent.Semaphore`一般用来限制并发而不是速率，虽然两者也是用一定的关联的（http://en.wikipedia.org/wiki/Little%27s_law"）
一般情况下，permit以一个固定的速率分发，如果我们一分钟分发5个permit，那么我们每200ms能获取到一个permit。当然，它也支持一种预热期，在一段时间内，分发的速率慢慢达到预定的值。
需要注意的是，一次性请求几个permit不会对本次请求造成额外的限制，比如一次请求1个permit或者1000个permit，但是会对下一个请求造成影响。例如当一个RateLimiter空闲的时候，来了一个大任务（需要较多的permit）先来的时候，会先得到保障来被执行，这时候在它后面到来的任务则需要进行额外的等待前面一个任务的permit完全获取到，这就是其需要额外付出的代价。需要注意的是，RateLimiter并不能保证公平性。
##### 原理
* 保持分发的速率，以一定速率分发令牌，比如我们设置`permitsPerSecond`为500的话，则每2毫秒产生一个令牌
* 令牌会存储，若一定时间没有请求，可用令牌会存储下来，当然会有一个上限值，当下次来请求的时候，优先使用现有的存储的令牌
* 会有一个`nextFreeTicketMicros`来记录下次有可用令牌的时间戳，在这个时间之前，所有的请求均不能通过
##### 核心方法
`public static RateLimiter create(double permitsPerSecond)`
该方法会创建一个RateLimiter实例，其每秒产生permitsPerSecond个令牌
`public double acquire(int permits)`
该方法是用于获取N个令牌的方法，如果系统内令牌不够，则一直等待直到有足够令牌可用
`public boolean tryAcquire(int permits, Duration timeout)`
该方法用户获取另外，如果在timeout时间内可以获取到足够的令牌，则等待，否则直接返回false

##### 代码实现
首先我们看下`acquire`方法
```java
  public double acquire(int permits) {
    long microsToWait = reserve(permits);
    stopwatch.sleepMicrosUninterruptibly(microsToWait);
    return 1.0 * microsToWait / SECONDS.toMicros(1L);
  }
```
代码比较简单，就如下3步
* 根据所需令牌计算等待时间
* 执行等待的动作
* 返回等待的毫秒数
计算等待时间的函数是`reserve`，其相关的实现
```java
  final long reserve(int permits) {
    checkPermits(permits);
    synchronized (mutex()) {
      return reserveAndGetWaitLength(permits, stopwatch.readMicros());
    }
  }
```
```java
  final long reserveAndGetWaitLength(int permits, long nowMicros) {
    long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
    return max(momentAvailable - nowMicros, 0);
  }
```
```java
  final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
    resync(nowMicros);//若当前时间大于nextFreeTicketMicros，则需要将storedPermits的值同步
    long returnValue = nextFreeTicketMicros;
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);//获取可使用的storedPermits
    double freshPermits = requiredPermits - storedPermitsToSpend;
    long waitMicros =
        storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
            + (long) (freshPermits * stableIntervalMicros);//如若需要新的令牌，则计算需要等待的时间

    this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);//重新计算下次令牌可用的时间
    this.storedPermits -= storedPermitsToSpend;
    return returnValue;
  }
```

还有一种`SmoothWarmingUp`的实现，比较复杂，下回有机会再看。
