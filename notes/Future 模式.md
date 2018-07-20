- [Future 模式](#future-模式)
    - [Callable 接口](#callable-接口)
    - [Future 接口](#future-接口)
    - [FutureTask](#futuretask)
        - [状态](#状态)
        - [几个重要的属性](#几个重要的属性)
        - [构造方法](#构造方法)
        - [WaitNode](#waitnode)
        - [执行任务](#执行任务)
        - [取消任务](#取消任务)
        - [获取结果](#获取结果)
    - [简单使用](#简单使用)
    - [总结](#总结)

# Future 模式
我们知道，线程运行是没有返回结果的，如果想获取结果，就必须通过共享变量或线程通信的方式来实现，这样实现是比较麻烦的，从 JDK 1.5 开始提供的 `Callable`、`Future` 以及 `FutureTask` 可以用来获取线程执行的结果。  

## Callable 接口

```java
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

这是一个函数式接口，也就是说可以在函数式编程中使用。只有一个方法 `call()`，有返回值，能够抛出异常。这个接口可以看成是 Runnable 接口的升级版。  

## Future 接口

```java
public interface Future<V> {

    /**
     * 尝试取消任务
     * 
     * 如果任务已经完成，已经取消或者因为其他原因无法取消，则会返回 false
     *
     * 如果执行成功，并且此时任务尚未启动，则任务永远不会运行
     * 如果任务已经启动，则根据参数 mayInterruptIfRunning 值来确定是否尝试中断线程
     *
     * 此方法返回后，调用 isDone 会始终返回 true
     */
    boolean cancel(boolean mayInterruptIfRunning);

    /**
     * 判断任务是否已经被取消
     *
     * @return {@code true} if this task was cancelled before it completed
     */
    boolean isCancelled();

    /**
     * 判断任务是否已经完成
     *
     * @return {@code true} if this task completed
     */
    boolean isDone();

    /**
     * 获取执行结果，如果执行没有结束则会阻塞
     */
    V get() throws InterruptedException, ExecutionException;

    /**
     * 在指定的时间内获取结果，如果超时时间结束没有返回结果，则抛出超时异常
     */
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

## FutureTask
![FutureTask](https://github.com/nekolr/java-notes/blob/master/images/Java%20并发/FutureTask.png)

### 状态

```java
// 任务的运行状态
private volatile int state;
// 初始状态
private static final int NEW          = 0;
// 完成期间使用，瞬时态
private static final int COMPLETING   = 1;
// 正常运行和结束
private static final int NORMAL       = 2;
// 异常
private static final int EXCEPTIONAL  = 3;
// 取消完成
private static final int CANCELLED    = 4;
// 中断期间使用，瞬时态
private static final int INTERRUPTING = 5;
// 中断完成
private static final int INTERRUPTED  = 6;
```
除了开始、结束、取消等这些“完成状态”，还定义了几个中间的瞬时状态。  

可能的状态转换包括：  
- NEW -> COMPLETING -> NORMAL
- NEW -> COMPLETING -> EXCEPTIONAL
- NEW -> CANCELLED
- NEW -> INTERRUPTING -> INTERRUPTED

### 几个重要的属性

```java
private Callable<V> callable;
/** 保存执行结果或者是 get 方法获取时的异常对象 */
private Object outcome; // non-volatile, protected by state reads/writes
/** 执行 call 方法的线程 */
private volatile Thread runner;
/** 记录在 Treiber 堆栈中等待的线程 */
private volatile WaitNode waiters;

// Unsafe mechanics
private static final sun.misc.Unsafe UNSAFE;
private static final long stateOffset;
private static final long runnerOffset;
private static final long waitersOffset;
static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> k = FutureTask.class;
        // 状态 state 的偏移量
        stateOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("state"));
        // 正在运行的线程 runner 的偏移量
        runnerOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("runner"));
        // waiters 的偏移量
        waitersOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("waiters"));
    } catch (Exception e) {
        throw new Error(e);
    }
}
```

### 构造方法

```java
/**
* 传入 Callable，初始化状态为 NEW
*/
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}

/**
* 传入 Runnable 和 一个结果，任务执行 Runnable 的 run 方法，
* 在执行结束时返回给定的结果。
*/
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

```java
public static <T> Callable<T> callable(Runnable task, T result) {
    if (task == null)
        throw new NullPointerException();
    return new RunnableAdapter<T>(task, result);
}
/**
* 默认实现了一套 Callable，call 方法执行 Runnable 的 run 方法，并在
* 执行完成后返回指定的结果
*/
static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }
```

特别的是，可以在构造时传入 `Runnable` 和一个特定的结果，运行时执行的是 `Runnable` 的 run 方法，并在执行完成，调用 get 方法时返回指定的结果。如果不需要特定的结果，可以写成：`Future<?> f = new FutureTask<Void>(runnable, null)`。  

### WaitNode
WaitNode 是 FutureTask 的静态内部类，简单实现一个链表，用来记录在 Treiber 堆栈中等待的线程。  

```java
static final class WaitNode {
    volatile Thread thread;
    volatile WaitNode next;
    WaitNode() { thread = Thread.currentThread(); }
}
```

### 执行任务
`FutureTask` 实现了 `RunnableFuture` 接口，而该接口实现了 `Runnable` 和 `Future` 接口，因此它的 run 方法就是任务真正执行的部分。  

```java
public void run() {
    // 如果状态不是 NEW 或者 CAS 更新 runner 为当前线程失败
    // 则什么也不执行
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                        null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                // 执行 call 方法，这里是同步执行，直到执行完成才会有返回值
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                // 异常，结果为 null
                result = null;
                ran = false;
                // 设置异常值，将结果置为异常对象
                setException(ex);
            }
            if (ran)
                // 设置结果
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        // 如果状态为正在中断或中断完成
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

```java
protected void setException(Throwable t) {
    // CAS 更新状态为正在完成状态
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        // 设置结果为异常对象
        outcome = t;
        // 将状态设置为 EXCEPTIONAL
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
        // 完成动作
        finishCompletion();
    }
}
```

```java
/**
* 完成时执行的动作
*/
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        // 更新 waiters 为 null
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            // 自旋
            for (;;) {
                // 获取等待的线程
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    // 唤醒线程
                    LockSupport.unpark(t);
                }
                // 指向下一个节点
                WaitNode next = q.next;
                // 没有下一个节点，跳出循环
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }
    // 当 isDone 为 true 时调用，默认为空方法，子类可以重写该方法
    // 来完成回调等操作，可以在此方法中查询状态，以确定是否已经取消任务
    done();

    callable = null;        // to reduce footprint
}
```

```java
/**
* 设置返回结果
*/
protected void set(V v) {
    // 设置状态为正在完成
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        // 保存结果
        outcome = v;
        // 设置状态为正常结束
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        // 执行完成动作
        finishCompletion();
    }
}
```

```java
private void handlePossibleCancellationInterrupt(int s) {
    //状态为中断中，则提醒可以让出线程占用
    if (s == INTERRUPTING)
        while (state == INTERRUPTING)
            Thread.yield(); // wait out pending interrupt
}
```

### 取消任务

```java
/**
* 取消任务，参数 true 表示如果任务在运行时是否处理中断
*/
public boolean cancel(boolean mayInterruptIfRunning) {
    // 状态为 NEW 并且根据参数值更新状态为中断中或已经取消
    if (!(state == NEW &&
            UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    // 设置中断状态
                    t.interrupt();
            } finally { // final state
                // 更新状态为中断完成
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        // 执行完成动作
        finishCompletion();
    }
    return true;
}
```

### 获取结果

```java
/**
* 获取结果
* @throws CancellationException {@inheritDoc}
*/
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    // 如果状态为 NEW 或者为正在完成，说明可能正在设置结果，则等待
    if (s <= COMPLETING)
        // 等待完成
        s = awaitDone(false, 0L);
    // 
    return report(s);
}
```

```java
/**
* 等待完成或者等待超时
*
* @param timed true if use timed waits
* @param nanos time to wait, if timed
* @return state upon completion
*/
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    // 自旋
    for (;;) {
        // 如果中断发生
        if (Thread.interrupted()) {
            //
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        // 如果状态不是 NEW 和 正在完成，直接返回状态
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        // 正在完成，能做的只有让出时间片
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)
            q = new WaitNode();
        // 如果没有排队的节点
        else if (!queued)
            // 更新等待的节点
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                    q.next = waiters, q);
        else if (timed) {
            // 计算剩余时间
            nanos = deadline - System.nanoTime();
            // 如果已经超时，则移除等待节点，返回状态
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            // 阻塞当前线程 nanos 时间
            LockSupport.parkNanos(this, nanos);
        }
        else
            // 阻塞当前线程，即阻塞调用 get 方法的线程
            LockSupport.park(this);
    }
}
```

```java
/**
* 移除等待节点
*/
private void removeWaiter(WaitNode node) {
    if (node != null) {
        node.thread = null;
        retry:
        for (;;) {          // restart on removeWaiter race
            for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                s = q.next;
                if (q.thread != null)
                    pred = q;
                else if (pred != null) {
                    pred.next = s;
                    if (pred.thread == null) // check for race
                        continue retry;
                }
                else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                        q, s))
                    continue retry;
            }
            break;
        }
    }
}
```

```java
private V report(int s) throws ExecutionException {
    // 获取结果值
    Object x = outcome;
    // 如果状态为正常结束，返回结果
    if (s == NORMAL)
        return (V)x;
    // 如果状态为取消、中断中、中断完成，则抛出取消异常
    if (s >= CANCELLED)
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}
```

```java
public V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    if (unit == null)
        throw new NullPointerException();
    int s = state;
    // 超时时间结束还没有结果，则抛出超时异常
    if (s <= COMPLETING &&
        (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
        throw new TimeoutException();
    return report(s);
}
```

## 简单使用

```java
FutureTask<Integer> task = new FutureTask<>(() -> {
    int sum = 0;
    for (int i = 0; i < 100; i++) {
        sum += i;
    }
    return sum;
});

Thread thread = new Thread(task, "thread-1");
thread.start();

Integer sum = task.get();
System.out.println(sum);
```

```java
FutureTask<Integer> task = new FutureTask<>(() -> {
    int sum = 0;
    for (int i = 0; i < 100; i++) {
        sum += i;
    }
    return sum;
});

ExecutorService executor = Executors.newSingleThreadExecutor();
executor.submit(task);

Integer sum = task.get();
System.out.println(sum);
```

## 总结

我们使用线程执行任务的入口就是 Runnable 的 run 方法，既然 run 方法没有返回值，自然想到在 run 方法执行完成后赋值给成员变量，然后直接获取该成员变量的值即可。这就是 Future 模式的思想，当然，说起来容易，Future 处理了几个重要的点。  

- 由于需要实现 Runnable 接口（FutureTask），所以具体的任务就需要一个新的接口来传入，这就是 Callable 接口出现的原因。  
- 如果不加处理，在线程执行完成后，直接获取成员变量的值，很大可能是得不到返回值的。因此就需要某种机制，当线程没有执行完成时，获取值会阻塞调用获取值的线程，直到有返回值或取消了任务。  
FutureTask 使用 LockSupport 实现线程的阻塞和唤醒，同时使用 WaitNode 简单实现了一个等待队列