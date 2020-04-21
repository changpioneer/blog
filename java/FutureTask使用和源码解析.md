

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



















