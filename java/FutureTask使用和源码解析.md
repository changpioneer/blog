

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


































