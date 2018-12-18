- [CyclicBarrier](#cyclicbarrier)
    - [属性和内部类](#属性和内部类)
    - [构造方法](#构造方法)
    - [await](#await)
    - [其他方法](#其他方法)
    - [简单示例](#简单示例)

CyclicBarrier 同样也是 JDK 1.5 提供的一个同步辅助工具类，从字面理解为回环栅栏，它允许一组线程互相等待，直到它们都到达某个状态之后再全部同时执行。这个状态被称为公共屏障点（common barrier point），而叫做回环是因为当所有等待的线程都被释放后，CyclicBarrier 可以被重用。  

需要注意的是，CyclicBarrier 与 CountDownLatch 比较容易混淆，它俩的不同之处在于：  

- CountDownLatch 是允许 1 个或多个线程等待其他线程执行完毕；而 CyclicBarrier 是允许多个线程互相等待。  
- CountDownLatch 的计数器无法被重置；而 CyclicBarrier 的计数器可以被重置后重新使用。  

# 属性和内部类
```java
/** 
* 协调多个线程进行同步的锁
*/
private final ReentrantLock lock = new ReentrantLock();
/**
* 线程等待的条件
*/
private final Condition trip = lock.newCondition();
/**
* 需要同时到达 barrier 的线程个数
*/
private final int parties;
/** 
* 当 parties 个线程都到达公共屏障点（都调用了 await 方法）时，最后一个调用 await 的线程会执行的动作
*/
private final Runnable barrierCommand;
/**
* 当前的 Generation，每当屏障打破之后都会重新更新，从而实现重置
*/
private Generation generation = new Generation();
/**
* 剩下还未到达 barrier 状态的线程数量
*/
private int count;
```

```java
/**
* 用来判断 CyclicBarrier 的状态
*/
private static class Generation {
    // 表示当前的屏障是否被打破
    boolean broken = false;
}
```

# 构造方法
CyclicBarrier 的构造方法有两个，其中有一个调用的是另一个构造方法。  

```java
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    // 设置需要同时到达 barrier point 的线程个数
    this.parties = parties;
    // 设置正在处于等待状态的线程个数
    this.count = parties;
    // 当 parties 个线程到达公共屏障点时会执行的动作
    this.barrierCommand = barrierAction;
}

public CyclicBarrier(int parties) {
    // 当 parties 个线程到达公共屏障点时并不执行预定义的动作
    this(parties, null);
}
```

# await
await 方法用来使当前线程处于等待状态，直到当前线程被中断，超时时间到或者有 parties 个线程到达公共屏障点这三种情况的某一种发生时，线程才会恢复执行。  

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        // cannot happen
        throw new Error(toe);
    }
}
```

```java
/**
* timed 表示是否响应超时，nanos 表示超时等待时间
*/
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
            TimeoutException {
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        final Generation g = generation;
        // 判断屏障是否被打破
        if (g.broken)
            throw new BrokenBarrierException();
        // 如果有其他线程中断当前线程，则屏障被打破
        if (Thread.interrupted()) {
            // 打破屏障
            breakBarrier();
            // 抛出中断异常
            throw new InterruptedException();
        }

        // 剩余未到达 barrier 状态的线程数减一
        int index = --count;
        // 如果剩余数量为 0，则表示全部到达 barrier 状态，tripped
        if (index == 0) {
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    // 最后一个线程执行预定义的动作
                    command.run();
                ranAction = true;
                // 重置当前这一轮的 CyclicBarrier 的状态，开始新的一轮
                nextGeneration();
                // 返回 0 表示最后一个线程也已经到达 barrier 状态
                return 0;
            } finally {
                // 如果动作执行完毕，则当前的 CyclicBarrier 也为被打破状态
                if (!ranAction)
                    breakBarrier();
            }
        }

        // 自旋等待直到以下情况发生：
        // 1. 所有线程都到达 barrier 状态，并且最后一个线程成功执行了预定义动作
        // 2. 当前线程被其他线程中断
        // 3. 超时
        for (;;) {
            try {
                // 如果不响应超时，则会调用 await 一直等待
                if (!timed)
                    trip.await();
                // 如果响应超时并且预设的超时时间大于 0，则等待 nanos 时间，
                // 如果超时时间到，则会恢复执行
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                // 如果等待的过程中，当前线程被中断，则执行下面的逻辑
                
                /** 
                *  如果 g == generation 则表示这一轮的 CyclicBarrier 还没结束
                *  如果 broken 为 false，表示没有其他线程执行 breakBarrier 打破屏障
                */ 
                if (g == generation && ! g.broken) {
                    // 执行打破屏障
                    breakBarrier();
                    // 抛出中断异常
                    throw ie;
                } else {
                    // 执行到这里表示，此时可能当前轮已经结束，则直接返回剩余未达到 barrier
                    // 的线程个数。也有可能是之前的某个线程执行了 breakBarrier 打破了屏障，
                    // 这样后续的线程都会执行到这里，重置中断状态。
                    Thread.currentThread().interrupt();
                }
            }

            // 打破了屏障，则直接抛出异常
            if (g.broken)
                throw new BrokenBarrierException();

            // 表示此时这一轮已经结束，直接返回剩余未达到 barrier 的线程个数
            if (g != generation)
                return index;
            // 如果超时，则打破屏障，并抛出超时异常
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```

在 `await` 方法中需要牢记的就是三种打破屏障的情况。  

```java
private void nextGeneration() {
    // 唤醒所有等待的线程
    trip.signalAll();
    // 重置未到达 barrier 状态的线程个数
    count = parties;
    // 重置当前 CyclicBarrier 的状态
    generation = new Generation();
}
```

其中 `nextGeneration` 方法的作用主要是在屏障被打破之后，重置状态以待下次使用。  

```java
private void breakBarrier() {
    // 设置为打破状态
    generation.broken = true;
    // 未到达 barrier 状态的线程个数重置为起始线程个数
    count = parties;
    // 唤醒所有在条件上等待的线程
    trip.signalAll();
}
```

其中 `breakBarrier` 方法的主要作用是在屏障被打破之后设置打破状态，同时通知并唤醒其他线程。  

# 其他方法
CyclicBarrier 除了提供 await，还提供了几个方法供外部调用。  

```java
/**
*  判断当前一轮是否已经被打破了屏障
*/ 
public boolean isBroken() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return generation.broken;
    } finally {
        lock.unlock();
    }
}
```

```java
/**
*  手动重置，开始新的一轮，所有还在等待状态的线程会抛出 BrokenBarrierException 异常
*/ 
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 打破屏障
        breakBarrier();
        // 开始新的一轮
        nextGeneration();
    } finally {
        lock.unlock();
    }
}
```

```java
/**
*  获取当前还在 barrier 状态等待的线程
*/
public int getNumberWaiting() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return parties - count;
    } finally {
        lock.unlock();
    }
}
```

# 简单示例
```java
public static void main(String[] args) {

    final int NUM = 5;

    CyclicBarrier cyclicBarrier = new CyclicBarrier(NUM, () -> 
            System.out.println(Thread.currentThread().getName() + " 所有线程数据准备完毕..."));

    for (int i = 0; i < NUM; i++) {
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + " 正在准备数据...");
            try {
                // 模拟准备数据
                TimeUnit.SECONDS.sleep(3);
                // 相互等待
                cyclicBarrier.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }, "thread-" + i).start();
    }
}
```