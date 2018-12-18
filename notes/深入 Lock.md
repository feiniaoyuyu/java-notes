
<details>
<summary>synchronized 的缺陷</summary>

- [synchronized 的缺陷](#synchronized-的缺陷)

</details>
<details>
<summary>Lock 接口</summary>

- [Lock 接口](#lock-接口)
    - [lock](#lock)
    - [lockInterruptibly](#lockinterruptibly)
    - [tryLock](#trylock)
    - [unlock](#unlock)
    - [newCondition](#newcondition)
    - [Lock 接口的使用方式](#lock-接口的使用方式)

</details>
<details>
<summary>LockSupport（JDK 1.8）</summary>

- [LockSupport（JDK 1.8）](#locksupportjdk-18)
    - [几个关键属性](#几个关键属性)
    - [许可](#许可)
    - [几个重要的方法](#几个重要的方法)
        - [setBlocker() 和 getBlocker()](#setblocker-和-getblocker)
        - [park()](#park)
        - [unpark()](#unpark)
        - [LockSupport 的特性](#locksupport-的特性)

</details>
<details>
<summary>深入 AbstractQueuedSynchronizer（JDK 1.8）</summary>

- [深入 AbstractQueuedSynchronizer（JDK 1.8）](#深入-abstractqueuedsynchronizerjdk-18)
    - [几个重要属性](#几个重要属性)
    - [Node](#node)
    - [同步队列](#同步队列)
        - [入队列](#入队列)
        - [出队列](#出队列)
    - [状态](#状态)
    - [独占模式](#独占模式)
        - [独占式获取锁](#独占式获取锁)
            - [独占式不响应中断获取锁](#独占式不响应中断获取锁)
            - [独占式响应中断获取锁](#独占式响应中断获取锁)
            - [独占式响应中断和超时获取锁](#独占式响应中断和超时获取锁)
        - [独占式释放锁](#独占式释放锁)
    - [共享模式](#共享模式)
        - [共享式获取锁](#共享式获取锁)
            - [共享式不响应中断获取锁](#共享式不响应中断获取锁)
            - [共享式响应中断获取锁](#共享式响应中断获取锁)
            - [共享式响应中断和超时获取锁](#共享式响应中断和超时获取锁)
        - [共享式释放锁](#共享式释放锁)
    - [Condition](#condition)
        - [Condition 接口](condition-接口)
        - [ConditionObject](#conditionobject)
            - [入队、出队操作](#入队出队操作)
            - [awaitUninterruptibly](#awaituninterruptibly)
            - [await](#await)
            - [awaitNanos](#awaitnanos)
            - [awaitUntil](#awaituntil)
            - [signal](#signal)
            - [signalAll](#signalall)

</details>

# 深入 Lock
除了基于对象监视器锁实现的 `synchronized`，还有基于 `AbstractQueuedSynchronizer` 实现的锁 `Lock`。  

# synchronized 的缺陷

- **不可中断**  
不能中断正在等待获取锁的线程。  
- **不可超时**  
在请求锁失败后，会无限等待（自旋超时后会进入阻塞状态）。  
- **自动释放锁**  
一般情况下，自动释放锁算是一个优点，相比手动释放锁代码要更优雅，并且不用担心因为忘记手动释放锁而造成死锁。但是如果一个获取到锁的线程因为等待 IO 或者其他原因而阻塞了，但是因为线程方法没有结束，也没有发生异常，所以对象锁是没有释放的，这时其他线程只能干等。因此有时手动释放锁更具灵活性。  
- **不可伸缩**  
无法细粒度控制锁，没有读锁和写锁的分类。比如多个线程读写文件时，读操作和写操作、写操作和写操作会发生冲突，而读操作和读操作不会发生冲突，而使用内部锁时，一个线程在做读操作，其他同样做读操作的线程只能等待。  
- **性能问题**  
在没有线程竞争的情况下，由于 JDK1.6 优化的原因，内部锁的性能与 `ReentrantLock` 的性能基本持平。一旦有线程竞争时，内部锁最终会膨胀为重量级锁，性能严重下降。  

# Lock 接口

![Lock接口](https://github.com/nekolr/java-notes/blob/master/images/Java%20并发/Jay.png)

## lock
获取锁的方法。  

```java
/**
* 获取锁。
*
* 如果锁不可用（被占用），则当前线程会休眠，直到获取到锁。
*
* 该方法的实现可能能够检测到锁的错误使用，比如死锁，并且可能会在这种情况下抛出
* 未检查的异常。此时该实现需要用文档注明这种情况以及其可能出现的异常。
*/ 
void lock();
```

## lockInterruptibly
可中断的锁获取操作，可以在锁获取的过程中断当前获取锁的线程。  

```java
/**
* 可中断地获取锁，即在获取锁的过程中可以中断当前线程。
*
* 如果锁可用会立即返回；
* 如果锁不可用，则当前线程会处于休眠状态，直到发生下面两种情况之一会返回：
* 1 锁被当前线程获取；
* 2 其他线程中断当前线程。
* 
* 在一些实现中中断获取锁是不可能的，即使可能的话也可能是昂贵的操作。
*/
void lockInterruptibly() throws InterruptedException;
```

## tryLock
`tryLock()` 方法提供可定时和可轮询的锁获取方式。可定时是指在指定的时间内获取锁，在超时时间结束立即返回。可轮询是指程序可以通过循环配合 `tryLock()` 来不断尝试获取锁。  

```java
/**
* 获取锁，如果锁可用立即返回 true；如果锁不可用，立即返回 false。
* 
* 下面这种用法可以确保解锁发生在获取锁后，如果没有获取到锁，则不会尝试解锁。
* Lock lock = ...;
* if (lock.tryLock()) {
*   try {
*     // manipulate protected state
*   } finally {
*     lock.unlock();
*   }
* } else {
*   // perform alternative actions
* }}
* 
*/
boolean tryLock();
```

```java
/**
* 如果在给定的时间内空闲并且当前线程未被中断，则获取锁。
*
* 如果锁可用，则立即返回 true；
* 如果锁不可用，则当前线程会处于休眠状态，直到发生下面三种情况之一会返回：
* 1 当前线程在超时时间内获取到锁，返回 true；
* 2 其他线程中断当前线程；
* 3 指定的时间已过，如果获取到锁，则返回 true；如果还是没有获取到锁，返回 false。
*
*/
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
```

## unlock
释放锁的方法。  

```java
/**
* Releases the lock.
* 释放锁
*/
void unlock();
```

## newCondition
`Condition` 可以实现更为灵活的锁获取与释放的条件控制。  

```java
/**
* Returns a new {@link Condition} instance that is bound to this
* {@code Lock} instance.
*
* 返回一个绑定到 Lock 实例的新的 Condition 实例。
* 在调用等待方法之前，当前线程必须持有锁。调用 Condition.await() 将使线程释放锁，
* 并在等待返回之前重新获取锁。
*/
Condition newCondition();
```

## Lock 接口的使用方式
加锁需要与解锁成对出现。  

```java
Lock lock = ...;
lock.lock();
try {
    // do somethings
} finally {
    lock.unlock();
}
```

<br>[⬆ Back to top](#深入-lock)

# LockSupport（JDK 1.8）
`LockSupport` 是一个线程阻塞工具类，能够在线程内任意位置让线程阻塞和释放，底层使用 `Unsafe` 实现，我们通常不会直接使用它，而是在锁实现中作为阻塞工具使用。  

这个类与每个使用它的线程通过 permit 相关联，这个 permit 从某种意义上可以理解为 Semaphore 类。  

## 几个关键属性

```java
// Hotspot implementation via intrinsics API
private static final sun.misc.Unsafe UNSAFE;
private static final long parkBlockerOffset;
private static final long SEED;
private static final long PROBE;
private static final long SECONDARY;
static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> tk = Thread.class;
        // 这里是获取 Thread 类中 parkBlocker 字段的偏移量
        parkBlockerOffset = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("parkBlocker"));
        SEED = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSeed"));
        PROBE = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomProbe"));
        SECONDARY = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
    } catch (Exception ex) { throw new Error(ex); }
}
```

`LockSupport` 中有个重要的属性 `parkBlockerOffset`，它在类加载时通过 Unsafe 类的 `objectFieldOffset()` 方法获取到 Thread 类中 parkBlocker 字段的偏移量，拿到偏移量后，可以通过 `UNSAFE.putObject()` 给该字段赋值。  

> 注：parkBlocker 是阻塞对象，即表示线程被谁阻塞。  

## 许可
`LockSupport` 与每个使用它的线程通过 permit 关联，这个 permit 从某种意义上可以认为是 Semaphore 类，但是区别是：permit 的值最多为 1，即重复调用 `unpark()` 也不会累加。  

当调用 `unpark()` 时，permit 加 1；调用 `park()` 方法，permit 被消费掉，此时 permit = 0，再次调用 `park()`，该线程会被阻塞，因为没有许可可以使用。  

## 几个重要的方法
包括 `setBlocker()`、`getBlocker()`、`park()` 和 `unpark()` 方法。  

### setBlocker() 和 getBlocker()

```java
private static void setBlocker(Thread t, Object arg) {
    // Even though volatile, hotspot doesn't need a write barrier here.
    UNSAFE.putObject(t, parkBlockerOffset, arg);
}

public static Object getBlocker(Thread t) {
    if (t == null)
        throw new NullPointerException();
    return UNSAFE.getObjectVolatile(t, parkBlockerOffset);
}
```

`setBlocker()` 方法使用 `UNSAFE.putObject(t, parkBlockerOffset, arg)`，表示通过 Unsafe 类的 `putObject()` 方法根据偏移量找到将线程 t 的 parkBlocker 字段，并赋给值 arg。  

`getBlocker()` 方法返回最近一次调用 `park()` 方法并且尚未解除阻塞的线程的 parkBlocker 对象，如果线程没有被阻塞则返回 null。  

### park()
该方法用于等待许可，在第一次使用时默认是没有许可的，也就是在使用该方法之前需要先使用 `unpark()` 方法。  

在调用 `park()` 时，当许可可用时，立即返回并消费掉该许可；当许可不用时，当前线程被阻塞，直到发生下面三种情况之一：  
- 其他线程调用 `unpark()` 方法给当前线程提供许可。
- 其他线程中断当前线程。
- 虚假调用（无理由）返回。  

```java
public static void park(Object blocker) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 设置阻塞对象
    setBlocker(t, blocker);
    // 阻塞当前线程
    UNSAFE.park(false, 0L);
    // 清空阻塞对象值
    setBlocker(t, null);
}

public static void parkNanos(Object blocker, long nanos) {
    if (nanos > 0) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        // 阻塞 nanos 纳秒后返回
        UNSAFE.park(false, nanos);
        setBlocker(t, null);
    }
}

public static void parkUntil(Object blocker, long deadline) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    // 如果想实现从现在开始 3000 毫秒后返回，则 deadline 的值为
    // System.currentTimeMillis() + 3000
    UNSAFE.park(true, deadline);
    setBlocker(t, null);
}
```

## unpark()
该方法在线程尚没有得到许可时，为该线程提供许可。如果线程被 `park()` 方法阻塞，那么该方法将解除阻塞。  

```java
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

## LockSupport 的特性
`LockSupport` 不支持可重入，可以响应中断，但是不会抛出中断异常。  

```java
public static void main(String[] args) {
    Thread currentThread = Thread.currentThread();
    // 提供许可 permit + 1 = 1
    LockSupport.unpark(currentThread);
    // permit 的值不变
    LockSupport.unpark(currentThread);
    // 第一次调用能够执行，permit - 1 = 0
    LockSupport.park();
    System.out.println("执行第一次 park");
    // 线程会一直阻塞在该方法，因为此时 permit = 0
    LockSupport.park();
    System.out.println("执行第二次 park");
}
```

```java
public static void main(String[] args) {
    Thread thread = new Thread(() -> LockSupport.park(), "thread-1");
    thread.start();
    thread.interrupt();
}
```

<br>[⬆ Back to top](#深入-lock)

# 深入 AbstractQueuedSynchronizer（JDK 1.8）
AQS（AbstractQueuedSynchronizer）是用来构建锁和同步器（Synchronizer）的框架，它是 J.U.C（java.util.concurrent）的基础，该包下大部分的同步器都是基于这个框架构建的。  

AQS 中使用一个 volatile 修饰的 int 型变量表示同步状态，通过 FIFO 同步队列实现线程的排队，通过 Unsafe 类实现底层的等待和唤醒操作，通过 ConditionObject 实现条件变量控制。  

## 几个重要属性
AQS 中维护着几个重要的变量，包括同步状态、FIFO 队列头节点和尾节点、一些偏移量。  

```java
/**
* 队列头节点，在第一次调用 enq() 方法时会初始化，可以通过setHead() 方法修改
*/
private transient volatile Node head;

/**
* 队列尾节点，同样会在第一次调用 enq() 方法时初始化
*/
private transient volatile Node tail;

/**
* 同步状态，不是布尔值是因为该状态表示可用资源数
*/
private volatile int state;

private static final Unsafe unsafe = Unsafe.getUnsafe();
// 线程状态 state 的偏移量
private static final long stateOffset;
// 队列头节点 head 的偏移量
private static final long headOffset;
// 队列尾节点 tail 的偏移量
private static final long tailOffset;
// 节点中 waitStatus 的偏移量
private static final long waitStatusOffset;
// 节点中 next 的偏移量
private static final long nextOffset;

static {
    try {
        stateOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
        headOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
        tailOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
        waitStatusOffset = unsafe.objectFieldOffset
            (Node.class.getDeclaredField("waitStatus"));
        nextOffset = unsafe.objectFieldOffset
            (Node.class.getDeclaredField("next"));

    } catch (Exception ex) { throw new Error(ex); }
}

protected final boolean compareAndSetState(int expect, int update) {
    // 使用 CAS 操作更新 state 的值
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}

private final boolean compareAndSetHead(Node update) {
    // 使用 CAS 操作更新 head 的值，只在第一次调用 enq() 时调用
    return unsafe.compareAndSwapObject(this, headOffset, null, update);
}

private final boolean compareAndSetTail(Node expect, Node update) {
    // 使用 CAS 操作更新 tail 的值
    return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}

private static final boolean compareAndSetWaitStatus(Node node,
                                                        int expect,
                                                        int update) {
    // 使用 CAS 操作更新 Node 节点中的 waitStatus 的值
    return unsafe.compareAndSwapInt(node, waitStatusOffset,
                                    expect, update);
}

private static final boolean compareAndSetNext(Node node,
                                                Node expect,
                                                Node update) {
    // 使用 CAS 操作更新 Node 节点中的 next 的值
    return unsafe.compareAndSwapObject(node, nextOffset, expect, update);
}
```

## Node
需要排队的线程，在入队之前都会被封装成一个 Node 节点，每一个 Node 节点中都维护着一个前驱节点和一个后继节点。  

```java
static final class Node {
    /** 标记，指示节点正在以共享模式等待 */
    static final Node SHARED = new Node();
    /** 标记，指示节点正在以独占模式等待 */
    static final Node EXCLUSIVE = null;

    /** waitStatus 可能的值，表示线程（节点）已取消 */
    static final int CANCELLED =  1;
    /** waitStatus 可能的值，表示后继线程（节点）需要许可（unpark） */
    static final int SIGNAL    = -1;
    /** waitStatus 可能的值，表示线程（节点）在条件上等待，即此节点当前处于条件队列中 */
    static final int CONDITION = -2;
    /**
    * waitStatus 可能的值，表示下一个 acquireShared 应该无条件传播。
    * 当前节点获得锁或释放锁时，共享模式下节点的最终状态就是 PROPAGATE
    */
    static final int PROPAGATE = -3;

    /**
    * 状态，仅取以下几个值：
    *
    * 1、SIGNAL
    * 表示此节点的后继节点将会或者很快会被阻塞（通过 park() 方法），
    * 因此当前节点在释放或取消时必须调用 unpark() 方法给后继节点许可。
    * 为了避免竞争，获取方法必须首先表明它们需要一个信号，然后重试原子
    * 获取方法，然后在失败时阻塞。
    *  
    * 2、CANCELLED
    * 表示此节点被取消（由于超时或中断）。一个被取消节点的线程不会再次阻塞。
    *
    * 3、CONDITION
    * 表示此节点当前处于条件队列中。此状态不会存在于 CLH 锁同步队列中，
    * 只用于条件阻塞队列。
    * 
    * 4、PROPAGATE
    * 一个 releaseShared 应该被传播到其他节点，在 doReleaseShared 中设置
    * （仅用于头节点）来确保传播继续。
    *
    * 5、0
    * 非负值意味着节点不需要信号，正常同步节点初始化时为 0。
    *
    * 在独占模式中可能的状态：SIGNAL、CANCELLED、0
    * 在共享模式中可能的状态：SIGNAL、CANCELLED、PROPAGATE、0
    *
    */ 
    volatile int waitStatus;

    /**
    * 链接到当前节点的前驱节点，用于检查 waitStatus 的值，在排队期间分配，
    * 并在出队时清空。在取消了前驱节点后，我们总能找到一个未被取消的节点，
    * 因为 Head 节点永远不会被取消。一个线程只能取消自己，取消的线程永远
    * 不会成功获取。
    * 
    * 前驱节点的主要作用是在循环中跳过 CANCELLED 状态的节点。
    */
    volatile Node prev;

    /**
    * 链接到当前节点释放后并调用 unpark 的后继节点。在排队期间分配，在跳过
    * 取消的前驱节点时进行调整，并在出队列时清除。enq() 操作会在 CAS 更新
    * 队列尾节点成功后才会将前驱节点的 next 设为后继节点。
    */
    volatile Node next;

    /** 关联的线程 */ 
    volatile Thread thread;

    /**
    * 链接处于条件阻塞队列的节点或者是特殊值 SHARED。
    * 实际就是用来标记 Node 是共享模式还是独占模式，
    * 独占模式时为 null，共享模式时为 SHARED，在条件
    * 阻塞队列中指向下一个节点。
    */
    Node nextWaiter;

    /**
    * 判断是否是共享模式
    */
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    /**
    * 返回前驱节点
    */
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }
    
    Node() {
    }

    /** 同步阻塞队列中 addWaiter 使用 */
    Node(Thread thread, Node mode) {
        this.nextWaiter = mode;
        this.thread = thread;
    }

    /** 条件阻塞队列中使用 */
    Node(Thread thread, int waitStatus) {
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```
## 同步队列
AQS 中的同步队列是一个 CLH 锁队列的变种，是由内部类 Node 组成的 FIFO 双向链表队列。  

### 入队列

```java
/**
* 入队操作，根据给定的模式，将当前线程包装成 Node 节点放入队列
*/
private Node addWaiter(Node mode) {
    // 选择模式，包装当前线程
    Node node = new Node(Thread.currentThread(), mode);
    // 记录原 tail
    Node pred = tail;
    // pred 为 null 说明队列还没初始化，直接执行 enq() 
    if (pred != null) {
        node.prev = pred;
        // CAS 更新 tail，如果失败，进入 enq() 方法自旋入队
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

/**
* 自旋方式入队或初始化队列，采用自旋是为了保证成功，自旋一定会成功
*/
private Node enq(final Node node) {
    // 自旋
    for (;;) {
        Node t = tail;
        // tail 为 null，则需要初始化队列
        if (t == null) {
            // CAS 更新 head 节点，更新成功后设置 tail 节点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 当节点在同步队列中时，prev 一定是非空的；
            // 当 prev 是非空的时，节点不一定在同步队列中，因为
            // 此时 CAS 更新尾节点可能没有成功。
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

关于入队，有以下几个要点：  
- 入队需要使用 CAS 操作，而 CAS 操作可能会失败，因此使用自旋 CAS 操作保证一定会成功。
- 当 tail 为 null 时，表示队列还没有初始化，此时需要通过自旋 CAS 操作保证初始化一定成功。
- head 节点在队列中一直作为一个 dummy 节点，不存储 thread，但会记录 waitStatus 作为唤醒后继节点的标识。
- 当节点在同步队列中时，prev 一定非空；但是 prev 非空，节点不一定在对列中，因为 CAS 更新尾节点操作可能会失败。  

### 出队列
出队列分为独占模式下成功获取锁后出队列、共享模式下成功获取锁后出队列以及因为超时或中断导致获取锁失败而出队列。  

- 独占模式出队列  
```java
/**
* 设置队列头为传入的节点，从而使该节点出队列
*/
private void setHead(Node node) {
    head = node;
    // 清空引用
    node.thread = null;
    node.prev = null;
}
// 清空该节点的前驱节点的后继节点
p.next = null;
```

- 共享模式出队列  
```java
/**
* 共享模式出队列，设置头节点并在满足条件时继续唤醒后续节点
*
* @param node the node
* @param propagate the return value from a tryAcquireShared
*/
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    // 设置头节点
    setHead(node);
    /*
    * 这三个条件，主要是为了判断是否还有剩余资源一唤醒后续节点
    * propagete > 0 意味着还有剩余资源（sate > 0），在共享模式下需要继续唤醒
    * h == null 这个判断官方的原话为：we don't know, because it appears null
    * h.waitStatus < 0 头节点状态为负数（尤其是PROPAGATE），说明需要唤醒后继节点
    */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        // 如果后继节点是独占模式，则不会唤醒后续节点
        if (s == null || s.isShared())
            // 唤醒后继节点
            doReleaseShared();
    }
}
```

- 超时或中断后出队列
```java
/**
* 取消获取锁
*/
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

    node.thread = null;

    Node pred = node.prev;
    while (pred.waitStatus > 0)
        /** 
        * 一直沿着 prev 方向遍历，直到找到一个不是取消状态的节点，
        * 同时清除路径上出现的取消状态的节点
        */
        node.prev = pred = pred.prev;

    Node predNext = pred.next;

    // 将节点状态设置为 CANCELLED，之后该节点除了被清除不再参与任何操作
    node.waitStatus = Node.CANCELLED;

    // 如果需要清除的节点正好是尾节点，则需要通过 CAS 操作将前驱节点设置为尾节点
    // 该操作失败会转入 else 执行
    if (node == tail && compareAndSetTail(node, pred)) {
        // 将前驱节点的 next 置为空，即使失败了也没有多大影响，判断线程节点能否获取锁
        // 的依据是其前驱节点的状态，与 next 没有关系
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        /** 
        * 1 前驱节点不是 head
        *
        * 2 前驱节点是 SIGNAL 状态，或者状态为非取消时，使用 CAS 操作设置状态为 SIGNAL，
        *   总之最后前驱节点的状态为 SIGNAL
        *
        * 3 前驱节点的 thread 不为 null，这个很奇怪，除了头节点，别的节点的 thread 应该都不为空
        */
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
                (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            // 如果后继节点为非取消状态，说明后继节点想获取锁，此时需要将前驱节点的 next 设为该后继节点
            if (next != null && next.waitStatus <= 0)
                // 允许失败，失败了影响也不大
                compareAndSetNext(pred, predNext, next);
        } else {
            // 如果前驱节点是 head，则执行 unparkSuccessor()
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

## 状态
state 用来记录线程获取锁的次数，该状态使用 volatile 保证可见性，并使用 CAS 操作进行更新。当 state > 0 时，表示当前线程已经获取到锁；当 state = 0 时，表示当前线程还没有获得锁。在独占模式中，state 的值最大为 1；在共享模式中，state 的值可以大于 1。  

## 独占模式
独占模式下，同一时间只有一个线程获得锁运行，state 的值最大为 1。  

### 独占式获取锁
独占模式获取锁共有三种方式：  
- 不响应中断获取锁（acquire）
线程在获取锁的过程中被中断后还可以被重新唤醒继续获取锁。  
- 响应中断获取锁（acquireInterruptibly）
线程在获取锁的过程中被中断则立即抛出异常。  
- 响应中断和超时获取锁（tryAcquireNanos）
线程在获取锁的过程中被中断则会立即抛出异常，在超时时间结束还没有获取到锁，则返回 false。  

**成功获取锁的标志是 tryAcquire() 方法是否成功**，该方法需要子类实现。  

#### 独占式不响应中断获取锁

```java
/**
* 以独占模式获取锁，忽略中断（不处理，只标记中断状态）。
*
* 成功获取锁的标志是至少一次调用 tryAcquire() 方法成功返回 true
* 调用失败将进入同步队列，期间可能会多次阻塞和解除阻塞（解除阻塞后竞争锁失败后又进入队列阻塞）
* 这个方法可以用来实现 Lock 接口的 lock 方法
*
* @param arg 期望的 state 的值
*/
public final void acquire(int arg) {
    // addWaiter 方法是入队方法，返回新入队的 Node
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 线程成功获取锁，但是在获取锁的过程中被中断，则重新设置中断状态
        selfInterrupt();
}

/**
* 为已经在同步队列中的线程提供独占式不响应中断的获取锁
*
* @param node the node
* @param arg the acquire argument
* @return {@code true} if interrupted while waiting
*/
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        // 在获取锁的过程中是否有被中断
        boolean interrupted = false;
        // 自旋尝试获取锁
        for (;;) {
            // 记录前驱节点
            final Node p = node.predecessor();
            // 前驱节点是 head 时，尝试获取锁
            if (p == head && tryAcquire(arg)) {
                // 成功获取锁，则要将该节点出队
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 线程被中断过
                interrupted = true;
        }
    } finally {
        // 获取锁失败
        if (failed)
            cancelAcquire(node);
    }
}

/**
* 线程无法获取到锁时，前驱节点的状态处理
*
* @param pred node's predecessor holding status
* @param node the node
* @return {@code true} if thread should block
*/
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 前驱节点的状态为 SIGNAL，说明当前节点可以安全的执行 park()
    // 只有前驱节点在获取锁之后更新状态为 SIGNAL 后，才说明前驱节点的后继节点可以
    // 被唤醒
    if (ws == Node.SIGNAL)
        return true;
    // 说明状态为 CANCELLED，此时需要重选一个非取消状态的节点
    if (ws > 0) {
        do {
            // 循环判断前驱节点的状态，已被取消的前驱节点形成一条引用链，最终被 GC 回收
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
        * 此时 waitStatus 的值为 0 或者是 PROPAGATE
        * 需要更新 waitStatus 的值为 SIGNAL，这样前驱节点获取到锁后就可以通知后继节点
        */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    // 前驱节点不是 SIGNAL 状态的，后继节点都不能 park()
    return false;
}

/**
* 使线程进入等待状态，返回线程是否被中断过
* @return {@code true} if interrupted
*/
private final boolean parkAndCheckInterrupt() {
    // LockSupport 与 线程绑定，使用 this 作为阻塞对象，此时阻塞在此处
    LockSupport.park(this);
    // 前驱节点使用 unpark() 后，后继节点才能继续执行此处的代码
    // Thread.interrupted() 会清除中断状态，如果线程被中断，则需要重新设置中断状态
    return Thread.interrupted();
}
```

#### 独占式响应中断获取锁

```java
/**
* 独占式响应中断获取锁
* 该方法可以实现 Lock 接口的 lockInterruptibly 方法
*
* @param arg the acquire argument.
* @throws InterruptedException if the current thread is interrupted
*/
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    // 线程被中断立即抛出中断异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 获取锁，失败进入同步队列
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}

/**
* Acquires in exclusive interruptible mode.
* 独占式响应中断获取锁
* @param arg the acquire argument
*/
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    // 节点入队
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        // 自旋获取锁
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                // 成功获取锁，则要将该节点出队
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 线程被中断过，则直接抛出中断异常
                throw new InterruptedException();
        }
    } finally {
        // 线程被中断，取消节点
        if (failed)
            cancelAcquire(node);
    }
}
```

#### 独占式响应中断和超时获取锁

```java
/**
* 独占式影响超时获取锁
*/
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    // 线程被中断立即抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}

/**
* 自旋超时的阈值，1000 纳秒
*/
static final long spinForTimeoutThreshold = 1000L;

/**
* 独占式影响超时获取锁
*
* @param arg the acquire argument
* @param nanosTimeout max wait time
* @return {@code true} if acquired
*/
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    // 计算截止时间
    final long deadline = System.nanoTime() + nanosTimeout;
    // 当前线程入队
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        // 自旋获取锁
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                // 成功获取锁，出队
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            // 计算过去了多长时间
            nanosTimeout = deadline - System.nanoTime();
            // 已经超时
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                // 剩余时间超过自旋的阈值，则等待一段时间（nanosTimeout），时间结束自动唤醒
                LockSupport.parkNanos(this, nanosTimeout);
            // 被中断直接抛出异常
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        // 线程被中断或超时，取消节点获取锁
        if (failed)
            cancelAcquire(node);
    }
}
```

### 独占式释放锁
其中，释放锁的方法 `tryRelease()` 需要子类实现。  

```java
/**
* 独占式释放锁
* 可以作为 Lock 接口的 unlock 实现
*
* @param arg the release argument. 释放锁个数
* @return the value returned from {@link #tryRelease}
*/
public final boolean release(int arg) {
    // tryRelease() 判断是否成功释放锁，即 state 的值被更新为 0
    if (tryRelease(arg)) {
        Node h = head;
        // head 不为 null，并且头节点的状态不是 0
        if (h != null && h.waitStatus != 0)
            // 需要唤醒同步队列中后继节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}

/**
* 唤醒后继节点
*
* @param node the node
*/
private void unparkSuccessor(Node node) {
    /*
    * 状态值小于 0，表示节点还没有取消，需要将状态修改为 0
    */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
    * 一般情况下，需要唤醒的就是节点的后继者，但是如果后继节点
    * 已经取消或者为空，则需要沿着尾节点向前找一个未被取消的节点
    */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 后继节点非空，直接唤醒
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

## 共享模式
共享模式下，同一时间可以有多个线程获得锁运行，state 的值可以大于 1。  

### 共享式获取锁
共享模式获取锁共有三种方式：  
- 不响应中断获取锁（acquireShared）
- 响应中断获取锁（acquireSharedInterruptibly）
- 响应中断和超时获取锁（tryAcquireSharedNanos）  

共享模式下获取锁的原理与独占模式相似，主要区别是获取锁的方法不再是 tryAcquire()，而是 tryAcquireShared()，该方法需要子类实现。还有一点不同是节点出队方式不同，当线程成功获取锁出队时，会传播唤醒后继节点。  

#### 共享式不响应中断获取锁

```java
/**
* 共享模式获取锁，不响应中断
*
* @param arg the acquire argument.
*/
public final void acquireShared(int arg) {
    // 小于 0 表示没有获取到锁
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

/**
* 不响应中断共享模式获取锁
* @param arg the acquire argument
*/
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                // 前驱节点是 head，则再次获取锁
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 获取锁成功后，出队列
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    /**
                    * 线程被中断，则设置中断状态
                    *
                    * 这里直接执行中断方法，而独占模式需要返回中断状态。
                    * 由于条件队列只能用于独占模式，因为条件队列的前提就是首先需要
                    * 获取到锁，在条件队列中会根据条件判断是否要执行 selfInterrupt()
                    */
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

#### 共享式响应中断获取锁

```java
/**
* 响应中断获取锁
*
* @param arg the acquire argument.
* @throws InterruptedException if the current thread is interrupted
*/
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

/**
* 共享模式响应中断获取锁
* @param arg the acquire argument
*/
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 直接抛出异常
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

#### 共享式响应中断和超时获取锁

```java
/**
* 响应中断和超时获取锁
*/
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquireShared(arg) >= 0 ||
        doAcquireSharedNanos(arg, nanosTimeout);
}

/**
* Acquires in shared timed mode.
*/
private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### 共享式释放锁
释放锁的 `tryReleaseShared()` 的方法需要子类去实现。  

```java
/**
* tryReleaseShared 返回 true 时解除一个或多个线程的阻塞
*
* @param arg the release argument.
* @return the value returned from {@link #tryReleaseShared}
*/
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

private void doReleaseShared() {
    // 自旋释放锁
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            // head 节点状态为 SIGNAL 时唤醒后继节点
            if (ws == Node.SIGNAL) {
                // 需要先将状态值修改为 0，才会去唤醒后继节点
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            // 将状态设置为 PROPAGATE，以区别独占模式中的 0
            else if (ws == 0 &&
                        !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        // head 没有变化，说明唤醒结束
        if (h == head)                   // loop if head changed
            break;
    }
}
```

## Condition
使用 Condition 来实现类似 `Object.wait()`、`Object.notify()`、`Object.notifyAll()` 等方法的效果。  

## Condition 接口
条件变量（或条件队列）为一个线程提供暂停执行的手段，直到某个状态条件当前为真时，另一个线程通知唤醒该线程。由于多线程场景中该条件是共享的，所以需要条件需要与锁关联来保证原子性，比如创建条件就使用 `Condition condition = lock.newCondition()`。需要注意的是，只有在独占模式中才能使用条件变量。  

```java
public interface Condition {

    /**
    * 导致当前线程等待，直到其他线程发出信号唤醒或者中断该线程
    */
    void await() throws InterruptedException;

    /**
    * 导致当前线程等待，直到其他线程发出信号唤醒该线程
    */
    void awaitUninterruptibly();

    /**
    * 导致当前线程等待直到其他线程发出信号唤醒或中断该线程，或超时自动唤醒
    */
    long awaitNanos(long nanosTimeout) throws InterruptedException;

    /**
    * 该方法等同于 awaitNanos(unit.toNanos(time)) > 0
    *
    * @param time the maximum time to wait
    * @param unit the time unit of the {@code time} argument
    * @return {@code false} if the waiting time detectably elapsed
    *         before return from the method, else {@code true}
    */
    boolean await(long time, TimeUnit unit) throws InterruptedException;

    /**
    * 导致当前线程等待直到其他线程发出信号唤醒或中断该线程，或超时自动唤醒
    *
    * @param deadline the absolute time to wait until
    * @return {@code false} if the deadline has elapsed upon return, else
    *         {@code true}
    * @throws InterruptedException if the current thread is interrupted
    *         (and interruption of thread suspension is supported)
    */
    boolean awaitUntil(Date deadline) throws InterruptedException;

    /**
    * 唤醒在条件队列中等待的一个线程
    */
    void signal();

    /**
    * 唤醒在条件队列中等待的所有线程
    */
    void signalAll();
}
```

## ConditionObject
`ConditionObject` 是 AQS 的一个内部类，它实现了 `Condition` 接口。内部维护了一个 FIFO 双向链表作为条件队列，当条件不满足时线程释放锁入队，条件满足时出队并重新尝试获取锁。  

```java
/** 条件队列头节点 */
private transient Node firstWaiter;
/** 条件队列尾节点 */
private transient Node lastWaiter;
/** 
* Mode meaning to reinterrupt on exit from wait
* 退出时重新中断
*/
private static final int REINTERRUPT =  1;
/** 
* Mode meaning to throw InterruptedException on exit from wait 
* 退出时抛出中断异常
*/
private static final int THROW_IE    = -1;
```
还有一个字段为 `nextWaiter`，是 Node 内部类的属性，只有在使用条件变量时才会使用它。  

### 入队、出队操作

```java
/**
* 封装新节点入队列
* @return its new wait node
*/
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    // 如果节点状态非 CONDITION，则需要清除所有非 CONDITION 节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        // 清除所有的非 CONDITION 节点
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 封装节点
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    // 队列尾为 null，说明队列为空
    if (t == null)
        firstWaiter = node;
    else
        // 将新节点放到 lastWaiter 之后
        t.nextWaiter = node;
    // 新节点作为尾节点
    lastWaiter = node;
    return node;
}
```

```java
/**
* 节点出队，此操作完成后，队列中的 lastWaiter 只可能为 null
* 或者 CONDITION
*/
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null;
    while (t != null) {
        Node next = t.nextWaiter;
        // 节点状态不为 CONDITION，则出队
        if (t.waitStatus != Node.CONDITION) {
            // 断开 t 与 nextWaiter 的连接
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            // trail 不为 null，说明队列中有状态不为 CONDITION 的节点，
            // 并且 trail 就是上一个不为 CONDITION 的节点
            else
                trail.nextWaiter = next;
            // next 为 null，表示 t 为队列尾
            if (next == null)
                lastWaiter = trail;
        }
        // 状态为 CONDITION，记录一下
        else
            trail = t;
        // 下一个循环
        t = next;
    }
}
```
### awaitUninterruptibly
不响应中断阻塞当前线程，直到其他线程唤醒。  

```java
/**
* 不响应中断阻塞当前线程
*/
public final void awaitUninterruptibly() {
    // 节点入队
    Node node = addConditionWaiter();
    // 释放锁
    int savedState = fullyRelease(node);
    boolean interrupted = false;
    while (!isOnSyncQueue(node)) {
        // 只要节点不在同步队列中，就一直阻塞
        LockSupport.park(this);
        if (Thread.interrupted())
            // 途中遇到中断，则记录一下
            interrupted = true;
    }
    // 节点在同步队列中，独占式不响应中断的获取锁
    if (acquireQueued(node, savedState) || interrupted)
        // 重新设置中断状态
        selfInterrupt();
}

/**
* 用当前状态值调用 release
* @param node the condition node for this wait
* @return previous sync state
*/
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        // 获取当前 state 的值
        int savedState = getState();
        // 独占式释放锁
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}

/**
* 如果节点（始终是最初放在条件队列中的节点）正在同步队列中，
* 则返回 true
* @param node the node
* @return true if is reacquiring
*/
final boolean isOnSyncQueue(Node node) {
    // prev 为 null，说明该节点一定不在同步队列中
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    // 如果 next 不为 null，说明节点一定在同步队列中
    if (node.next != null) // If has successor, it must be on queue
        return true;
    /*
    * prev 非空，节点不一定在同步队列中，因为节点入队（同步队列）使用
    * CAS 操作更新可能会失败
    */
    return findNodeFromTail(node);
}

/**
* 从后往前沿着 prev 遍历，搜索节点是否在同步队列中
* @return true if present
*/
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```

### await
响应中断阻塞线程，直到其他线程中断该线程或唤醒该线程。  

```java
/**
* 响应中断阻塞线程
*/
public final void await() throws InterruptedException {
    // 中断发生，立即抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 节点入队
    Node node = addConditionWaiter();
    // 释放锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 判断节点是否在同步队列中
    while (!isOnSyncQueue(node)) {
        // 不在同步队列中就阻塞
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 尝试获取同步锁，acquireQueued 会在有中断发生时返回 true
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    // 如果当前节点存在后继节点，需要将节点出队
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    // 中断处理
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

/**
* 检测中断，如果发生中断则将因中断唤醒的节点转移
* 如果没有中断发生，返回 0
* 如果有中断发生，则转移节点
*/
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}

/**
* 将条件队列中因为中断或者超时而唤醒的节点进行转移
*
* @param node the node
* @return true if cancelled before the node was signalled
*/
final boolean transferAfterCancelledWait(Node node) {
    // 将 waitStatus 设置为 0 才能进入同步队列
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
    /*
    * 如果 CAS 操作失败，并且节点不在同步队列中
    */
    while (!isOnSyncQueue(node))
        // 提醒释放资源
        Thread.yield();
    return false;
}

/**
* 中断处理
*/
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    // 抛出中断异常
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    // 重设中断状态
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}
```

### awaitNanos
响应中断和超时阻塞线程。  

```java
/**
* 响应中断和超时阻塞线程
* @param nanosTimeout the maximum time to wait, in nanoseconds
*/
public final long awaitNanos(long nanosTimeout)
        throws InterruptedException {
    // 中断发生，立即抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 节点入队
    Node node = addConditionWaiter();
    // 释放锁
    int savedState = fullyRelease(node);
    // 计算截止时间
    final long deadline = System.nanoTime() + nanosTimeout;
    int interruptMode = 0;
    // 判断节点是否在同步队列中
    while (!isOnSyncQueue(node)) {
        // 超时就转移节点（这里的 nanosTimeout 是超时时间，可以为负值）
        if (nanosTimeout <= 0L) {
            transferAfterCancelledWait(node);
            break;
        }
        // 如果超过自旋的阈值，则阻塞 nanosTimeout 的时间
        if (nanosTimeout >= spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanosTimeout);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
        // 剩余时间
        nanosTimeout = deadline - System.nanoTime();
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    return deadline - System.nanoTime();
}
```

### awaitUntil
响应中断和超时阻塞线程。  

```java
/**
* 响应中断和超时阻塞线程
*/
public final boolean awaitUntil(Date deadline)
        throws InterruptedException {
    long abstime = deadline.getTime();
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    boolean timedout = false;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        // 超时，转移节点
        if (System.currentTimeMillis() > abstime) {
            timedout = transferAfterCancelledWait(node);
            break;
        }
        LockSupport.parkUntil(this, abstime);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    return !timedout;
}
```

### signal
从条件队列中唤醒一个节点（头节点）。  

```java
/**
* 从条件队列中唤醒一个节点
* @throws IllegalMonitorStateException if {@link #isHeldExclusively}
*         returns {@code false}
*/
public final void signal() {
    // 条件队列只适用于独占模式，且只有持有锁的线程执行唤醒操作
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

/**
* Removes and transfers nodes until hit non-cancelled one or
* null. Split out from signal in part to encourage compilers
* to inline the case of no waiters.
* @param first (non-null) the first node on condition queue
*/
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
        // 将节点转移到同步队列并唤醒
    } while (!transferForSignal(first) &&
                (first = firstWaiter) != null);
}

/**
* 将节点从条件队列中转移到同步队列
* @param node the node
* @return true if successfully transferred (else the node was
* cancelled before signal)
*/
final boolean transferForSignal(Node node) {
    /*
    * 进入同步队列需要将节点 waitStatus 设置为 0
    */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
    * 进入同步队列
    */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

### signalAll
唤醒全部。  

```java
/**
* Moves all threads from the wait queue for this condition to
* the wait queue for the owning lock.
*
* @throws IllegalMonitorStateException if {@link #isHeldExclusively}
*         returns {@code false}
*/
public final void signalAll() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignalAll(first);
}

/**
* Removes and transfers all nodes.
* @param first (non-null) the first node on condition queue
*/
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    // 沿着 nextWaiter 遍历唤醒
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```

<br>[⬆ Back to top](#深入-lock)
