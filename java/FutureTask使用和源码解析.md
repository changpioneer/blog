

# 4、源码

### 4.1、构造函数
先从FutureTask的构造函数看起，FutureTask有两个构造函数，其中一个如下：
```java
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
```
这个构造函数会把传入的`Callable`变量保存在`this.callable`字段中，该字段定义为`private Callable<V> callable;`用来保存底层的调用，在被执行完成以后会指向`null`,接着会初始化`state`字段为`NEW`。`state`字段用来保存`FutureTask`内部的任务执行状态，一共有7中状态，每种状态及其对应的值如下：
```java
    /**
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
```
其中需要注意的是`state`是`volatile`类型的，也就是说只要有任何一个线程修改了这个变量，那么其他所有的线程都会知道最新的值。

状态解释：

    - NEW: 表示是个新的任务或者还没被执行完的任务。这是初始状态。

    - COMPLETING: 任务已经执行完成或者执行任务的时候发生异常，但是任务执行结果或者异常原因还没有保存到`outcome`字段(outcome字段用来保存任务执行结果，如果发生异常，则用来保存异常原因)的时候，状态会从`NEW`变更到`COMPLETING`。但是这个状态会时间会比较短，属于中间状态。

    - NORMAL: 任务已经执行完成并且任务执行结果已经保存到`outcome`字段，状态会从`COMPLETING`转换到`NORMAL`。这是一个最终态。

    - EXCEPTIONAL: 任务执行发生异常并且异常原因已经保存到`outcome`字段中后，状态会从`COMPLETING`转换到`EXCEPTIONAL`。这是一个最终态。

    - CANCELLED: 任务还没开始执行或者已经开始执行但是还没有执行完成的时候，用户调用了`cancel(false)`方法取消任务且不中断任务执行线程，这个时候状态会从`NEW`转化为`CANCELLED`状态。这是一个最终态。

    - INTERRUPTING:  任务还没开始执行或者已经执行但是还没有执行完成的时候，用户调用了`cancel(true)`方法取消任务并且要中断任务执行线程但是还没有中断任务执行线程之前，状态会从`NEW`转化为`INTERRUPTING`。这是一个中间状态。

    - INTERRUPTED: 调用`interrupt()`中断任务执行线程之后状态会从`INTERRUPTING`转换到`INTERRUPTED`。这是一个最终态。

有一点需要注意的是，所有值大于`COMPLETING`的状态都表示任务已经执行完成(任务正常执行完成，任务执行异常或者任务被取消)。


另外一个构造函数如下:
```java
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }
```
这个构造函数会把传入的`Runnable`封装成一个`Callable`对象保存在`callable`字段中，同时如果任务执行成功的话就会返回传入的`result`。这种情况下如果不需要返回值的话可以传入一个`null`。

