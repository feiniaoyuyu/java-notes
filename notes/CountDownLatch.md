- [CountDownLatch](#countdownlatch)
    - [基本原理](#基本原理)
    - [构造函数](#构造函数)
    - [countDown](#countdown)
    - [await](#await)
    - [使用说明](#使用说明)
    - [简单示例](#简单示例)

JDK 从 1.5 开始提供了一个很有用的并发辅助工具类 CountDownLatch，它允许一个或多个线程一直等待，直到其他线程的操作完成后再执行。该类位于 `java.util.concurrent` 包下，因此我们很容易想到它的实现依赖的是 AQS。  

# 基本原理
CountDownLatch 内部维护了一个计数器，计数器的初始值为线程的数量。每当一个线程完成了自己的任务后，主动调用计数器减 1 的操作。当计数器的值到达 0 时，它表示所有的线程已经完成了任务，此时等待的线程就可以恢复执行。  

![CountDownLatch](https://github.com/nekolr/java-notes/blob/master/images/Java%20并发/CountDownLatch.png)

# 构造函数
```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

这个 count 其实就是我们所说的计数器，用来表示需要等待的线程数量。在构造函数中，使用 count 构造了一个 Sync 对象，而该类继承自 AQS。  

```java
private static final class Sync extends AbstractQueuedSynchronizer {
    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

    /**
    * 尝试获取共享锁，当 state 的值为 0 时，表示可以获取到锁；否则不能
    */
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    /**
    * 尝试释放共享锁，以自旋的方式执行，返回值为是否完全释放所有的锁
    */
    protected boolean tryReleaseShared(int releases) {
        for (;;) {
            // 获取 state 的值
            int c = getState();
            // 如果值为 0，表示全部释放完毕
            if (c == 0)
                return false;
            // 如果值不为 0，则将值减一，并尝试通过比较和交换修改 state 的值
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

我们可以发现，计数器其实赋给了 AQS 的 state，该变量是用来记录线程获取到锁的次数的，当它的值大于 0 时，表示当前线程获取到了锁；当它的值等于 0 时，表示当前线程还没有获取到锁。在独占模式中，state 的值最大为 1；而在共享模式中，state 的值可以大于 1。  

# countDown
countDown 方法用来释放锁，对应的 state 的值会减去 1。  

```java
public void countDown() {
    sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
    // 尝试释放锁，因为是自旋，所以一定会成功释放锁
    if (tryReleaseShared(arg)) {
        // 修改队列中节点状态，唤醒后继节点
        doReleaseShared();
        return true;
    }
    return false;
}
```

# await
await 方法用来使线程等待其他线程，通过不断尝试获取锁来实现。  

```java
public void await() throws InterruptedException {
    // 响应中断地获取共享锁
    sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 尝试获取共享锁，如果值小于 0 则表示没有获取到锁
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 当前线程入队列
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        // 自旋
        for (;;) {
            // 获取前驱节点
            final Node p = node.predecessor();
            // 如果前驱节点是头节点
            if (p == head) {
                // 尝试获取锁
                int r = tryAcquireShared(arg);
                // 如果值大于等于 0 则表示获取锁成功
                if (r >= 0) {
                    // 共享模式出队列
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        // 如果失败，则取消获取锁
        if (failed)
            cancelAcquire(node);
    }
}
```

# 使用说明
在使用 CountDownLatch 时，在其他线程全部启动以后，主线程必须立即调用 CountDownLatch 的 await 方法，让主线程去自旋尝试获取共享锁。在其他线程执行完毕后，需要它们通知 CountDownLatch 对象它们已经完成了任务，这就需要通过 CountDownLatch 的 countDown 方法来完成。每调用一次，计数器的值就减一，当所有的 n 个线程都调用了这个方法之后，计数器的值就为 0，此时主线程就可以获取到共享锁，从而恢复执行。  

# 简单示例
统计所有线程的耗时。  

```java
public static void main(String[] args) throws InterruptedException {

    final CountDownLatch latch = new CountDownLatch(10);

        long start = System.currentTimeMillis();

        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                long count = 0;
                for (int j = 0; j < 90000000; j++) {
                    count += j;
                }
                System.out.println(Thread.currentThread().getName() + "：count=" + count);
                latch.countDown();
            }, "thread-" + i).start();
        }
        // 当前线程阻塞
        latch.await();

        System.out.println(Thread.currentThread().getName() +
                "：所有线程执行耗时(ms)：" + (System.currentTimeMillis() - start));
}
```