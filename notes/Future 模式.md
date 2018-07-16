- [Future 模式](#future-模式)

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
     * @return the computed result
     * @throws CancellationException if the computation was cancelled
     * @throws ExecutionException if the computation threw an
     * exception
     * @throws InterruptedException if the current thread was interrupted
     * while waiting
     */
    V get() throws InterruptedException, ExecutionException;

    /**
     * 在指定的时间内获取结果，如果超时时间结束没有返回结果，则抛出超时异常
     * @param timeout the maximum time to wait
     * @param unit the time unit of the timeout argument
     * @return the computed result
     * @throws CancellationException if the computation was cancelled
     * @throws ExecutionException if the computation threw an
     * exception
     * @throws InterruptedException if the current thread was interrupted
     * while waiting
     * @throws TimeoutException if the wait timed out
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

## 几个重要的属性

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

## 构造方法

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
* @param runnable the runnable task
* @param result the result to return on successful completion. If
* you don't need a particular result, consider using
* constructions of the form:
* {@code Future<?> f = new FutureTask<Void>(runnable, null)}
* @throws NullPointerException if the runnable is null
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

## WaitNode
WaitNode 是 FutureTask 的静态内部类，简单实现一个链表，用来记录在 Treiber 堆栈中等待的线程。  

```java
static final class WaitNode {
    volatile Thread thread;
    volatile WaitNode next;
    WaitNode() { thread = Thread.currentThread(); }
}
```

## run 方法
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
                // 执行 call 方法
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
    // 设置状态为
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}
```