### 4.2、run方法
```java
public void run() {
    // 1. 状态如果不是NEW，说明任务或者已经执行过，或者已经被取消，直接返回
    // 2. 状态如果是NEW，则尝试把当前执行线程保存在runner字段中
    // 如果赋值失败则直接返回
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
                // 3. 执行任务
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                // 4. 任务异常
                setException(ex);
            }
            if (ran)
                // 4. 任务正常执行完毕
                set(result);
        }
    } finally {
        runner = null;
        int s = state;
        // 5. 如果任务被中断，执行中断处理
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

### 4.3.1、setException()方法如下：
```java
protected void setException(Throwable t) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = t;
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
        finishCompletion();
    }
}
```

在setException()方法中

- 首先会以原子形式的把当前的状态从`NEW`变更为`COMPLETING`状态。
- 把异常原因保存在`outcome`字段中，`outcome`字段用来保存任务执行结果或者异常原因。
- 以原子形式的把当前任务状态从`COMPLETING`变更为`EXCEPTIONAL`。
调用`finishCompletion()`。关于这个方法后面在分析。

### 4.3.2、set()
如果任务成功执行则调用set()方法设置执行结果，该方法实现如下:
```java
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}
```
这个方法跟上面分析的setException()差不多，

- 首先会CAS的把当前的状态从`NEW`变更为`COMPLETING`状态。
- 把任务执行结果保存在`outcome`字段中。
- CAS的把当前任务状态从`COMPLETING`变更为`NORMAL`。调用`finishCompletion()`。

## 发起任务线程跟执行任务线程通常情况下都不会是同一个线程，在任务执行线程执行任务的时候，任务发起线程可以查看任务执行状态、获取任务执行结果、取消任务等等操作，接下来分析下这些操作。

### 4.4、get()
任务发起线程可以调用`get()`方法来获取任务执行结果，如果此时任务已经执行完毕则会直接返回任务结果，如果任务还没执行完毕，则调用方会阻塞直到任务执行结束返回结果为止。`get()`方法实现如下:
```java
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
```
get()方法实现比较简单，会
- 判断任务当前的`state <= COMPLETING`是否成立。前面分析过，`COMPLETING`状态是任务是否执行完成的临界状态。
- 如果成立，表明任务还没有结束(这里的结束包括任务正常执行完毕，任务执行异常，任务被取消)，则会调用`awaitDone()`进行阻塞等待。
- 如果不成立表明任务已经结束，调用`report()`返回结果。

### 4.4.1、awaitDone()
当调用get()获取任务结果但是任务还没执行完成的时候，调用线程会调用awaitDone()方法进行阻塞等待，该方法定义如下:
```java
private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
    // 计算等待截止时间
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        // 1. 判断阻塞线程是否被中断,如果被中断则在等待队
        // 列中删除该节点并抛出InterruptedException异常
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }
 
        // 2. 获取当前状态，如果状态大于COMPLETING
        // 说明任务已经结束(要么正常结束，要么异常结束，要么被取消)
        // 则把thread显示置空，并返回结果
        int s = state;
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        // 3. 如果状态处于中间状态COMPLETING
        // 表示任务已经结束但是任务执行线程还没来得及给outcome赋值
        // 这个时候让出执行权让其他线程优先执行
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        // 4. 如果等待节点为空，则构造一个等待节点
        else if (q == null)
            q = new WaitNode();
        // 5. 如果还没有入队列，则把当前节点加入waiters首节点并替换原来waiters
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                    q.next = waiters, q);
        else if (timed) {
            // 如果需要等待特定时间，则先计算要等待的时间
            // 如果已经超时，则删除对应节点并返回对应的状态
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            // 6. 阻塞等待特定时间
            LockSupport.parkNanos(this, nanos);
        }
        else
            // 6. 阻塞等待直到被其他线程唤醒
            LockSupport.park(this);
    }
}
```
awaitDone()中有个死循环，每一次循环都会

- 1. 判断调用`get()`的线程是否被其他线程中断，如果是的话则在等待队列中删除对应节点然后抛出`InterruptedException`异常。
- 2. 获取任务当前状态，如果当前任务状态大于`COMPLETING`则表示任务执行完成，则把`thread`字段置`null`并返回结果。
- 3. 如果任务处于`COMPLETING`状态，则表示任务已经处理完成(正常执行完成或者执行出现异常)，但是执行结果或者异常原因还没有保存到`outcome`字段中。这个时候调用线程让出执行权让其他线程优先执行。
- 4. 如果等待节点为空，则构造一个等待节点`WaitNode`。
- 5. 如果第四步中新建的节点还没如队列，则CAS的把该节点加入`waiters`队列的首节点。
- 6. 阻塞等待。

假设当前`state=NEW`且`waiters`为`NULL`,也就是说还没有任何一个线程调用`get()`获取执行结果，这个时候有两个线程`threadA`和`threadB`先后调用`get()`来获取执行结果。再假设这两个线程在加入阻塞队列进行阻塞等待之前任务都没有执行完成且`threadA`和`threadB`都没有被中断的情况下(因为如果`threadA`和`threadB`在进行阻塞等待结果之前任务就执行完成或线程本身被中断的话，`awaitDone()`就执行结束返回了)，执行过程是这样的，以threadA为例:

    - 第一轮for循环，执行的逻辑是`q == null`, 所以这时候会新建一个节点q。第一轮循环结束。
    - 第二轮for循环，执行的逻辑是`!queue`，这个时候会把第一轮循环中生成的节点的`next`指针指向`waiters`，然后CAS的把节点q替换waiters。也就是把新生成的节点添加到`waiters`链表的首节点。如果替换成功，queued=true。第二轮循环结束。
    - 第三轮for循环，进行阻塞等待。要么阻塞特定时间，要么一直阻塞知道被其他线程唤醒。

### 4.5、cancel(boolean)
用户可以调用`cancel(boolean)`方法取消任务的执行，`cancel()`实现如下:
```java
public boolean cancel(boolean mayInterruptIfRunning) {
    // 1. 如果任务已经结束，则直接返回false
    if (state != NEW)
        return false;
    // 2. 如果需要中断任务执行线程
    if (mayInterruptIfRunning) {
        // 2.1. 把任务状态从NEW转化到INTERRUPTING
        if (!UNSAFE.compareAndSwapInt(this, stateOffset, NEW, INTERRUPTING))
            return false;
        Thread t = runner;
        // 2.2. 中断任务执行线程
        if (t != null)
            t.interrupt();
        // 2.3. 修改状态为INTERRUPTED
        UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED); // final state
    }
    // 3. 如果不需要中断任务执行线程，则直接把状态从NEW转化为CANCELLED
    else if (!UNSAFE.compareAndSwapInt(this, stateOffset, NEW, CANCELLED))
        return false;
    // 4.
    finishCompletion();
    return true;
}

public boolean cancel(boolean mayInterruptIfRunning) {
    // 1. 如果任务状态不是NEW，即已经结束，则直接返回false
    // 2. 更新任务状态，
    // 2.1 如果不需要中断任务执行线程，则直接把状态从NEW转化为CANCELLED
    // 2.2 如果需要中断任务执行线程，则把状态从NEW转化为INTERRUPTING
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    
        // 3. 如果需要中断任务执行线程
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                // 3.1. 中断任务执行线程
                if (t != null)
                    t.interrupt();
            } finally { 
                // 3.2. 修改状态为INTERRUPTED
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        // 4.
        finishCompletion();
    }
    return true;
}
```

### 4.6、finishCompletion()

根据前面的分析，不管是任务执行异常还是任务正常执行完毕，或者取消任务，最后都会调用finishCompletion()方法，该方法实现如下:
```java
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }

    done();

    callable = null;        // to reduce footprint
}
```
这个方法的实现比较简单，依次遍历`waiters`链表，唤醒节点中的线程，然后把`callable`置空。
被唤醒的线程会各自从`awaitDone()`方法中的`LockSupport.park*()`阻塞中返回，然后会进行新一轮的循环。在新一轮的循环中会返回执行结果(或者更确切的说是返回任务的状态)。

### 4.7、report()
`report()`方法用在`get()`方法中，作用是把不同的任务状态映射成任务执行结果。实现如下：
```java
private V report(int s) throws ExecutionException {
    Object x = outcome;
    // 1. 任务正常执行完成，返回任务执行结果
    if (s == NORMAL)
        return (V)x;
    // 2. 任务被取消，抛出CancellationException异常
    if (s >= CANCELLED)
        throw new CancellationException();
    // 3. 其他状态，抛出执行异常ExecutionException
    throw new ExecutionException((Throwable)x);
}
```

### 4.8、isCancelled()和isDone()
这两个方法分别用来判断任务是否被取消和任务是否执行完成，实现都比较简单，代码如下:
```java
public boolean isCancelled() {
    return state >= CANCELLED;
}

public boolean isDone() {
    return state != NEW;
}
```

总结下，其实FutureTask的实现还是比较简单的，当用户实现Callable()接口定义好任务之后，把任务交给其他线程进行执行。FutureTask内部维护一个任务状态，任何操作都是围绕着这个状态进行，并随时更新任务状态。任务发起者调用get*()获取执行结果的时候，如果任务还没有执行完毕，则会把自己放入阻塞队列中然后进行阻塞等待。当任务执行完成之后，任务执行线程会依次唤醒阻塞等待的线程。调用cancel()取消任务的时候也只是简单的修改任务状态，如果需要中断任务执行线程的话则调用Thread.interrupt()中断任务执行线程。

有个值得关注的问题就是当任务还在执行的时候用户调用cancel(true)方法能否真正让任务停止执行呢？
在前面的分析中我们直到，当调用cancel(true)方法的时候，实际执行还是Thread.interrupt()方法，而interrupt()方法只是设置中断标志位，如果被中断的线程处于sleep()、wait()或者join()逻辑中则会抛出InterruptedException异常。

因此结论是:cancel(true)并不一定能够停止正在执行的异步任务。
















